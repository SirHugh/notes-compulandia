# Docker Compose — Todos los servicios

---

## Estructura de archivos

```
monitoring-network/
├── docker-compose.yml
├── alloy/
│   └── config.alloy
├── snmp_exporter/
│   └── snmp.yml
├── mktxp/
│   ├── _mktxp.conf
│   └── mktxp.conf
└── smokeping/
    └── config.yml
```

```bash
mkdir -p monitoring-network/{alloy,snmp_exporter,mktxp,smokeping}
cd monitoring-network

# Crear config.alloy vacío antes del primer up
# (si no existe, Docker lo crea como directorio y falla el mount)
touch alloy/config.alloy
```

---

## docker-compose.yml

```yaml
services:

  snmp-exporter:
    image: prom/snmp-exporter:v0.26.0
    container_name: snmp-exporter
    restart: unless-stopped
    volumes:
      - ./snmp_exporter/snmp.yml:/etc/snmp_exporter/snmp.yml:ro
    command:
      - "--config.file=/etc/snmp_exporter/snmp.yml"
    networks:
      - monitoring

  mktxp:
    image: ghcr.io/akpw/mktxp:latest
    container_name: mktxp
    restart: unless-stopped
    user: root
    volumes:
      - ./mktxp:/root/mktxp:rw    # con user: root la config se lee desde /root/mktxp
    networks:
      - monitoring

  speedtest-exporter:
    image: miguelndecarvalho/speedtest-exporter:latest
    container_name: speedtest-exporter
    restart: unless-stopped
    networks:
      - monitoring

  smokeping:
    image: quay.io/superq/smokeping-prober:latest
    container_name: smokeping
    restart: unless-stopped
    privileged: true              # requerido para enviar paquetes ICMP raw
    environment:
      - GOMAXPROCS=1
    volumes:
      - ./smokeping/config.yml:/etc/smokeping/config.yml:ro
    command:
      - --config.file=/etc/smokeping/config.yml
      - --web.listen-address=0.0.0.0:9374
    networks:
      - monitoring

  alloy:
    image: grafana/alloy:v1.14.1
    container_name: alloy
    restart: unless-stopped
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
    command:
      - run
      - /etc/alloy/config.alloy
      - --server.http.listen-addr=0.0.0.0:12345
    ports:
      - "12345:12345"
    networks:
      - monitoring
    depends_on:
      - snmp-exporter
      - mktxp
      - speedtest-exporter
      - smokeping

networks:
  monitoring:
    name: monitoring-network
    driver: bridge
```

> Solo `alloy` y `smokeping` exponen puertos — Alloy para su UI, smokeping por el flag `privileged` que requiere modo especial. El resto comunica solo por red interna.

---

## snmp_exporter/snmp.yml

> Usar OIDs numéricos en `walk`. El contenedor no resuelve nombres simbólicos.

```yaml
auths:
  compulandia_v2:
    community: compulandia_ro
    security_level: noAuthNoPriv
    version: 2

modules:

  if_mib:
    walk:
      - 1.3.6.1.2.1.1.3        # sysUpTime
      - 1.3.6.1.2.1.2          # interfaces
      - 1.3.6.1.2.1.31.1.1     # ifXTable
      - 1.3.6.1.2.1.25.3.3.1.2 # hrProcessorLoad
      - 1.3.6.1.2.1.25.2.3     # hrStorage
    get:
      - 1.3.6.1.2.1.1.1.0      # sysDescr
      - 1.3.6.1.2.1.1.5.0      # sysName
    metrics:
      - name: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
        help: Estado operacional (1=up, 2=down)
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      - name: ifHCInOctets
        oid: 1.3.6.1.2.1.31.1.1.1.6
        type: counter
        help: Bytes recibidos (64-bit)
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
        help: Bytes enviados (64-bit)
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
        help: Errores de entrada
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
        help: Errores de salida
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
      - name: hrProcessorLoad
        oid: 1.3.6.1.2.1.25.3.3.1.2
        type: gauge
        help: Carga de CPU (%)
        indexes:
          - labelname: hrDeviceIndex
            type: gauge
      - name: hrStorageUsed
        oid: 1.3.6.1.2.1.25.2.3.1.6
        type: gauge
        help: Almacenamiento usado
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
        help: Almacenamiento total
        indexes:
          - labelname: hrStorageIndex
            type: gauge
        lookups:
          - labels: [hrStorageIndex]
            labelname: hrStorageDescr
            oid: 1.3.6.1.2.1.25.2.3.1.3
            type: DisplayString

  mikrotik:
    walk:
      - 1.3.6.1.4.1.14988.1
    get:
      - 1.3.6.1.2.1.1.5.0
    metrics:
      - name: mtxrGaugeCpuLoad
        oid: 1.3.6.1.4.1.14988.1.1.3.1.0
        type: gauge
        help: CPU total (%)
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
        help: Temperatura (°C)
      - name: mtxrInterfaceStatsRxBytes
        oid: 1.3.6.1.4.1.14988.1.1.14.1.1.31
        type: counter
        help: Bytes recibidos por interfaz
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
        help: Bytes enviados por interfaz
        indexes:
          - labelname: mtxrInterfaceStatsIndex
            type: gauge
        lookups:
          - labels: [mtxrInterfaceStatsIndex]
            labelname: mtxrInterfaceStatsName
            oid: 1.3.6.1.4.1.14988.1.1.14.1.1.2
            type: DisplayString
```

