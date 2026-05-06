# Guía: Agregar un Nuevo Peer WireGuard en MikroTik

**Aplica a:** RouterOS 7.x  
**Interfaz WireGuard:** `wgAdelantos`  
**Puerto:** `13232 UDP`  
**IP pública del router:** `181.94.197.29`  
**Subred del túnel:** `172.16.1.0/27`  
**Red LAN principal:** `192.168.0.0/24`

---

## Contexto de la red

```
Internet (ether8 → 181.94.197.29)
         │
      MikroTik
      ├── bridge  → 192.168.0.0/24   (LAN principal)
      ├── bridge2 → 192.168.1.0/24   (LAN secundaria)
      ├── bridge3 → 192.168.2.0/24   (LAN terciaria)
      └── wgAdelantos → 172.16.1.2/27
               │
               └──► Servidor remoto 143.202.209.230 (172.16.1.1)
               └──► Peer 1: 172.16.1.3/32 (Portátil Windows)
               └──► Peer N: 172.16.1.x/32 (nuevos dispositivos)
```

> **Nota:** El MikroTik actúa simultáneamente como cliente del servidor remoto y como receptor de peers entrantes en la misma interfaz `wgAdelantos`.

---

## Tabla de IPs del túnel asignadas

|IP del túnel|Dispositivo|
|---|---|
|`172.16.1.1/32`|Servidor remoto|
|`172.16.1.2/32`|MikroTik (este router)|
|`172.16.1.3/32`|Portátil Windows|
|`172.16.1.4/32`|Próximo dispositivo...|

---

## Paso 1 — Agregar el peer en el MikroTik

Desde la **terminal de WinBox**, ejecutar:

```routeros
/interface wireguard peers add \
  interface=wgAdelantos \
  allowed-address=172.16.1.X/32 \
  comment="Nombre-Dispositivo"
```

> Reemplazar `172.16.1.X` con la IP libre siguiente en la tabla de arriba y `Nombre-Dispositivo` con una descripción clara (ej. `Portatil-Gerencia`, `PC-Soporte`).

Verificar que el peer fue creado correctamente:

```routeros
/interface wireguard peers print detail
```

Anotar los siguientes valores del peer recién creado — son necesarios para el archivo de configuración del cliente:

|Campo|Dónde encontrarlo|
|---|---|
|`private-key`|Campo `private-key` del peer|
|`public-key`|Campo `public-key` del peer|

---

## Paso 2 — Verificar reglas de firewall

### 2.1 Permitir entrada UDP al puerto WireGuard

```routeros
/ip firewall filter add \
  chain=input \
  protocol=udp \
  dst-port=13232 \
  action=accept \
  comment="WireGuard - entrada UDP" \
  place-before=0
```

> Si ya existe esta regla de una configuración previa, omitir este paso. Verificar con:
> 
> ```routeros
> /ip firewall filter print where comment~"WireGuard"
> ```

### 2.2 Permitir forward del peer hacia la LAN

```routeros
/ip firewall filter add \
  chain=forward \
  src-address=172.16.1.X/32 \
  dst-address=192.168.0.0/24 \
  action=accept \
  comment="VPN Nombre-Dispositivo hacia LAN" \
  place-before=0
```

> Reemplazar `172.16.1.X` con la IP asignada al peer. Agregar reglas adicionales si el peer necesita acceso a `192.168.1.0/24` o `192.168.2.0/24`.

---

## Paso 3 — Crear el archivo de configuración del cliente

Crear un archivo de texto con extensión `.conf` y el siguiente contenido:

```ini
[Interface]
PrivateKey = <private-key-del-peer>
Address = 172.16.1.X/32
DNS = 192.168.0.1

[Peer]
PublicKey = ulQ.....n2E=
Endpoint = 181.94.197.29:13232
AllowedIPs = 192.168.0.0/24, 172.16.1.0/27
PersistentKeepalive = 25
```

### Descripción de cada campo

|Campo|Descripción|
|---|---|
|`PrivateKey`|Clave privada generada en el paso 1 (del peer, no del router)|
|`Address`|IP del túnel asignada a este peer|
|`DNS`|IP del router como resolver DNS interno|
|`PublicKey`|Clave pública del MikroTik (interfaz `wgAdelantos`)|
|`Endpoint`|IP pública del MikroTik y puerto WireGuard|
|`AllowedIPs`|Subredes que se enrutarán por el túnel|
|`PersistentKeepalive`|Mantiene el túnel activo desde detrás de NAT (recomendado: 25s)|

### AllowedIPs según el acceso requerido

|Acceso necesario|Valor de AllowedIPs|
|---|---|
|Solo LAN principal|`192.168.0.0/24`|
|LAN principal + túnel remoto|`192.168.0.0/24, 172.16.1.0/27`|
|Todas las LANs|`192.168.0.0/24, 192.168.1.0/24, 192.168.2.0/24, 172.16.1.0/27`|
|Todo el tráfico (full tunnel)|`0.0.0.0/0`|

---

## Paso 4 — Instalar y configurar el cliente en Windows

1. Descargar el cliente oficial desde **https://wireguard.com/install**
2. Instalar y abrir la aplicación
3. Clic en **"Import tunnel(s) from file"**
4. Seleccionar el archivo `.conf` creado en el paso 3
5. Clic en **"Activate"** para conectar

Una vez activado, el estado debe cambiar a **Active** y mostrar tráfico enviado y recibido.

---

## Paso 5 — Verificar la conexión desde el MikroTik

Confirmar que el peer está conectado y viendo tráfico:

```routeros
/interface wireguard peers print
```

El campo `last-handshake` debe mostrar un timestamp reciente (menos de 3 minutos). Si aparece vacío, el cliente aún no ha establecido el túnel.

Para verificar conectividad desde el router hacia el peer:

```routeros
/ping 172.16.1.X
```

---

## Troubleshooting rápido

|Síntoma|Causa probable|Acción|
|---|---|---|
|No hay handshake|Puerto UDP 13232 bloqueado|Verificar firewall input en el MikroTik|
|Handshake OK pero sin tráfico|Falta regla forward o NAT|Revisar paso 2|
|Llega al túnel pero no a la LAN|Falta ruta o regla forward|Agregar regla forward `172.16.1.X → 192.168.0.0/24`|
|DNS no resuelve|DNS incorrecto en `[Interface]`|Verificar que `192.168.0.1` responde DNS|
|Conflicto de IP en el túnel|IP de túnel duplicada|Asignar IP libre de la tabla del paso 1|

---

## Referencia rápida — Clave pública del MikroTik

Para obtener la clave pública del MikroTik en cualquier momento:

```routeros
/interface wireguard print
```

Campo `public-key` de la interfaz `wgAdelantos`.