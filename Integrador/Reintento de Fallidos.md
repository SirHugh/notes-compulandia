# Sistema de Gestión de Sincronizaciones Fallidas [[RetryJobsCommand.canvas|RetryJobsCommand]]

## Problema

El sistema acumula sincronizaciones fallidas (~4,600 actualmente) que no se procesan de forma automatizada. Los reintentos manuales pueden inundar las colas y no distinguen entre errores recuperables y no recuperables.

## Análisis de Errores Actuales

### Distribución por Canal

| Canal          | Fallidos | % del Total |
| -------------- | -------- | ----------- |
| Tienda Naranja | 2,952    | 63%         |
| Contimarket    | 1,156    | 25%         |
| WooCommerce    | 307      | 7%          |
| Medusa         | 244      | 5%          |
  
### Clasificación de Errores

#### Retryable (~50% - 2,310 jobs)

Errores temporales que se pueden resolver con reintentos:

- **SIN IMAGEN** (1,168): Producto sin imagen, reintentar cuando tenga

- **DESINCRONIZADO/409** (613): Producto existe en destino, reintentar sincroniza

- **Connection refused/timeout** (~350): Errores de red temporales

- **Too many attempts** (147): Jobs que fallaron por reintentos agotados

- **Stock reservado** (34): Condición temporal

  

#### No Retryable (~35% - 1,612 jobs)

Errores que requieren intervención manual o corrección de datos:
- **SIN CATEGORIA** (1,540): Requiere mapeo de categoría en el sistema

- **No autorizado** (60): Problema de permisos en plataforma destino

- **Nombre duplicado** (10): Conflicto de nombre en TiendaNaranja

- **SKU duplicado** (2): Conflicto de SKU en WooCommerce

- **Client error 4xx** (7): Errores de validación de datos

  

#### Requiere Investigación (~15% - 737 jobs)

Errores que necesitan análisis caso por caso:

- **Producto no encontrado** (271): ¿Eliminado en destino?

- **ID no válido** (291): ¿Producto eliminado en WooCommerce?

- **FAILED genérico** (142): Sin mensaje descriptivo
## Solución Propuesta

### Componentes


```

app/

├── Enums/

│   └── SyncErrorCategory.php          # Categorías de error

├── Services/

│   └── Sync/

│       └── FailedSyncClassifier.php   # Clasificador de errores

├── Console/Commands/

│   └── ProcessFailedSyncsCommand.php  # Comando principal

└── Models/

    └── ProductSyncLog.php             # (modificar) agregar scopes

```
### 1. Enum de Categorías
```php

enum SyncErrorCategory: string

{

    case MISSING_IMAGE = 'missing_image';

    case MISSING_CATEGORY = 'missing_category';

    case NETWORK_ERROR = 'network_error';

    case DESYNC = 'desync';

    case NOT_FOUND = 'not_found';

    case PERMISSION = 'permission';

    case DUPLICATE = 'duplicate';

    case RATE_LIMIT = 'rate_limit';

    case UNKNOWN = 'unknown';

  

    public function isRetryable(): bool

    {

        return match($this) {

            self::MISSING_IMAGE,

            self::NETWORK_ERROR,

            self::DESYNC,

            self::RATE_LIMIT => true,

            default => false,

        };

    }

}

```

### 2. Clasificador de Errores

Servicio que analiza `response_message` y `response_code` para determinar la categoría:

```php

class FailedSyncClassifier

{

    public function classify(ProductSyncLog $log): SyncErrorCategory

    {

        $message = $log->response_message ?? '';

        $code = $log->response_code;

  

        // Patrones de clasificación por mensaje

        return match(true) {

            str_contains($message, 'SIN IMAGEN') => SyncErrorCategory::MISSING_IMAGE,

            str_contains($message, 'SIN CATEGORIA') => SyncErrorCategory::MISSING_CATEGORY,

            str_contains($message, 'DESINCRONIZADO') => SyncErrorCategory::DESYNC,

            str_contains($message, 'Connection refused') => SyncErrorCategory::NETWORK_ERROR,

            // ... más patrones

            default => SyncErrorCategory::UNKNOWN,

        };

    }

}

```
### 3. Comando de Procesamiento

```bash

php artisan sync:process-failed [opciones]

```

**Opciones:**

- `--channel=` : Filtrar por canal (tn, contimarket, woocommerce, medusa)

- `--limit=20` : Jobs a procesar por ejecución

- `--category=` : Filtrar por categoría de error

