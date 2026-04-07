**Fecha:** 19 de marzo de 2026  
**Servidor:** Dell PowerEdge R740xd  
**Elaborado a partir de:** relevamiento directo del sistema

---

## Resumen del Objetivo

Redistribuir la memoria del servidor para aumentar la RAM disponible para la VM SAP HANA (Hana), reduciendo la asignada a la VM Windows (Win2k22), manteniendo el aislamiento NUMA estricto y cumplimiento de licenciamiento SAP.

---

## Estado Actual vs Estado Objetivo

| Componente              | Estado Actual      | Estado Objetivo                        |
| ----------------------- | ------------------ | -------------------------------------- |
| Licencia SAP HANA       | 64 GB              | ≥ 128 GB (gestión comercial)           |
| global_allocation_limit | No configurado     | 65.536 MB → luego ampliar con licencia |
| DIMMs socket A (Hana)   | 4 × 32 GB = 128 GB | 6 × 32 GB = 192 GB                     |
| DIMMs socket B (Win)    | 4 × 32 GB = 128 GB | 2 × 32 GB = 64 GB                      |
| RAM VM Hana             | 115 GB             | 192 GB                                 |
| RAM VM Win2k22          | 115 GB             | 64 GB                                  |
| HugePages nodo 0        | 115                | 192                                    |
| HugePages nodo 1        | 115                | 64                                     |

---

## Fases del Plan

El plan se divide en tres fases secuenciales. **No se puede avanzar a la siguiente fase sin completar la anterior.**

```
Fase 1 → Fase 2 → Fase 3
Compliance   Física    Software
  SAP       Hardware    KVM
```

---

## Fase 1 — Regularización de Licenciamiento SAP

**Prerrequisito de todo lo demás. No es opcional.**

### 1.1 Configurar global_allocation_limit inmediatamente

Aplicar el límite de memoria en HANA para alinearse con la licencia actual de 64 GB. Esto detiene el crecimiento del uso y demuestra acción correctiva ante una eventual auditoría SAP.

```bash
# Conectarse al guest Hana
ssh <usuario>@<ip-hana>
su - ndbadm

# Aplicar límite (en caliente, sin reiniciar HANA)
hdbsql -i 00 -u SYSTEM -p <password> \
  "ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM') \
   SET ('memorymanager', 'global_allocation_limit') = '65536' \
   WITH RECONFIGURE;"
```

**Verificar que quedó aplicado:**

```bash
grep -i global_allocation_limit \
  /usr/sap/NDB/SYS/global/hdb/custom/config/global.ini
```

Debe devolver:

```
global_allocation_limit = 65536
```

**Verificar que HANA lo reconoce:**

```sql
SELECT PRODUCT_LIMIT, PRODUCT_USAGE, ENFORCED, LOCKED_DOWN
FROM SYS.M_LICENSE;
```

> ⚠️ **Advertencia de impacto:** al aplicar el límite de 64 GB, HANA liberará ~27 GB de datos que hoy tiene en RAM. Esto generará mayor actividad de I/O mientras el sistema restablece su estado. Se recomienda aplicarlo en horario de bajo uso.

---

### 1.2 Gestión comercial — ampliar licencia

Contactar al partner SAP para ampliar la licencia de 64 GB al valor que soporte el nuevo dimensionamiento objetivo.

|Escenario|Licencia recomendada|
|---|---|
|Objetivo conservador|128 GB|
|Objetivo con margen de crecimiento|192 GB|

La licencia define el techo real — no tiene sentido asignar más RAM física a la VM si SAP no puede usarla dentro del compliance.

**Documentación a presentar al partner:**

- Hardware Key actual: `R3559236425`
- Sistema ID: NDB
- Nro. instalación: 0021280667
- Uso actual medido: 91 GB
- Capacidad física objetivo: 192 GB

---

### 1.3 Actualizar global_allocation_limit con nueva licencia

Una vez instalada la nueva licencia, ajustar el límite al nuevo valor:

```sql
-- Ejemplo para licencia de 192 GB:
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM')
SET ('memorymanager', 'global_allocation_limit') = '196608'
WITH RECONFIGURE;
```

|Licencia objetivo|Valor en MB|
|---|---|
|128 GB|131072|
|192 GB|196608|

---

## Fase 2 — Intervención Física de Hardware

**Ejecutar solo después de tener la nueva licencia SAP instalada.**

### 2.1 Preparación previa

```bash
# En el host — backup de XMLs de ambas VMs
export LIBVIRT_DEFAULT_URI=qemu:///system
virsh dumpxml Hana   > /root/backup-hana-$(date +%Y%m%d).xml
virsh dumpxml Win2k22 > /root/backup-win2k22-$(date +%Y%m%d).xml

# Verificar que los backups existen
ls -lh /root/backup-*.xml
```

### 2.2 Apagado ordenado

