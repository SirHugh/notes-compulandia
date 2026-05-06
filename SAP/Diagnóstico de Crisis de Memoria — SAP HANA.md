# Diagnóstico de Crisis de Memoria — SAP HANA + SAP Business One Service Layer

**Fecha del incidente:** 2026-04-09 **Servidor:** `sap-linux` — SUSE Linux Enterprise Server 15 SP4 **RAM física:** 112 GB **Swap:** 112 GB (`/dev/dm-0`) **Componentes afectados:** SAP HANA 2.0 (tenant NDB) + SAP Business One Service Layer

---

## 1. El Síntoma

El servidor presentaba un estado de _thrashing_ severo: el sistema operativo movía páginas de memoria al swap y desde el swap continuamente, a una velocidad de 170–260 MB/s en ambas direcciones. Los indicadores concretos eran:

- RAM disponible: menos de 700 MB sobre 112 GB físicos
- Swap en uso: 77–82 GB
- `vmstat` mostraba `si`/`so` sostenidos por encima de 150 MB/s
- 2–3 procesos permanentemente bloqueados en I/O (`b > 0` en vmstat)
- I/O wait de CPU entre 6–12%

El sistema seguía funcionando pero con una degradación severa del rendimiento. Cualquier operación de SAP Business One o consulta a HANA tenía que esperar a que el kernel rescatara páginas desde el disco.

---

## 2. La Arquitectura del Sistema

Para entender el problema hay que comprender cómo están organizados los componentes:

**SAP HANA** corre bajo el usuario `ndbadm` y gestiona dos instancias lógicas dentro del mismo proceso físico: el SYSTEMDB (base de datos de sistema) y el tenant NDB (donde viven los datos de SAP B1). HANA reserva memoria de forma agresiva al arrancar y la mantiene ocupada incluso cuando no está en uso activo — esto es comportamiento esperado y por diseño.

**SAP Business One Service Layer** corre bajo el usuario `b1service0` e implementa una API REST para todos los clientes de SAP B1. Internamente usa Apache httpd en modo `mpm_prefork` — el modelo de proceso más antiguo de Apache, donde cada petición HTTP es atendida por un proceso separado, no un thread. Esto significa que si hay 10 workers activos, hay 10 procesos independientes, cada uno con su propia copia del estado en memoria.

La arquitectura del Service Layer en este servidor tiene cuatro capas:

```
Puerto 50000 — Load Balancer (httpd mpm_event, hasta 390 threads)
    ├── Puerto 50001 — Node 1 (httpd mpm_prefork, hasta 24 workers)
    ├── Puerto 50002 — Node 2 (httpd mpm_prefork, hasta 24 workers)
    ├── Puerto 50003 — Node 3 (httpd mpm_prefork, hasta 24 workers)
    └── Puerto 50004 — Node 4 (httpd mpm_prefork, hasta 24 workers)
```

Cada nodo es una instancia independiente de Apache que carga el módulo `libServiceLayerApache.so` — la biblioteca nativa de SAP B1 que implementa toda la lógica de negocio. La configuración `b1s.conf` apunta a una única conexión HANA (`NDB@sap-linux:30013`), lo que significa que cada worker accede a todas las empresas disponibles en el tenant.

---

## 3. La Causa Raíz

El problema no era HANA. La distribución real de memoria era:

|Usuario|RAM (RSS)|Descripción|
|---|---|---|
|`b1service0`|**64.6 GB**|Service Layer — Apache workers|
|`ndbadm`|**28.6 GB**|SAP HANA (SYSTEMDB + NDB + servicios XS)|
|`ndbxsa`|2.7 GB|XS Engine (servicios adicionales de HANA)|
|Resto del sistema|~0.5 GB|OS, agentes, exporters|

El Service Layer consumía más de la mitad de toda la RAM del servidor. La causa específica era el comportamiento de los workers `mpm_prefork` bajo carga sostenida: cada proceso worker de Apache que atiende sesiones de SAP B1 carga datos de empresa en memoria a medida que los usuarios trabajan. Con múltiples empresas configuradas en el tenant NDB, un worker que ya atendió muchas sesiones acumula el footprint de memoria de todas esas empresas.