- `--retry-only` : Solo procesar retryables

- `--archive-non-retryable` : Marcar no-retryables como archivados

- `--dry-run` : Mostrar qué haría sin ejecutar

- `--report` : Generar reporte sin procesar

**Comportamiento:**

1. Consulta jobs fallidos (más antiguos primero)

2. Clasifica cada error

3. Filtra según opciones

4. Despacha jobs retryables con delay entre cada uno

5. Marca no-retryables como archivados (opcional)

6. Genera reporte de ejecución

### 4. Migración (Opcional)

```php

Schema::table('product_sync_logs', function (Blueprint $table) {

    $table->string('error_category', 30)->nullable()->after('response_message');

    $table->unsignedTinyInteger('retry_count')->default(0)->after('error_category');

    $table->timestamp('archived_at')->nullable()->after('retry_count');

  

    $table->index(['status', 'error_category']);

    $table->index(['status', 'archived_at']);

});

```

## Configuración de Ejecución

### Schedule (routes/console.php)

```php

// Procesar fallidos retryables cada 10 minutos

Schedule::command('sync:process-failed --retry-only --limit=15')

    ->everyTenMinutes()

    ->withoutOverlapping()

    ->runInBackground();

  

// Reporte diario de estado

Schedule::command('sync:process-failed --report')

    ->dailyAt('08:00')

    ->emailOutputTo('admin@compulandia.com.py');

```

### Parámetros Recomendados

| Parámetro | Valor | Justificación |
|-----------|-------|---------------|
| Batch size | 15-20 | Evita inundar colas |
| Intervalo | 10 min | Procesamiento gradual |
| Max reintentos | 3 | Evita loops infinitos |
| Delay entre jobs | 2 seg | Respeta rate limits |
## Reportes

### Reporte de Ejecución

```

=== Sync Failed Jobs Report ===

Fecha: 2026-01-05 10:30:00

  

Procesados: 20

  - Reintentados: 12

  - Archivados: 5

  - Omitidos: 3

  

Por Categoría:

  - missing_image: 8 (reintentados)

  - network_error: 4 (reintentados)

  - missing_category: 5 (archivados)

  - unknown: 3 (omitidos)

  

Pendientes:

  - Tienda Naranja: 2,940

  - Contimarket: 1,144

  - WooCommerce: 307

  - Medusa: 244

```

### Reporte de Estado Global

```

=== Failed Syncs Status ===

  

Total Fallidos: 4,635

  

Por Categoría:

  - Retryable: 2,310 (50%)

  - No Retryable: 1,612 (35%)

  - Investigar: 713 (15%)

  

Tendencia (últimos 7 días):

  - Nuevos fallidos: +245

  - Resueltos: -180

  - Neto: +65

```

## Flujo de Procesamiento

  

```

┌─────────────────┐

│  Cron (10 min)  │

└────────┬────────┘

         │

         ▼

┌─────────────────────┐

│ ProcessFailedSyncs  │

│   --retry-only      │

│   --limit=15        │

└────────┬────────────┘

         │

         ▼

┌─────────────────────┐

│  Query failed logs  │

│  (oldest first)     │

└────────┬────────────┘

         │

         ▼

┌─────────────────────┐

│  Classify each      │◄─── FailedSyncClassifier

│  error              │

└────────┬────────────┘

         │

         ▼

    ┌────┴────┐

    │ Retryable? │

    └────┬────┘

    yes  │  no

    ▼    │    ▼

┌───────┐│┌──────────┐

│Dispatch││ Archive/ │

│ Job   │││ Skip     │

└───┬───┘│└────┬─────┘

    │    │     │

    ▼    ▼     ▼

┌─────────────────────┐

│  Update log status  │

│  + retry_count      │

└────────┬────────────┘

         │

         ▼

┌─────────────────────┐

│  Generate report    │

└─────────────────────┘

```
 
## Consideraciones
 ### Rate Limiting

- Delay de 2 segundos entre dispatches

- Máximo 15-20 jobs por ejecución

- withoutOverlapping() previene ejecuciones paralelas

### Priorización

- Jobs más antiguos primero (FIFO)

- Opción de priorizar por canal si es necesario

### Monitoreo

- Logs en canal dedicado `SYNC_RETRY_MANAGER`

- Métricas: procesados, reintentados, archivados

- Alertas si pendientes superan umbral

### Limpieza

- Archivados se pueden purgar después de 30 días

- Comando separado `sync:cleanup-archived --older-than=30`