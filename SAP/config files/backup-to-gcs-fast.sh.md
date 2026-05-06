
``` bash
backup-to-gcs-fast.sh
#!/bin/bash
# Upload optimizado para velocidad

BACKUP_SOURCE="/hana/shared/NDB/HDB00/backup/data"
GCS_BUCKET="gs://sap-backups-prod"
DATE=$(date +%Y%m%d)
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
LOG_FILE="/usr/sap/NDB/home/backup-upload-fast.log"
GSUTIL_WRAPPER="/usr/sap/NDB/home/gsutil-wrapper.sh"

log() {
    echo "[$TIMESTAMP] $1" | tee -a "$LOG_FILE"
}

log "=== Upload OPTIMIZADO de backup HANA ==="

GCS_PATH="$GCS_BUCKET/$(date +%Y/%m/%d)"

# Upload SIN throttling y CON paralelismo máximo
log "Iniciando upload rápido..."
"$GSUTIL_WRAPPER" -m -o "GSUtil:parallel_thread_count=4" cp \
  "$BACKUP_SOURCE"/*/COMPLETE_DATA_BACKUP_${DATE}_* \
  "$GCS_PATH/" >> "$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
    log "Upload completado exitosamente"
else
    log "ERROR: Fallo en upload"
fi
```
