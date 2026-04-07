# Zabbix — Guía de Instalación con Docker
 
## Requisitos 
- Docker 20.x o superior
- Docker Compose plugin
- 2GB RAM mínimo (recomendado 4GB)
- 20GB disco

---

## Levantar Zabbix

```bash
# 1. Crear carpeta del proyecto
mkdir zabbix && cd zabbix

# 2. Copiar el docker-compose.yml aquí

# 3. Levantar todos los contenedores
docker compose up -d

# 4. Ver que todos están corriendo
docker compose ps

# 5. Ver logs si algo falla
docker compose logs -f
```

La primera vez tarda 2-3 minutos mientras inicializa la base de datos.

---

## Acceder a la interfaz

Abrir el navegador en:
```
http://localhost:8080
```

**Credenciales por defecto:**
- Usuario: `Admin`
- Password: `zabbix`

> ⚠️ Cambiar el password inmediatamente después del primer login.

---

## Configurar descubrimiento de red

### Paso 1 — Crear regla de descubrimiento
1. Ir a **Configuration → Discovery**
2. Click en **Create discovery rule**
3. Configurar:
   - Name: `Red Interna`
   - IP range: `192.168.88.1-254` (ajustar a tu rango)
   - Update interval: `1h`
   - Checks: agregar `ICMP ping` y `SNMP agent`
4. Guardar

### Paso 2 — Crear acción automática
1. Ir a **Configuration → Actions → Discovery actions**
2. Click en **Create action**
3. Configurar para que al descubrir un host:
   - Lo agregue automáticamente a Zabbix
   - Le asigne un template básico (Linux, Windows, o SNMP device)

---

## Configurar MikroTik con SNMP

### En el MikroTik (terminal o Winbox)

```
# Habilitar SNMP
/snmp set enabled=yes

# Configurar community (string de acceso)
/snmp community set name=public addresses=192.168.88.0/24

# Verificar
/snmp print
```

### En Zabbix
1. Ir a **Configuration → Hosts → Create host**
2. Configurar:
   - Hostname: `MikroTik-Core`
   - IP: IP de tu MikroTik
   - Template: buscar `MikroTik` (viene incluido en Zabbix 7.0)
3. En la pestaña **SNMP interfaces** agregar la IP del router
4. Guardar

---

## Instalar agente Zabbix en PCs/Laptops (opcional)

### Windows
Descargar desde: https://www.zabbix.com/download_agents
Instalar y configurar con la IP del servidor Zabbix.

### Linux / Ubuntu
```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo apt update
sudo apt install zabbix-agent2 -y

# Editar configuración
sudo nano /etc/zabbix/zabbix_agent2.conf
# Cambiar: Server=<IP_DEL_SERVIDOR_ZABBIX>

sudo systemctl enable zabbix-agent2 --now
```

---

## Comandos útiles

```bash
# Detener Zabbix
docker compose down

# Reiniciar
docker compose restart

# Ver logs del servidor
docker compose logs zabbix-server -f

# Ver logs de la web
docker compose logs zabbix-web -f

# Actualizar imágenes
docker compose pull
docker compose up -d
```

---

## Notas importantes

- Cambiar todas las passwords del `docker-compose.yml` antes de usar en producción
- El puerto `8080` puede cambiarse si ya está en uso
- Los datos de PostgreSQL persisten en el volumen `postgres-data` aunque reinicies los contenedores
- Para acceso desde otros equipos de la red, usar la IP del servidor en lugar de `localhost`