# Grafana Agent — Guía de instalación y configuración

## Requisitos previos

- Ubuntu/Debian 20.04+
- Acceso sudo
- Conectividad al servidor Prometheus:
	- https://prometheus.compulandia.com.py
- Exporters corriendo en los servidores a monitorear (node_exporter `:9100`, mysql_exporter `:9104`)
- Puerto `9100` abierto en el firewall de cada servidor monitoreado hacia la IP del agente

---

## 1. Instalación

```bash
# Agregar repositorio de Grafana
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Instalar
sudo apt update
sudo apt install grafana-agent -y
```

Verificar la versión instalada:

```bash
grafana-agent --version
```

---

## 2. Directorios y permisos

```bash
# Crear directorio WAL con permisos correctos
sudo mkdir -p /var/lib/grafana-agent/wal
sudo chown -R grafana-agent:grafana-agent /var/lib/grafana-agent/
```

---

## 3. Configuración del servicio

El archivo `/etc/default/grafana-agent` controla los parámetros de arranque del servicio. Editar:

```bash
sudo nano /etc/default/grafana-agent
```

Contenido:

```bash
CONFIG_FILE="/etc/grafana-agent.yaml"
CUSTOM_ARGS="-server.http.address=0.0.0.0:12345 -server.grpc.address=127.0.0.1:9091"
RESTART_ON_UPGRADE=true
```

> El flag `-server.http.address=0.0.0.0:12345` es necesario para poder consultar el estado del agente desde otras máquinas. Por defecto escucha solo en `127.0.0.1`.

---

## 4. Archivo de configuración

Editar `/etc/grafana-agent.yaml`:

```bash
sudo nano /etc/grafana-agent.yaml
```

```yaml
server:
  log_level: info

metrics:
  wal_directory: /var/lib/grafana-agent/wal
  global:
    scrape_interval: 60s
    remote_write:
      - url: http://192.168.0.201:9090/api/v1/write

  configs:
    - name: red-nombre          # identificador de la red
      scrape_configs:

        # Node exporter
        - job_name: 'nombre-servidor-node'
          basic_auth:
            username: prometheus
            password: 'password'
          static_configs:
            - targets: ['IP_SERVIDOR:9100']
              labels:
                red: 'red-nombre'
                servidor: 'nombre-servidor'
                exporter: 'node'

        # MySQL exporter (si aplica)
        - job_name: 'nombre-servidor-mysql'
          basic_auth:
            username: prometheus
            password: 'password'
          static_configs:
            - targets: ['IP_SERVIDOR:9104']
              labels:
                red: 'red-nombre'
                servidor: 'nombre-servidor'
                exporter: 'mysql'
```

### Labels obligatorias por target

|Label|Descripción|Ejemplo|
|---|---|---|
|`red`|Identificador de la red|`red-aviadores`|
|`servidor`|Nombre del servidor|`dev`, `stg`, `prod`|
|`exporter`|Tipo de exporter|`node`, `mysql`|

---

## 5. Verificar la configuración antes de iniciar

```bash
sudo grafana-agent --config.file /etc/grafana-agent.yaml
```

Si no hay errores de YAML, detener con `Ctrl+C` e iniciar el servicio.

---

## 6. Iniciar y habilitar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-agent
sudo systemctl start grafana-agent
sudo systemctl status grafana-agent
```

Estado esperado: `active (running)`.

---

## 7. Verificación

**Logs del agente:**

```bash
sudo journalctl -u grafana-agent -f
```

No deben aparecer líneas con `level=error`. El warning `Expected to have read whole segment` al iniciar es normal y desaparece solo.

**Targets scrapeados por el agente:**

```bash
curl -s http://localhost:12345/agent/api/v1/metrics/targets | jq '.data[] | {job: .labels.job, health: .health, lastError: .lastError}'
```

**Verificar que las métricas llegaron a Prometheus:**

```bash
curl -s 'http://192.168.0.201:9090/api/v1/query?query=up{red="red-nombre"}' | jq '.data.result[] | {instance: .metric.instance, up: .value[1]}'
```

Valor `"1"` = scraping exitoso. Valor `"0"` = target caído o inaccesible.

---

## 8. Firewall en servidores monitoreados

Cada servidor que expone exporters debe permitir acceso desde la IP del agente:

```bash
# Permitir acceso al node_exporter desde el agente
sudo ufw allow from IP_AGENTE to any port 9100

# Permitir acceso al mysql_exporter desde el agente
sudo ufw allow from IP_AGENTE to any port 9104
```

---

## Troubleshooting

|Síntoma|Causa probable|Solución|
|---|---|---|
|Servicio en `failed` al iniciar|`CUSTOM_ARGS` inválido en `/etc/default/grafana-agent`|Revisar y corregir el archivo, luego `systemctl reset-failed`|
|`permission denied` en WAL|Ownership incorrecto|`chown -R grafana-agent:grafana-agent /var/lib/grafana-agent/`|
|`up=0` en Prometheus|Firewall bloqueando el puerto|Abrir puerto en UFW desde la IP del agente|
|`connection refused` al remote write|Prometheus no tiene `--web.enable-remote-write-receiver`|Agregar el flag en `docker-compose.yml` y reiniciar|
|No aparece en `/targets` de Prometheus|Comportamiento normal con remote write|Los datos llegan igual — consultar via PromQL|