---

## mktxp/_mktxp.conf

> Debe incluir **todos los campos**. Campos faltantes causan `KeyError` al iniciar. Campo crítico: `compact_default_conf_values`.

```ini
[MKTXP]
    listen = '0.0.0.0:9436'
    socket_timeout = 5
    initial_delay_on_failure = 120
    max_delay_on_failure = 900
    delay_inc_div = 5
    bandwidth = False
    bandwidth_test_dns_server = 8.8.8.8
    bandwidth_test_interval = 600
    minimal_collect_interval = 5
    verbose_mode = False
    fetch_routers_in_parallel = False
    max_worker_threads = 5
    max_scrape_duration = 30
    total_max_scrape_duration = 90
    http_server_threads = 16
    persistent_router_connection_pool = True
    persistent_dhcp_cache = True
    compact_default_conf_values = False
    prometheus_headers_deduplication = False
    probe_connection_pool = False
    probe_connection_pool_ttl = 300
    probe_connection_pool_max_size = 128
```

---

## mktxp/mktxp.conf

> La sección `[default]` define valores base para todos los routers. Cada sección de router solo necesita sobreescribir lo que difiere.

```ini
[Compulandia-MikroTik]
    hostname = 192.168.0.1
    port = 8728
    username = prometheus
    password = CAMBIAR_PASSWORD
    use_ssl = False
    no_ssl_certificate = False
    plaintext_login = True

[default]
    enabled = True
    module_only = False
    hostname = localhost
    port = 8728
    username = username
    password = password
    credentials_file = ""
    custom_labels = None
    use_ssl = False
    no_ssl_certificate = False
    ssl_certificate_verify = False
    ssl_check_hostname = True
    ssl_ca_file = ""
    plaintext_login = True
    health = True
    installed_packages = False
    dhcp = True
    dhcp_lease = True
    connections = False
    connection_stats = False
    interface = True
    route = True
    pool = True
    firewall = True
    neighbor = True
    address_list = None
    dns = False
    ipv6_route = False
    ipv6_pool = False
    ipv6_firewall = False
    ipv6_neighbor = False
    ipv6_address_list = None
    poe = False
    monitor = True
    netwatch = True
    public_ip = True
    wireless = False
    wireless_clients = False
    capsman = False
    capsman_clients = False
    w60g = False
    eoip = False
    gre = False
    ipip = False
    lte = False
    ipsec = False
    switch_port = False
    kid_control_assigned = False
    kid_control_dynamic = False
    user = True
    queue = True
    bfd = False
    bgp = False
    routing_stats = False
    certificate = False
    container = False
    remote_dhcp_entry = None
    remote_capsman_entry = None
    interface_name_format = name
    check_for_updates = False
```

---

## smokeping/config.yml

> Agregar o quitar hosts según necesidad. El prober envía 1 ping por segundo a cada host de forma continua.

```yaml
targets:
  - hosts:
      - 192.168.0.1        # Gateway MikroTik
    interval: 1s
    network: ip4
    protocol: icmp
    size: 56

  - hosts:
      - 8.8.8.8            # Google DNS
      - 1.1.1.1            # Cloudflare DNS
    interval: 1s
    network: ip4
    protocol: icmp
    size: 56
```

---

## Levantar el stack

```bash
docker compose up -d

# Verificar los cinco contenedores
docker compose ps

# snmp_exporter
curl -s "http://localhost:9116/snmp?target=192.168.0.1&module=if_mib&auth=compulandia_v2" | head -5

# mktxp (esperar ~10s)
curl -s http://localhost:9436/metrics | grep mktxp_system_uptime

# speedtest (tarda ~60s en completar el test)
curl -s http://localhost:9798/metrics | grep speedtest_download

# smokeping
curl -s http://localhost:9374/metrics | grep smokeping_requests_total
```

> Para recrear un contenedor específico tras cambiar su config:
> 
> ```bash
> docker compose up -d --force-recreate <nombre-servicio>
> ```

---

## Documentación relacionada

- [[00-arquitectura]] — Visión general y diagrama del stack
- [[01-mikrotik-routeros]] — Configuración previa en el router
- [[03-alloy]] — Configuración de Alloy (scrape + remote_write)
- [[04-grafana-alertas]] — Dashboards, alertas y PromQL