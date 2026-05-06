# Chatwoot v4.8.0 — Despliegue de Producción (Debian + Docker)

Stack: Chatwoot v4.8.0 · Postgres 16 + pgvector · Redis 7 · publicado vía Cloudflare Tunnel corriendo en un servidor separado dentro de la misma LAN.

**Topología real del despliegue:**

```
Internet
   ↓ HTTPS
chatwoot.compulandia.com.py (DNS → Cloudflare edge)
   ↓ HTTPS
Cloudflare Tunnel (servidor separado, ej. 192.168.0.50)
   ↓ HTTP + Host: chatwoot.compulandia.com.py
192.168.0.201:3700 (servidor Chatwoot, filtrado por UFW)
   ↓ Docker bridge
chatwoot-rails → postgres + redis
```

---

## 1. Prerrequisitos del servidor

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Docker Engine + Compose plugin (oficial)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install -y docker-compose-plugin

# Validar versiones
docker --version          # >= 20.10
docker compose version    # >= 2.14

# Agregar usuario al grupo docker (re-login después)
sudo usermod -aG docker $USER
```

**Dimensionamiento**: 4 vCPU, 8 GB RAM, 50 GB SSD recomendado. Mínimo viable 2 vCPU / 4 GB.

---

## 2. Preparar directorio y archivos

```bash
sudo mkdir -p /opt/chatwoot && sudo chown $USER:$USER /opt/chatwoot
cd /opt/chatwoot
# Copiar docker-compose.yaml y .env a este directorio
```

### Generar secretos

Usar **solo caracteres hex** para evitar problemas de URL-encoding en `REDIS_URL` y de escapes de shell en los healthchecks:

```bash
# SECRET_KEY_BASE (64 bytes hex)
openssl rand -hex 64

# POSTGRES_PASSWORD (32 bytes hex)
openssl rand -hex 32

# REDIS_PASSWORD (32 bytes hex)
openssl rand -hex 32

# FB_VERIFY_TOKEN / IG_VERIFY_TOKEN / WhatsApp verify tokens
openssl rand -hex 24
```

Reemplazar los `CHANGE_ME_*` en `.env`. **Importante**: el password de Redis debe aparecer idéntico en `REDIS_URL` (formato `redis://:PASS@redis:6379`) y en `REDIS_PASSWORD`.

---

## 3. Puerto de publicación

En `docker-compose.yaml`, el servicio `rails` publica en el puerto `3700` del host (Grafana ya ocupa el 3000 en este servidor):

```yaml
ports:
  - "3700:3000"
```

**Nota sobre bind address**: el puerto se publica en `0.0.0.0:3700` porque el servidor de Cloudflare Tunnel corre en otra máquina de la LAN. El acceso lo restringimos vía UFW (paso 9), no con bind-to-loopback.

---

## 4. Inicialización de la base de datos

```bash
cd /opt/chatwoot

# Descargar imágenes
docker compose pull

# Crear schema + habilitar pgvector + correr migraciones (PRIMERA vez)
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
```

Si falla con `PG::UndefinedFile: could not open extension control file "vector.control"`, verificar que la imagen en `docker-compose.yaml` sea `pgvector/pgvector:pg16`, no `postgres:*`.

---

## 5. Levantar la plataforma

```bash
docker compose up -d

# Esperar ~1-2 minutos a que Rails precompile assets y boote
docker compose logs -f rails
# Esperado al final:
#   * Listening on http://0.0.0.0:3000

# Validar desde el mismo servidor
curl -I http://127.0.0.1:3700
# Esperado: HTTP/1.1 302 Found
#           location: http://127.0.0.1:3700/installation/onboarding
# (el redirect a onboarding confirma que DB + Redis + Rails están OK)
```

**Nota**: el `302` redirigiendo a `127.0.0.1:3700` es **correcto** cuando se consulta con `Host: 127.0.0.1:3700`. Vía Cloudflare Tunnel el `Host` se sobrescribe a `chatwoot.compulandia.com.py` y Rails arma los redirects con el dominio correcto.

---

## 6. VAPID keys (push notifications)

**Chatwoot v4 autogenera las VAPID keys** en el primer boot y las guarda en la base de datos (tabla `global_configs`). No requiere rake task ni variables de entorno (las del `.env` son opcionales, solo para forzar keys específicas al migrar desde otra instancia).

Validar que se generaron:

```bash
docker compose exec rails bundle exec rails runner \
  "puts 'public: ' + GlobalConfig.get_value('VAPID_PUBLIC_KEY').to_s[0..30] + '...'; \
   puts 'private set: ' + GlobalConfig.get_value('VAPID_PRIVATE_KEY').present?.to_s"
```

