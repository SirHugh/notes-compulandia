
```bash
check-backup-status.sh
#!/bin/bash
# Script de verificación de backup diario

DATE_YESTERDAY=$(date +%Y/%m/%d --date="yesterday")
DATE_TODAY=$(date +%Y-%m-%d)
LOG_FILE="/usr/sap/NDB/home/backup-upload-fast.log"
GSUTIL_WRAPPER="/usr/sap/NDB/home/gsutil-wrapper.sh"

echo "=== Verificación de Backup SAP HANA → GCS ==="
echo "Fecha: $DATE_TODAY"
echo ""

# 1. Verificar última ejecución del script
echo "1. Último backup ejecutado:"
if [ -f "$LOG_FILE" ]; then
    LAST_BACKUP=$(tail -20 "$LOG_FILE" | grep "Upload completado exitosamente" | tail -1)
    if [ ! -z "$LAST_BACKUP" ]; then
        echo "✅ ÉXITO: $LAST_BACKUP"
    else
        echo "❌ ERROR: No se encontró confirmación de backup exitoso reciente"
    fi
else
    echo "❌ ERROR: No existe archivo de log"
fi

echo ""

# 2. Verificar archivos en GCS del día anterior
echo "2. Archivos en GCS (día anterior: $DATE_YESTERDAY):"
GCS_FILES=$("$GSUTIL_WRAPPER" ls gs://sap-backups-prod/$DATE_YESTERDAY/ 2>/dev/null | grep "COMPLETE_DATA_BACKUP" | wc -l)

if [ "$GCS_FILES" -ge 5 ]; then
    echo "✅ ÉXITO: $GCS_FILES archivos encontrados en GCS"
else
    echo "❌ ERROR: Solo $GCS_FILES archivos en GCS (esperado: 7)"
fi

echo ""

# 3. Verificar espacio en bucket
echo "3. Estado del bucket:"
BUCKET_SIZE=$("$GSUTIL_WRAPPER" du -s gs://sap-backups-prod 2>/dev/null | awk '{print $1}')
if [ ! -z "$BUCKET_SIZE" ]; then
    echo "✅ Tamaño total del bucket: $BUCKET_SIZE bytes"
else
    echo "❌ ERROR: No se pudo verificar tamaño del bucket"
fi

echo ""
echo "=== Verificación completada ==="
```