Los tres workers más grandes al momento del diagnóstico mostraban:

```
PID 25567 — RSS 21.3 GB + Swap 25.0 GB = 46.3 GB totales
PID 25587 — RSS 20.7 GB + Swap 25.6 GB = 46.3 GB totales
PID 28579 — RSS 18.6 GB + Swap 25.8 GB = 44.4 GB totales
```

Estos tres procesos solos representaban ~137 GB de espacio de direcciones — más que toda la RAM física del servidor. Habían arrancado apenas 90 minutos antes. El resto de los workers del sistema eran jóvenes y aún no habían acumulado estado, pero crecerían de la misma manera con el tiempo.

Un segundo hallazgo importante fue la existencia de **dos procesos `hdbindexserver` simultáneos**: el del tenant NDB (activo, con alta actividad) y el del SYSTEMDB (antiguo, con 9 días de vida y 9 GB en swap). El SYSTEMDB indexserver había sido completamente desplazado a swap por la competencia de memoria, agravando el thrashing de HANA.

---

## 4. Por Qué el Sistema Llegó a Este Estado

Hay una incompatibilidad fundamental entre la configuración del Service Layer y los recursos disponibles. El archivo `httpd-b1s-lb-member-common.conf` define `MaxRequestWorkers 24` por nodo. Con cuatro nodos activos, el sistema tiene potencial para 96 workers simultáneos. Si cada worker maduro ocupa ~25 GB, el escenario teórico máximo requeriría 2.4 TB de RAM — 21 veces la RAM disponible.

En la práctica el sistema no llega a 96 workers simultáneos, pero la configuración no impone ningún límite que tenga en cuenta la memoria física disponible. Los workers crecen sin restricción hasta que el kernel comienza a paginar, y para cuando el thrashing es visible, el daño ya está hecho.

El valor de `swappiness` ya estaba configurado correctamente en 10 (adecuado para SAP), lo que indica que alguien ya había atendido este problema antes o siguió las recomendaciones de SUSE. Sin embargo, con una presión de memoria tan extrema, `swappiness` solo puede influir en _cuándo_ el kernel empieza a swapear, no en _si_ lo hace.

---

## 5. La Intervención

La acción de mayor impacto fue detener los nodos 3 y 4 del Service Layer, que concentraban todos los workers maduros:

```bash
systemctl stop b1s50004.service
systemctl stop b1s50003.service
```

El efecto fue inmediato:

|Métrica|Antes|Después|
|---|---|---|
|RAM usada|111 GB|45 GB|
|RAM libre|~700 MB|**67 GB**|
|Swap usado|77 GB|36 GB|
|`vmstat so`|200 MB/s|**0**|
|Procesos bloqueados|2–3|0|

El thrashing se detuvo completamente en segundos. Los 36 GB de swap restantes corresponden principalmente al SYSTEMDB indexserver y servicios XS de HANA que fueron desplazados durante la crisis — se recuperarán solos en memoria a medida que sean accedidos.

---

## 6. Estado Actual y Riesgo Residual

Con dos nodos activos (Node1 en puerto 50001 y Node2 en puerto 50002), el sistema está estable. Sin embargo, el riesgo de recurrencia es alto porque los workers de Node1 y Node2 eventualmente madurarán y alcanzarán el mismo footprint de 20–25 GB. El tiempo hasta la próxima crisis depende del volumen de sesiones que atiendan estos workers.

Los nodos 3 y 4 están deshabilitados para inicio automático pero el problema estructural permanece sin resolver.

---

## 7. Acciones Pendientes

Las siguientes acciones están ordenadas por urgencia e impacto:

**Inmediato (esta semana)**

Auditar cuántas empresas están configuradas en el tenant NDB y cuántas están activamente en uso. El tamaño del footprint por worker es directamente proporcional al número de empresas a las que los usuarios se conectan. Si hay empresas inactivas o de prueba en la misma instancia, separarlas reduciría el consumo.