Si ambos tienen valor, push notifications listas.

---

## 7. Cloudflare Tunnel — Public Hostname

En el panel de Zero Trust → **Networks → Tunnels → tu tunnel → Public Hostnames → Add**:

|Campo|Valor|
|---|---|
|Subdomain|`chatwoot`|
|Domain|`compulandia.com.py`|
|Path|_(vacío)_|
|Type|`HTTP`|
|URL|`192.168.0.201:3700`|

### Additional application settings — HTTP Settings

|Campo|Valor|
|---|---|
|**HTTP Host Header**|`chatwoot.compulandia.com.py`|
|Disable Chunked Encoding|OFF|
|Connection timeout|90s|
|Keep-Alive Connections|100|

**Crítico**: el `HTTP Host Header` es lo que hace que Rails arme URLs correctas. Sin ese header sobrescrito, los redirects, cookies de session y links absolutos apuntan a `192.168.0.201:3700` y rompen todo el flujo.

### TLS Settings

|Campo|Valor|
|---|---|
|Origin Server Name|`chatwoot.compulandia.com.py`|
|No TLS Verify|ON _(el origin es HTTP plano, no TLS)_|

### WebSockets

Habilitado por defecto en cloudflared moderno. Requerido para ActionCable (chat en vivo).

### Si tenés acceso al `config.yml` de cloudflared

```yaml
ingress:
  - hostname: chatwoot.compulandia.com.py
    service: http://192.168.0.201:3700
    originRequest:
      httpHostHeader: chatwoot.compulandia.com.py
      noTLSVerify: true
      connectTimeout: 30s
      keepAliveTimeout: 90s
```

---

## 8. Cloudflare dashboard — ajustes del dominio

En `compulandia.com.py`:

### SSL/TLS → Overview

- **Encryption mode**: `Full` (NO `Flexible`, produce redirect loop con `FORCE_SSL=true`)

### Speed → Optimization

- **Rocket Loader**: OFF para este hostname
    - Page Rules / Configuration Rules → Match `chatwoot.compulandia.com.py/*` → Rocket Loader: Off
    - Rocket Loader rompe el Vue.js del frontend de Chatwoot

### Security → WAF

Cuando se registren webhooks de Meta/Twilio, la WAF puede bloquear los POST entrantes. Crear **Custom Rule** preventiva:

```
Name: Chatwoot webhooks bypass
Expression:
  (http.host eq "chatwoot.compulandia.com.py" and
   starts_with(http.request.uri.path, "/webhooks/"))
  or
  (http.host eq "chatwoot.compulandia.com.py" and
   http.request.uri.path eq "/twilio/callback")
Action: Skip → All remaining custom rules + Managed Rules
```

Este es el mismo patrón que usamos para el Grafana Agent remote_write.

---

## 9. UFW — restringir el acceso al puerto 3700

El puerto `3700` está publicado en `0.0.0.0` a nivel Docker, así que cualquier cosa en la LAN puede alcanzarlo si no filtramos. Permitir **solo** la IP del servidor de Cloudflare Tunnel:

```bash
# Reemplazar 192.168.0.50 por la IP real del servidor de cloudflared
sudo ufw allow from 192.168.0.50 to any port 3700 proto tcp \
  comment 'Chatwoot via CF Tunnel'

# SSH
sudo ufw allow 22/tcp

# Defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Activar (si no estaba ya activo)
sudo ufw enable

# Verificar
sudo ufw status numbered
```

**Importante sobre Docker + UFW**: Docker por defecto inserta reglas en `iptables` que pueden **saltear UFW**. Si después de aplicar lo anterior podés alcanzar `192.168.0.201:3700` desde una IP que no es `192.168.0.50`, revisar `/etc/docker/daemon.json` y considerar el workaround de UFW-Docker: https://github.com/chaifeng/ufw-docker

En la mayoría de despliegues con `ports: "3700:3000"` + UFW default deny, la regla `allow from IP` funciona correctamente.

---

## 10. Primer login

Ir a `https://chatwoot.compulandia.com.py`. Se abre `/installation/onboarding` donde se crea la cuenta admin inicial. **Esta es la única vez** que el signup funciona con `ENABLE_ACCOUNT_SIGNUP=false` — el primer usuario siempre puede registrarse para bootstrap.

Después, para operaciones de superadmin: `/super_admin` (mismas credenciales del primer admin).

---

## 11. Configurar canales

### 11.1 WhatsApp Cloud API

Requisitos previos en Meta:

- Cuenta en business.facebook.com
- WhatsApp Business Account con Phone Number ID
- App en developers.facebook.com con producto **WhatsApp** agregado
- **Permanent Access Token** (System User → Generate Token, permisos: `whatsapp_business_messaging`, `whatsapp_business_management`)

