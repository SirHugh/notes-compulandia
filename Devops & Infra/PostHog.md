# PostHog — Despliegue Self-Hosted con Docker

> **Requisitos previos:** Docker Engine + Docker Compose Plugin instalados · Linux Debian/Ubuntu · `cloudflared` configurado  
> **Stack:** Docker Compose · Cloudflare Tunnel (TLS gestionado por Cloudflare Edge)

---

## Requisitos del servidor

|Recurso|Mínimo|
|---|---|
|vCPU|4|
|RAM|16 GB|
|Disco|30 GB+|

Los 10 contenedores del stack (PostHog web, worker, plugins, PostgreSQL, Redis, ClickHouse, Kafka, ZooKeeper, MinIO, Caddy) demandan recursos desde el primer arranque. **ClickHouse es el servicio más intensivo** — operar por debajo del mínimo de RAM produce OOM kills bajo carga normal.

> El contenedor **Caddy** forma parte del stack por defecto pero no cumple ninguna función en este despliegue — el TLS y el enrutamiento los gestiona Cloudflare Tunnel. Ver sección [Cloudflare Tunnel](https://claude.ai/chat/c54e2650-b058-4813-bc6a-a60e721cc3fc#cloudflare-tunnel).

---

## Antes de instalar

### 1. Cloudflare Tunnel configurado

El dominio de PostHog se publica exclusivamente a través del túnel. El servidor no expone ningún puerto hacia internet — la conexión es **saliente** desde `cloudflared` hacia el edge de Cloudflare.

Verificar que el tunnel esté activo antes de proceder:

```bash
cloudflared tunnel list
cloudflared tunnel info <nombre-del-tunnel>
```

### 2. No se requiere abrir puertos en el firewall

A diferencia de un despliegue directo, con Cloudflare Tunnel **no** es necesario abrir los puertos 80 ni 443. El stack de PostHog no necesita ser accesible desde internet de forma directa.

```bash
# El servidor solo necesita acceso SSH
sudo ufw status
# Suficiente con: 22/tcp ALLOW
```

---

## Instalación

```bash
mkdir -p /opt/posthog && cd /opt/posthog

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/posthog/posthog/HEAD/bin/deploy-hobby)"
```

El script solicita dos datos:

|Campo|Descripción|Ejemplo|
|---|---|---|
|Release tag|Tag de DockerHub a usar|`latest` o un tag fijo (recomendado para producción)|
|Domain|FQDN del dominio configurado en el túnel|`posthog.tudominio.com`|

El proceso tarda **5–10 minutos** mientras corren las migraciones de base de datos. Al finalizar, PostHog estará disponible internamente en el puerto `8000` — el acceso externo se habilita en el paso siguiente (configuración del túnel).

---

## Configuración — archivo `.env`

Ubicado en el directorio de trabajo (`/opt/posthog/.env`). Variables críticas:

```dotenv
# Generado automáticamente — NO modificar después del primer arranque
SECRET_KEY=<generado>

DOMAIN=posthog.tudominio.com

# Requerido con Cloudflare Tunnel — PostHog recibe HTTP plano desde el túnel
# y necesita respetar los headers X-Forwarded-For / X-Forwarded-Proto
IS_BEHIND_PROXY=true
DISABLE_SECURE_SSL_REDIRECT=true

# Email — necesario para invitaciones de usuarios
EMAIL_HOST=smtp.tudominio.com
EMAIL_PORT=587
EMAIL_HOST_USER=noreply@tudominio.com
EMAIL_HOST_PASSWORD=<password>
EMAIL_USE_TLS=true

# Deshabilitar signup público después de crear el usuario admin
DISABLE_SIGNUP=true
```

> ⚠️ **`SECRET_KEY` no debe cambiar nunca.** Hacerlo invalida todas las sesiones y puede romper datos cifrados existentes. Guardar una copia offline antes de cualquier otra acción.

Para aplicar cambios en `.env`:

```bash
cd /opt/posthog
docker compose down && docker compose up -d
```

---

## Cloudflare Tunnel

El tráfico fluye así: el usuario accede por HTTPS al edge de Cloudflare, que lo reenvía por el túnel hasta el contenedor `posthog-web` en HTTP plano (puerto `8000`). El servidor nunca recibe conexiones entrantes directas.

```
Usuario → HTTPS → Cloudflare Edge → cloudflared (túnel saliente) → http://localhost:8000
```

### Configurar el túnel para apuntar a PostHog

En Cloudflare Dashboard → Zero Trust → Networks → Tunnels → seleccionar el túnel → **Public Hostnames**, agregar:

|Campo|Valor|
|---|---|
|Subdomain|`posthog`|
|Domain|`tudominio.com`|
|Type|`HTTP`|
|URL|`localhost:8000`|

O en el archivo `config.yml` de cloudflared:

```yaml
ingress:
  - hostname: posthog.tudominio.com
    service: http://localhost:8000
  - service: http_status:404
```

```bash
# Aplicar y reiniciar el tunnel
sudo systemctl restart cloudflared
```

### Deshabilitar el contenedor Caddy (opcional pero recomendado)

Caddy no cumple ninguna función con este esquema y generará errores continuos en sus logs al intentar emitir un certificado Let's Encrypt sin el puerto 80 expuesto. Para evitar el ruido, comentar el servicio en `docker-compose.yml`:

```yaml
# services:
#   caddy:
#     ...
```

```bash
docker compose down && docker compose up -d
```

### WAF — regla de bypass para ingestión de eventos

Si el túnel tiene reglas WAF activas, los SDKs que envían eventos a `/e/`, `/batch/` y `/capture/` pueden recibir 403. Crear una regla de bypass:

```
URI Path contains: /e/ OR /batch/ OR /capture/
→ Action: Skip → WAF Managed Rules
```

---

## Verificación post-instalación

```bash
# Todos los contenedores deben aparecer como "Up" sin reinicios recientes
docker ps --format "table {{.Names}}\t{{.Status}}"

# Comprobar TLS
curl -I https://posthog.tudominio.com

# Monitorear uso de recursos
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}"
```

---

## Operaciones

### Actualizar

```bash
# Hacer backup de PostgreSQL antes de actualizar (ver sección Backups)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/posthog/posthog/HEAD/bin/upgrade-hobby)"
```

### Reiniciar el stack

```bash
cd /opt/posthog
docker compose down && docker compose up -d
```

### Logs

```bash
docker compose logs web -f
docker compose logs worker -f
docker compose logs clickhouse -f
```

### Auto-start en reboot del servidor

```bash
sudo crontab -e
# Agregar:
@reboot sleep 30 && cd /opt/posthog && docker compose up -d >> /var/log/posthog-start.log 2>&1
```

---

## Backups

PostHog no incluye backup automático. Mínimo recomendado:

```bash
# /opt/posthog/scripts/backup.sh
#!/bin/bash
BACKUP_DIR="/opt/backups/posthog"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p "$BACKUP_DIR"

# PostgreSQL
docker exec posthog-db pg_dump -U posthog posthog \
  | gzip > "$BACKUP_DIR/postgres_$DATE.sql.gz"

# Retener 7 días
find "$BACKUP_DIR" -name "postgres_*.sql.gz" -mtime +7 -delete
```

```bash
chmod +x /opt/posthog/scripts/backup.sh

# Cron diario a las 02:00
echo "0 2 * * * /opt/posthog/scripts/backup.sh >> /var/log/posthog-backup.log 2>&1" \
  | sudo crontab -
```

> **ClickHouse** almacena todos los eventos de analytics. Para producción, complementar con snapshots de disco a nivel de proveedor de nube (ej: GCP Persistent Disk Snapshots).

---

## Troubleshooting rápido

|Síntoma|Diagnóstico|Acción|
|---|---|---|
|Contenedor reiniciando|`docker compose logs <servicio>`|Ver errores en logs; OOM → revisar RAM disponible|
|Sitio no accesible desde internet|`cloudflared tunnel info <nombre>`|Verificar que el túnel está activo y apuntando a `localhost:8000`|
|Caddy con errores en logs|`docker compose logs caddy`|Esperado — Caddy no funciona en este esquema; deshabilitar el servicio|
|Ingestión de eventos falla (403)|Cloudflare WAF bloqueando|Agregar regla de bypass para paths `/e/`, `/batch/`, `/capture/`|
|Migración atascada al actualizar|`docker compose logs web`|Reintentar con `docker compose restart web`|
|Disco lleno|`docker system df`|`docker image prune -af` para liberar imágenes antiguas|

---

_compulandia.com.py — revisar ante cada upgrade mayor de PostHog_