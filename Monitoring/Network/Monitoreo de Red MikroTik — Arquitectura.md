**Stack:** Alloy + snmp_exporter + mktxp — despliegue nuevo  
**Red:** `10.100.102.x` (servidor on-prem con acceso al MikroTik)

---

## Flujo

```
MikroTik Router (10.100.102.1)
  │  SNMP UDP/161
  │  RouterOS API TCP/8728
  ▼
┌─────────────────────────────────────┐
│   Servidor on-prem — Docker Compose │
│                                     │
│  snmp-exporter:9116 ─┐              │
│                      ├─► Alloy ─────┼──► remote_write
│  mktxp:9436        ──┘              │    CF Access headers
│                                     │         │
│  (red interna Docker)               │         ▼
└─────────────────────────────────────┘  prometheus.compulandia.com.py
                                                 │
                                                 ▼
                                           Grafana :3200
```

- Los exporters **no exponen puertos al host** — Alloy los alcanza por red interna Docker.
- Prometheus (nube) **no requiere cambios** — solo recibe via remote_write.
- Headers de Cloudflare Access: los mismos Service Token que usa Alloy en `sap-linux`.

---

## Archivos de esta documentación

|Archivo|Contenido|
|---|---|
|`01-mikrotik-routeros.md`|SNMP y API en el router|
|`02-compose.md`|docker-compose.yml + configs de exporters|
|`03-alloy.md`|Configuración de Alloy (scrape + remote_write)|
|`04-grafana-alertas.md`|Dashboards, alertas y PromQL|