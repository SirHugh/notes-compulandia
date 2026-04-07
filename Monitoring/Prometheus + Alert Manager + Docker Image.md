## Instalación

**Servidor**: 192.168.0.201
**Directorio**: /opt/monitoring/prometheus
### Comfiguracion docker-compose.yml
``` yaml
services:
	prometheus:
	    image: prom/prometheus:latest
	    container_name: prometheus
	    restart: unless-stopped
	    ports:
	      - "9090:9090"
	    networks:
	      - monitoring
	    volumes:
	      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
	      - ./rules:/etc/prometheus/rules:ro
	      - ./data:/prometheus
	    command:
	      - --config.file=/etc/prometheus/prometheus.yml
	      - --storage.tsdb.path=/prometheus
	      - --storage.tsdb.retention.time=30d
	      - --web.enable-lifecycle

    alertmanager: 
	    image: prom/alertmanager:latest 
	    container_name: alertmanager 
	    volumes:
		    - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml 
	    ports: - "9093:9093" 
	    restart: unless-stopped
	    networks:
	      - monitoring	    
networks:
  monitoring:
    name: monitoring
```
### Permisos de datos al host
> $ sudo chown -R "$UID:$GID" /opt/monitoring/prometheus/data
> $ sudo chmod -R u+rwX,g+rwX /opt/monitoring/prometheus/data

## Configuracion de prometheus.yml

``` yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Apuntar a Alertmanager 
alerting: 
	alertmanagers: 
		- static_configs: 
		  - targets: 
		    - 'localhost:9093'

scrape_configs:
  # Prometheus se monitorea a sí mismo
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  # Ejemplo: node_exporter local o remoto(s)
  - job_name: node
    static_configs:
      - targets:
        - 'localhost:9100' # servidor local
        - '192.168.0.250:9100' # servidor staging
        - '192.168.0.220:9100' # servidor de desarrollo
        # - '10.0.0.12:9100'
rule_files:
  - /etc/prometheus/rules/*.yml
```
## Levantar el servicio
> $ docker compose up -d
> $ docker compose logs -f prometheus








