# 🎯 Backups Pendientes para Disaster Recovery

**Versión**: 1.0 **Fecha**: 06/05/2026 **Estado**: Pendiente de implementación

---

## 🎯 Propósito

Listado específico de backups que faltan implementar para poder restaurar SAP completamente en caso de desastre. Solo HANA no es suficiente.

---

## ✅ Lo que YA tenemos

- Backups de HANA (SYSTEMDB, DB_NDB, DB_COCKPITDB) en GCS

## ❌ Lo que FALTA implementar

---

## 📦 1. SAP Business One

### Qué respaldar:

- Archivos de **licencia SAP B1**
- **Add-ons** instalados
- **Customizaciones**: queries, reportes custom, layouts de impresión
- **Print layouts** personalizados
- Configuración del **SBO Mailer**
- Configuración de **impresoras**

### Por qué es crítico:

Sin esto, aunque restaures HANA, SAP B1 no funciona o pierde toda la personalización del cliente.

### Frecuencia: Diaria

### Retención GCS: 30 días

### Tamaño estimado: 100-500 MB/día

---

## 📦 2. Service Layer

### Qué respaldar:

- **Archivos de configuración** del Service Layer
- **Certificados SSL** (HTTPS)
- **Usuarios B1** mapeados para integración
- Configuración de **routing** y endpoints

### Por qué es crítico:

Sin Service Layer, las integraciones (CRM, OMS, apps) no pueden conectarse a SAP.

### Frecuencia: Diaria (configs) / Mensual (certificados)

### Retención GCS: 30 días

### Tamaño estimado: 50-200 MB/día

---

## 📦 3. Sistema Operativo y Configuración del Servidor

### Qué respaldar:

- **Cron jobs** de usuarios críticos (`crontab -l`)
- **Configuración de red**: IP, DNS, hosts
- **Reglas de firewall** (UFW/iptables)
- **Service accounts** y JSONs (cifrados)
- **Scripts custom** (`/usr/sap/NDB/home/*.sh`)
- **Parámetros de tunning HANA** a nivel OS

### Por qué es crítico:

Reconstruir esto manualmente lleva horas/días. Tener backup acelera DR significativamente.

### Frecuencia: Diaria (cron, scripts) / Semanal (red, firewall)

### Retención GCS: 60 días

### Tamaño estimado: 10-50 MB/día

---

## 📦 4. Aplicaciones e Integraciones

### Qué respaldar:

- **CRM/OMS Backend** (NestJS): config y variables de entorno
- **CRM/OMS Frontend** (Next.js): configuración
- **Cloudflare**: configuración de tunnels y DNS
- **Integraciones con partners**: CLX, MC Group, AEX
- **APIs externas**: configuración y credenciales
- **Webhooks** configurados

### Por qué es crítico:

Estas son las aplicaciones que el negocio usa día a día. Sin ellas, SAP funciona pero los procesos de negocio no.

### Frecuencia: Diaria (configs) / Mensual (integraciones)

### Retención GCS: 30 días

### Tamaño estimado: 50-200 MB/día

---

## 📊 Resumen de Volumen

|Categoría|Tamaño/día|Retención|Total GB|
|---|---|---|---|
|SAP B1|0.5 GB|30 días|15 GB|
|Service Layer|0.2 GB|30 días|6 GB|
|OS y Config|0.05 GB|60 días|3 GB|
|Apps e Integraciones|0.2 GB|30 días|6 GB|
|**TOTAL ADICIONAL**|**~1 GB**||**~30 GB**|

**Costo adicional estimado**: ~$1-2/mes

---

## 🗂️ Estructura GCS Propuesta

```
gs://sap-backups-prod/
├── YYYY/MM/DD/                ← HANA (ya implementado)
├── logs/                      ← HANA logs (ya implementado)
│
├── b1-config/YYYY/MM/DD/      ← PENDIENTE
├── service-layer/YYYY/MM/DD/  ← PENDIENTE
├── os-config/YYYY/MM/DD/      ← PENDIENTE
└── integrations/YYYY/MM/DD/   ← PENDIENTE
```

---

## 📋 Plan de Implementación

### Orden sugerido (por criticidad):

|Prioridad|Categoría|Razón|
|---|---|---|
|🔴 Alta|SAP B1|Sin esto SAP no opera correctamente|
|🔴 Alta|Service Layer|Sin esto las integraciones se rompen|
|🟡 Media|Aplicaciones|Importante pero más recuperable|
|🟢 Baja|OS y Config|Útil pero reconstruible|

### Para cada categoría, crear:

1. **Script de backup** (siguiendo patrón de los existentes)
2. **Subida automática a GCS** (usando gsutil-wrapper)
3. **Métricas Prometheus** (.prom file)
4. **Lifecycle policy** en GCS (retención)
5. **Integración con Grafana** (dashboard)
6. **Alertas** si falla

---

## 🎯 Resultado Esperado

Cuando esté completo:

✅ Restore total de SAP posible desde GCS ✅ Sin dependencia de tener acceso al servidor original ✅ Tiempo de recovery reducido (procedimientos claros) ✅ Todos los componentes críticos protegidos

---

## 🔗 Referencias

- [DISASTER-RECOVERY-PLAN.md](https://claude.ai/chat/DISASTER-RECOVERY-PLAN.md) - Plan DR completo
- [BACKUP-MONITORING-ACTIVITIES.md](https://claude.ai/chat/BACKUP-MONITORING-ACTIVITIES.md) - Actividades de seguimiento

---

## 📜 Historial

|Versión|Fecha|Cambios|
|---|---|---|
|1.0|06/05/2026|Versión inicial|
