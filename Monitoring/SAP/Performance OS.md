# SAP HANA — Cambios Aplicados (Bloque 1: OS)

**Servidor:** sap-linux | **OS:** SUSE Linux Enterprise (SLES) | **Fecha:** 2026-03-26

---

## 1. Transparent Huge Pages (THP)

**Problema:** THP activo (`always`) interfiere con el gestor de memoria de HANA.

```bash
# Aplicar en caliente
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
 
```

| Antes   | Después   |         |
| ------- | --------- | ------- |
| enabled | `always`  | `never` |
| defrag  | `madvise` | `never` |

---

## 2. vm.swappiness

**Problema:** Valor default (60) permite swap agresivo — crítico en base de datos in-memory.

```bash
# Aplicar en caliente
sysctl -w vm.swappiness=10
```

```bash
# Persistencia
cat > /etc/sysctl.d/99-sap-hana.conf << 'EOF'
# SAP HANA Performance Tuning - OS Parameters
vm.swappiness = 10
EOF

sysctl -p /etc/sysctl.d/99-sap-hana.conf
```

|Antes|Después|
|---|---|---|
|vm.swappiness|`60`|`10`|

---

## 3. Parámetros verificados — Sin cambio requerido

Los siguientes parámetros ya superaban los mínimos recomendados por SAP:

|Parámetro|Valor actual|Mínimo SAP|
|---|---|---|
|vm.dirty_ratio|`10`|`10`|
|vm.dirty_background_ratio|`5`|`5`|
|kernel.shmmni|`32768`|`4096`|
|fs.file-max|`9.2e18`|`200000`|
|fs.aio-max-nr|`max`|`max`|

---

## 4. I/O Scheduler — Sin cambio requerido

```bash
# Verificación
systemd-detect-virt   # → kvm
lsblk -d -o NAME,ROTA,TYPE,MODEL
```

Discos virtuales QEMU sobre KVM. El scheduler `none` es correcto — el hipervisor gestiona el I/O sobre el hardware físico real.

---

## 5. Mount options — noatime / nodiratime

**Problema:** `relatime` (default) genera escrituras innecesarias de timestamps en cada lectura.  
**Referencia:** Recomendación explícita SAP para todos los volúmenes HANA.

```bash
# Backup previo
cp /etc/fstab /etc/fstab.bak.20260326

# Editar /etc/fstab — cambiar defaults por defaults,noatime,nodiratime
# en los tres volúmenes HANA

# Verificar sintaxis
mount -a --fake -v | grep hana

# Aplicar en caliente
mount -o remount,noatime,nodiratime /hana/shared
mount -o remount,noatime,nodiratime /hana/log
mount -o remount,noatime,nodiratime /hana/data

# Verificar
mount | grep hana
```

**`/etc/fstab` resultante:**

```
UUID=9be3a6f1-...  /hana/shared  xfs  defaults,noatime,nodiratime  0  0
UUID=ee7de12d-...  /hana/log     xfs  defaults,noatime,nodiratime  0  0
UUID=c1fba068-...  /hana/data    xfs  defaults,noatime,nodiratime  0  0
```

|Antes|Después|
|---|---|---|
|/hana/log|`relatime`|`noatime,nodiratime`|
|/hana/data|`relatime`|`noatime,nodiratime`|
|/hana/shared|`relatime`|`noatime,nodiratime`|

---

## Baseline I/O — Prometheus (antes de noatime)

|Métrica|sda `/hana/log`|sdc `/hana/data`|
|---|---|---|
|Escrituras/s|8.07|8.57|
|Ratio escrituras/lecturas|62.9|17.9|
|Latencia escritura|0.16 ms|2.50 ms|
|Bytes escritos/s|85 KB/s|637 KB/s|

**Query de seguimiento:**

```promql
rate(node_disk_writes_completed_total{
  job="prometheus.scrape.sap_linux",
  instance="10.100.102.10:9100",
  exporter="node",
  red="red_sap",
  device=~"sda|sdc"
}[5m])
```