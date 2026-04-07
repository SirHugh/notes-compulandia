# Alloy — Configuración

Alloy scrape los cuatro exporters por red interna Docker y envía al Prometheus de la nube via `remote_write` con headers de Cloudflare Access.

---

## alloy/config.alloy

> Todos los valores en el map `headers` deben terminar con coma, incluyendo el último. Omitirla causa error de sintaxis al iniciar.

```hcl
// ─── remote_write — Prometheus en la nube ────────────────────────────────────

prometheus.remote_write "cloud" {
  endpoint {
    url = "https://prometheus.compulandia.com.py/api/v1/write"

    headers = {
      "CF-Access-Client-Id"     = "CLOUDFLARE_CLIENT_ID",
      "CF-Access-Client-Secret" = "CLOUDFLARE_CLIENT_SECRET",
    }
  }
}

// ─── snmp_exporter ────────────────────────────────────────────────────────────

prometheus.scrape "snmp_mikrotik" {
  targets = [{
    __address__    = "snmp-exporter:9116",
    __param_target = "192.168.0.1",
    __param_module = "mikrotik,if_mib",
    __param_auth   = "compulandia_v2",
    instance       = "192.168.0.1",
  }]

  metrics_path    = "/snmp"
  scrape_interval = "60s"
  scrape_timeout  = "30s"
  job_name        = "snmp_mikrotik"

  forward_to = [prometheus.remote_write.cloud.receiver]
}

// ─── mktxp ───────────────────────────────────────────────────────────────────

prometheus.scrape "mktxp" {
  targets = [{
    __address__ = "mktxp:9436",
  }]

  metrics_path    = "/metrics"
  scrape_interval = "60s"
  scrape_timeout  = "30s"
  job_name        = "mktxp"

  forward_to = [prometheus.remote_write.cloud.receiver]
}

// ─── speedtest-exporter ───────────────────────────────────────────────────────
// scrape_interval extendido a 1h — el exporter corre el test completo
// en cada scrape (~60s). No reducir el interval para no saturar el enlace.

prometheus.scrape "speedtest" {
  targets = [{
    __address__ = "speedtest-exporter:9798",
  }]

  metrics_path    = "/metrics"
  scrape_interval = "1h"
  scrape_timeout  = "2m"
  job_name        = "speedtest"

  forward_to = [prometheus.remote_write.cloud.receiver]
}

// ─── smokeping_prober ─────────────────────────────────────────────────────────
// Histogramas de latencia ICMP — 30s es suficiente dado que el prober
// acumula pings cada 1s internamente.

prometheus.scrape "smokeping" {
  targets = [{
    __address__ = "smokeping:9374",
  }]

  metrics_path    = "/metrics"
  scrape_interval = "30s"
  scrape_timeout  = "10s"
  job_name        = "smokeping"

  forward_to = [prometheus.remote_write.cloud.receiver]
}
```

---

## Aplicar cambios

```bash
docker compose up -d --force-recreate alloy
docker logs alloy --tail 20
```

Los logs deben mostrar los cinco nodos evaluados sin errores:

```
finished node evaluation  node_id=prometheus.remote_write.cloud
finished node evaluation  node_id=prometheus.scrape.snmp_mikrotik
finished node evaluation  node_id=prometheus.scrape.mktxp
finished node evaluation  node_id=prometheus.scrape.speedtest
finished node evaluation  node_id=prometheus.scrape.smokeping
```

---

## Verificar que las métricas llegan a Prometheus

```bash
# mktxp
curl -sg "https://prometheus.compulandia.com.py/api/v1/query?query=mktxp_system_uptime" \
  -H "CF-Access-Client-Id: CLOUDFLARE_CLIENT_ID" \
  -H "CF-Access-Client-Secret: CLOUDFLARE_CLIENT_SECRET" \
  | python3 -m json.tool | head -10

# smokeping
curl -sg "https://prometheus.compulandia.com.py/api/v1/query?query=smokeping_requests_total" \
  -H "CF-Access-Client-Id: CLOUDFLARE_CLIENT_ID" \
  -H "CF-Access-Client-Secret: CLOUDFLARE_CLIENT_SECRET" \
  | python3 -m json.tool | head -10

# speedtest (solo disponible después del primer scrape ~1h)
curl -sg "https://prometheus.compulandia.com.py/api/v1/query?query=speedtest_download_bits_per_second" \
  -H "CF-Access-Client-Id: CLOUDFLARE_CLIENT_ID" \
  -H "CF-Access-Client-Secret: CLOUDFLARE_CLIENT_SECRET" \
  | python3 -m json.tool
```

UI de Alloy: `http://192.168.0.201:12345`

---

## Troubleshooting

|Síntoma|Causa probable|Solución|
|---|---|---|
|`missing ',' in field list`|Falta coma en map de headers|Agregar `,` al final de cada línea del map|
|`remote_write` 403|Headers CF incorrectos|Verificar Client-Id y Client-Secret|
|`snmp-exporter: no such host`|Red Docker incorrecta|Verificar `networks: monitoring` en Compose|
|Alloy no inicia|`config.alloy` no existía antes del `up`|`touch alloy/config.alloy` antes de levantar|
|mktxp conecta a `192.168.88.1`|Volumen mal montado|Verificar path `/root/mktxp` con `user: root`|
|mktxp `KeyError` al iniciar|`_mktxp.conf` incompleto|Usar template completo de [[02-compose]]|
|speedtest sin datos|Primer scrape ocurre a la hora|Esperar hasta el primer intervalo de 1h|
|smokeping sin histogramas|Contenedor sin `privileged: true`|Verificar flag en docker-compose.yml|

---

## Documentación relacionada

- [[00-arquitectura]] — Visión general y diagrama del stack
- [[02-compose]] — docker-compose.yml y archivos de configuración
- [[04-grafana-alertas]] — Dashboards, alertas y PromQL