# Stack de Observabilidad — Documentación

## Descripción general

Este repositorio centraliza la configuración del stack de observabilidad basado en imágenes Docker stock. **No se construyen imágenes personalizadas.** Todo cambio se gestiona a través de archivos de configuración versionados en Git.

El modelo de recolección de métricas se basa en **Grafana Agent** instalado de forma nativa en un servidor por red. Cada agente scrapea los exporters locales de su red y envía las métricas a Prometheus centralizado vía **remote write**.

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SERVIDOR CENTRAL                             │
│                       192.168.0.201                                 │
│                                                                     │
│   ┌─────────────┐    ┌───────────────┐    ┌────────────────────┐   │
│   │  Prometheus │    │ Alertmanager  │    │      Grafana       │   │
│   │   :9090     │───►│    :9093      │    │      :3200         │   │
│   └──────┬──────┘    └───────────────┘    └────────────────────┘   │
│          │ remote_write receiver                                    │
└──────────┼──────────────────────────────────────────────────────────┘
           │ HTTP POST /api/v1/write
           │
    ┌──────┴────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
┌───────────────────────────┐           ┌───────────────────────────┐
│         RED-A             │           │         RED-B             │
│                           │           │                           │
│  ┌─────────────────────┐  │           │  ┌─────────────────────┐  │
│  │   Grafana Agent     │  │           │  │   Grafana Agent     │  │
│  │   (nativo)          │  │           │  │   (nativo)          │  │
│  └──────────┬──────────┘  │           │  └──────────┬──────────┘  │
│             │ scrape       │           │             │ scrape       │
│    ┌────────┴────────┐    │           │    ┌────────┴────────┐    │
│    ▼                 ▼    │           │    ▼                 ▼    │
│ :9100             :9104   │           │ :9100             :9104   │
│ node_exporter  mysql_exp  │           │ node_exporter  mysql_exp  │
│                           │           │                           │
│  srv1  srv2  srv3  ...    │           │  srv1  srv2  srv3  ...    │
└───────────────────────────┘           └───────────────────────────┘

                           ┌───────────────────────────┐
                           │         RED-C             │
                           │                           │
                           │  ┌─────────────────────┐  │
                           │  │   Grafana Agent     │  │
                           │  │   (nativo)          │  │
                           │  └──────────┬──────────┘  │
                           │             │ scrape       │
                           │    ┌────────┴────────┐    │
                           │    ▼                 ▼    │
                           │ :9100             :9104   │
                           │ node_exporter  mysql_exp  │
                           │                           │
                           │  srv1  srv2  srv3  ...    │
                           └───────────────────────────┘
```

---

## Modelo de comunicación

### Scraping tradicional vs Remote Write

| Scraping directo                        | Remote Write (modelo elegido) |                  |
| --------------------------------------- | ----------------------------- | ---------------- |
| Dirección                               | Prometheus hace pull          | Agente hace push |
| Aparece en `/targets`                   | ✅ Sí                          | ❌ No             |
| Aparece en métricas                     | ✅ Sí                          | ✅ Sí             |
| Requiere acceso de red desde Prometheus | ✅ Sí                          | ❌ No             |
| Agente requerido                        | ❌ No                          | ✅ Sí             |

### Por qué remote write

Los servidores a monitorear están en redes distintas. Con scraping directo, Prometheus necesitaría alcanzar cada exporter de cada red. Con remote write, **solo el agente necesita alcanzar a Prometheus** — invirtiendo la dirección del flujo y simplificando la conectividad de red.

### Flujo de datos

```
Exporter (:9100 / :9104)
        │
        │  pull (basic_auth)
        ▼
  Grafana Agent
  (instalado nativo en un servidor de la red)
        │
        │  POST /api/v1/write (HTTP)
        ▼
  Prometheus :9090
  (--web.enable-remote-write-receiver)
        │
        ▼
  Grafana :3200
  (visualización y dashboards)
```

### Labels por red

Cada agente etiqueta sus métricas para identificar el origen:

```yaml
labels:
  red: 'red-a'          # identifica la red
  servidor: 'srv1'      # identifica el host
  exporter: 'node'      # identifica el tipo de exporter
