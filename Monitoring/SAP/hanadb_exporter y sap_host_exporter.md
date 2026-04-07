# Fase 3 y 4 — hanadb_exporter y sap_host_exporter

**Fecha:** 18 Marzo 2026 · **Sistema:** SLES 15 SP4 · `sap-linux` · `10.100.102.10`

---

## Fase 3 — hanadb_exporter ⏸️ EN PAUSA

### Contexto

Exporter para métricas internas de SAP HANA: memoria, backups, conexiones y tenants. Requiere un usuario de base de datos con rol `MONITORING`.

### Instalación completada

El paquete no está en repos locales ni disponible via RPM para SLES 15 SP4. Se instaló desde código fuente siguiendo la documentación de SUSE (Sección 4.3):

```bash
# Prerequisitos
zypper --no-refresh install git
pip3 install virtualenv

# Clonar e instalar
cd /opt
git clone https://github.com/SUSE/hanadb_exporter
cd hanadb_exporter
virtualenv virt
source virt/bin/activate
pip install pyhdb
pip install shaptools        # dependencia faltante — instalada desde GitHub
pip install .
```

> **Nota:** `shaptools` no estaba en PyPI. Se instaló con: `pip install git+https://github.com/SUSE/shaptools.git`

### Archivos de configuración

```bash
mkdir -p /etc/hanadb_exporter
cp /opt/hanadb_exporter/config.json.example /etc/hanadb_exporter/config.json
cp /opt/hanadb_exporter/metrics.json /etc/hanadb_exporter/metrics.json
```

### Configuración pendiente (`/etc/hanadb_exporter/config.json`)

```json
{
  "listen_address": "0.0.0.0",
  "exposition_port": 9668,
  "multi_tenant": true,
  "timeout": 30,
  "hana": {
    "host": "localhost",
    "port": 30013,
    "user": "HANADB_EXPORTER_USER",
    "password": "<PASSWORD>",
    "ssl": false,
    "ssl_validate_cert": false
  }
}
```

### ⏸️ Punto de pausa — Pendiente administrador SAP

Crear usuario de monitoreo en **SYSTEMDB y en cada tenant** (SID: `NDB`):

```bash
su - ndbadm
hdbsql -u HANADB_EXPORTER_USER -p <password> -d NDB
```

```sql
CREATE USER HANADB_EXPORTER_USER PASSWORD <password> NO FORCE_FIRST_PASSWORD_CHANGE;
CREATE ROLE HANADB_EXPORTER_ROLE;
GRANT MONITORING TO HANADB_EXPORTER_ROLE;
GRANT HANADB_EXPORTER_ROLE TO HANADB_EXPORTER_USER;
```

> Repetir los mismos comandos conectando a cada tenant (`-d <TENANT>`).

### Activación (ejecutar cuando el usuario esté creado)

```bash
source /opt/hanadb_exporter/virt/bin/activate
systemctl start prometheus-hanadb_exporter@config
systemctl enable prometheus-hanadb_exporter@config
```

Agregar al Grafana Agent (`/etc/grafana-agent.yaml`):

```yaml
- job_name: 'hanadb'
  scrape_interval: 60s
  static_configs:
    - targets: ['10.100.102.10:9668']
      labels:
        red: 'red_sap'
        servidor: 'sap_linux'
        exporter: 'hanadb'
```

---

## Fase 4 — sap_host_exporter ✅

### Contexto

Exporter para estado de la instancia SAP: procesos, Enqueue Server y Dispatcher. Se conecta via SAPControl — no requiere credenciales de base de datos.

### Sistema identificado

|Elemento|Valor|
|---|---|
|SID|`NDB`|
|Instancia|`HDB00`|
|Número|`0`|
|Usuario admin|`ndbadm`|
|Socket SAPControl|`/tmp/.sapstream50013`|
|SAP HANA versión|`2.00.059` (SPS05)|

### Instalación

Paquete no disponible en repos locales ni en Open Build Service para SLES 15 SP4. Se compiló desde código fuente en un servidor Linux externo con internet:

```bash
# En servidor externo con internet (Go 1.23 requerido)
git clone https://github.com/SUSE/sap_host_exporter
cd sap_host_exporter

# Instalar Go 1.23 (el sistema tenía go1.15 — insuficiente)
curl -LO https://go.dev/dl/go1.23.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.23.0.linux-amd64.tar.gz
export PATH=/usr/local/go/bin:$PATH

# Instalar mockgen (requerido por make)
go install github.com/golang/mock/mockgen@v1.6.0
export PATH=$PATH:$(go env GOPATH)/bin

# Compilar
make
# Resultado: build/bin/sap_host_exporter-amd64
```

Transferir e instalar en la VM SAP:

```bash
# Desde servidor externo
scp build/bin/sap_host_exporter-amd64 usuario@10.100.102.10:/tmp/

# En VM SAP
mv /tmp/sap_host_exporter-amd64 /usr/local/bin/sap_host_exporter
chmod +x /usr/local/bin/sap_host_exporter
```

### Servicio systemd

```bash
cat > /etc/systemd/system/prometheus-sap_host_exporter.service << 'EOF'
[Unit]
Description=Prometheus exporter for SAP systems
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/sap_host_exporter --sap-control-uds /tmp/.sapstream50013
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start prometheus-sap_host_exporter
systemctl enable prometheus-sap_host_exporter
```

### Grafana Agent

```yaml
- job_name: 'sap_host_exporter'
  static_configs:
    - targets: ['10.100.102.10:9680']
      labels:
        red: 'red_sap'
        servidor: 'sap_linux'
        exporter: 'sap_host'
```

```bash
systemctl reload grafana-agent
```

### Verificación

```promql
sap_start_service_processes{servidor="sap_linux"}
```

**Resultado:** 14 procesos HANA reportando `status="Running"`:

|Proceso|PID|Estado|
|---|---|---|
|hdbdaemon|3316|Running|
|hdbnameserver|3342|Running|
|hdbcompileserver|3737|Running|
|hdbpreprocessor|3740|Running|
|hdbindexserver|3798|Running|
|hdbindexserver|3801|Running|
|hdbscriptserver|3804|Running|
|hdbxsengine|3807|Running|
|hdbdiserver|8248|Running|
|hdbdiserver|8251|Running|
|hdbwebdispatcher|8254|Running|
|hdbxscontroller|8258|Running|
|hdbxsexecagent|8261|Running|
|hdbxsuaaserver|8264|Running|

---

## Estado general

```
✅ FASE 1 — node_exporter        10.100.102.10:9100
✅ FASE 2 — windows_exporter     10.100.102.20:9182
⏸️  FASE 3 — hanadb_exporter     10.100.102.10:9668  (usuario HANA pendiente)
✅ FASE 4 — sap_host_exporter    10.100.102.10:9680
⬜ FASE 5 — blackbox_exporter
⬜ FASE 6 — Alertas
```