# Manual de Instalación: Node Exporter con Autenticación

Guía completa para instalar y configurar `prometheus-node-exporter` con autenticación Basic Auth en servidores Debian/Ubuntu.

---

## Requisitos previos

- Sistema operativo: Debian 11+ / Ubuntu 20.04+
- Acceso root o sudo
- IP del servidor Prometheus: `192.168.0.201` (ajustar según tu entorno)

---

## Paso 1: Instalación de Node Exporter

```bash
sudo apt-get update
sudo apt-get install -y prometheus-node-exporter
```

Habilitar e iniciar el servicio:

```bash
sudo systemctl enable --now prometheus-node-exporter
sudo systemctl status prometheus-node-exporter --no-pager
```

---

## Paso 2: Configurar Firewall

Exponer puerto 9100 **solo** para el servidor Prometheus:

```bash
# Permitir solo desde Prometheus
sudo ufw allow from 192.168.0.201 to any port 9100 proto tcp

# Denegar acceso desde cualquier otra IP (si UFW está activo)
sudo ufw status
```

---

## Paso 3: Configurar Autenticación Basic Auth

### 3.1 Instalar herramienta para generar hash

```bash
sudo apt-get install -y apache2-utils
```

### 3.2 Generar hash de la contraseña

```bash
htpasswd -nBC 12 "" | tr -d ':\n'; echo
```

- Ingresá la contraseña cuando lo solicite
- Copiá el hash generado (empieza con `$2y$12$...`)

### 3.3 Crear archivo de configuración web

```bash
sudo mkdir -p /etc/prometheus
sudo nano /etc/prometheus/web-config.yml
```

Contenido (reemplazar el hash):

```yaml
basic_auth_users:
  prometheus: '$2y$12$PEGAR_TU_HASH_AQUI'
```

Ajustar permisos:

```bash
sudo chown prometheus:prometheus /etc/prometheus/web-config.yml
sudo chmod 600 /etc/prometheus/web-config.yml
```

### 3.4 Configurar argumentos del servicio

```bash
sudo nano /etc/default/prometheus-node-exporter
```

Contenido:

```
ARGS="--web.listen-address=:9100 --web.config.file=/etc/prometheus/web-config.yml"
```

### 3.5 Configurar override de systemd (si es necesario)

Si el servicio no toma los ARGS automáticamente:

```bash
sudo systemctl edit prometheus-node-exporter
```

Pegar:

```ini
[Service]
EnvironmentFile=-/etc/default/prometheus-node-exporter
ExecStart=
ExecStart=/usr/bin/prometheus-node-exporter $ARGS
```

---

## Paso 4: Reiniciar y Verificar

### 4.1 Reiniciar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl restart prometheus-node-exporter
sudo systemctl status prometheus-node-exporter --no-pager
```

### 4.2 Verificar que escucha en el puerto

```bash
sudo ss -ltnp | grep :9100
```

Salida esperada:
```
LISTEN  0  4096  *:9100  *:*  users:(("prometheus-node",pid=XXXX,fd=3))
```

### 4.3 Probar autenticación

```bash
# Sin credenciales - debe dar error 401 Unauthorized
curl -s -o /dev/null -w "%{http_code}" http://localhost:9100/metrics
# Esperado: 401

# Con credenciales - debe mostrar métricas
curl -u prometheus:TU_CONTRASEÑA http://localhost:9100/metrics | head
```

---

## Paso 5: Configurar Prometheus

En el servidor donde corre Prometheus, editar `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: node
    basic_auth:
      username: prometheus
      password: 'TU_CONTRASEÑA'
    static_configs:
      - targets:
          - '192.168.0.201:9100'
          - '192.168.0.250:9100'
          # Agregar más servidores aquí
```

Recargar Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

Verificar en `http://<prometheus>:9090/targets` que el estado sea `UP`.

---

## Troubleshooting

### El servicio no inicia

```bash
# Ver logs detallados
journalctl -u prometheus-node-exporter --no-pager -n 50

# Verificar sintaxis del web-config.yml
cat /etc/prometheus/web-config.yml
```

Errores comunes:
- YAML mal formateado (espacios vs tabs)
- Hash de contraseña incorrecto o incompleto

### Prometheus no puede conectarse

```bash
# Desde el servidor Prometheus, probar conexión
curl -u prometheus:TU_CONTRASEÑA http://IP_SERVIDOR:9100/metrics | head

# Verificar firewall en el servidor destino
sudo ufw status
sudo iptables -L -n | grep 9100
```

### Error 401 con credenciales correctas

Regenerar el hash y actualizar `web-config.yml`:

```bash
htpasswd -nBC 12 "" | tr -d ':\n'; echo
sudo nano /etc/prometheus/web-config.yml
sudo systemctl restart prometheus-node-exporter
```

---

## Resumen de Archivos

| Archivo | Ubicación | Propósito |
|---------|-----------|-----------|
| Binario | `/usr/bin/prometheus-node-exporter` | Ejecutable |
| Argumentos | `/etc/default/prometheus-node-exporter` | Configuración ARGS |
| Autenticación | `/etc/prometheus/web-config.yml` | Usuario y hash |
| Servicio | `/lib/systemd/system/prometheus-node-exporter.service` | Unit systemd |
| Override | `/etc/systemd/system/prometheus-node-exporter.service.d/override.conf` | Personalización |

---

## Script de Instalación Rápida

Para automatizar todo, usar el script `install.sh` del repositorio:

```bash
git clone <tu-repo>/monitoring-config.git
cd monitoring-config/node-exporter
# Editar web-config.yml con el hash generado
sudo bash install.sh
```

---

## Notas de Seguridad

- La contraseña se almacena como hash bcrypt, no en texto plano
- El archivo `web-config.yml` tiene permisos 600 (solo lectura para prometheus)
- El puerto 9100 solo acepta conexiones desde el servidor Prometheus
- Usar la misma contraseña en todos los servidores simplifica la gestión

---

*Última actualización: Febrero 2026*
