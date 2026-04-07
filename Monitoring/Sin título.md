# Monitoreo de Red MikroTik con Prometheus y Grafana

**Infraestructura objetivo:** Compulandia — Stack observabilidad en Docker Compose  
**Redes:** `10.100.102.x` (on-premises) / `10.158.0.x` (GCP)  
**Versión del documento:** 1.0 — Marzo 2026

---

## Tabla de contenidos

1. [Arquitectura general](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#1-arquitectura-general)
2. [Requisitos previos](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#2-requisitos-previos)
3. [Opción A — snmp_exporter (SNMP)](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#3-opci%C3%B3n-a--snmp_exporter-snmp)
    - 3.1 [Habilitar SNMP en MikroTik](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#31-habilitar-snmp-en-mikrotik)
    - 3.2 [Despliegue con Docker Compose](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#32-despliegue-con-docker-compose)
    - 3.3 [Archivo snmp.yml (módulo MikroTik)](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#33-archivo-snmpyml-m%C3%B3dulo-mikrotik)
    - 3.4 [Configurar job en Prometheus](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#34-configurar-job-en-prometheus)
4. [Opción B — mktxp (RouterOS API)](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#4-opci%C3%B3n-b--mktxp-routeros-api)
    - 4.1 [Habilitar API en MikroTik](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#41-habilitar-api-en-mikrotik)
    - 4.2 [Crear usuario de monitoreo en RouterOS](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#42-crear-usuario-de-monitoreo-en-routeros)
    - 4.3 [Despliegue con Docker Compose](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#43-despliegue-con-docker-compose)
    - 4.4 [Configuración de mktxp](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#44-configuraci%C3%B3n-de-mktxp)
    - 4.5 [Configurar job en Prometheus](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#45-configurar-job-en-prometheus)
5. [Dashboard en Grafana](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#5-dashboard-en-grafana)
6. [Alertas recomendadas](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#6-alertas-recomendadas)
7. [Verificación y troubleshooting](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#7-verificaci%C3%B3n-y-troubleshooting)
8. [Métricas de referencia](https://claude.ai/chat/4ab0345a-b35c-4824-88a2-033ccb13cfde#8-m%C3%A9tricas-de-referencia)

---

## 1. Arquitectura general

```
┌─────────────────────────────────────────────┐
│            Red 10.100.102.x                 │
│                                             │
│  ┌─────────────┐     ┌──────────────────┐  │
│  │   MikroTik  │     │  Switches / APs  │  │
│  │   Router    │     │  (SNMP v2c)      │  │
│  │ SNMP + API  │     └────────┬─────────┘  │
│  └──────┬──────┘              │             │
└─────────┼─────────────────────┼─────────────┘
          │  SNMP UDP/161       │
          │  RouterOS API/8728  │
          ▼                     ▼
┌─────────────────────────────────────────────┐
│         VM Observabilidad (Docker)          │
│                                             │
│  ┌──────────────┐   ┌──────────────────┐   │
│  │ snmp_exporter│   │      mktxp       │   │
│  │  :9116       │   │     :9436        │   │
│  └──────┬───────┘   └────────┬─────────┘   │
│         │                    │              │
│         └─────────┬──────────┘              │
│                   ▼                         │
│           ┌──────────────┐                 │
│           │  Prometheus  │                 │
│           └──────┬───────┘                 │
│                  ▼                         │
│           ┌──────────────┐                 │
│           │   Grafana    │                 │
│           │   :3200      │                 │
│           └──────────────┘                 │
└─────────────────────────────────────────────┘
```

**Cuándo usar cada exporter:**

|Criterio|snmp_exporter|mktxp|
|---|---|---|
|También hay switches no-MikroTik|✅ Ideal|❌ Solo MikroTik|
|Detalle de colas QoS|Básico|✅ Completo|
|DHCP leases|❌|✅|
|BGP/OSPF peers|Básico|✅ Detallado|
|Configuración inicial|Media|Baja|
|Switches genéricos|✅|❌|

Ambos exporters pueden coexistir en el mismo `docker-compose.yml`.

---

## 2. Requisitos previos

### En la VM de observabilidad

- Docker Compose operativo con el stack actual (Prometheus, Grafana, Loki, Alertmanager)
- Acceso de red al router MikroTik desde la VM (`10.100.102.x`)
- Puerto UDP 161 accesible (SNMP) y/o TCP 8728 (RouterOS API)

### Verificar conectividad antes de continuar

```bash
# Verificar que el router responde SNMP (reemplazar IP y community)
snmpwalk -v2c -c public 10.100.102.1 sysDescr

# Verificar que el puerto API está accesible
nc -zv 10.100.102.1 8728
```

Si `snmpwalk` no está instalado en el host:

```bash
# Debian/Ubuntu
sudo apt install snmp snmp-mibs-downloader

# Probar desde contenedor temporal
docker run --rm --network host alpine/snmp \
  snmpwalk -v2c -c public 10.100.102.1 sysDescr
```

---

## 3. Opción A — snmp_exporter (SNMP)

### 3.1 Habilitar SNMP en MikroTik

Conectarse al router vía Winbox, SSH o WebFig y ejecutar:

```routeros
# Habilitar el servicio SNMP
/snmp set enabled=yes

# Configurar community (reemplazar "compulandia_ro" por el valor deseado)
/snmp community
set [ find default=yes ] name=public disabled=yes   # deshabilitar "public"
add name=compulandia_ro addresses=10.100.102.0/24 read-access=yes write-access=no

# Configurar información del sistema (opcional pero recomendado)
/snmp set contact="NOC Compulandia" location="Asuncion-DataCenter"

# Verificar
/snmp print
/snmp community print
```

> **Seguridad:** Restringir el campo `addresses` a la IP de la VM de observabilidad en lugar de toda la subred si es posible. Ej: `addresses=10.100.102.50/32`.

### 3.2 Despliegue con Docker Compose

Agregar el servicio `snmp-exporter` al `docker-compose.yml` existente:

```yaml
services:
  # ... servicios existentes (prometheus, grafana, loki, alertmanager) ...

  snmp-exporter:
    image: prom/snmp-exporter:v0.26.0
    container_name: snmp-exporter
    restart: unless-stopped
    ports:
      - "9116:9116"
    volumes:
      - ./snmp_exporter/snmp.yml:/etc/snmp_exporter/snmp.yml:ro
    command:
      - "--config.file=/etc/snmp_exporter/snmp.yml"
    networks:
      - monitoring

networks:
  monitoring:
    external: true
```

Crear el directorio de configuración:

```bash
mkdir -p ./snmp_exporter
```

### 3.3 Archivo snmp.yml (módulo MikroTik)

El proyecto `prometheus/snmp_exporter` provee módulos pre-generados. El módulo para MikroTik cubre las MIBs estándar IF-MIB, HOST-RESOURCES-MIB y las MIBs propietarias de RouterOS.

Crear `./snmp_exporter/snmp.yml` con el siguiente contenido mínimo funcional. Para producción se recomienda descargar el archivo completo del repositorio oficial.

```yaml
# ./snmp_exporter/snmp.yml
auths:
  public_v2:
    community: compulandia_ro   # debe coincidir con el community del router
    security_level: noAuthNoPriv
    version: 2

modules:
  # Módulo genérico para interfaces (aplica a cualquier dispositivo SNMP)
  if_mib:
    walk:
      - sysUpTime
      - interfaces
      - ifXTable
    get:
      - sysDescr
      - sysName
      - sysContact
      - sysLocation
    metrics:
      - name: sysUpTime
        oid: 1.3.6.1.2.1.1.3.0
        type: gauge
        help: Uptime del sistema en centésimas de segundo
      - name: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
        help: Estado operacional de la interfaz (1=up, 2=down)
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifInOctets
        oid: 1.3.6.1.2.1.2.2.1.10
        type: counter
        help: Bytes recibidos por interfaz
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifOutOctets
        oid: 1.3.6.1.2.1.2.2.1.16
        type: counter
        help: Bytes enviados por interfaz
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifInErrors
        oid: 1.3.6.1.2.1.2.2.1.14
        type: counter
        help: Errores de entrada por interfaz
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifOutErrors
        oid: 1.3.6.1.2.1.2.2.1.20
        type: counter
        help: Errores de salida por interfaz
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifInDiscards
        oid: 1.3.6.1.2.1.2.2.1.13
        type: counter
        help: Paquetes descartados en entrada
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifOutDiscards
        oid: 1.3.6.1.2.1.2.2.1.19
        type: counter
        help: Paquetes descartados en salida
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      # Contadores de 64 bits (ifXTable) — preferir sobre los de 32 bits
      - name: ifHCInOctets
        oid: 1.3.6.1.2.1.31.1.1.1.6
        type: counter
        help: Bytes recibidos (64-bit counter)
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifHCOutOctets
        oid: 1.3.6.1.2.1.31.1.1.1.10
        type: counter
        help: Bytes enviados (64-bit counter)
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      # CPU y Memoria via HOST-RESOURCES-MIB
      - name: hrProcessorLoad
        oid: 1.3.6.1.2.1.25.3.3.1.2
        type: gauge
        help: Uso de CPU por procesador (%)
        indexes:
          - labelname: hrDeviceIndex
            type: gauge
      - name: hrStorageUsed
        oid: 1.3.6.1.2.1.25.2.3.1.6
        type: gauge
        help: Almacenamiento usado (unidades definidas por hrStorageAllocationUnits)
        indexes:
          - labelname: hrStorageIndex
            type: gauge
        lookups:
          - labels: [hrStorageIndex]
            labelname: hrStorageDescr
            oid: 1.3.6.1.2.1.25.2.3.1.3
            type: DisplayString
      - name: hrStorageSize
        oid: 1.3.6.1.2.1.25.2.3.1.5
        type: gauge
        help: Tamaño total del almacenamiento
        indexes:
          - labelname: hrStorageIndex
            type: gauge
        lookups:
          - labels: [hrStorageIndex]
            labelname: hrStorageDescr
            oid: 1.3.6.1.2.1.25.2.3.1.3
            type: DisplayString

  # Módulo MikroTik — OIDs propietarios
  mikrotik:
    walk:
      - 1.3.6.1.4.1.14988.1   # MikroTik enterprise OID tree
    get:
      - sysDescr
      - sysName
    metrics:
      - name: mtxrInterfaceStatsRxBytes
        oid: 1.3.6.1.4.1.14988.1.1.14.1.1.31
        type: counter
        help: Bytes recibidos por interfaz (MikroTik)
        indexes:
          - labelname: mtxrInterfaceStatsIndex
            type: gauge
        lookups:
          - labels: [mtxrInterfaceStatsIndex]
            labelname: mtxrInterfaceStatsName
            oid: 1.3.6.1.4.1.14988.1.1.14.1.1.2
            type: DisplayString
      - name: mtxrInterfaceStatsTxBytes
        oid: 1.3.6.1.4.1.14988.1.1.14.1.1.32
        type: counter
        help: Bytes enviados por interfaz (MikroTik)
        indexes:
          - labelname: mtxrInterfaceStatsIndex
            type: gauge
        lookups:
          - labels: [mtxrInterfaceStatsIndex]
            labelname: mtxrInterfaceStatsName
            oid: 1.3.6.1.4.1.14988.1.1.14.1.1.2
            type: DisplayString
      - name: mtxrGaugeCpuFrequency
        oid: 1.3.6.1.4.1.14988.1.1.3.14.0
        type: gauge
        help: Frecuencia de CPU (MHz)
      - name: mtxrGaugeCpuLoad
        oid: 1.3.6.1.4.1.14988.1.1.3.1.0
        type: gauge
        help: Uso de CPU (%)
      - name: mtxrGaugeMemAvail
        oid: 1.3.6.1.4.1.14988.1.1.3.6.0
        type: gauge
        help: Memoria disponible (KB)
      - name: mtxrGaugeMemTotal
        oid: 1.3.6.1.4.1.14988.1.1.3.5.0
        type: gauge
        help: Memoria total (KB)
      - name: mtxrGaugeBoardTemperature
        oid: 1.3.6.1.4.1.14988.1.1.3.10.0
        type: gauge
        help: Temperatura de la placa (°C) — solo hardware físico
```

> **Nota:** Para obtener el `snmp.yml` completo y actualizado con todos los módulos MikroTik, ejecutar el generador oficial:
> 
> ```bash
> # Descargar el generador
> git clone https://github.com/prometheus/snmp_exporter
> cd snmp_exporter/generator
> 
> # Descargar MIBs de MikroTik
> make mibs
> 
> # Generar snmp.yml
> go run . generate
> ```

### 3.4 Configurar job en Prometheus

Agregar al archivo `prometheus.yml`:

```yaml
scrape_configs:
  # ... jobs existentes ...

  # Monitoreo SNMP — MikroTik Router
  - job_name: snmp_mikrotik
    scrape_interval: 60s
    scrape_timeout: 30s
    static_configs:
      - targets:
          - 10.100.102.1    # Router principal
          # - 10.100.102.2  # Segundo router (si aplica)
        labels:
          site: onprem
          device_type: router
    metrics_path: /snmp
    params:
      module: [mikrotik, if_mib]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116

  # Monitoreo SNMP — Switches genéricos (solo IF-MIB)
  - job_name: snmp_switches
    scrape_interval: 60s
    scrape_timeout: 30s
    static_configs:
      - targets:
          - 10.100.102.5    # Switch capa de acceso
          # agregar IPs adicionales según corresponda
        labels:
          site: onprem
          device_type: switch
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
```

Recargar Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload

# Verificar que el target aparece como UP
curl -s http://localhost:9090/api/v1/targets | \
  python3 -m json.tool | grep -A3 snmp_mikrotik
```

---

## 4. Opción B — mktxp (RouterOS API)

### 4.1 Habilitar API en MikroTik

```routeros
# Habilitar el servicio API (puerto 8728, sin TLS)
/ip service enable api

# Para entornos donde se requiera TLS (recomendado en producción)
/ip service enable api-ssl

# Verificar estado
/ip service print where name=api or name=api-ssl
```

> **Seguridad:** Restringir el acceso al puerto 8728 solo desde la IP de la VM de observabilidad con una regla de firewall:
> 
> ```routeros
> /ip firewall filter
> add chain=input protocol=tcp dst-port=8728 \
>     src-address=10.100.102.50 action=accept comment="API Prometheus" place-before=0
> ```

### 4.2 Crear usuario de monitoreo en RouterOS

```routeros
# Crear grupo con permisos de solo lectura
/user group add name=prometheus policy=read,api,!write,!policy,!test,!winbox,!password,!web,!sniff,!sensitive,!romon,!tikapp,!rest-api

# Crear usuario
/user add name=prometheus group=prometheus password=CAMBIAR_PASSWORD comment="Prometheus monitoring"

# Verificar
/user print where name=prometheus
```

### 4.3 Despliegue con Docker Compose

```yaml
services:
  # ... servicios existentes ...

  mktxp:
    image: ghcr.io/akpw/mktxp:latest
    container_name: mktxp
    restart: unless-stopped
    user: root
    ports:
      - "9436:9436"
    volumes:
      - ./mktxp:/home/mktxp/mktxp:rw
    networks:
      - monitoring
```

Crear estructura de directorios:

```bash
mkdir -p ./mktxp
```

### 4.4 Configuración de mktxp

Crear `./mktxp/mktxp.conf`:

```ini
# ./mktxp/mktxp.conf
# Configuración global del exporter
[MKTXP]
    # Puerto de exposición de métricas
    port = 9436
    # Intervalo de scrape (segundos)
    socket_timeout = 2

# ─── Definición de routers a monitorear ────────────────────────────────────────
# Cada sección [NombreDispositivo] define un target.
# El nombre debe ser único y descriptivo.

[Compulandia-MikroTik]
    enabled = True

    # Conexión
    hostname = 10.100.102.1
    port = 8728
    username = prometheus
    password = CAMBIAR_PASSWORD

    # TLS (activar si se usa api-ssl en puerto 8729)
    use_ssl = False
    no_ssl_certificate = False
    ssl_certificate_authority = ''

    # Módulos de métricas a recolectar
    # Activar/desactivar según los recursos del router
    dhcp = True
    dhcp_lease = True
    connections = True
    connection_stats = False   # costoso en CPU, desactivar si hay muchas conexiones
    pool = True
    interface = True
    firewall = True             # contadores de reglas de firewall
    neighbor = True             # vecinos CDP/LLDP
    ipv6_neighbor = False
    routes = True               # tabla de rutas
    queue = True                # simple queues y queue trees
    dns = False                 # cache DNS (verbose)
    bgp = False                 # activar si se usa BGP
    capsman = False             # activar si hay APs CAPsMAN
    wireless = False            # activar si hay interfaces wireless locales
    wireless_clients = False
    monitor = True              # estadísticas de interfaces en tiempo real
    poe = False                 # activar si el router tiene puertos PoE
    netwatch = True             # estado de hosts monitoreados por Netwatch
    public_ip = True            # IP pública detectada
    arp = False                 # tabla ARP (verbose)
    kid_control_devices = False
    user = True                 # sesiones de usuarios activos
    mktxp_fetch_routers_in_parallel = True

# Agregar más routers replicando el bloque anterior con diferente nombre
# [Compulandia-MikroTik-Sucursal]
#     enabled = True
#     hostname = 10.100.102.3
#     port = 8728
#     username = prometheus
#     password = CAMBIAR_PASSWORD
#     ...
```

### 4.5 Configurar job en Prometheus

```yaml
scrape_configs:
  # ... jobs existentes ...

  - job_name: mktxp
    scrape_interval: 60s
    scrape_timeout: 30s
    static_configs:
      - targets:
          - mktxp:9436
        labels:
          site: onprem
    metrics_path: /metrics
```

Recargar Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 5. Dashboard en Grafana

### Para snmp_exporter

Importar el dashboard de la comunidad Grafana:

- **Dashboard ID:** `11169` — SNMP Stats (genérico, funciona con IF-MIB)
- **Dashboard ID:** `14857` — MikroTik RouterOS SNMP

En Grafana → Dashboards → Import → ingresar el ID → seleccionar datasource Prometheus.

> Los dashboards de la comunidad frecuentemente tienen variables de datasource mal configuradas. Si no cargan métricas, ir a Settings → Variables y corregir los filtros (ver notas de infraestructura).

### Para mktxp

mktxp incluye dashboards oficiales que coinciden exactamente con las métricas generadas:

```bash
# Clonar el repositorio para obtener los JSON de dashboards
git clone https://github.com/akpw/mktxp.git /tmp/mktxp-repo

# Copiar los JSON al directorio de provisioning de Grafana
cp /tmp/mktxp-repo/extras/grafana/mktxp_*.json ./grafana/provisioning/dashboards/
```

Los dashboards incluidos son:

- `mktxp_main_dashboard.json` — vista general multi-router
- `mktxp_queue_dashboard.json` — análisis de QoS y colas
- `mktxp_wireless_dashboard.json` — clientes WiFi (si aplica)

### Provisioning como código (recomendado)

Agregar al archivo de provisioning de dashboards existente o crear `./grafana/provisioning/dashboards/network.yml`:

```yaml
apiVersion: 1

providers:
  - name: network-monitoring
    orgId: 1
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: true
```

---

## 6. Alertas recomendadas

Agregar al archivo de reglas de Alertmanager (por ejemplo `./prometheus/rules/network.yml`):

```yaml
groups:
  - name: network.rules
    interval: 60s
    rules:

      # ── Disponibilidad ──────────────────────────────────────────────────────
      - alert: InterfazDeRedCaida
        expr: ifOperStatus{job=~"snmp.*", ifName!~"lo.*|sit.*|tun.*"} == 2
        for: 2m
        labels:
          severity: critical
          team: infra
        annotations:
          summary: "Interfaz {{ $labels.ifName }} caída en {{ $labels.instance }}"
          description: >
            La interfaz {{ $labels.ifName }} del dispositivo {{ $labels.instance }}
            está en estado DOWN hace más de 2 minutos.

      # ── Tráfico / saturación ────────────────────────────────────────────────
      - alert: InterfazSaturada
        expr: >
          (rate(ifHCInOctets[5m]) * 8)
          / (ifHighSpeed * 1000000)
          > 0.85
        for: 5m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "Interfaz {{ $labels.ifName }} al {{ $value | humanizePercentage }}"
          description: >
            La interfaz {{ $labels.ifName }} en {{ $labels.instance }} supera
            el 85% de su capacidad nominal.

      # ── Errores de capa 2 ───────────────────────────────────────────────────
      - alert: ErroresDeRed
        expr: rate(ifInErrors[5m]) + rate(ifOutErrors[5m]) > 10
        for: 5m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "Errores en interfaz {{ $labels.ifName }} — {{ $labels.instance }}"
          description: >
            Tasa de errores: {{ $value | humanize }} errores/s en {{ $labels.ifName }}.

      # ── CPU del router (mktxp) ──────────────────────────────────────────────
      - alert: RouterCPUAlto
        expr: mktxp_cpu_load > 85
        for: 5m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "CPU alta en router {{ $labels.routerboard_name }}"
          description: "CPU al {{ $value }}% en {{ $labels.routerboard_name }}."

      - alert: RouterCPUCritico
        expr: mktxp_cpu_load > 95
        for: 2m
        labels:
          severity: critical
          team: infra
        annotations:
          summary: "CPU crítica en router {{ $labels.routerboard_name }}"
          description: "CPU al {{ $value }}% en {{ $labels.routerboard_name }}."

      # ── Memoria del router (mktxp) ──────────────────────────────────────────
      - alert: RouterMemoriaAlta
        expr: >
          (mktxp_memory_total_bytes - mktxp_memory_available_bytes)
          / mktxp_memory_total_bytes > 0.90
        for: 5m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "Memoria alta en router {{ $labels.routerboard_name }}"
          description: >
            Uso de memoria al {{ $value | humanizePercentage }}
            en {{ $labels.routerboard_name }}.

      # ── SNMP exporter ───────────────────────────────────────────────────────
      - alert: SNMPTargetInaccesible
        expr: up{job=~"snmp.*"} == 0
        for: 3m
        labels:
          severity: critical
          team: infra
        annotations:
          summary: "Target SNMP inaccesible: {{ $labels.instance }}"
          description: >
            El dispositivo {{ $labels.instance }} no responde SNMP
            hace más de 3 minutos.
```

Referenciar el archivo desde `prometheus.yml`:

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml
```

---

## 7. Verificación y troubleshooting

### Verificar que los exporters responden

```bash
# snmp_exporter — consultar métricas de un target específico
curl "http://localhost:9116/snmp?target=10.100.102.1&module=if_mib&auth=public_v2" | head -40

# mktxp — métricas directas
curl http://localhost:9436/metrics | head -40

# Ver logs de cada contenedor
docker logs snmp-exporter --tail 50
docker logs mktxp --tail 50
```

### Verificar en Prometheus

```bash
# Targets activos
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | \
  grep -E '"job"|"health"|"lastError"'

# Query de prueba — tráfico de interfaces
curl -sg 'http://localhost:9090/api/v1/query?query=ifHCInOctets' | \
  python3 -m json.tool | head -30
```

### Problemas comunes

|Síntoma|Causa probable|Solución|
|---|---|---|
|`snmp_exporter` no devuelve métricas|Community string incorrecto|Verificar `/snmp community print` en MikroTik|
|Timeout en scrape SNMP|Firewall bloqueando UDP 161|Abrir puerto en router y en el host|
|`mktxp` error de autenticación|Password o usuario incorrecto|Verificar `/user print` en RouterOS|
|Interfaces muestran solo índice numérico|Lookup OID no resuelve|Verificar que el módulo incluye la sección `lookups`|
|Métricas de CPU no aparecen|RouterOS no expone esa MIB|Usar mktxp en lugar de SNMP para CPU/RAM|
|`up=0` en Prometheus|Exporter caído o red|Revisar `docker ps` y logs del contenedor|

### Comandos de diagnóstico SNMP desde línea de comandos

```bash
# Listar todas las interfaces del router
snmpwalk -v2c -c compulandia_ro 10.100.102.1 ifDescr

# Ver uso de CPU via SNMP
snmpget -v2c -c compulandia_ro 10.100.102.1 1.3.6.1.4.1.14988.1.1.3.1.0

# Ver memoria disponible
snmpget -v2c -c compulandia_ro 10.100.102.1 1.3.6.1.4.1.14988.1.1.3.6.0

# Explorar el árbol MikroTik completo (verbose)
snmpwalk -v2c -c compulandia_ro 10.100.102.1 1.3.6.1.4.1.14988
```

---

## 8. Métricas de referencia

### Métricas snmp_exporter (IF-MIB)

|Métrica|Tipo|Descripción|
|---|---|---|
|`ifOperStatus`|gauge|Estado operacional (1=up, 2=down)|
|`ifHCInOctets`|counter|Bytes recibidos (64-bit)|
|`ifHCOutOctets`|counter|Bytes enviados (64-bit)|
|`ifInErrors`|counter|Errores de entrada|
|`ifOutErrors`|counter|Errores de salida|
|`ifInDiscards`|counter|Paquetes descartados en entrada|
|`ifOutDiscards`|counter|Paquetes descartados en salida|
|`hrProcessorLoad`|gauge|Carga CPU por procesador (%)|
|`hrStorageUsed`|gauge|Almacenamiento usado|

### Métricas MikroTik propietarias (SNMP)

|Métrica|OID|Descripción|
|---|---|---|
|`mtxrGaugeCpuLoad`|`14988.1.1.3.1.0`|CPU total (%)|
|`mtxrGaugeMemAvail`|`14988.1.1.3.6.0`|RAM disponible (KB)|
|`mtxrGaugeMemTotal`|`14988.1.1.3.5.0`|RAM total (KB)|
|`mtxrGaugeBoardTemperature`|`14988.1.1.3.10.0`|Temperatura (°C)|

### Métricas mktxp principales

|Métrica|Descripción|
|---|---|
|`mktxp_cpu_load`|CPU (%) — por router|
|`mktxp_memory_available_bytes`|RAM libre|
|`mktxp_memory_total_bytes`|RAM total|
|`mktxp_interface_rx_byte`|Bytes recibidos por interfaz|
|`mktxp_interface_tx_byte`|Bytes enviados por interfaz|
|`mktxp_interface_running`|Estado de interfaz (1=activa)|
|`mktxp_dhcp_lease_count`|Cantidad de leases DHCP activos|
|`mktxp_route_count`|Entradas en tabla de rutas|
|`mktxp_firewall_filter_bytes`|Bytes matcheados por regla firewall|
|`mktxp_queue_queued_bytes`|Bytes en cola (QoS)|
|`mktxp_uptime`|Uptime del router (segundos)|
|`mktxp_public_ip_status`|Estado de IP pública (1=alcanzable)|

### Queries PromQL útiles

```promql
# Tráfico de entrada en Mbps por interfaz (snmp_exporter)
rate(ifHCInOctets{job="snmp_mikrotik"}[5m]) * 8 / 1e6

# Porcentaje de uso de interfaz
rate(ifHCInOctets[5m]) * 8 / (ifHighSpeed * 1e6) * 100

# Memoria usada en el router (mktxp)
(mktxp_memory_total_bytes - mktxp_memory_available_bytes)
  / mktxp_memory_total_bytes * 100

# DHCP leases activos
mktxp_dhcp_lease_count{status="bound"}

# Top 5 interfaces por tráfico de salida
topk(5, rate(mktxp_interface_tx_byte[5m]))
```

---

_Documento generado para infraestructura Compulandia — Stack Prometheus/Grafana_  
_Referencia: SUSE Linux Enterprise for SAP Applications 15 SP6 — SAP Monitoring Guide_