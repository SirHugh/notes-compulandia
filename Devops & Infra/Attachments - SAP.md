# Troubleshooting — SAP B1 Service Layer: Fallo en montaje CIFS y subida de Attachments

**Servidor:** `sap-linux` (SUSE Linux, IP: 10.100.102.10)  
**Fecha de incidencia:** 31 de marzo de 2026  
**Componentes afectados:** Samba 4.15.13, CIFS loopback mount, SAP B1 Service Layer

---

## 1. Síntomas

- `mount -t cifs //sap-linux/B1_SHF/ANEXOS/Attachments` retornaba `mount error(13): Permission denied`
- `POST /b1s/v2/Attachments2` retornaba HTTP 400 con error `-5002`: _"Unable to copy file to Attachments directory. Access is denied."_
- Archivos subidos aparecían en el directorio destino con tamaño 0 bytes
- Comandos como `df -h` y `ls` se colgaban indefinidamente sobre el punto de montaje

---

## 2. Diagnóstico

### 2.1 Fallo de autenticación NTLMSSP

```
gensec_spnego_client_negTokenTarg_step: SPNEGO(ntlmssp) login failed: NT_STATUS_INVALID_PARAMETER
```

**Causa:** El usuario `root` no existía en la base de datos de Samba (`pdbedit -L` sin salida). Sin cuenta Samba, la autenticación NTLMSSP falla con `NT_STATUS_INVALID_PARAMETER`.

**Causa raíz adicional:** El servidor `sap-linux` resuelve a su propia IP (`10.100.102.10`), por lo que el mount es un **loopback CIFS** — el cliente y servidor son la misma máquina. Este escenario requiere configuración especial en Samba.

### 2.2 Deadlock en operaciones sobre el mount

Con la cuenta Samba creada el mount procedía, pero cualquier operación de directorio (`ls`, `df`) se colgaba indefinidamente. Causa: Samba con `unix extensions = yes` (default) genera deadlocks en conexiones loopback.

### 2.3 Permisos incorrectos en el mount CIFS

El mount se realizó sin especificar `uid`/`gid`, resultando en:

```
uid=0,gid=0,noforceuid,noforcegid
```

El Service Layer corre bajo el usuario `b1service0` (UID 1002). Al intentar escribir en un directorio montado como `root`, recibe "Access is denied" — error `-5002`.

### 2.4 Directorio de destino con permisos insuficientes

```
drwxr-xr-x 2 root root  Attachments  ← sin bit de escritura para b1service0
```

El directorio físico `/usr/sap/SAPBusinessOne/B1_SHF/ANEXOS/Attachments` tenía dueño `root` y modo `755`, impidiendo la escritura por parte de `b1service0`.

### 2.5 Crashes del Service Layer — `double free or corruption`

Los logs de `/usr/sap/SAPBusinessOne/ServiceLayer/logs/error_5000*` revelaron:

```
double free or corruption (!prev)
child pid XXXXX exit signal Aborted (6)
```

Los workers del Service Layer (puertos 50001–50004) abortaban vía SIGABRT al procesar requests de `Attachments2`. Los crashes comenzaron el **31 de marzo a las 11:53**, coincidiendo con la actualización reciente de SAP B1. El sistema además opera con presión de memoria (~104 GiB / 112 GiB usados + 6.6 GiB en swap).

---

## 3. Resolución

### Paso 1 — Crear cuenta Samba para root

```bash
smbpasswd -a root
pdbedit -L  # verificar que root aparece en la lista
```

### Paso 2 — Corregir smb.conf para loopback CIFS

Agregar en la sección `[global]` de `/etc/samba/smb.conf`:

```ini
unix extensions = no
allow insecure wide links = yes
```

Agregar en la sección `[B1_SHF]`:

```ini
oplocks = no
level2 oplocks = no
```

Aplicar cambios:

```bash
testparm -s
systemctl restart smb
```

### Paso 3 — Corregir permisos del directorio físico

```bash
chown b1service0:b1service0 /usr/sap/SAPBusinessOne/B1_SHF/ANEXOS/Attachments
chmod 775 /usr/sap/SAPBusinessOne/B1_SHF/ANEXOS/Attachments
```

### Paso 4 — Remontar con uid/gid correcto

```bash
umount -l /mnt/anexos/Attachments

mount -t cifs //sap-linux/B1_SHF/ANEXOS/Attachments /mnt/anexos/Attachments \
  -o username=root,password=<password>,vers=3.0,noperm,noserverino,cache=none,\
uid=$(id -u b1service0),gid=$(id -g b1service0),file_mode=0755,dir_mode=0775
```

Verificación:

```bash
sudo -u b1service0 touch /mnt/anexos/Attachments/test.txt && echo "OK escritura"
```

### Paso 5 — Persistencia en /etc/fstab

```bash
# Crear archivo de credenciales
echo -e "username=root\npassword=<password>" > /etc/samba/creds_b1shf
chmod 600 /etc/samba/creds_b1shf

# Entrada fstab
echo "//sap-linux/B1_SHF/ANEXOS/Attachments /mnt/anexos/Attachments cifs \
credentials=/etc/samba/creds_b1shf,vers=3.0,noperm,noserverino,cache=none,\
uid=$(id -u b1service0),gid=$(id -g b1service0),file_mode=0755,dir_mode=0775,_netdev 0 0" \
>> /etc/fstab

mount -a && echo "fstab OK"
```

---

## 4. Estado pendiente

|Item|Estado|Acción requerida|
|---|---|---|
|Mount CIFS loopback|✅ Resuelto|—|
|Escritura por `b1service0`|✅ Resuelto|—|
|Error -5002 Attachments2|✅ Resuelto|—|
|Crashes `double free` en Service Layer|⚠️ Activo|Identificar versión exacta del SL y buscar SAP Note aplicable|
|Presión de memoria (104/112 GiB)|⚠️ Monitorear|Evaluar expansión de RAM o rebalanceo de servicios|

---

## 5. Lecciones aprendidas

- En Samba, los usuarios Linux **no son automáticamente cuentas Samba** — deben crearse explícitamente con `smbpasswd -a`.
- Los mounts CIFS loopback (mismo host como cliente y servidor) requieren `unix extensions = no` y `oplocks = no` para evitar deadlocks.
- Al montar CIFS para uso por un usuario no-root, siempre especificar `uid=` y `gid=` explícitamente.
- Los crashes `double free` en el Service Layer de SAP B1 Linux son bugs conocidos asociados a versiones específicas — verificar SAP Notes tras actualizaciones.