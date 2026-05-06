# Auditoría de Almacenamiento — Debian

## Discos y particiones

```bash
df -hT                        # Uso de filesystems montados
df -i                         # Uso de inodos
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID
fdisk -l                      # Tabla de particiones
parted -l
```

## Uso por directorios

```bash
du -hx --max-depth=1 / | sort -rh | head -10
du -ahx /var | sort -rh | head -20
du -sh /home /var /opt /tmp
```

## Archivos grandes

```bash
find / -xdev -type f -size +100M 2>/dev/null | xargs ls -lh | sort -k5 -rh
```

## Logs y temporales

```bash
journalctl --disk-usage
du -sh /var/log/* | sort -rh | head -10
du -sh /tmp /var/tmp
```

## LVM / Snapshots

```bash
lvs && vgs && pvs
lvs -o +snap_percent
```

## Archivos eliminados pero aún abiertos

```bash
lsof +L1                      # Lista archivos eliminados con handles abiertos
lsof +L1 | awk 'NR>1 {sum += $7} END {print sum/1024/1024 " MB retenidos"}'
```

## Herramientas interactivas

```bash
apt install ncdu -y && ncdu /  # Navegación visual
apt install duf -y  && duf     # Alternativa moderna a df
```

---

## Flujo de diagnóstico

```
df -hT
  → du -hx --max-depth=1 <mountpoint>
      → ncdu <directorio>
          → lsof +L1   # si el espacio no cuadra tras borrar archivos
```