```bash
# 1. Apagar SAP HANA desde adentro del guest (preferible a shutdown de VM)
# Conectarse al guest:
su - ndbadm
HDB stop

# 2. Apagar VMs desde el host
virsh shutdown Hana
virsh shutdown Win2k22

# Esperar confirmación:
watch virsh list --all
# Ambas deben quedar en state "shut off"

# 3. Apagar el servidor desde iDRAC o:
sudo shutdown -h now
```

### 2.3 Movimiento de DIMMs

**Mover exactamente estos dos módulos:**

|DIMM origen|DIMM destino|Capacidad|
|---|---|---|
|B4|A3|32 GB|
|B5|A6|32 GB|

**Distribución resultante:**

|Socket|Slots ocupados|Canales activos|RAM total|
|---|---|---|---|
|CPU A (Hana)|A1, A2, A3, A4, A5, A6|6 de 6 ✓|**192 GB**|
|CPU B (Win)|B1, B2|2 de 6|**64 GB**|

> ⚠️ **Regla Dell R740xd:** poblar siempre los slots de pestaña blanca antes que los negros. A3 y A6 son pestañas blancas — correcto para esta configuración.

> ⚠️ **ESD:** manipular DIMMs con pulsera antiestática. El servidor debe estar completamente desenergizado (cable de poder desconectado) antes de tocar los módulos.

### 2.4 Verificación post-arranque en iDRAC

Antes de encender el servidor, verificar en iDRAC:

```
System → Hardware → Memory
```

Confirmar:

- Slots A1–A6: 32 GB c/u → Total 192 GB en CPU A
- Slots B1–B2: 32 GB c/u → Total 64 GB en CPU B
- Sin errores de memoria reportados

### 2.5 Verificar en el OS del host

```bash
# Topología NUMA post-cambio (debe mostrar ~192 GB en nodo 0):
numactl --hardware

# Resultado esperado:
# node 0 size: ~192000 MB
# node 1 size: ~64000 MB
```

---

## Fase 3 — Reconfiguración Software

**Ejecutar con el servidor encendido y VMs aún apagadas.**

### 3.1 Ajustar HugePages por nodo NUMA

```bash
# Liberar hugepages actuales y reasignar
# Nodo 0 → 192 hugepages (192 GB para Hana)
echo 192 | sudo tee \
  /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages

# Nodo 1 → 64 hugepages (64 GB para Win2k22)
echo 64 | sudo tee \
  /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages

# Verificar:
cat /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
# → 192
cat /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages
# → 64
grep -i huge /proc/meminfo
# HugePages_Total debe mostrar 256 (192+64)
```

**Hacer el cambio persistente en el host:**

```bash
# Editar /etc/sysctl.conf o crear archivo dedicado:
sudo vi /etc/sysctl.d/99-hugepages.conf

# Contenido:
vm.nr_hugepages = 256

# Aplicar:
sudo sysctl -p /etc/sysctl.d/99-hugepages.conf
```

> ⚠️ El ajuste por nodo via `/sys/devices/system/node/` es necesario además del global, ya que el sistema debe saber cuántas asignar por nodo NUMA para respetar el pinning estricto de las VMs.

---

### 3.2 Ajustar XML de VM Hana

```bash
export LIBVIRT_DEFAULT_URI=qemu:///system
virsh edit Hana
```

Modificar los siguientes bloques:

**Memoria principal** (cambiar 120586240 → 201326592, equivalente a 192 GB en KiB):

```xml
<!-- Antes: -->
<memory unit='KiB'>120586240</memory>
<currentMemory unit='KiB'>120586240</currentMemory>

<!-- Después: -->
<memory unit='KiB'>201326592</memory>
<currentMemory unit='KiB'>201326592</currentMemory>
```

**Celda NUMA interna de la VM:**

```xml
<!-- Antes: -->
<cell id='0' cpus='0-39' memory='120586240' unit='KiB'>

<!-- Después: -->
<cell id='0' cpus='0-39' memory='201326592' unit='KiB'>
```

**Verificar que numatune sigue en nodo 0** (no cambiar):

```xml
<numatune>
  <memory mode='strict' nodeset='0'/>
  <memnode cellid='0' mode='strict' nodeset='0'/>
</numatune>
```

---

### 3.3 Ajustar XML de VM Win2k22

```bash
virsh edit Win2k22
```

Cambiar a 64 GB (67108864 KiB):

```xml
<!-- Antes: -->
<memory unit='KiB'>120586240</memory>
<currentMemory unit='KiB'>120586240</currentMemory>

<!-- Después: -->
<memory unit='KiB'>67108864</memory>
<currentMemory unit='KiB'>67108864</currentMemory>
```

**Celda NUMA interna:**

```xml
<!-- Antes: -->
<cell id='0' cpus='0-19' memory='120586240' unit='KiB'>

<!-- Después: -->
<cell id='0' cpus='0-19' memory='67108864' unit='KiB'>
```

