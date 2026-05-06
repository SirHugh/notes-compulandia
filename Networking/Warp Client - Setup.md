# Cloudflare WARP — Configuración de acceso a redes privadas vía Zero Trust

**Fecha:** Abril 2026  
**Plataforma cliente:** Windows 10/11  
**Entorno:** Cloudflare Zero Trust (Cloudflare One)

---

## Prerequisitos

- Cuenta de Cloudflare con Zero Trust activo (`one.dash.cloudflare.com`)
- Túnel `cloudflared` operativo con al menos un conector en estado **Healthy**
- Correo electrónico con acceso al panel de administración

---

## 1. Panel de administración

El panel de Zero Trust es independiente del dashboard general de Cloudflare.

|Panel|URL|
|---|---|
|Dashboard general|`dash.cloudflare.com`|
|**Zero Trust (Cloudflare One)**|`one.dash.cloudflare.com`|

> Toda la configuración de WARP, enrolamiento y rutas se realiza exclusivamente en `one.dash.cloudflare.com`.

---

## 2. Configurar permisos de enrolamiento de dispositivos

```
Zero Trust → Settings → (sección) Admin Control & Permissions
  → Device enrollment permissions → Manage
```

Crear una regla de inclusión:

|Campo|Valor|
|---|---|
|**Rule name**|nombre descriptivo|
|**Include**|Emails → correo del usuario o dominio completo (`*@dominio.com`)|
|**Authentication method**|One-time PIN (si no hay IdP configurado)|

---

## 3. Obtener el Team Name

El Team Name es requerido al enrolar el cliente manualmente.

```
Zero Trust → Settings → (sección) Team Name
```

Formato: `nombre-equipo` (prefijo de `nombre-equipo.cloudflareaccess.com`)

---

## 4. Instalar y enrolar el cliente

### Descarga

Cliente oficial (Cloudflare One Client, anteriormente WARP):  
`https://developers.cloudflare.com/cloudflare-one/connections/connect-devices/warp/download-warp/`

### Enrolamiento manual en Windows

1. Abrir el cliente instalado
2. Seleccionar **"Zero Trust security"**
3. Ingresar el **Team Name**
4. Completar autenticación en el navegador (One-time PIN al correo)
5. El cliente queda conectado automáticamente

### Verificar conexión

```powershell
warp-cli.exe status
# Resultado esperado: Connected

warp-cli.exe tunnel stats
# Muestra latencia estimada y pérdida de paquetes hacia Cloudflare
```

---

## 5. Declarar rutas de red privada

Las redes privadas accesibles a través del túnel se declaran en:

```
Zero Trust → Networks → Routes → Add CIDR route
```

|Campo|Valor|
|---|---|
|**Network (CIDR)**|p.ej. `10.100.102.0/24` o `/32` para host específico|
|**Tunnel**|seleccionar el túnel `cloudflared` activo|
|**Description**|nombre descriptivo del recurso|

> **Importante:** Una ruta en esta sección le dice a Cloudflare cómo reenviar el tráfico, pero no es suficiente por sí sola. También es necesario el paso 6.

---

## 6. Configurar Split Tunnel en el perfil del cliente

Las rutas declaradas en el paso 5 no son enrutadas por el cliente hasta que se agreguen explícitamente al perfil de Split Tunnel.

```
Zero Trust → Team & Resources → Devices → Device profiles
  → Default → Configure → Split Tunnels → Manage
```

- Modo recomendado para acceso a redes privadas: **Include IPs and domains**
- Agregar cada CIDR de red privada que deba atravesar el túnel

> **Nota:** Cambios en el perfil pueden tardar hasta 10 minutos en propagarse. Si el cliente no actualiza las rutas inmediatamente, reiniciar el equipo o el servicio de Windows es suficiente para forzar la sincronización.

### Verificar rutas activas en el cliente

```powershell
warp-cli tunnel ip list
# Debe mostrar los CIDRs agregados en Split Tunnels
```

---

## 7. Verificar conectividad a recursos internos

El protocolo ICMP (ping) no siempre atraviesa el túnel de forma confiable. La verificación correcta es por puerto TCP:

```powershell
# Verificar acceso a un servicio específico
Test-NetConnection -ComputerName 10.100.102.10 -Port 22
Test-NetConnection -ComputerName 10.100.102.10 -Port 80
Test-NetConnection -ComputerName 10.100.102.20 -Port 3389
```

Resultado esperado: `TcpTestSucceeded : True`

---

## 8. Auditoría de rendimiento y latencia

### Desde el cliente (tiempo real)

```powershell
# Latencia y pérdida de paquetes hacia el PoP de Cloudflare
warp-cli.exe tunnel stats

# Verificación completa de conectividad
warp-cli.exe debug connectivity-check

# Información del registro activo
warp-cli.exe registration show
```

### Desde el panel (histórico)

```
Zero Trust → DEX → Remote Captures → Run diagnostics
```

Genera un archivo `.zip` con logs de las últimas 96 horas, capturas de red (PCAP) y análisis con IA disponible directamente en el dashboard.

---

## Resumen de rutas configuradas (ejemplo)

|CIDR|Descripción|Tunnel|
|---|---|---|
|`10.100.102.10/32`|Suse Linux VM|tunnel-principal|
|`10.100.102.20/32`|SAP Windows Server|tunnel-principal|
|`192.168.0.0/24`|Red LAN principal (aviadores-network)|tunnel-principal|

---

## Notas operativas

- El cliente WARP instala un adaptador de red virtual. Verificar conflictos con otras VPNs usando `route print` si hay problemas de enrutamiento.
- El tráfico hacia las rutas del Split Tunnel pasa por Cloudflare antes de llegar a `cloudflared` en la red local. El conector actúa como proxy — debe tener conectividad directa hacia los recursos declarados.
- ICMP (ping) puede no funcionar como indicador de conectividad a través del túnel. Usar siempre `Test-NetConnection` con puerto específico para verificar acceso real.
- NetBIOS over TCP/IP está deshabilitado por defecto en versiones recientes del cliente Windows. Habilitarlo desde el perfil de dispositivo si se requiere acceso a recursos SMB.