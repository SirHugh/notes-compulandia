**Fecha:** Marzo 2026 · **Sistema:** KVM Guest · SLES for SAP · `/dev/mapper/system-root`

---

## Contexto

El volumen raíz `/` estaba al **98% de capacidad**, bloqueando la instalación de exportadores Prometheus para el monitoreo SAP.

---

## Diagnóstico

```bash
df -h              # estado de filesystems
vgdisplay          # espacio libre en Volume Group
lsblk              # estructura de discos y LVM
df -Th /           # tipo de filesystem del volumen raíz
```

### Resultado clave

|Elemento|Valor|
|---|---|
|Volumen problemático|`/dev/mapper/system-root`|
|Filesystem|`xfs`|
|Tamaño inicial|`10 GB`|
|Uso|`9.8 GB (98%)`|
|Libre en VG `system`|**52 GB disponibles**|
|Swap asignado|`113 GB` (desproporcionado para HANA)|

### Estructura LVM identificada

```
sde (200G)
└─sde2 (200G)
  ├─system-swap   113G   [SWAP]
  ├─system-root    10G   /        ← problema
  └─system-home    25G   /home
  
  Free PE: 13312 / 52 GB disponibles en VG
```

> **Causa raíz:** el volumen `system-root` fue aprovisionado con solo 10 GB en la instalación original. El swap de 113 GB consumió la mayor parte del espacio disponible del VG.

---

## Solución — Expansión LVM Online

Sin downtime · Sin reiniciar SAP HANA · 3 comandos

### Paso 1 — Extender el Logical Volume

```bash
lvextend -L +30G /dev/mapper/system-root
```

```
Size of logical volume system/root changed from 10.00 GiB to 40.00 GiB.
Logical volume system/root successfully resized.
```

### Paso 2 — Expandir el filesystem XFS en caliente

```bash
xfs_growfs /
```

### Paso 3 — Verificar

```bash
df -h /
```

```
Filesystem               Size  Used Avail Use%  Mounted on
/dev/mapper/system-root   40G   9.8G   31G  25%  /
```

---

## Resultado

```
ANTES →  system-root   10G   9.8G   249M   98%   ⛔
DESPUÉS → system-root   40G   9.8G    31G   25%   ✅
```

|Métrica|Antes|Después|
|---|---|---|
|Tamaño `/`|10 GB|40 GB|
|Espacio libre|249 MB|31 GB|
|Uso|98%|25%|
|VG libre restante|52 GB|22 GB|

---

## Notas

- `xfs_growfs` opera **online** — no requiere desmontar ni reiniciar
- El VG `system` conserva **22 GB libres** para expansiones futuras
- El swap de **113 GB** es excesivo para SAP HANA (SAP recomienda mínimo o deshabilitado). Reducción planificada para etapa posterior.
- Próximo paso: instalación de `node_exporter` — **Fase 1 del plan de observabilidad**