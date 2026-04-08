# SAP B1 — Registrar Cliente OIDC en Keycloak

**Admin Console:** `https://<servidor>:40020/auth/admin/master/console/`

> ⚠️ Siempre operar en el realm **`sapb1`**, no en `master`.

---

## Pasos

**1. Crear cliente** `Clients → Create client`

- Client type: `OpenID Connect`
- Client ID: nombre único sin espacios 



**2. Capacidades**

|Opción|Valor|
|---|---|
|Client authentication|**ON**|
|Standard flow|OFF|
|Direct access grants|OFF|
|Service accounts roles|**ON**|

**3. Redirect URIs** En la sección **Login settings**, agregar los endpoints de callback de la aplicación:

- Valid redirect URIs: `https://<app>/callback`
- Valid post logout redirect URIs: `https://<app>/logout`
- Web origins: `https://<app>` (o `+` para permitir todos los redirect URIs registrados)

> ⚠️ Sin al menos un redirect URI configurado Keycloak rechaza el registro del cliente.

**4. Obtener credenciales** Pestaña **Credentials**:

- Copiar **Client ID** y **Client secret**
- El secret se puede regenerar con **Regenerate** — invalida el anterior de inmediato

**5. Asignar roles de Service Account** `Pestaña Service Account Roles → Assign role`

- Filtrar por cliente `b1-B1ServiceLayers-...`
- Asignar `B1ServiceLayer.Full` (o el rol disponible equivalente)

**6. Audience Mapper** _(solo si el Service Layer sigue devolviendo 401 tras asignar roles)_ `Client Scopes → <cliente>-dedicated → Add mapper → Audience`

- Included Client Audience: `b1-B1ServiceLayers-<XXXX>-main-<empresa>`
- Add to access token: ON

---

## Token endpoint de referencia

```
POST https://<servidor>:40020/auth/realms/sapb1/protocol/openid-connect/token

grant_type=client_credentials
&client_id=<client_id>
&client_secret=<client_secret>
```

---

## Errores comunes

|Error|Causa|
|---|---|
|`400 invalid_client`|client_id / secret incorrectos, o Client authentication OFF|
|`400 unauthorized_client`|Service accounts roles no habilitado|
|`401` en Service Layer|Roles de service account no asignados o falta audience mapper|
|`404` en SLD API|Usar puerto `40020`, no `40000`|