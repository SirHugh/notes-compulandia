# Runbook: Instalación de Taiga con Docker + Cloudflare Tunnel

**Fecha:** Abril 2026 **Entorno:** Servidor Linux con Docker instalado **Acceso público:** Cloudflare Tunnel

---

## Requisitos previos

- Docker Engine 19.03+ y Docker Compose plugin instalados
- Git instalado
- Mínimo 2 GB RAM, 2 vCPUs, 20 GB disco
- `cloudflared` instalado y autenticado

---

## 1. Clonar repositorio oficial (rama stable)

```bash
cd /opt
sudo git clone -b stable https://github.com/taigaio/taiga-docker.git taiga
cd /opt/taiga
```

> La rama `stable` es la recomendada para entornos de producción.

---

## 2. Configurar el archivo `.env`

```bash
nano /opt/taiga/.env
```

### Variables críticas — nombres exactos requeridos por Taiga v6+

> ⚠️ **Importante:** Los nombres de variables deben ser exactamente los siguientes. Variables como `TAIGA_SCHEME` o `TAIGA_DOMAIN` no son reconocidas en v6+ y causan que la página cargue en blanco.

```env
# ── Dominio ────────────────────────────────────────────────────────
TAIGA_SITES_SCHEME=https                       # "http" o "https"
TAIGA_SITES_DOMAIN=projects.compulandia.com.py # dominio público exacto
SUBPATH=""                                     # dejar vacío si no hay subpath

# ── WebSockets ─────────────────────────────────────────────────────
WEBSOCKETS_SCHEME=wss   # "ws" para HTTP, "wss" para HTTPS (debe coincidir con TAIGA_SITES_SCHEME)

# ── Base de datos ──────────────────────────────────────────────────
POSTGRES_USER=taiga
POSTGRES_PASSWORD=<password_seguro>

# ── Clave secreta Django ───────────────────────────────────────────
SECRET_KEY=<generar con: python3 -c "import secrets; print(secrets.token_hex(32))">

# ── RabbitMQ ───────────────────────────────────────────────────────
RABBITMQ_USER=taiga
RABBITMQ_PASS=<password_seguro>

# ── Email (requerido para invitaciones) ────────────────────────────
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
DEFAULT_FROM_EMAIL=taiga@compulandia.com.py
EMAIL_HOST=smtp.tuservidor.com
EMAIL_PORT=587
EMAIL_HOST_USER=usuario@compulandia.com.py
EMAIL_HOST_PASSWORD=<password_email>
EMAIL_USE_TLS=True

# ── Telemetría ─────────────────────────────────────────────────────
ENABLE_TELEMETRY=False
```

**Generar SECRET_KEY:**

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

### Regla de consistencia HTTP/HTTPS

|`TAIGA_SITES_SCHEME`|`WEBSOCKETS_SCHEME`|
|---|---|
|`http`|`ws`|
|`https`|`wss`|

---

## 3. Resolver conflicto de puerto (si aplica)

El gateway de Taiga usa el puerto `9000` por defecto. Si ya está ocupado:

**Identificar el conflicto:**

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep 9000
sudo ss -tlnp | grep 9000
```

**Cambiar el puerto en `docker-compose.yml`:**

```yaml
taiga-gateway:
  ports:
    - "9100:80"   # cambiar 9000 por un puerto libre
```

> No es necesario cambiar `TAIGA_SITES_DOMAIN` si se accede mediante Cloudflare Tunnel, ya que el túnel apunta directamente a `ip:puerto` interno.

---

## 4. Levantar el stack

```bash
cd /opt/taiga
docker compose up -d
```

**Verificar que todos los contenedores estén en estado `Up`:**

```bash
docker compose ps
```

|Contenedor|Rol|
|---|---|
|`taiga-db`|PostgreSQL|
|`taiga-back`|Backend Django|
|`taiga-async`|Celery worker|
|`taiga-async-rabbitmq`|Cola async|
|`taiga-events`|WebSockets|
|`taiga-events-rabbitmq`|Cola de eventos|
|`taiga-front`|Frontend Angular|
|`taiga-protected`|Archivos protegidos|
|`taiga-gateway`|Nginx interno|

---

## 5. Crear superusuario

```bash
cd /opt/taiga
docker compose -f docker-compose.yml -f docker-compose-inits.yml \
  run --rm taiga-manage createsuperuser
