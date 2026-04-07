# Documentación de Infraestructura SAP y Plan de Monitoreo

**Versión:** 1.0 · **Estado:** Borrador  
**Stack:** Prometheus · Grafana · Alertmanager

---

## 1. Descripción del Sistema

### 1.1 Topología General

```
┌─────────────────────────────────────────────────────────────┐
│  SERVIDOR FÍSICO  (Dual-CPU · Bare Metal · Certificado SAP) │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  SUSE Linux Enterprise Server (Host Base)            │  │
│  │  SLES for SAP 15 SP4+ · SAP Host Agent               │  │
│  │                                                      │  │
│  │  ┌─────────────────────┐  ┌───────────────────────┐  │  │
│  │  │  Sistema 1 (Linux)  │  │  Sistema 2 (Windows)  │  │  │
│  │  │                     │  │                       │  │  │
│  │  │  SUSE Linux Guest   │  │  Windows Server       │  │  │
│  │  │  ├─ SAP HANA DB     │  │  ├─ SAP GUI           │  │  │
│  │  │  │   └─ SYSTEMDB    │  │  ├─ SAP HANA Studio   │  │  │
│  │  │  │   └─ Tenant(s)   │  │  ├─ SAP Business Cl.  │  │  │
│  │  │  └─ SAP Service     │  │  └─ RDP (acceso       │  │  │
│  │  │       Layer (SL)    │  │       remoto)         │  │  │
│  │  └─────────────────────┘  └───────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Componentes y Puertos

| Componente               | Sistema   | Puerto        | Rol                         |
| ------------------------ | --------- | ------------- | --------------------------- |
| SAP HANA Index Server    | Sistema 1 | `30015`       | Base de datos principal     |
| SAP HANA SYSTEMDB        | Sistema 1 | `30013`       | Administración multi-tenant |
| SAP HANA Statistics Srv. | Sistema 1 | `30060`       | Performance interno         |
| SAP XS Engine            | Sistema 1 | `8000`        | HTTP embebido               |
| SAP Service Layer        | Sistema 1 | `40000`       | API / integración B1        |
| SAP Host Agent (SAPCtrl) | Host SUSE | `1128/1129`   | Control instancias          |
| SAP GUI / RDP            | Sistema 2 | `3389 / 32xx` | Interfaz gráfica remota     |
| SAP HANA Studio          | Sistema 2 | `4300`        | Administración y dev        |

### 1.3 Layout de Sistema de Archivos (Sistema 1)

|Ruta|Propósito|Dimensionamiento|
|---|---|---|
|`/hana/data/<SID>`|Volúmenes de datos persistentes|≥ tamaño de RAM|
|`/hana/log/<SID>`|Redo logs / transaction logs|≥ 0.5× RAM|
|`/hana/shared`|Binarios y configuración compartida|~1 TB|
|`/usr/sap/<SID>`|Instancia SAP HANA|~50 GB|
|`/hanabackup`|Backups locales|≥ tamaño `/hana/data`|
|`/usr/sap/SAPService`|SAP Service Layer|~20 GB|

---

## 2. Estrategia de Monitoreo

El monitoreo se organiza en **6 capas** cubriendo desde el hardware hasta la conectividad de red.

```
Capa 6 ── Red y Conectividad          ← blackbox_exporter
Capa 5 ── Windows Server (RDP/GUI)    ← windows_exporter
Capa 4 ── SAP Service Layer + App     ← sap_host_exporter · sapnwrfc_exporter
Capa 3 ── SAP HANA Database           ← hanadb_exporter · hana_sql_exporter
Capa 2 ── SUSE Linux (Sistema 1+Host) ← node_exporter
Capa 1 ── Hardware Físico             ← node_exporter (hwmon) · IPMI
```

### 2.1 Exportadores Requeridos

| Exportador          | Puerto | Sistema Objetivo         | Capa |
| ------------------- | ------ | ------------------------ | ---- |
| `node_exporter`     | `9100` | Host SUSE + Sistema 1    | 1, 2 |
| `hanadb_exporter`   | `9968` | Sistema 1 — SAP HANA     | 3    |
| `hana_sql_exporter` | `9658` | Sistema 1 — SAP HANA     | 3    |
| `sap_host_exporter` | `9680` | Sistema 1 — SAP App      | 4    |
| `sapnwrfc_exporter` | `9663` | Sistema 1 — SAP RFC      | 4    |
| `windows_exporter`  | `9182` | Sistema 2 — Windows      | 5    |
| `blackbox_exporter` | `9115` | Red / endpoints HTTP/TCP | 6    |

### 2.2 Métricas Críticas por Capa

**Capa 1-2 — Hardware y OS Linux**

|Métrica|Umbral Crítico|Umbral Warning|
|---|---|---|
|CPU total del servidor|> 90% por 5 min|> 75% por 10 min|
|RAM disponible|< 10%|< 20%|
|`/hana/data` uso de disco|> 85%|> 75%|
|`/hana/log` uso de disco|> 80%|> 70%|
|Temperatura CPU|> 80 °C|> 70 °C|
|Servicios SAP (systemd)|Estado `failed`|—|

**Capa 3 — SAP HANA Database**

|Métrica|Umbral Crítico|Umbral Warning|
|---|---|---|
|Memoria HANA usada|> 90% asignada|> 80%|
|Estado último backup|`!= successful`|—|
|Antigüedad de backup|> 24 horas|> 12 horas|
|Jobs ABAP cancelados|—|> 5 en 1 hora|
|Lag replicación HSR|> 60 s|> 30 s|

**Capa 4 — SAP Service Layer**

|Métrica|Umbral Crítico|Umbral Warning|
|---|---|---|
|Work processes dialog|`= 0`|< 2 disponibles|
|Instancia SAP estado|`!= GREEN`|—|
|Service Layer HTTP|`!= 200` en `:40000`|Latencia > 3 s|
|Lock table entries|—|> 500 entradas|

**Capa 5 — Windows Server**

|Métrica|Umbral Crítico|Umbral Warning|
|---|---|---|
|Puerto RDP `:3389`|No responde|—|
|CPU Windows|> 90% por 5 min|> 80%|
|Disco C:\|> 90%|> 80%|
|Servicio `sapgui.exe`|Proceso ausente|—|

---

## 3. Plan de Implementación

### Fase 1 — Infraestructura Base Linux (Semana 1) - [[Node Exporter]] 

> Objetivo: cobertura completa de hardware, OS y sistema de archivos SAP.

- [ ] Instalar `node_exporter` en **Host SUSE** y en **Sistema 1** (SUSE guest)
- [ ] Configurar scrape en `prometheus.yml` — jobs `sap-host-os` y `sap-vm1-os`
- [ ] Verificar métricas de disco montadas: `/hana/data`, `/hana/log`, `/hanabackup`
- [ ] Importar dashboard Grafana: **[[Node Exporter]] Full** (ID `1860`)
- [ ] Reglas de alerta: CPU, RAM, disco, temperatura

```yaml
# prometheus.yml — Fase 1
- job_name: 'sap-host-linux'
  static_configs:
    - targets: ['<IP_HOST>:9100']
      labels: { host: 'sap-host', role: 'hypervisor' }

