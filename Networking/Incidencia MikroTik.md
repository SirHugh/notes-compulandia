# Incidencia MikroTik — DNS y Seguridad

**Fecha:** 2026-03-31  
**Equipo:** MikroTik RB5009UG+S+ — RouterOS 7.16.2  
**Red LAN:** 192.168.100.0/24 — WAN: 181.94.237.126

---

## Síntomas reportados

- Clientes en la red no podían resolver nombres de dominio
- Configurando DNS manualmente en los equipos la salida funcionaba
- Intentos de login fallidos por SSH (usuario `root`) visibles en consola

---

## Diagnóstico

### 1 — Cache DNS saturado

El cache DNS estaba al 100% de capacidad (`2048KiB / 2048KiB`). Con el cache lleno el router no podía procesar nuevas consultas correctamente, resultando en resolución intermitente o fallida para los clientes.

### 2 — DHCP client conflictivo en ether8

La interfaz WAN (`ether8`) tenía configurada una IP estática (`181.94.237.126/24`) y simultáneamente un DHCP client activo en estado `searching`. Este conflicto en la interfaz WAN afectaba el stack de red e impedía que las consultas DNS salieran correctamente hacia los servidores upstream.

### 3 — Malware infatica detectado

Se encontró una regla `layer7-protocol` con el patrón `infatica` — malware conocido en RouterOS que intercepta y redirige tráfico DNS/HTTP de los clientes hacia servidores externos maliciosos.

### 4 — Usuarios no autorizados con acceso total

Se encontraron dos usuarios desconocidos con grupo `full` en el sistema:

- `eratzlaff` — último acceso julio 2025
- Intentos de acceso por fuerza bruta desde `45.163.19.147` vía API

### 5 — Servicios de administración expuestos a internet

Todos los servicios de administración (API, SSH, WinBox, Telnet, FTP, WWW) estaban accesibles desde internet sin restricción de IP.

---

## Acciones aplicadas

### DNS restaurado

```bash
/ip dhcp-client disable [find interface=ether8]
/ip dns set cache-size=4096KiB
/ip dns cache flush
```

### Malware eliminado

```bash
/ip firewall filter remove [find layer7-protocol=infatica_pattern]
/ip firewall layer7-protocol remove [find name=infatica_pattern]
```

### Usuarios eliminados

```bash
/user remove [find name=eratzlaff]
```

### Servicios asegurados

Se deshabilitaron todos los servicios innecesarios. Solo WinBox permanece activo:

```bash
/ip service set telnet disabled=yes
/ip service set ftp disabled=yes
/ip service set www disabled=yes
/ip service set ssh disabled=yes
/ip service set api disabled=yes
/ip service set api-ssl disabled=yes
```

### Cuenta admin deshabilitada

La cuenta `admin` no había sido utilizada desde diciembre 2024 y estaba marcada como expirada. Se deshabilitó para eliminar un vector de acceso innecesario:

```bash
/user disable admin
```

### IP atacante bloqueada

```bash
/ip firewall address-list add list=blacklist address=45.163.19.147 comment="Brute force API"
/ip firewall filter add chain=input src-address-list=blacklist action=drop place-before=0
```

---

## Estado final verificado

|Componente|Antes|Después|
|---|---|---|
|Resolución DNS|❌ Fallida|✅ Funcional|
|Cache DNS|❌ 100% lleno (2048KiB)|✅ 2% usado (4096KiB)|
|DHCP client ether8|❌ Conflicto con IP estática|✅ Deshabilitado|
|Malware infatica|❌ Presente|✅ Eliminado|
|Usuario eratzlaff|❌ Acceso full|✅ Eliminado|
|Usuario admin|❌ Activo sin uso desde dic 2024|✅ Deshabilitado|

---

## Recomendaciones

- Auditar usuarios periódicamente con `/user print`
- Restringir WinBox a IP pública específica: `/ip service set winbox address=192.168.100.0/24,TU.IP/32`
- Monitorear el log regularmente: `/log print follow where topics~"system,error"`
- Considerar netinstall (reinstalación limpia) dado el compromiso confirmado del router
- Cambiar contraseña de `admin` y del usuario administrador activo