# MikroTik — Plan de Segmentación de Red por Bridges

**Fecha:** 2026-03-27  
**Equipo:** MikroTik (RouterOS)  
**Objetivo:** Dividir la red LAN en subredes independientes para distribuir el pool DHCP y evitar el agotamiento de IPs.

---

## Contexto y problema resuelto

El pool DHCP original (`dhcp_pool`) de 200 IPs se agotaba por la combinación de:

- **Lease-time de 12 horas** → dispositivos desconectados retenían su IP por demasiado tiempo
- **Todos los puertos LAN en un solo bridge** → un único pool absorbía todos los dispositivos simultáneamente

**Solución:** Crear bridges adicionales con subredes y pools DHCP independientes.

---

## Mapa de infraestructura

|Interfaz|Rol|Red|
|---|---|---|
|`ether8`|WAN — Internet (Personal)|—|
|`bridge`|LAN principal|192.168.0.0/24|
|`bridge2`|LAN secundaria|192.168.1.0/24|
|`bridge3`|LAN terciaria|192.168.2.0/24|
|`wgAdelantos`|VPN WireGuard|—|

### Distribución de puertos

```
bridge  → 192.168.0.1/24
├── ether1
├── ether2
├── ether3
└── ether4  ← servidores con IP estática (.220 y .250) — NO mover

bridge2 → 192.168.1.1/24
├── ether5
└── ether6

bridge3 → 192.168.2.1/24
├── ether7
└── sfp-sfpplus1
```

> ⚠️ **ether4 no se toca.** Contiene los servidores `192.168.0.220` (SSH) y `192.168.0.250` (HTTP/HTTPS) con port-forwards activos.

---

## Servidores con IP estática y port-forwards

| IP            | MAC               | Servicio   | Regla NAT                |
| ------------- | ----------------- | ---------- | ------------------------ |
| 192.168.0.220 | BC:24:11:5E:46:B3 | SSH        | Puerto externo 777 → :22 |
| 192.168.0.250 | BC:24:11:F8:9B:60 | HTTP/HTTPS | Puerto 80 y 443          |

Ambos conectados por **ether4** → permanecen en `bridge` (192.168.0.x).

---

## Etapa 1 — Configurar bridge2 (192.168.1.x)

### Paso 1 — Asignar IP al router en bridge2

```bash
/ip address add address=192.168.1.1/24 interface=bridge2
```

### Paso 2 — Crear pool de IPs

```bash
/ip pool add name=dhcp_pool2 ranges=192.168.1.2-192.168.1.254
```

### Paso 3 — Crear red DHCP

```bash
/ip dhcp-server network add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=8.8.8.8,8.8.4.4
```

### Paso 4 — Crear servidor DHCP

```bash
/ip dhcp-server add name=dhcp2 interface=bridge2 address-pool=dhcp_pool2 lease-time=1h disabled=no
```

### Paso 5 — Mover puertos al bridge2 ⚡ (interrupción breve ~10-30 seg)

```bash
/interface bridge port remove [find interface=ether5]
/interface bridge port remove [find interface=ether6]
/interface bridge port add interface=ether5 bridge=bridge2
/interface bridge port add interface=ether6 bridge=bridge2
```

### Paso 6 — Agregar bridge2 a la lista LAN para NAT

```bash
/interface list member add list=LAN interface=bridge2
```

---

## Etapa 2 — Configurar bridge3 (192.168.2.x)

### Paso 1 — Asignar IP al router en bridge3

```bash
/ip address add address=192.168.2.1/24 interface=bridge3
```

### Paso 2 — Crear pool de IPs

```bash
/ip pool add name=dhcp_pool3 ranges=192.168.2.2-192.168.2.254
```

### Paso 3 — Crear red DHCP

```bash
/ip dhcp-server network add address=192.168.2.0/24 gateway=192.168.2.1 dns-server=8.8.8.8,8.8.4.4
```

### Paso 4 — Crear servidor DHCP

```bash
/ip dhcp-server add name=dhcp3 interface=bridge3 address-pool=dhcp_pool3 lease-time=1h disabled=no
```

### Paso 5 — Mover puertos al bridge3 ⚡ (interrupción breve ~10-30 seg)

```bash
/interface bridge port remove [find interface=ether7]
/interface bridge port remove [find interface=sfp-sfpplus1]
/interface bridge port add interface=ether7 bridge=bridge3
/interface bridge port add interface=sfp-sfpplus1 bridge=bridge3
```

### Paso 6 — Agregar bridge3 a la lista LAN para NAT

```bash
/interface list member add list=LAN interface=bridge3
```

---

## Ajuste en bridge original (192.168.0.x)

Reducir lease-time para evitar que el pool se vuelva a agotar:

```bash
/ip dhcp-server set [find name=defconf] lease-time=1h
```

Limpiar leases vencidos si los hubiera:

```bash
/ip dhcp-server lease remove [find where status=expired]
/ip dhcp-server lease remove [find where status=declined]
/ip dhcp-server lease remove [find where status=waiting]
```

---

## Comandos de auditoría y verificación

### Verificar que los 3 servidores DHCP están activos

```bash
/ip dhcp-server print
```

Confirmar que `defconf`, `dhcp2` y `dhcp3` aparecen sin la flag `X` (disabled).

### Ver leases activos por servidor

```bash
/ip dhcp-server lease print where server=dhcp2
/ip dhcp-server lease print where server=dhcp3
```

### Verificar pools y uso

```bash
/ip pool print
/ip pool used print
```

### Verificar IPs asignadas al router en cada bridge

```bash
/ip address print
```

Debe aparecer `192.168.0.1`, `192.168.1.1` y `192.168.2.1`.

### Verificar distribución de puertos en bridges

```bash
/interface bridge port print
```

### Verificar que bridge2 y bridge3 están en la lista LAN

```bash
/interface list member print where list=LAN
```

### Ver log DHCP en tiempo real (errores)

```bash
/log print follow where topics~"dhcp"
```

### Ver tabla ARP por interfaz (detectar anomalías)

```bash
/ip arp print where interface=bridge2
/ip arp print where interface=bridge3
```

### Verificar conectividad desde el router hacia cada subred

```bash
/ping 192.168.1.1
/ping 192.168.2.1
```

---

## Errores comunes y solución

|Error en log|Causa|Solución|
|---|---|---|
|`pool <x> is empty`|Pool agotado|Limpiar leases vencidos o ampliar pool|
|`failed to give out IP`|Sin servidor DHCP en la interfaz|Verificar que el servidor está activo y asignado al bridge correcto|
|Dispositivo no recibe IP|Puerto no movido al nuevo bridge|Verificar con `bridge port print`|
|Sin internet en nueva subred|bridge2/bridge3 no está en lista LAN|Agregar con `interface list member add`|
|Port-forward roto|Servidor movido de ether4|No mover ether4 del bridge original|

---

## Resumen de capacidad post-segmentación

|Red|Puertos|IPs disponibles|Lease-time|
|---|---|---|---|
|192.168.0.x (bridge)|ether1-4|253|1h|
|192.168.1.x (bridge2)|ether5-6|253|1h|
|192.168.2.x (bridge3)|ether7, sfp|253|1h|
|**Total**||**759 IPs**||