- job_name: 'sap-vm1-linux'
  static_configs:
    - targets: ['<IP_VM1>:9100']
      labels: { host: 'sap-vm1', role: 'hana' }
```

---

### Fase 2 — SAP HANA Database (Semana 2)

> Objetivo: visibilidad de memoria, backups, conexiones y tenants.

- [ ] Instalar y configurar `hanadb_exporter` en Sistema 1
    - Crear usuario de solo lectura en HANA con rol `MONITORING`
    - Configurar `config.json` con conexión a SYSTEMDB
- [ ] Instalar `hana_sql_exporter` para métricas custom (backups, jobs)
- [ ] Agregar jobs a `prometheus.yml` — `hanadb` y `hana-sql`
- [ ] Importar dashboard SUSE para SAP HANA (Grafana Community)
- [ ] Reglas de alerta: backup fallido, memoria crítica, jobs cancelados

```bash
# Usuario mínimo para hanadb_exporter
CREATE USER prometheus_mon PASSWORD '<pwd>' NO FORCE_FIRST_PASSWORD_CHANGE;
GRANT MONITORING TO prometheus_mon;
```

```yaml
# prometheus.yml — Fase 2
- job_name: 'hanadb'
  scrape_interval: 60s
  static_configs:
    - targets: ['<IP_VM1>:9968']
      labels: { sid: '<SID>', db: 'hana' }
```

---

### Fase 3 — SAP Application Layer (Semana 3)

> Objetivo: monitoreo de procesos SAP, Service Layer y health checks HTTP.

- [ ] Instalar `sap_host_exporter` en Sistema 1
    - Conectar vía SAPControl socket: `/tmp/.sapstream50013`
- [ ] Instalar `sapnwrfc_exporter` (requiere SAP NWRFC SDK 7.50 PL3+)
    - Crear usuario SAP con acceso de lectura a módulos RFC
- [ ] Configurar `blackbox_exporter` para:
    - Health check HTTP: `http://<IP_VM1>:40000` (Service Layer)
    - Health check TCP: `<IP_VM1>:30015` (HANA Index Server)
- [ ] Importar dashboard: SAP NetWeaver / S/4HANA (SUSE Community)
- [ ] Reglas de alerta: instancia DOWN, work processes a cero

```yaml
# prometheus.yml — Fase 3
- job_name: 'sap-host-exporter'
  static_configs:
    - targets: ['<IP_VM1>:9680']
      labels: { sid: '<SID>', component: 'netweaver' }

- job_name: 'blackbox-sap'
  metrics_path: /probe
  params: { module: [http_2xx] }
  static_configs:
    - targets:
        - 'http://<IP_VM1>:40000'   # Service Layer
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: '<IP_PROMETHEUS>:9115'
```

---

### Fase 4 — Windows Server (Semana 4)

