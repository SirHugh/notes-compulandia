# SAP Business One — Agregar Usuarios IDP en el SLD

## Requisitos previos

- SAP Business One 10.0 FP 2208 o superior
- IAM habilitado en el SLD (Identity and Authentication Management activo)
- Al menos un IDP registrado y **activo** en el SLD Control Center
- Acceso al SLD Control Center con rol **Landscape Administrator** (B1SiteUser)
- El usuario a registrar debe tener cuenta activa en el IDP configurado

---

## Puertos de referencia

|Servicio|Puerto típico|
|---|---|
|SLD Control Center UI|`40000`|
|SLD API / Auth Service|`40020`|
|Service Layer|`50000`|

> ⚠️ Confirmar puertos reales en: SLD Control Center → **System Configuration → SLD**.

---

## Paso 1 — Registrar el usuario en el SLD

1. Acceder al SLD Control Center:
    
    ```
    https://<servidor>:40000/ControlCenter
    ```
    
2. Ir a la pestaña **Users**
3. Clic en **Add**
4. Completar:
    - **IDP**: seleccionar el proveedor de identidad configurado
    - **IDP User**: identificador del usuario en el IDP  
        → Si el `Claim Name` configurado es `upn`: ingresar el UPN (`usuario@dominio.com`)  
        → Si es `email`: ingresar el email del usuario en el IDP

> ⚠️ El valor ingresado debe coincidir **exactamente** con el claim que el IDP envía en el token. Una discrepancia causa fallo de autenticación sin mensaje claro.

---

## Paso 2 — Vincular (bind) el usuario IDP con un usuario SAP B1

Este paso es **obligatorio**. Sin el binding el token OIDC se genera pero el Service Layer devuelve 401/403 porque no puede resolver a qué usuario B1 corresponde.

1. En la pestaña **Users**, seleccionar el usuario registrado
2. Clic en **Bind**
3. Seleccionar:
    - **Company Database**: la empresa SAP B1 correspondiente
    - **B1 User**: el usuario SAP B1 al que se vincula
4. Repetir para cada empresa a la que el usuario necesite acceso

### Lo que devuelve el endpoint de verificación cuando el binding es correcto

```
GET https://<servidor>:40020/sld/sld0100.svc/CurrentUserInfo?IncludeB1UserBinding=true
Authorization: Bearer <token>
```

Respuesta esperada:

```json
{
  "UserName": "usuario@dominio.com",
  "B1UserBindings": [
    {
      "CompanyDB": "SBO_EMPRESA",
      "B1UserName": "manager",
      "UserType": "Normal"
    }
  ]
}
```

---

## Paso 3 — Verificar estado del IDP

Antes de que cualquier usuario pueda autenticarse, el IDP debe estar en estado **Active**.

1. SLD Control Center → pestaña **Identity Providers**
2. Verificar que el IDP tenga estado `Active`
3. Si está `Inactive`: seleccionarlo y clic en **Activate**

> ⚠️ **Advertencia crítica**: al activar un IDP, los usuarios de SAP B1 que no tengan binding configurado **no podrán iniciar sesión**. Configurar todos los bindings antes de activar.

---

## Consideraciones importantes

### Sobre el Claim Name

El campo `Claim Name` definido al registrar el IDP en el SLD determina qué atributo del token OIDC se usa como identificador del usuario:

- AD FS / Entra ID: generalmente `upn`
- SAP IAS: generalmente `email`

El valor del IDP User en el SLD debe coincidir con el valor real de ese claim en el token.

### Sobre licencias

Las licencias asignadas a los usuarios SAP B1 no se modifican al habilitar IAM. El binding vincula identidades, no reasigna licencias.

### Sobre múltiples empresas

Un mismo usuario IDP puede tener binding con usuarios en distintas compañías (company databases). Se debe hacer un binding por cada compañía a la que necesite acceso.

### Sobre usuarios de integración backend

Para integraciones service-to-service (sin usuario humano interactivo), el binding se configura con un usuario B1 dedicado para la integración, no con usuarios nominales. Ese usuario B1 debe tener los permisos suficientes en SAP B1 para las operaciones que la integración ejecutará.

### Gestión centralizada desde el SLD

Desde la pestaña Users del SLD se pueden realizar:

- Resetear contraseñas de usuarios unificados
- Activar / desactivar cuentas de usuario
- Asignar rol de Landscape Administrator

---

## Troubleshooting común

|Síntoma|Causa probable|
|---|---|
|401 al llamar Service Layer con token válido|Falta de binding del usuario IDP con usuario B1|
|404 en `CurrentUserInfo`|Puerto incorrecto — usar `40020`, no `40000`|
|Login redirige pero falla con "user not found"|El `Claim Name` del IDP no coincide con el IDP User registrado en SLD|
|IDP activo pero sin opción de login|Usuario no registrado en la pestaña Users del SLD|