En Chatwoot: **Settings → Inboxes → Add Inbox → WhatsApp → WhatsApp Cloud**:

- Phone Number
- Phone Number ID
- Business Account ID
- API Key (el permanent token)
- Webhook Verify Token (inventar y copiar a Meta)

Webhook a registrar en Meta → WhatsApp → Configuration:

```
URL: https://chatwoot.compulandia.com.py/webhooks/whatsapp/<PHONE_NUMBER_ID>
Verify Token: <el que pusiste en Chatwoot>
Fields suscritos: messages
```

### 11.2 Facebook Messenger

1. Meta for Developers → App type: Business. Agregar producto **Messenger**.
2. Configurar `FB_APP_ID` y `FB_APP_SECRET` en `.env`, reiniciar: `docker compose up -d rails sidekiq`
3. Chatwoot → **Settings → Inboxes → Add → Facebook Messenger** → login con la cuenta que administra la página.
4. Meta → Messenger → Webhooks, subscribir la Page con fields: `messages`, `messaging_postbacks`, `message_deliveries`, `message_reads`.

Callback URL: `https://chatwoot.compulandia.com.py/webhooks/facebook` Verify Token: valor de `FB_VERIFY_TOKEN`

### 11.3 Instagram DM

Requiere cuenta de Instagram **Business** vinculada a una Facebook Page. En Meta agregar el producto **Instagram** a la misma app.

Chatwoot → **Settings → Inboxes → Add → Instagram** → login. Subscribir en Meta con fields: `messages`, `messaging_postbacks`.

Callback: `https://chatwoot.compulandia.com.py/webhooks/instagram`

### 11.4 Email (IMAP + SMTP)

Se configura **por inbox** desde la UI: **Settings → Inboxes → Add → Email**:

- IMAP: `imap.gmail.com:993` SSL (o tu servidor de correo)
- SMTP: si el global del `.env` alcanza, dejar ese; si no, configurar aquí

Para Gmail con 2FA, generar App Password.

### 11.5 SMS (Twilio)

Chatwoot → **Settings → Inboxes → Add → SMS → Twilio**:

- Account SID, Auth Token, número Twilio

En consola de Twilio, número → Messaging → "A message comes in":

```
Webhook: https://chatwoot.compulandia.com.py/twilio/callback
Method: POST
```

---

## 12. Backups

### 12.1 Postgres

Script `/opt/chatwoot/backup-pg.sh`:

```bash
#!/bin/bash
set -e
DEST=/var/backups/chatwoot
mkdir -p "$DEST"
TS=$(date +%Y%m%d-%H%M)
cd /opt/chatwoot
docker compose exec -T postgres \
  pg_dump -U chatwoot -d chatwoot -F c -f - > "$DEST/chatwoot-$TS.dump"
# Rotación: 14 días
find "$DEST" -name "chatwoot-*.dump" -mtime +14 -delete
```

```bash
chmod +x /opt/chatwoot/backup-pg.sh
# Cron diario 02:30
echo "30 2 * * * root /opt/chatwoot/backup-pg.sh" | \
  sudo tee /etc/cron.d/chatwoot-backup
```

### 12.2 Storage (adjuntos)

```bash
docker run --rm \
  -v chatwoot_storage_data:/data:ro \
  -v /var/backups/chatwoot:/backup \
  alpine tar czf /backup/storage-$(date +%Y%m%d).tar.gz -C /data .
```

Sincronizar `/var/backups/chatwoot` a GCS con `gcloud storage rsync` o `rclone` en el mismo cron para backup off-site.

---

## 13. Actualizaciones

```bash
cd /opt/chatwoot

# 1. Backup primero (siempre)
./backup-pg.sh

# 2. Bumpear versión en docker-compose.yaml (v4.8.0 → v4.x.0)
# 3. Pull + migrar
docker compose pull
docker compose run --rm rails bundle exec rails db:migrate
docker compose up -d

# 4. Verificar
docker compose logs -f rails sidekiq
curl -I http://127.0.0.1:3700
```

---

## 14. Monitoreo (integración con stack Prometheus existente)

- **Postgres**: `postgres_exporter` → `postgresql://chatwoot:PASS@localhost:5432/chatwoot`
- **Redis**: `redis_exporter` → `REDIS_ADDR=redis://:PASS@localhost:6379`
- **Rails**: `blackbox_exporter` → probe HTTP `https://chatwoot.compulandia.com.py/health`
- **Sidekiq**: panel web en `/sidekiq` (requiere auth de super_admin)

Alertas sugeridas en Alertmanager:

