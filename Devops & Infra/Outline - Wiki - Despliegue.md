# Outline Wiki — Documentación de Implementación

> **Fecha:** Abril 2025  
> **Responsable:** Infraestructura / Compulandia  
> **Estado:** ✅ En producción

---

## Índice

1. [Descripción general](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#descripci%C3%B3n-general)
2. [Arquitectura](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#arquitectura)
3. [Requisitos previos](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#requisitos-previos)
4. [Autenticación con Google OAuth](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#autenticaci%C3%B3n-con-google-oauth)
5. [Estructura de archivos](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#estructura-de-archivos)
6. [docker-compose.yml](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#docker-composeyml)
7. [Variables de entorno (.env)](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#variables-de-entorno-env)
8. [Cloudflare Tunnel](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#cloudflare-tunnel)
9. [Despliegue](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#despliegue)
10. [Problemas encontrados y soluciones](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#problemas-encontrados-y-soluciones)
11. [Backup y recuperación](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#backup-y-recuperaci%C3%B3n)
12. [Mantenimiento](https://claude.ai/chat/0b390fad-39de-4783-a1ea-4ddb9ea0c1fb#mantenimiento)

---

## Descripción general

Outline es una plataforma open source de gestión de documentación y wiki para equipos. Permite crear colecciones de documentos, colaborar en tiempo real, buscar contenido con full-text search y gestionar permisos por roles.

**URL de acceso:** `https://wiki.compulandia.com.py`  
**Versión:** `outlinewiki/outline:latest`  
**Licencia:** BSL 1.1 (convierte a Apache 2.0 tras 3 años)

---

## Arquitectura

```
Internet
   │
   ▼
Cloudflare (DNS + Túnel)
   │
   ▼
VM Google Cloud
   │
   └── Docker Compose Stack
         ├── outline:3000       ← Aplicación principal
         ├── postgres:5432      ← Base de datos + búsqueda full-text
         └── redis:6379         ← Caché y colas
```

**Almacenamiento de archivos:** Local (volumen Docker `outline-data`)  
**Reverse proxy / TLS:** Cloudflare Tunnel (no se expone el puerto 3000 al exterior)

---

## Requisitos previos

|Requisito|Detalle|
|---|---|
|Servidor|VM en Google Cloud con Docker y Docker Compose instalados|
|RAM|Mínimo 2 GB (recomendado 4 GB)|
|Disco|Mínimo 10 GB libres|
|Dominio|Subdominio `wiki.compulandia.com.py` administrado en Cloudflare|
|Autenticación|Google OAuth (Google Cloud Console)|
|Cloudflare|Tunnel configurado hacia la VM|

---

## Autenticación con Google OAuth

Outline **no tiene autenticación propia por usuario/contraseña**. Requiere un proveedor externo. En este caso se usa **Google OAuth**.

### Pasos para configurar la app en Google Cloud

1. Ingresar a [console.cloud.google.com](https://console.cloud.google.com/)
2. Ir a **APIs & Services → Credentials**
3. Clic en **Create Credentials → OAuth 2.0 Client ID**
4. Tipo de aplicación: **Web application**
5. En **Authorized redirect URIs** agregar:
    
    ```
    https://wiki.compulandia.com.py/auth/google.callback
    ```
    
6. Copiar y guardar de forma segura:
    - `Client ID` → se usa en `GOOGLE_CLIENT_ID`
    - `Client Secret` → se usa en `GOOGLE_CLIENT_SECRET`

> ⚠️ **Nota de seguridad:** Por defecto, cualquier cuenta de Google puede autenticarse si conoce la URL. Se recomienda agregar una regla en **Cloudflare Access** para restringir el acceso únicamente a cuentas `@compulandia.com.py`.

---

## Estructura de archivos

```
/opt/outline/
├── docker-compose.yml
└── .env
```

---

## docker-compose.yml

```yaml
services:
  outline:
    image: outlinewiki/outline:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      SECRET_KEY: ${SECRET_KEY}
      UTILS_SECRET: ${UTILS_SECRET}
      DATABASE_URL: postgres://outline:${POSTGRES_PASSWORD}@postgres:5432/outline
      REDIS_URL: redis://redis:6379
      URL: ${URL}
      PORT: 3000
      # Autenticación Google
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      # Almacenamiento local
      FILE_STORAGE: local
      FILE_STORAGE_LOCAL_ROOT_DIR: /var/lib/outline/data
      FILE_STORAGE_UPLOAD_MAX_SIZE: 26214400
      # Reverse proxy — Cloudflare maneja TLS
      FORCE_HTTPS: "false"
      PGSSLMODE: disable
      DEFAULT_LANGUAGE: en_US
    volumes:
      - outline-data:/var/lib/outline/data
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: outline
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: outline
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U outline -d outline"]
      interval: 10s
      timeout: 3s
      retries: 3

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  outline-data:
  postgres-data:
  redis-data:
```

---

## Variables de entorno (.env)

Ubicación: `/opt/outline/.env`

```bash
# ─── Secrets (generar con: openssl rand -hex 32) ───────────────────────────
SECRET_KEY=REEMPLAZAR_CON_HEX_64_CHARS
UTILS_SECRET=REEMPLAZAR_CON_HEX_64_CHARS

# ─── Base de datos ──────────────────────────────────────────────────────────
POSTGRES_PASSWORD=PASSWORD_FUERTE_AQUI

# ─── URL pública (sin trailing slash) ──────────────────────────────────────
URL=https://wiki.compulandia.com.py

# ─── Google OAuth ───────────────────────────────────────────────────────────
GOOGLE_CLIENT_ID=tu_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=tu_client_secret
```

### Generar los secrets

```bash
openssl rand -hex 32  # → copiar resultado en SECRET_KEY
openssl rand -hex 32  # → copiar resultado en UTILS_SECRET
```

> ⚠️ **Importante:** Una vez inicializado el volumen de PostgreSQL, cambiar `POSTGRES_PASSWORD` en el `.env` **no actualiza** la contraseña en la base de datos existente. Ver sección de problemas conocidos.

---

## Cloudflare Tunnel

El puerto `3000` **no se expone al exterior**. El tráfico llega a través de un Cloudflare Tunnel configurado en **Zero Trust → Networks → Tunnels**.

### Configuración de la ruta pública

|Campo|Valor|
|---|---|
|Subdomain|`wiki`|
|Domain|`compulandia.com.py`|
|Type|`HTTP`|
|URL|`localhost:3000`|

> El túnel actúa como reverse proxy interno. Cloudflare gestiona el certificado TLS del subdominio. Por eso en el contenedor se configura `FORCE_HTTPS: "false"`.

---

## Despliegue

### Primera vez

```bash
# 1. Crear directorio de trabajo
mkdir -p /opt/outline && cd /opt/outline

# 2. Crear docker-compose.yml y .env con los valores correctos

# 3. Generar secrets
openssl rand -hex 32  # SECRET_KEY
openssl rand -hex 32  # UTILS_SECRET

# 4. Levantar el stack
docker compose up -d

# 5. Verificar estado de los servicios
docker compose ps

# 6. Seguir logs de Outline
docker compose logs -f outline
```

El **primer usuario** que inicie sesión con Google queda automáticamente como **administrador**.

### Comandos de gestión habituales

```bash
# Ver estado
docker compose ps

# Ver logs en tiempo real
docker compose logs -f outline

# Reiniciar solo Outline (sin tocar DB ni Redis)
docker compose restart outline

# Bajar todo el stack
docker compose down

# Actualizar imagen de Outline
docker compose pull outline
docker compose up -d outline
```

---

## Problemas encontrados y soluciones

### ❌ `password authentication failed for user "outline"`

**Causa:** Mismatch entre la contraseña guardada en el volumen de PostgreSQL y el valor actual de `POSTGRES_PASSWORD` en el `.env`. Ocurre cuando el volumen fue creado previamente con una contraseña diferente o cuando se modifica el `.env` después de la primera inicialización.

**Síntoma en logs:**

```
SequelizeConnectionError: password authentication failed for user "outline"
outline exited with code 1 (restarting)
```

**Solución** (solo si no hay datos que conservar):

```bash
cd /opt/outline

# 1. Bajar el stack
docker compose down

# 2. Eliminar el volumen de postgres
docker volume rm outline_postgres-data

# 3. Volver a levantar (se reinicializa con la contraseña del .env)
docker compose up -d

# 4. Verificar
docker compose logs -f outline
```

> ⚠️ Este procedimiento **elimina todos los datos de la base de datos**. Solo aplicar en instalaciones nuevas o si se cuenta con un backup.

---

## Backup y recuperación

### Backup de la base de datos

```bash
docker compose exec postgres pg_dump -U outline outline > outline_backup_$(date +%Y%m%d).sql
```

### Backup de archivos subidos

```bash
# Copiar el volumen de datos a una ubicación externa
docker run --rm \
  -v outline_outline-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/outline-files-$(date +%Y%m%d).tar.gz /data
```

### Backup del .env

```bash
cp /opt/outline/.env /ruta/segura/outline.env.backup
```

### Restaurar la base de datos

```bash
cat outline_backup_YYYYMMDD.sql | docker compose exec -T postgres psql -U outline outline
```

---

## Mantenimiento

### Actualizar Outline

```bash
cd /opt/outline
docker compose pull outline
docker compose up -d outline
docker compose logs -f outline  # verificar que migraciones corran correctamente
```

> Outline ejecuta migraciones de DB automáticamente al iniciar. Esperar 30-60 segundos en la primera ejecución tras una actualización.

### Monitorear uso de disco

```bash
# Tamaño de los volúmenes Docker
docker system df -v | grep outline
```

---

_Documento generado como registro interno de la implementación. Actualizar ante cualquier cambio de configuración._