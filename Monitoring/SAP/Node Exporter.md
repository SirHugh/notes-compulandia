# Fase 1 — node_exporter en VM SAP (SUSE Linux)

**Fecha:** 17 Marzo 2026 · **Sistema:** SLES 15 SP4 · `sap-linux` · `10.100.102.10`

---

## Contexto

Instalación de `node_exporter` en la VM de SAP HANA para exponer métricas de sistema operativo (CPU, RAM, disco, red) hacia el stack Prometheus existente.

---

## Reconocimiento previo

```bash
cat /etc/os-release | grep -E "NAME|VERSION"
curl -v --max-time 5 https://github.com       # acceso internet
zypper repos                                  # repositorios disponibles
which node_exporter || echo "no instalado"    # instalación previa
uname -m                                      # arquitectura
```

### Resultados clave

|Elemento|Valor|
|---|---|
|Sistema operativo|SLES 15 SP4 · x86_64|
|Acceso internet (HTTPS)|❌ Puerto 443 bloqueado|
|Ping externo|✅ Funciona|
|DNS|✅ Resuelve (`10.0.0.5`)|
|node_exporter previo|No instalado|

> **Puerto 443 bloqueado:** `scc.suse.com` inaccesible. Los repos SUSE no pueden refrescarse desde internet, pero los repos tipo `Pool` tienen su índice cacheado localmente.

---

## Instalación

El paquete está disponible en los repos Pool locales de SUSE Package Hub. El flag `--no-refresh` evita intentar contactar `scc.suse.com`.

```bash
zypper --no-refresh install golang-github-prometheus-node_exporter
```

> **`golang-github-prometheus-node_exporter`** es el nombre RPM de SUSE para el `node_exporter` oficial de Prometheus. Mismo binario, empaquetado por SUSE. Instala automáticamente binario, servicio systemd y usuario de sistema `prometheus`.

### Archivos instalados

|Ruta|Descripción|
|---|---|
|`/usr/bin/node_exporter`|Binario principal|
|`/usr/lib/systemd/system/prometheus-node_exporter.service`|Unidad systemd|
|`/etc/sysconfig/prometheus-node_exporter`|Configuración|

---

## Activación del servicio

```bash
systemctl start prometheus-node_exporter
systemctl enable prometheus-node_exporter
systemctl status prometheus-node_exporter
```

### Salida status

```
Active: active (running) since Tue 2026-03-17 14:13:26 -03
Listening on [::]:9100
TLS is disabled.
```

---

## Verificación local

```bash
curl -s http://localhost:9100/metrics | head -5
```

Responde con métricas en formato Prometheus — servicio operativo.

---

## Firewall

```bash
systemctl status firewalld   # inactive (dead)
iptables -L INPUT             # policy ACCEPT — sin restricciones
```

Sin firewall interno activo. Puerto `9100` accesible desde la red sin configuración adicional.

---

## Configuración en Grafana Agent

El agente Grafana en la red SAP (`/etc/grafana-agent.yaml`) ya tenía el target configurado con la IP de esta VM:

```yaml
- name: red_sap
  scrape_configs:
    - job_name: 'sap_linux'
      static_configs:
        - targets: ['10.100.102.10:9100']
          labels:
            red: 'red_sap'
            servidor: 'sap_linux'
            exporter: 'node'
```

Recarga de configuración:

```bash
systemctl reload grafana-agent
```

---

## Resultado

Métricas de `sap-linux` visibles en Prometheus. Target `sap_linux` activo.

```
✅ node_exporter instalado     (zypper --no-refresh)
✅ Servicio activo y habilitado (systemctl enable)
✅ Métricas en :9100           (curl localhost:9100/metrics)
✅ Firewall                    (sin restricciones)
✅ Grafana Agent               (systemctl reload)
✅ Métricas en Prometheus      (target UP)
```

---

## Siguiente paso

**Fase 2 — `hanadb_exporter`:** métricas de SAP HANA Database (memoria, backups, conexiones, tenants).