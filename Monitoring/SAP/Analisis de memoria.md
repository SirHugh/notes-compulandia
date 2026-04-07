# SAP HANA — Análisis de Memoria

**Fecha:** 2026-03-26 | **Servidor:** sap-linux

---

## Configuración actual

|Parámetro|Valor|
|---|---|
|RAM total servidor|112 GB|
|`global_allocation_limit` (DATABASE layer)|65536 MB (64 GB) — ignorado|
|Límite efectivo calculado por HANA|~104.8 GB (automático)|

> HANA calcula el límite automáticamente cuando no existe configuración en SYSTEM layer. El valor en DATABASE layer no tiene efecto en este caso.

---

## Estado de memoria por servicio

|Servicio|Puerto|Límite|Usado|% Usado|
|---|---|---|---|---|
|indexserver|30003|51.7 GB|29.3 GB|57%|
|nameserver|30001|29.7 GB|7.6 GB|26%|
|xsengine|30007|28.0 GB|5.6 GB|20%|
|scriptserver|30004|25.8 GB|3.3 GB|13%|
|webdispatcher|30006|24.6 GB|2.2 GB|9%|
|diserver|30025|24.4 GB|2.0 GB|8%|
|preprocessor|30002|23.0 GB|0.6 GB|3%|
|compileserver|30010|22.8 GB|0.4 GB|2%|
|**Total**|||**~51 GB**||

---

## Conclusiones

- ✅ **Sin presión de memoria** — 51 GB usados sobre 112 GB disponibles
- ✅ **indexserver saludable** — 57% de su límite, 7.9 GB libres
- ✅ **94% en Grafana es normal** — HANA pre-reserva un pool grande al iniciar; el OS lo ve como "usado" aunque HANA lo reporte internamente como disponible
- ✅ **global_allocation_limit no requiere ajuste** — HANA gestiona el límite automáticamente de forma correcta

---

## ✅ Aplicado — Expensive Statement Tracing

```sql
ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM')
SET ('expensive_statement', 'enable') = 'true'
WITH RECONFIGURE;

ALTER SYSTEM ALTER CONFIGURATION ('global.ini', 'SYSTEM')
SET ('expensive_statement', 'threshold_duration') = '5000000'
WITH RECONFIGURE;
```

|Parámetro|Valor|
|---|---|
|`enable`|`true`|
|`threshold_duration`|`5000000` µs = 5 segundos|

> Queries que superen 5 segundos quedarán registradas en `M_EXPENSIVE_STATEMENTS`.

**Query de consulta:**

```sql
SELECT TOP 10
    START_TIME,
    DURATION_MICROSEC/1000000          AS DURACION_SEG,
    CPU_TIME/1000000                   AS CPU_SEG,
    MEMORY_SIZE/1048576                AS MEM_MB,
    SUBSTRING(STATEMENT_STRING, 1, 120) AS QUERY
FROM M_EXPENSIVE_STATEMENTS
ORDER BY DURATION_MICROSEC DESC;
```

---

## Pendiente — Nivel SAP HANA

- [ ] Ejecutar Mini Checks (SAP Note 1969700)
- [ ] Revisar estado de Delta Merge