```

Esto permite filtrar en Grafana por red, servidor o tipo de exporter de forma independiente.

---

## Estructura del repositorio

```
monitoring/
├── docker-compose.yml           # Stack unificado
├── prometheus.yml               # Configuración de scraping directo + remote write receiver
├── rules/
│   └── alerts.yml               # Reglas de alertas
├── alertmanager/
│   └── alertmanager.yml         # Rutas y receptores de alertas
├── grafana/
│   └── provisioning/
│       ├── datasources/         # Datasources automáticos
│       └── dashboards/          # Dashboards como código
└── .github/
    └── workflows/
        └── deploy.yml           # CI/CD pipeline
```

---

## Stack de servicios (servidor central)

|Servicio|Imagen|Puerto|Propósito|
|---|---|---|---|
|Prometheus|`prom/prometheus:latest`|9090|Almacenamiento y query de métricas|
|Alertmanager|`prom/alertmanager:latest`|9093|Gestión de alertas|
|Grafana|`grafana/grafana:latest`|3200|Visualización|

Prometheus debe tener el flag `--web.enable-remote-write-receiver` activo para recibir métricas de los agentes.

### Agregar un nuevo servicio (ej. Loki)

```yaml
loki:
  image: grafana/loki:latest
  profiles: ["loki"]
  ports:
    - "3100:3100"
  networks:
    - monitoring
```

```bash
docker compose --profile loki up -d
```

---

## Grafana Agent (por red)

Instalado como servicio nativo (`systemd`) en un servidor por red. Configuración en `/etc/grafana-agent.yaml`:

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
    - name: red-a
      scrape_configs:
        - job_name: 'a-srv1-node'
          basic_auth:
            username: prometheus
            password: 'password'
          static_configs:
            - targets: ['192.168.x.x:9100']
              labels:
                red: 'red-a'
                servidor: 'srv1'
                exporter: 'node'
```

El agente **no corre en Docker** — corre nativo para tener acceso directo al filesystem del host si se habilita el node_exporter integrado.

---

## Despliegue automático (CI/CD)

El pipeline se activa en cada push a `main`. Ejecuta las siguientes acciones vía SSH:

1. `git pull` — sincroniza los archivos de configuración en el servidor
2. Hot-reload de Prometheus y Alertmanager sin reiniciar contenedores
3. `docker compose up -d --remove-orphans` — levanta servicios nuevos si los hay

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy al servidor
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/monitoring
            git pull
            curl -s -X POST http://localhost:9090/-/reload
            curl -s -X POST http://localhost:9093/-/reload
            docker compose up -d --remove-orphans
```

### Secrets requeridos en GitHub

|Secret|Descripción|
|---|---|
|`SERVER_HOST`|IP o hostname del servidor central|
|`SERVER_USER`|Usuario SSH|
|`SSH_KEY`|Llave privada SSH|

---

## Hot-reload vs reinicio

|Servicio|Soporta hot-reload|Endpoint|
|---|---|---|
|Prometheus|✅ Sí|`POST /prometheus/-/reload`|
|Alertmanager|✅ Sí|`POST /alertmanager/-/reload`|
|Grafana|❌ No*|Requiere `docker compose restart grafana`|

> *Grafana aplica cambios de provisioning solo al iniciar.

---

## Flujo de trabajo recomendado

```
Cambio de config
      │
      ▼
 Editar archivo localmente
      │
      ▼
 git commit + git push → main
      │
      ▼
 GitHub Actions dispara el pipeline
      │
      ▼
 SSH → git pull → hot-reload / docker compose up
      │
      ▼
 Cambio aplicado sin intervención manual
```

---

## Variables de entorno sensibles

Las credenciales no deben versionarse. Usar un archivo `.env` en el servidor (no en el repo):

```env
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=tu_password_aqui
```

Referenciar en `docker-compose.yml`:

```yaml
env_file:
  - .env
```

Agregar `.env` al `.gitignore`.