---

### 3.4 Arrancar VMs y verificar

```bash
# Arrancar Hana primero
virsh start Hana

# Verificar que levantó correctamente
virsh dommemstat Hana
# actual debe mostrar ~201326592

# Arrancar Win2k22
virsh start Win2k22

virsh dommemstat Win2k22
# actual debe mostrar ~67108864
```

**Verificar hugepages post-arranque:**

```bash
grep -i huge /proc/meminfo
# HugePages_Free debe ser 0 (todas asignadas)
# HugePages_Total debe ser 256
```

---

### 3.5 Verificar HANA post-arranque

```bash
# Dentro del guest Hana:
su - ndbadm
hdbsql -i 00 -u SYSTEM -p <password> \
  "SELECT PRODUCT_LIMIT, PRODUCT_USAGE, ENFORCED, LOCKED_DOWN FROM SYS.M_LICENSE;"
```

```sql
-- Verificar memoria disponible:
SELECT
  ROUND(FREE_PHYSICAL_MEMORY/1024/1024/1024,1) AS FREE_GB,
  ROUND(USED_PHYSICAL_MEMORY/1024/1024/1024,1) AS USED_GB,
  ROUND(ALLOCATION_LIMIT/1024/1024/1024,1) AS LIMIT_GB
FROM SYS.M_HOST_RESOURCE_UTILIZATION;
```

```bash
# Verificar que el global_allocation_limit está activo:
grep -i global_allocation_limit \
  /usr/sap/NDB/SYS/global/hdb/custom/config/global.ini
```

---

## Tabla de Valores de Referencia

### Conversión de memoria a KiB (para XMLs)

|GB|KiB|
|---|---|
|64 GB|67.108.864|
|115 GB|120.586.240 (valor actual)|
|128 GB|134.217.728|
|192 GB|201.326.592|

### Conversión a MB (para global_allocation_limit)

|GB|MB|
|---|---|
|64 GB|65.536|
|128 GB|131.072|
|192 GB|196.608|

---

## Checklist de Ejecución

### Fase 1 — Compliance SAP

- [ ] Aplicar `global_allocation_limit = 65536` en horario de bajo uso
- [ ] Verificar que `global.ini` refleja el cambio
- [ ] Verificar `M_LICENSE` — confirmar `ENFORCED = TRUE`, `LOCKED_DOWN = FALSE`
- [ ] Contactar partner SAP con Hardware Key y datos de uso
- [ ] Obtener e instalar nueva licencia
- [ ] Actualizar `global_allocation_limit` al nuevo valor de licencia

### Fase 2 — Hardware

- [ ] Backup de XMLs de ambas VMs
- [ ] Coordinar ventana de mantenimiento con usuarios de Win2k22
- [ ] Detener SAP HANA correctamente desde el guest
- [ ] Apagar ambas VMs con `virsh shutdown`
- [ ] Apagar el servidor físicamente
- [ ] Mover DIMM B4 → A3
- [ ] Mover DIMM B5 → A6
- [ ] Encender servidor
- [ ] Verificar en iDRAC: 6 slots en CPU A, 2 slots en CPU B, sin errores
- [ ] Verificar en OS: `numactl --hardware` muestra ~192 GB en nodo 0

### Fase 3 — Software

- [ ] Ajustar hugepages nodo 0: 192
- [ ] Ajustar hugepages nodo 1: 64
- [ ] Hacer hugepages persistentes en `/etc/sysctl.d/`
- [ ] Editar XML Hana: memoria → 201326592 KiB
- [ ] Editar XML Win2k22: memoria → 67108864 KiB
- [ ] Iniciar VM Hana — verificar `dommemstat`
- [ ] Iniciar VM Win2k22 — verificar `dommemstat`
- [ ] Verificar `M_LICENSE` en HANA
- [ ] Verificar `M_HOST_RESOURCE_UTILIZATION` en HANA
- [ ] Monitorear swap_in en las horas siguientes al arranque

---

## Consideraciones de Riesgo

|Riesgo|Probabilidad|Impacto|Mitigación|
|---|---|---|---|
|Impacto de performance al limitar a 64 GB (Fase 1)|Alta|Medio|Aplicar en horario nocturno, monitorear swap|
|HANA no levanta por hugepages insuficientes|Baja|Alto|Verificar hugepages antes de arrancar VMs|
|Error en XML de VM (VM no arranca)|Baja|Alto|Backup de XMLs previo, probar en staging si es posible|
|DIMM mal asentado (POST error)|Baja|Alto|Verificar en iDRAC antes de arrancar OS|
|SAP bloquea sistema antes de ampliar licencia|Media|Muy alto|Ejecutar Fase 1.1 hoy como acción de emergencia|

---

_Plan elaborado el 19/03/2026 — basado en relevamiento directo del sistema_