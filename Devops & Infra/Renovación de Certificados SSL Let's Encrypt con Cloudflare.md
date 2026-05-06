## Contexto

Servidor Apache con certificado Let's Encrypt detrás de proxy Cloudflare.

**Archivos de configuración:**

- Certificado: `/etc/letsencrypt/live/compulandia.com.py/fullchain.pem`
- Clave privada: `/etc/letsencrypt/live/compulandia.com.py/privkey.pem`
- Config renovación: `/etc/letsencrypt/renewal/compulandia.com.py.conf`

---

## Errores comunes y soluciones

### Error 403 - WAF bloqueando el challenge

```
Invalid response from http://.../.well-known/acme-challenge/...: 403
```

**Causa:** Reglas de Cloudflare (WAF, Bot Fight Mode, reglas custom) bloquean el bot de Let's Encrypt.

**Solución:**

1. Cloudflare → Security → WAF → Custom rules
2. Crear regla "Allow ACME Challenge":
    - Expression: `(http.request.uri.path contains "/.well-known/acme-challenge/")`
    - Action: Skip → seleccionar todas las opciones
3. **Importante:** Esta regla debe estar ARRIBA de otras reglas como "Challenge Non-Regional Traffic"

**obs:** ya existe la regla, esta desactivada. Se debe activar y subir al primer lugar entre las reglas. 

---

### Error 526 - Certificado SSL inválido en origen

```
Invalid response from https://.../.well-known/acme-challenge/...: 526
```

**Causa:** Cloudflare redirige HTTP a HTTPS, pero el certificado del origen está vencido.

**Solución permanente (Page Rule):**

1. Cloudflare → Rules → Page Rules → Create Page Rule
2. URL: `*compulandia.com.py/.well-known/acme-challenge/*`
3. Settings:
    - SSL: Off
    - Automatic HTTPS Rewrites: Off

**Solución temporal:**

1. SSL/TLS → Edge Certificates → Desactivar "Always Use HTTPS"
2. Renovar certificado
3. Volver a activar "Always Use HTTPS"

---

### Error Rate Limit

```
too many failed authorizations (5) ... retry after [FECHA_UTC]
```

**Causa:** Demasiados intentos fallidos en poco tiempo.

**Solución:** Esperar hasta la fecha/hora indicada (convertir de UTC a hora local, Paraguay = UTC-3).

---

## Comandos útiles

```bash
# Probar renovación sin ejecutarla
sudo certbot renew --dry-run

# Renovar certificado
sudo certbot renew

# Recargar Apache después de renovar
sudo systemctl reload apache2

# Ver estado del timer de renovación automática
sudo systemctl list-timers certbot.timer

# Ver certificados instalados y fechas de vencimiento
sudo certbot certificates

# Ver logs de certbot
sudo cat /var/log/letsencrypt/letsencrypt.log
```

---

## Renovación automática

- Certbot se ejecuta **2 veces al día** vía systemd timer
- Los certificados duran **90 días**
- Certbot renueva cuando quedan **30 días o menos**

---

## Checklist preventivo

- [ ] Regla WAF "Allow ACME Challenge" creada y en primer lugar
- [ ] Page Rule para SSL Off en `/.well-known/acme-challenge/*`
- [ ] Timer de certbot activo: `systemctl is-active certbot.timer`

---

## Configuración Cloudflare requerida

### Orden de reglas WAF (de arriba a abajo):

1. Allow ACME Challenge ← **debe estar primero**
2. Challenge Non-Regional Traffic
3. (otras reglas)

### Page Rule activa:

- URL: `*compulandia.com.py/.well-known/acme-challenge/*`
- SSL: Off