> Objetivo: monitoreo de interfaz gráfica remota y salud del sistema Windows.

- [ ] Instalar `windows_exporter` en Sistema 2 (Windows Server)
    - Habilitar collectors: `cpu,memory,logical_disk,service,net,process`
- [ ] Abrir puerto `9182` en firewall Windows
- [ ] Configurar blackbox TCP para verificar RDP `:3389`
- [ ] Agregar job `sap-windows` en `prometheus.yml`
- [ ] Dashboard Grafana: Windows Exporter (ID `14694`)
- [ ] Reglas: RDP caído, disco lleno, CPU sostenido

```yaml
# prometheus.yml — Fase 4
- job_name: 'sap-windows'
  static_configs:
    - targets: ['<IP_VM2>:9182']
      labels: { host: 'sap-vm2', role: 'gui-rdp', os: 'windows' }

- job_name: 'blackbox-rdp'
  metrics_path: /probe
  params: { module: [tcp_connect] }
  static_configs:
    - targets: ['<IP_VM2>:3389']
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: '<IP_PROMETHEUS>:9115'
```

---

### Fase 5 — Alertas y Notificaciones (Semana 5)

> Objetivo: ruleset completo con notificaciones a Slack y email.

- [ ] Crear archivo `rules/sap-critical.yml` con alertas de Fase 1–4
- [ ] Crear archivo `rules/sap-warning.yml` con umbrales preventivos
- [ ] Configurar rutas en Alertmanager: críticas → canal `#sap-alerts`, warnings → `#sap-monitoring`
- [ ] Establecer inhibiciones: no alertar warning si ya existe crítico del mismo host
- [ ] Prueba de alertas con `amtool alert add`

**Alertas mínimas requeridas al finalizar esta fase:**

|Regla|Severidad|
|---|---|
|`SAP_HANA_Down`|critical|
|`SAP_ServiceLayer_Down`|critical|
|`HANA_Backup_Failed`|critical|
|`HANA_Data_Disk_Full` (< 15% libre)|critical|
|`RDP_Port_Down`|critical|
|`HANA_Memory_High` (> 85%)|warning|
|`Host_CPU_Sustained` (> 90% / 5m)|warning|
|`SSL_Cert_Expiring` (< 30 días)|warning|

---

### Fase 6 — Logs con Loki (Semana 6+)

> Objetivo: centralización de logs SAP HANA, Service Layer y sistema.

- [ ] Desplegar **Grafana Loki** en el stack Docker Compose existente
- [ ] Instalar **Promtail** en Sistema 1 apuntando a:
    - `/usr/sap/<SID>/HDB*/trace/*.trc` — Trazas SAP HANA
    - `/var/log/messages` — Sistema operativo SUSE
    - `/usr/sap/SAPService/logs/` — Service Layer
- [ ] Instalar **Promtail** en Sistema 2 (Windows) con Loki Windows Plugin
- [ ] Crear dashboard Grafana con correlación métricas + logs

```yaml
# docker-compose.yml — agregar a stack existente
loki:
  image: grafana/loki:latest
  ports: ["3100:3100"]
  volumes:
    - ./loki/config.yml:/etc/loki/local-config.yaml
    - loki_data:/loki

promtail:
  image: grafana/promtail:latest
  volumes:
    - /usr/sap/<SID>:/sap-logs:ro
    - /var/log:/var/log:ro
    - ./promtail/config.yml:/etc/promtail/config.yml
```

---

## 4. Configuración de Red — Puertos a Habilitar

### En el Host SUSE / Firewall

```bash
# Exporters (acceso desde Prometheus)
ufw allow from <IP_PROMETHEUS> to any port 9100   # node_exporter
ufw allow from <IP_PROMETHEUS> to any port 9968   # hanadb_exporter
ufw allow from <IP_PROMETHEUS> to any port 9658   # hana_sql_exporter
ufw allow from <IP_PROMETHEUS> to any port 9680   # sap_host_exporter
ufw allow from <IP_PROMETHEUS> to any port 9663   # sapnwrfc_exporter
```

### En Windows Server (firewall)

```powershell
# windows_exporter
New-NetFirewallRule -DisplayName "Prometheus windows_exporter" `
  -Direction Inbound -Protocol TCP -LocalPort 9182 `
  -RemoteAddress <IP_PROMETHEUS> -Action Allow
```

---

## 5. Resumen del Roadmap

|Semana|Fase|Cobertura al Finalizar|
|---|---|---|
|1|Hardware + OS Linux|CPU, RAM, disco, temperatura, servicios systemd|
|2|SAP HANA DB|Memoria, backups, conexiones, tenants|
|3|SAP App + Service Layer|Work processes, instancia, health checks HTTP|
|4|Windows Server|RDP, GUI, disco, CPU Windows|
|5|Alertas y notificaciones|Ruleset completo, Slack/email operativo|
|6+|Logs con Loki|Trazas SAP HANA, logs sistema, correlación|

---

_Basado en: documentación oficial SAP HANA, SUSE SAP Monitoring Guide, Prometheus exporters SUSE/Community_