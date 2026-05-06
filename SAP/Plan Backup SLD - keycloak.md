# Plan de Implementación — Backup y Restauración de SLD / Keycloak

**Entorno:** SAP Business One 10.0 sobre HANA, SUSE Linux Enterprise Server **Alcance:** Configuración y datos del SLD y del Authentication Service (Keycloak)

---

## 1. Contexto

El stack de autenticación y paisaje de SAP B1 se compone de:

- **SLD (System Landscape Directory):** registro de servidores, bases de datos, servicios y mappings. Datos en schema HANA `SLDDATA`, configs en `/usr/sap/SAPBusinessOne/ServerTools/SLD/conf/`.
- **Keycloak (Authentication Service):** IdP que gestiona usuarios, realms, clients y credenciales. Datos en schema HANA `B1AS` y schemas auxiliares `SBSS_*`. Configs en `/usr/sap/SAPBusinessOne/Common/keycloak/conf/`.

La pérdida de cualquiera de estos componentes impide el arranque del landscape y el login de usuarios al cliente SAP B1.

---

## 2. Estado actual del backup

|Componente|Ubicación|¿Cubierto por backup HANA diario?|
|---|---|---|
|Schema `SLDDATA`|HANA|✅ Sí|
|Schema `B1AS`|HANA|✅ Sí|
|Schemas `SBSS_*` (secure storage)|HANA|✅ Sí|
|Schema `SBOCOMMON`|HANA|✅ Sí|
|Schemas de negocio (`COMPU`, `ZZ_COMPU*`, `B1_*`)|HANA|✅ Sí|
|Configs SLD (XMLs)|Filesystem|❌ No|
|Configs Keycloak (`env.conf`, keystore, XMLs)|Filesystem|❌ No|
|Certificados PKCS12 / JKS|Filesystem|❌ No|
|Shared folder del SLD|Filesystem|❌ No|
|Licencias|Filesystem|❌ No|

**Conclusión:** los datos están cubiertos; falta implementar backup de filesystem para los archivos de configuración y certificados.

---

## 3. Componentes a respaldar (filesystem)

### 3.1 Críticos — irrecuperables sin backup

|Archivo / Directorio|Contenido|
|---|---|
|`/usr/sap/SAPBusinessOne/Common/keycloak/conf/env.conf`|Credenciales cifradas de HANA y keystore|
|`/usr/sap/SAPBusinessOne/Common/keycloak/conf/keycloak.jks`|Keystore Java con certificado HTTPS|
|`/usr/sap/SAPBusinessOne/Common/keycloak/conf/sapb1.conf`|Config específica SAP para Keycloak|
|`/usr/sap/SAPBusinessOne/Common/keycloak/conf/hana-cache.xml`|Config del cache distribuido|
|`/usr/sap/SAPBusinessOne/Common/keycloak/conf/cache-ispn.xml`|Config Infinispan|
|`/usr/sap/SAPBusinessOne/Common/keycloak/conf/truststores/`|Truststores del servicio|
|`/usr/sap/SAPBusinessOne/ServerTools/SLD/conf/sld.xml`|Configuración principal del SLD|
|`/usr/sap/SAPBusinessOne/ServerTools/SLD/conf/ControlCenter.xml`|Configuración del Control Center|
|`/usr/sap/SAPBusinessOne/ServerTools/License/`|Archivos de licencia|
|`/etc/systemd/system/sapb1*.service`|Unit files de systemd|

### 3.2 No requieren backup

- `providers/`, `themes/`, `lib/`, `bin/` → regenerables vía reinstalación.
- `data/log`, `data/tmp`, `data/transaction-logs` → logs y temporales.

---

## 4. Estrategia de backup en tres capas

### Capa 1 — Datos (HANA)

- **Ya implementado.** Backup diario existente cubre `SLDDATA`, `B1AS`, `SBSS_*` y `SBOCOMMON`.
- Validar periódicamente que el job incluya estos schemas y que la retención sea adecuada (mínimo recomendado: 30 días).

### Capa 2 — Configuración (filesystem)

- **A implementar.** Job diario que empaquete los archivos listados en la sección 3.1.
- Ventana: coordinada con el backup HANA, idealmente ejecutarse antes para mantener coherencia temporal.
- Retención recomendada: 30 días en local + copia remota.
- Destino: disco separado del server de producción o storage de red; nunca en el mismo filesystem.

### Capa 3 — Export del realm Keycloak (opcional, semanal)

