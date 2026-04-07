# Resumen Ejecutivo — Estado Actual SAP HANA

**Fecha de relevamiento:** 19 de marzo de 2026  
**Servidor:** Dell PowerEdge R740xd  
**Sistema:** SAP HANA NDB — instancia HDB00

---

## 1. Infraestructura del Servidor

|Atributo|Valor|
|---|---|
|Modelo|Dell PowerEdge R740xd|
|Procesadores|2 × Intel (20 cores físicos c/u, 40 threads c/u)|
|Total cores lógicos|80 (40 físicos + HyperThreading)|
|RAM total instalada|256 GB|
|Slots de memoria|24 (8 ocupados, 16 vacíos)|
|Configuración DIMMs|8 × 32 GB DDR4 2666 MT/s RDIMM|
|Arquitectura NUMA|2 nodos (1 por socket)|
|RAM por nodo NUMA|~128.5 GB (nodo 0) / ~128.6 GB (nodo 1)|
|Virtualización|KVM sobre SUSE Linux Enterprise|

### Distribución física de DIMMs

|Socket|Slots ocupados|Slots vacíos|Total RAM|
|---|---|---|---|
|CPU A (nodo NUMA 0)|A1, A2, A4, A5|A3, A6–A12|128 GB|
|CPU B (nodo NUMA 1)|B1, B2, B4, B5|B3, B6–B12|128 GB|

---

## 2. Máquinas Virtuales

|Atributo|Hana (KVM 1)|Win2k22 (KVM 2)|
|---|---|---|
|OS Guest|SUSE Linux Enterprise 15.6|Windows Server 2022|
|RAM asignada|115 GB|115 GB|
|vCPUs|40|20|
|Nodo NUMA|Estricto — nodo 0|Estricto — nodo 1|
|HugePages|Sí (1 GB c/u)|Sí (1 GB c/u)|
|HugePages asignadas|115|115|
|Estado|Running|Running|

### HugePages del host

|Parámetro|Valor|
|---|---|
|Tamaño de página|1 GB (no las comunes de 2 MB)|
|Total reservadas|230|
|Libres|0 — **100% asignadas**|
|Nodo 0|115 hugepages → 115 GB|
|Nodo 1|115 hugepages → 115 GB|

---

## 3. Estado de Memoria — VM Hana

|Métrica|Valor|Observación|
|---|---|---|
|RAM asignada a la VM|115 GB|Configuración KVM|
|Disponible en guest|~118 GB|Lo que el SO ve|
|En uso por procesos HANA|102.2 GB|`M_HOST_RESOURCE_UTILIZATION`|
|Libre en VM|10.5 GB|Margen crítico|
|Límite interno HANA|104.8 GB|~90% de RAM de VM|
|Swap utilizado (swap_in)|~3.8 GB|**Señal de presión de memoria**|

El swap activo indica que la VM ya está bajo presión de memoria con la configuración actual.

---

## 4. Estado de Licenciamiento SAP HANA ⚠️

| Campo                                           | Valor                  |
| ----------------------------------------------- | ---------------------- |
| Sistema                                         | NDB                    |
| Producto                                        | SAP-HANA-ENF           |
| Nro. instalación                                | 0021280667             |
| Memoria licenciada (`PRODUCT_LIMIT`)            | **64 GB**              |
| Memoria en uso medida por SAP (`PRODUCT_USAGE`) | **91 GB**              |
| Exceso sobre licencia                           | **+27 GB (+42%)**      |
| Licencia válida (`VALID`)                       | TRUE                   |
| Enforcement activo (`ENFORCED`)                 | **TRUE**               |
| Sistema bloqueado (`LOCKED_DOWN`)               | FALSE                  |
| Último chequeo exitoso                          | 2026-03-19 (hoy)       |
| Vencimiento                                     | Sin fecha (permanente) |

### Interpretación

SAP está midiendo activamente el uso de memoria y detecta que el sistema supera la licencia contratada en 27 GB. El enforcement está activo pero el sistema no está bloqueado, lo que indica que está dentro del **período de gracia** establecido por SAP.

Superado el período de gracia sin regularización, SAP puede bloquear el sistema automáticamente, impidiendo nuevas conexiones salvo acceso de administrador.