- Postgres down, conexiones > 80% del max
- Sidekiq queue latency > 60s
- Rails 5xx rate > 1% en 5min
- Volumen `chatwoot_storage_data` > 85% lleno
- Cloudflare Tunnel status (vía API de CF a `/accounts/:id/cfd_tunnel/:tunnel_id`)

---

## 15. Troubleshooting — casos reales encontrados en este despliegue

### `container chatwoot-redis is unhealthy`

**Causa**: el `command` original usaba `sh -c` y Docker Compose interpolaba el password antes del shell, rompiendo cuando el password tenía caracteres especiales (`$`, `/`, `+`).

**Fix**: usar `command` como lista de args (sin `sh -c`) y en el healthcheck usar `$$REDIS_PASSWORD` (doble dólar) para que lo resuelva el shell del contenedor vía env var. Ya aplicado en el `docker-compose.yaml` entregado.

**Prevención**: generar el password con `openssl rand -hex 32` (solo caracteres `0-9a-f`).

### `Bind for 0.0.0.0:3000 failed: port is already allocated`

**Causa**: Grafana ya ocupa el 3000 en este servidor.

**Fix**: cambiar a puerto 3700. Solo se cambia el puerto del host (izquierda), el del contenedor (derecha) sigue siendo 3000:

```yaml
ports:
  - "3700:3000"
```

### `rake aborted! Don't know how to build task 'webpush:generate_keys'`

**Causa**: esa tarea existe en Mastodon, no en Chatwoot. Chatwoot v4 autogenera las VAPID keys en el primer boot y las guarda en DB.

**Fix**: no ejecutar esa tarea. Validar con el runner del paso 6.

### `ERR_CONNECTION_REFUSED` desde otra máquina de la LAN

**Causa**: el `ports` estaba en `127.0.0.1:3700:3000`, que solo acepta conexiones desde loopback del mismo servidor. Cloudflare Tunnel corre en otro servidor de la LAN y no podía conectar.

**Fix**: cambiar a `"3700:3000"` (bind `0.0.0.0`) + UFW allow desde la IP específica del servidor de cloudflared.

### Rails redirige a `http://127.0.0.1:3700/installation/onboarding`

**Causa**: Rails arma URLs con el `Host` header que recibe. Si el tunnel no sobrescribe el Host, Rails usa `192.168.0.201:3700` o lo que le llegue.

**Fix**: configurar **HTTP Host Header** = `chatwoot.compulandia.com.py` en el Public Hostname del tunnel (paso 7).

### Webhooks de WhatsApp/Meta dan 403 o timeout

**Causa**: WAF de Cloudflare bloqueando POST a `/webhooks/*`.

**Fix**: Custom Rule de skip del paso 8 (WAF).

### Login redirige infinitamente (`ERR_TOO_MANY_REDIRECTS`)

**Causa**: `FORCE_SSL=true` + SSL mode en Cloudflare = `Flexible`. Cloudflare manda HTTP al origin, Rails fuerza HTTPS con un 301, y el ciclo se repite.

**Fix**: cambiar SSL mode a `Full` en el dashboard de Cloudflare.

### Sidekiq no procesa jobs

**Causa típica**: `REDIS_URL` mal formateado. Si el password tiene caracteres especiales, hay que URL-encodearlos dentro del URL pero no dentro de `REDIS_PASSWORD`. Mejor: regenerar con `openssl rand -hex 32`.

**Fix**: `docker compose logs sidekiq` para ver el error exacto.

---

## 16. Configuración final consolidada — referencia rápida

### `/opt/chatwoot/docker-compose.yaml` (puntos clave)

- Imagen Chatwoot: `chatwoot/chatwoot:v4.8.0`
- Imagen Postgres: `pgvector/pgvector:pg16` (no postgres vanilla)
- Redis `command` como lista de args, healthcheck con `$$REDIS_PASSWORD`
- Rails `ports: "3700:3000"` (bind 0.0.0.0, filtrado por UFW)
- Volúmenes: `postgres_data`, `redis_data`, `storage_data`

### `.env` (puntos clave)

- `FRONTEND_URL=https://chatwoot.compulandia.com.py`
- `FORCE_SSL=true`
- `ENABLE_ACCOUNT_SIGNUP=false`
- `DEFAULT_LOCALE=es`
- Secretos generados con `openssl rand -hex N`
- SMTP de salida configurado (Gmail con App Password recomendado)

### Cloudflare

- Public Hostname con **HTTP Host Header** override
- SSL mode: **Full**
- Rocket Loader: **OFF** para el hostname
- WAF Custom Rule para `/webhooks/*` y `/twilio/callback`

### Firewall

- UFW `allow from <IP-cloudflared> to any port 3700`
- UFW `allow 22/tcp`
- UFW `default deny incoming`