- Export en formato JSON vía `kc.sh export` del realm `SBOCommon`.
- Ventaja: formato portable y versionable; útil para migración a hardware nuevo.
- Limitación: requiere detener el servicio de autenticación (~30-60s).
- Consideración: el `ClientSecret` del Service Layer se exporta enmascarado — debe registrarse por separado.

---

## 5. Consideraciones de sistema

### 5.1 Permisos y propietario

- Los archivos de Keycloak pertenecen al usuario `b1service0` (grupo `b1service0`).
- `env.conf` tiene permisos `400` (solo lectura para el owner) por contener credenciales.
- El backup debe ejecutarse como `root` para garantizar acceso de lectura a todos los archivos sin alterar permisos.
- Los archivos de backup generados deben tener permisos `600` (contienen credenciales cifradas).

### 5.2 Credenciales cifradas y portabilidad

- Las passwords en `env.conf` están ofuscadas por `SLDInstallerTool.jar` de SAP.
- **Riesgo en DR:** al restaurar en hardware nuevo, las credenciales protegidas pueden no descifrarse correctamente si el binding con el host original se pierde.
- **Mitigación:** documentar en gestor de secretos (fuera del server) las credenciales en claro de:
    - Usuario HANA usado por Keycloak
    - Password del keystore `keycloak.jks`
    - `ClientID` y `ClientSecret` del Service Layer
    - Password de `B1SiteUser`

### 5.3 Secuencia de dependencias en arranque

Al restaurar un server desde cero, el orden de inicio de servicios debe ser:

1. SAP HANA (`sapinit.service`)
2. `sapb1servertools-authentication.service` (Keycloak)
3. Servicio SLD
4. Service Layer, Web Client, resto de componentes

### 5.4 Coherencia entre backup de datos y backup de config

- El backup de HANA y el de filesystem deben estar temporalmente cercanos. Un desfase mayor a 24h entre ambos puede producir inconsistencias tras una restauración (p. ej. una credencial rotada en HANA pero el `env.conf` apuntando al valor viejo).
- Recomendación: ambos jobs en la misma ventana nocturna, filesystem antes que HANA.

### 5.5 Versiones

- El backup de filesystem debe re-capturarse tras cada **upgrade de patch level** de SAP B1, ya que los archivos en `conf/` pueden cambiar de formato.
- Conservar al menos un backup pre-upgrade y uno post-upgrade de forma permanente.

---

## 6. Procedimiento de restauración

### 6.1 Escenario A — Corrupción de configuración (mismo server)

1. Detener servicios: authentication, SLD.
2. Restaurar los archivos del último backup de filesystem.
3. Verificar permisos (`b1service0:b1service0`, `env.conf` en `400`).
4. Reiniciar servicios en el orden definido en 5.3.

### 6.2 Escenario B — Pérdida de datos (schemas corruptos)

1. Detener servicios de autenticación y SLD.
2. Restaurar schemas afectados desde backup HANA.
3. Reiniciar servicios.

### 6.3 Escenario C — Disaster recovery (server nuevo)

1. Instalar SUSE + HANA + ServerTools SAP B1 con **misma versión y patch exactos**.
2. Restaurar backup HANA completo (todos los schemas).
3. Restaurar backup de filesystem sobre las rutas originales.
4. Si las credenciales cifradas fallan, reconfigurarlas usando las credenciales en claro del gestor de secretos.
5. Regenerar Client Secrets afectados y actualizar integraciones externas (Service Layer, tienda web).
6. Validar login con `B1SiteUser` y con usuarios de compañía.

---

## 7. Validación del plan

Antes de considerar el plan operativo:

- [ ] Ejecutar restauración de prueba en ambiente no productivo.
- [ ] Verificar login al SLD Control Center tras la restauración.
- [ ] Verificar login al cliente SAP B1 con usuario de compañía.
- [ ] Verificar conectividad del Service Layer (Client Secret intacto).
- [ ] Documentar el tiempo real de restauración para definir el RTO.
- [ ] Programar revisión semestral del procedimiento.

---

## 8. Próximos pasos

1. Implementar script de backup de filesystem (Capa 2) y programar en cron.
2. Configurar copia remota de los archivos de backup.
3. Cargar credenciales críticas en gestor de secretos corporativo.
4. Ejecutar una restauración de prueba completa en ambiente de test.
5. Documentar RTO/RPO resultantes y comunicar a stakeholders.