---

## 5. Configuración de Límite de Memoria HANA ⚠️

```
global_allocation_limit: NO CONFIGURADO
```

El archivo `/usr/sap/NDB/SYS/global/hdb/custom/config/global.ini` no contiene sección `[memorymanager]`. Esto significa que HANA no tiene ningún límite interno configurado por el administrador y puede crecer hasta el techo que el sistema operativo le permita (~104.8 GB según el límite automático del 90%).

---

## 6. Brecha entre uso real y licencia

```
Licencia contratada:          64 GB
Uso medido por SAP:           91 GB   ← lo que SAP factura/audita
Uso físico total HANA:       102 GB   ← procesos + datos + overhead
Límite configurado por admin:  NO HAY

Brecha de licencia:          +27 GB
Brecha real de recursos:     +38 GB sobre licencia
```

---

## 7. Hallazgos Críticos

### Prioridad Alta — Acción Inmediata

**7.1 global_allocation_limit no configurado**  
HANA puede consumir toda la RAM de la VM sin restricción. Debe configurarse inmediatamente en 65.536 MB (64 GB) para alinearse con la licencia contratada.

Comando a ejecutar como `ndbadm`:

```sql
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM')
SET ('memorymanager', 'global_allocation_limit') = '65536'
WITH RECONFIGURE;
```

**Advertencia:** aplicar este límite forzará a HANA a liberar ~27 GB de datos que hoy tiene en RAM. Habrá impacto en performance mientras el sistema se estabiliza.

**7.2 Licencia excedida en 42%**  
El sistema opera con 91 GB de uso medido contra 64 GB de licencia. SAP detecta esta situación en cada chequeo diario. Se requiere gestión comercial urgente con el partner SAP para ampliar la licencia.

### Prioridad Media — Planificación

**7.3 Objetivo original: aumentar RAM de Hana**  
El objetivo de mover DIMMs físicos del socket B al A para ampliar la VM Hana de 115 GB a 192 GB queda condicionado a la resolución del licenciamiento. Ampliar la RAM de la VM sin ampliar la licencia agrava la situación de compliance.

**7.4 Win2k22 — pérdida de estadísticas de balloon**  
`virsh dommemstat Win2k22` retorna solo `actual` y `rss` sin datos de uso interno, lo que sugiere que el balloon driver de Windows no está reportando correctamente. Dificulta evaluar el uso real de memoria de esa VM.

---

## 8. Plan de Acción Recomendado

|Orden|Acción|Urgencia|Responsable|
|---|---|---|---|
|1|Configurar `global_allocation_limit = 65536` en HANA|Inmediata|Administrador SAP|
|2|Verificar `M_LICENSE` post-configuración|Inmediata|Administrador SAP|
|3|Contactar partner SAP para ampliar licencia|Esta semana|Área comercial|
|4|Definir nuevo valor de licencia objetivo (128 GB o más)|Esta semana|Dirección + SAP|
|5|Una vez ampliada la licencia: proceder con cambio físico de DIMMs|Posterior|Infraestructura|
|6|Post cambio de DIMMs: ajustar hugepages y XML de VMs|Posterior|Infraestructura|

---

## 9. Arquitectura Objetivo (Post-Resolución de Licencia)

Si se amplía la licencia a ≥ 128 GB, el cambio físico consiste en:

**Mover:** DIMM B4 → slot A3 y DIMM B5 → slot A6

|Socket|Slots|RAM|VM asignada|
|---|---|---|---|
|CPU A (nodo 0)|A1,A2,A3,A4,A5,A6|**192 GB**|Hana (SAP)|
|CPU B (nodo 1)|B1, B2|**64 GB**|Win2k22|

Esto mantiene aislamiento NUMA estricto para ambas VMs y optimiza el ancho de banda de memoria para HANA (6 canales activos vs 4 actuales).

Los ajustes de sistema requeridos post-cambio físico serían:

```bash
# Hugepages por nodo:
node 0: 192 hugepages (192 GB)
node 1:  64 hugepages ( 64 GB)

# XML Hana:  memoria → 192 GB
# XML Win2k22: memoria → 64 GB
```

---

_Documento generado a partir de relevamiento directo del sistema — 19/03/2026_