```

Se solicitará email y contraseña del administrador.


---

Identificó usuarios válidos y consideró problema de autenticación

Identificó usuarios válidos y consideró problema de autenticación

Los usuarios existen. El username es `hquintero` y `mvaliente` — intentá loguearte con esos exactamente.

Si sigue fallando, el problema puede ser la contraseña. Resetéala directamente:

bash

```bash
docker compose exec taiga-back python manage.py shell -c "
from taiga.users.models import User
u = User.objects.get(username='hquintero')
u.set_password('Compu2026#')
u.is_active = True
u.save()
print('Password actualizado')
"
```

Luego logueate con `hquintero` y `NuevoPassword123`.
---

## 6. Publicar con Cloudflare Tunnel

En el dashboard de **Cloudflare Zero Trust → Tunnels**, configurar el servicio público:

|Campo|Valor|
|---|---|
|Subdominio|`projects`|
|Dominio|`compulandia.com.py`|
|Servicio|`http://IP_SERVIDOR:9000` (o el puerto configurado)|

> Cloudflare Tunnel gestiona el SSL automáticamente. Taiga no necesita certificado propio.

Asegurarse que `.env` refleje el dominio público y esquema HTTPS:

```env
TAIGA_SITES_SCHEME=https
TAIGA_SITES_DOMAIN=projects.compulandia.com.py
WEBSOCKETS_SCHEME=wss
```

Tras cualquier cambio en `.env`, reiniciar el stack completo:

```bash
docker compose down && docker compose up -d
```

---

## 7. Gestión de usuarios

### Crear el primer superusuario (por consola)

Ver paso 5. Solo es necesario hacerlo una vez.

### Invitar usuarios desde un proyecto (sin consola)

1. Ingresar al proyecto → **Settings** → **Members**
2. Clic en **Add members**
3. Ingresar el email del usuario y asignar rol:

|Rol|Permisos|
|---|---|
|Owner|Control total del proyecto|
|Admin|Gestión de miembros y configuración|
|Member|Crear y editar tareas|
|Restricted|Solo lectura|

4. Taiga envía un email de invitación automáticamente (requiere email configurado en `.env`)

> Si el email no está configurado, el link de invitación puede copiarse manualmente desde **Settings → Members** y compartirse por otro medio.

### Gestión avanzada desde el panel de administración

Acceder a `https://projects.compulandia.com.py/admin/` con el superusuario:

- Crear, activar o desactivar usuarios
- Cambiar contraseñas
- Asignar permisos de staff o superusuario
- Ver todos los proyectos y miembros

---

## 8. Inicio automático con systemd

```bash
sudo nano /etc/systemd/system/taiga.service
```

```ini
[Unit]
Description=Taiga Project Management
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/taiga
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable taiga
```

---

## 9. Backup de datos

```bash
# Base de datos PostgreSQL
docker compose exec taiga-db pg_dump -U taiga taiga \
  > /opt/backups/taiga_db_$(date +%F).sql

# Archivos media (adjuntos, avatares)
docker run --rm \
  -v taiga_taiga-static-data:/data \
  -v /opt/backups:/backup \
  alpine tar czf /backup/taiga_media_$(date +%F).tar.gz /data
```

---

## Referencia rápida de comandos

```bash
# Ver logs en tiempo real
docker compose logs -f taiga-back

# Reiniciar un servicio específico
docker compose restart taiga-back

# Reinicio completo (requerido tras cambios en .env)
docker compose down && docker compose up -d

# Actualizar Taiga a nueva versión
cd /opt/taiga
git pull origin stable
docker compose pull
docker compose up -d

# Acceder al shell Django
docker compose exec taiga-back python manage.py shell
```

---

## Problemas conocidos y soluciones

|Síntoma|Causa|Solución|
|---|---|---|
|Puerto ya en uso al levantar|Otro contenedor usa el mismo puerto|Cambiar puerto en `docker-compose.yml`|
|Página en blanco al acceder por dominio|Variables mal nombradas en `.env`|Usar `TAIGA_SITES_SCHEME` y `TAIGA_SITES_DOMAIN`|
|Página en blanco con HTTPS|`WEBSOCKETS_SCHEME=ws` en lugar de `wss`|Cambiar a `WEBSOCKETS_SCHEME=wss`|
|Error "Something happened"|`TAIGA_SITES_DOMAIN` no coincide con URL de acceso|Verificar dominio exacto en `.env`|
|Invitaciones no llegan por email|Email no configurado en `.env`|Configurar variables `EMAIL_*`|