Revisar `MaxConnectionsPerChild` — actualmente en 1024. Este parámetro controla cuántas peticiones puede atender un worker antes de ser reemplazado por uno nuevo. Un valor más bajo (por ejemplo 100–200) forzaría rotación más frecuente de workers, limitando la acumulación de estado. La contrapartida es mayor overhead de creación de procesos.

**Corto plazo (este mes)**

Evaluar si dos nodos son suficientes para la carga operativa real. Si el negocio requiere más de 48 workers simultáneos (2 nodos × 24 workers), es necesario agregar RAM al servidor. La recomendación de SAP para instalaciones con múltiples empresas y Service Layer en el mismo servidor que HANA es generalmente 256 GB o más.

Implementar alertas de Prometheus sobre el crecimiento de workers — específicamente sobre el RSS de los procesos `httpd` bajo `b1service0`. Una alerta cuando cualquier worker supera los 10 GB daría tiempo de reacción antes de que el sistema entre en thrashing.

**Mediano plazo (próximos meses)**

La solución arquitectónica correcta es **separar el Service Layer de HANA en servidores distintos**. HANA es una base de datos in-memory que necesita toda la RAM disponible para funcionar eficientemente. El Service Layer es una aplicación de aplicación que tiene sus propias demandas de memoria. Compartir el mismo servidor crea competencia inevitable por los recursos.

Alternativamente, si la separación física no es posible a corto plazo, se puede considerar limitar el número de empresas que cada nodo del Service Layer tiene permitido cargar simultáneamente, o implementar un límite de memoria por proceso mediante cgroups.

---

## 8. Referencia Rápida de Diagnóstico

Para detectar recurrencia de este problema antes de que llegue a crisis:

```bash
# Estado general
free -h
vmstat 1 5   # si/so deben ser 0; wa debe ser <1%

# Workers pesados del Service Layer
ps -o pid,rss,etime,comm -u b1service0 | grep httpd | sort -k2 -rn | head -10
# Alerta si RSS de algún worker supera 5 GB

# Swap por usuario
awk -v uid="$(id -u b1service0)" '
  FNR==1{rss=0;swap=0}
  /^Uid/{cur=$2}
  /^VmRSS/{rss=$2}
  /^VmSwap/{swap=$2}
  cur==uid&&/^VmSwap/{sr+=rss;ss+=swap}
  END{printf "b1service0 RSS=%.1fGB SWAP=%.1fGB\n",sr/1024/1024,ss/1024/1024}
' /proc/*/status 2>/dev/null

# Verificar que solo Node1 y Node2 están corriendo
systemctl is-active b1s50001.service b1s50002.service b1s50003.service b1s50004.service
```

---

## 9. Lecciones Aprendidas

`mpm_prefork` es fundamentalmente incompatible con aplicaciones que cargan grandes volúmenes de datos en memoria por proceso, a menos que el número de workers esté estrictamente acotado por la RAM disponible y no por la carga de usuarios esperada. La combinación de `mpm_prefork` con una aplicación como SAP B1 Service Layer que carga datos de múltiples empresas por worker requiere planificación explícita de capacidad.

El swap en Linux no es un error ni una falla — es una herramienta del kernel para manejar presión de memoria. El thrashing ocurre cuando la demanda de memoria supera tanto la RAM que el kernel pasa más tiempo moviendo páginas que ejecutando código útil. La detección temprana (workers con RSS > umbral, `vmstat so > 0` sostenido) permite intervenir antes de que el sistema se degrade completamente.

HANA y SAP Business One Service Layer en el mismo servidor de 112 GB es una configuración que opera sin margen de seguridad. Cualquier pico de actividad en el Service Layer afecta directamente el rendimiento de HANA y viceversa.

---

_Relacionado: [[roadmap-observabilidad]] · [[arquitectura-sap]] · [[monitoreo-hana]]_