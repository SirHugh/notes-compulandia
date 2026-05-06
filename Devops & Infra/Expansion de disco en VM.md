# Expansión de disco en VM Proxmox con LVM

## Contexto

Aplica cuando se amplía el disco de una VM desde Proxmox pero el sistema operativo invitado no refleja el nuevo tamaño. El espacio agregado queda sin usar hasta completar la expansión dentro de la VM.

Layout típico afectado:

```
sda
├─sda1          /boot
├─sda2          Extended (contenedor MBR)
└─sda5          LVM PV
  ├─<vg>-root   /
  └─<vg>-swap
```

---

## 1. Diagnóstico inicial

```bash
lsblk        # verificar layout y tamaños actuales
df -h        # ver uso del filesystem
```

Confirmá que `sda` ya muestra el nuevo tamaño (ej. 130G o 180G) pero `sda5` sigue con el tamaño anterior.

---

## 2. Liberar espacio si el disco está al 100%

Si el disco está lleno y no hay espacio para instalar paquetes:

```bash
# Eliminar paquetes huérfanos y kernels viejos
sudo apt autoremove -y

# Limpiar caché de descargas de apt
sudo apt clean

# Eliminar logs antiguos (opcional, con precaución)
sudo journalctl --vacuum-time=7d
```

---

## 3. Instalar herramientas necesarias

```bash
sudo apt install parted cloud-guest-utils -y
```

> `parted` expande la partición Extended. `growpart` (incluido en `cloud-guest-utils`) expande la partición lógica.

---

## 4. Expandir particiones

### 4.1 Expandir la partición Extended (sda2)

```bash
sudo parted /dev/sda resizepart 2 100%
```

> Si pregunta `Fix/Ignore?`, responder `Fix`.

### 4.2 Expandir la partición lógica (sda5)

```bash
sudo growpart /dev/sda 5
```

Resultado esperado: `CHANGED: partition=5 ...`

---

## 5. Expandir la cadena LVM

```bash
# Notificar al PV del nuevo tamaño
sudo pvresize /dev/sda5

# Extender el Logical Volume usando todo el espacio libre
sudo lvextend -l +100%FREE /dev/<nombre-vg>/root

# Redimensionar el filesystem (ext4)
sudo resize2fs /dev/mapper/<nombre--vg>-root
```

> Reemplazar `<nombre-vg>` con el nombre real del Volume Group (visible en `lsblk`, ej. `ecommerce-vg` o `integrador-vg`).  
> En el mapper, los guiones del VG se duplican: `ecommerce-vg` → `ecommerce--vg`.

---

## 6. Verificación

```bash
df -h /
```

El filesystem `/` debe reflejar el nuevo tamaño total disponible.

---

## Referencia rápida — Comandos completos en orden

```bash
sudo apt autoremove -y && sudo apt clean
sudo apt install parted cloud-guest-utils -y
sudo parted /dev/sda resizepart 2 100%
sudo growpart /dev/sda 5
sudo pvresize /dev/sda5
sudo lvextend -l +100%FREE /dev/<vg>/root
sudo resize2fs /dev/mapper/<vg--name>-root
df -h /
```