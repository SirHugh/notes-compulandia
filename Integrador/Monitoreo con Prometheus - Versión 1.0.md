# Implementación de Monitoreo con Prometheus - Versión 1.0

**Fecha**: 2025-01-21

**Versión**: 1.0

**Estado**: Propuesta Técnica

---

## 1. Resumen Ejecutivo

Este documento describe la implementación inicial de monitoreo de métricas del sistema Compulandia Integrador utilizando Prometheus. La versión 1 se enfoca en:

- Métricas de estado de sincronizaciones con proveedores

- Métricas de colas y Horizon

- Métricas de inventario y productos

- Configuración de seguridad para el endpoint de scraping


---
## 2. Stack Tecnológico

### 2.1 Paquete Seleccionado

**spatie/laravel-prometheus** v1.x

| Característica    | Detalle                                      |
| ----------------- | -------------------------------------------- |
| Repositorio       | https://github.com/spatie/laravel-prometheus |
| Licencia          | MIT                                          |
| Compatibilidad    | Laravel 10+ / PHP 8.1+                       |
| Métricas Horizon  | Incluidas                                    |
| Métricas de Colas | Incluidas                                    |

### 2.2 Justificación

- Soporte nativo para Laravel Horizon (ya utilizado en el proyecto)

- API declarativa simple para métricas personalizadas

- Mantenimiento activo por Spatie

- Configuración de seguridad integrada

---

## 3. Endpoint de Scraping

  

### 3.1 Configuración

  

| Parámetro                | Valor                                     |
| ------------------------ | ----------------------------------------- |
| **URL**                  | `/metrics/prometheus`                     |
| **Método HTTP**          | GET                                       |
| **Formato de respuesta** | text/plain (Prometheus exposition format) |
| **Puerto**               | 80/443 (mismo que la aplicación)          |

### 3.2 Ejemplo de Scrape Config (prometheus.yml)
  

```yaml

scrape_configs:

  - job_name: 'integrador'

    scrape_interval: 30s

    scrape_timeout: 10s

    metrics_path: '/metrics/prometheus'

    scheme: https

    static_configs:

      - targets: ['integrador.compulandia.com.py']

    basic_auth:

      username: 'prometheus'

      password_file: '/etc/prometheus/integrador_password'

```
---
## 4. Autenticación y Seguridad

### 4.1 Método de Autenticación

**Autenticación por IP + Token Bearer**

Se implementará una capa de seguridad en dos niveles:

| Nivel | Método              | Descripción                                      |
| ----- | ------------------- | ------------------------------------------------ |
| 1     | Lista blanca de IPs | Solo IPs del servidor Prometheus pueden acceder  |
| 2     | Token Bearer        | Header `Authorization: Bearer <token>` requerido |
### 4.2 Configuración de Seguridad

  

```php

// config/prometheus.php

  

return [

    'enabled' => env('PROMETHEUS_ENABLED', true),

  

    'urls' => [

        'default' => 'metrics/prometheus',

    ],

  

    // Nivel 1: Lista blanca de IPs

    'allowed_ips' => array_filter([

        env('PROMETHEUS_ALLOWED_IP_1'),

        env('PROMETHEUS_ALLOWED_IP_2'),

        // IPs adicionales según sea necesario

    ]),

  

    'middleware' => [

        \Spatie\Prometheus\Http\Middleware\AllowIps::class,

        \App\Http\Middleware\PrometheusTokenAuth::class, // Nivel 2

    ],

  

    'default_namespace' => 'integrador',

];

```

  

### 4.3 Middleware de Token (Nivel 2)

  

```php

// app/Http/Middleware/PrometheusTokenAuth.php

  

<?php

  

namespace App\Http\Middleware;

  

use Closure;

use Illuminate\Http\Request;

use Symfony\Component\HttpFoundation\Response;

  

class PrometheusTokenAuth

{

    public function handle(Request $request, Closure $next): Response

    {

        $expectedToken = config('prometheus.bearer_token');

  

        if (empty($expectedToken)) {

            // Si no hay token configurado, permitir (solo aplica IP whitelist)

            return $next($request);

        }

  

        $providedToken = $request->bearerToken();

  

        if ($providedToken !== $expectedToken) {

            return response('Unauthorized', 401);

        }

  

        return $next($request);

    }

}

```

  

### 4.4 Variables de Entorno

  

```env

# .env

  

PROMETHEUS_ENABLED=true

PROMETHEUS_ALLOWED_IP_1=10.0.0.50        # IP del servidor Prometheus

PROMETHEUS_ALLOWED_IP_2=                  # IP adicional (opcional)

PROMETHEUS_BEARER_TOKEN=your-secure-token-here

```

  

---

  

## 5. Métricas a Observar

  

### 5.1 Métricas de Horizon (Incluidas Automáticamente)

  

El paquete incluye 7 collectors de Horizon:

  

| Métrica                                   | Tipo  | Descripción                                          |
| ----------------------------------------- | ----- | ---------------------------------------------------- |
| `integrador_horizon_status`               | Gauge | Estado de Horizon (-1=inactivo, 0=pausado, 1=activo) |
| `integrador_horizon_jobs_per_minute`      | Gauge | Jobs procesados por minuto                           |
| `integrador_horizon_failed_jobs_per_hour` | Gauge | Jobs fallidos por hora                               |
| `integrador_horizon_current_workload`     | Gauge | Jobs esperando por cola                              |
| `integrador_horizon_current_processes`    | Gauge | Procesos activos por cola                            |
| `integrador_horizon_recent_jobs`          | Gauge | Jobs recientes procesados                            |
| `integrador_horizon_master_supervisors`   | Gauge | Supervisores master activos                          |
### 5.2 Métricas de Colas (Incluidas Automáticamente)

  

| Métrica                         | Tipo  | Labels | Descripción         |
| ------------------------------- | ----- | ------ | ------------------- |
| `integrador_queue_size`         | Gauge | queue  | Tamaño de cada cola |
| `integrador_queue_pending_jobs` | Gauge | queue  | Jobs pendientes     |
| `integrador_queue_delayed_jobs` | Gauge | queue  | Jobs retrasados     |
### 5.3 Métricas Personalizadas - Inventario
 

| Métrica                                   | Tipo  | Labels   | Descripción                      |
| ----------------------------------------- | ----- | -------- | -------------------------------- |
| `integrador_supplier_products_total`      | Gauge | supplier | Total de productos por proveedor |
| `integrador_supplier_products_active`     | Gauge | supplier | Productos activos por proveedor  |
| `integrador_supplier_products_with_stock` | Gauge | supplier | Productos con stock > 0          |
| `integrador_product_items_total`          | Gauge | -        | Total de ProductItems unificados |
| `integrador_product_items_published`      | Gauge | -        | ProductItems publicados          |
| `integrador_products_total`               | Gauge | -        | Total de Products padre          |

  

### 5.4 Métricas Personalizadas - Sincronizaciones

  

| Métrica                              | Tipo    | Labels       | Descripción                         |
| ------------------------------------ | ------- | ------------ | ----------------------------------- |
| `integrador_sync_last_run_timestamp` | Gauge   | sync_command | Timestamp última ejecución          |
| `integrador_sync_products_synced`    | Gauge   | partner      | Productos sincronizados por partner |
| `integrador_sync_pending_count`      | Gauge   | partner      | Sincronizaciones pendientes         |
| `integrador_sync_failed_24h`         | Counter | partner      | Sincronizaciones fallidas (24h)     |
### 5.5 Mapeo de Proveedores y Partners


**Proveedores (Suppliers):**

| ID  | Código | Nombre           |
| --- | ------ | ---------------- |
| 1   | CL     | Compulandia API  |
| 2   | FX     | Fastrax          |
| 888 | NGO    | NGO              |
| 999 | CP     | Compras Paraguai |
  

**Partners (Destinos de Sync):**

| ID  | Código | Nombre                  |
| --- | ------ | ----------------------- |
| 1   | TN     | TiendaNaranja           |
| 2   | CL     | Compulandia/WooCommerce |

  

---

  

## 6. Implementación de Métricas

### 6.1 Service Provider


```php

// app/Providers/PrometheusServiceProvider.php

  

<?php

  

namespace App\Providers;

  

use Illuminate\Support\ServiceProvider;

use Spatie\Prometheus\Facades\Prometheus;

use App\Models\SupplierProduct;

use App\Models\ProductItem;

use App\Models\Product;

use App\Constants\SupplierConstants;

use App\Constants\PartnerConstants;

  

class PrometheusServiceProvider extends ServiceProvider

{

    public function boot(): void

    {

        $this->registerInventoryMetrics();

        $this->registerSyncMetrics();

    }

  

    private function registerInventoryMetrics(): void

    {

        // Productos por proveedor

        $suppliers = [

            SupplierConstants::CL => 'compulandia',

            SupplierConstants::FX => 'fastrax',

            SupplierConstants::NGO => 'ngo',

        ];

  

        foreach ($suppliers as $supplierId => $supplierName) {

            Prometheus::addGauge("supplier_products_total")

                ->label('supplier', $supplierName)

                ->value(fn() => SupplierProduct::where('supplier_id', $supplierId)->count());

  

            Prometheus::addGauge("supplier_products_active")

                ->label('supplier', $supplierName)

                ->value(fn() => SupplierProduct::where('supplier_id', $supplierId)

                    ->where('publicable', true)

                    ->count());

  

            Prometheus::addGauge("supplier_products_with_stock")

                ->label('supplier', $supplierName)

                ->value(fn() => SupplierProduct::where('supplier_id', $supplierId)

                    ->where('stock', '>', 0)

                    ->count());

        }

  

        // Totales de ProductItem

        Prometheus::addGauge("product_items_total")

            ->value(fn() => ProductItem::count());

  

        Prometheus::addGauge("product_items_published")

            ->value(fn() => ProductItem::where('publish', true)->count());

  

        // Total de Products

        Prometheus::addGauge("products_total")

            ->value(fn() => Product::count());

    }

  

    private function registerSyncMetrics(): void

    {

        $partners = [

            PartnerConstants::TN => 'tiendanaranja',

            PartnerConstants::CL => 'compulandia',

        ];

  

        foreach ($partners as $partnerId => $partnerName) {

            Prometheus::addGauge("sync_products_synced")

                ->label('partner', $partnerName)

                ->value(fn() => $this->getSyncedCount($partnerId));

        }

    }

  

    private function getSyncedCount(int $partnerId): int

    {

        // Implementar según la lógica de tracking de syncs

        return 0; // Placeholder

    }

}

```

  

### 6.2 Registro del Provider

  

```php

// bootstrap/providers.php

  

return [

    // ... otros providers

    App\Providers\PrometheusServiceProvider::class,

];

```

  

---

  

## 7. Ejemplo de Respuesta del Endpoint

  

```prometheus

# HELP integrador_horizon_status Status of Horizon (-1=inactive, 0=paused, 1=running)

# TYPE integrador_horizon_status gauge

integrador_horizon_status 1

  

# HELP integrador_horizon_jobs_per_minute Jobs processed per minute

# TYPE integrador_horizon_jobs_per_minute gauge

integrador_horizon_jobs_per_minute 142

  

# HELP integrador_horizon_failed_jobs_per_hour Failed jobs in the last hour

# TYPE integrador_horizon_failed_jobs_per_hour gauge

integrador_horizon_failed_jobs_per_hour 3

  

# HELP integrador_supplier_products_total Total products by supplier

# TYPE integrador_supplier_products_total gauge

integrador_supplier_products_total{supplier="compulandia"} 15234

integrador_supplier_products_total{supplier="fastrax"} 8567

integrador_supplier_products_total{supplier="ngo"} 2341

  

# HELP integrador_supplier_products_with_stock Products with stock > 0

# TYPE integrador_supplier_products_with_stock gauge

integrador_supplier_products_with_stock{supplier="compulandia"} 12456

integrador_supplier_products_with_stock{supplier="fastrax"} 7234

integrador_supplier_products_with_stock{supplier="ngo"} 1987

  

# HELP integrador_product_items_total Total unified product items

# TYPE integrador_product_items_total gauge

integrador_product_items_total 18542

  

# HELP integrador_product_items_published Published product items

# TYPE integrador_product_items_published gauge

integrador_product_items_published 14230

  

# HELP integrador_queue_size Current queue sizes

# TYPE integrador_queue_size gauge

integrador_queue_size{queue="default"} 45

integrador_queue_size{queue="sync"} 123

integrador_queue_size{queue="images"} 67

```

  

---
 
## 8. Plan de Instalación

  

### 8.1 Pasos de Implementación

  

| Paso | Descripción                     | Comando/Acción                                                         |
| ---- | ------------------------------- | ---------------------------------------------------------------------- |
| 1    | Instalar paquete                | `composer require spatie/laravel-prometheus`                           |
| 2    | Publicar configuración          | `php artisan vendor:publish --tag=prometheus-config`                   |
| 3    | Crear middleware de token       | Crear `app/Http/Middleware/PrometheusTokenAuth.php`                    |
| 4    | Registrar middleware            | Agregar a `bootstrap/app.php`                                          |
| 5    | Crear PrometheusServiceProvider | Crear provider con métricas                                            |
| 6    | Registrar provider              | Agregar a `bootstrap/providers.php`                                    |
| 7    | Configurar .env                 | Agregar variables de entorno                                           |
| 8    | Verificar endpoint              | `curl -H "Authorization: Bearer TOKEN" https://app/metrics/prometheus` |
### 8.2 Verificación

```bash

# Test local (sin restricción de IP)

curl http://localhost:8000/metrics/prometheus

  

# Test con autenticación

curl -H "Authorization: Bearer your-token" https://integrador.compulandia.com.py/metrics/prometheus

```

  

---

  

## 9. Configuración de Prometheus Server

  

### 9.1 Job Configuration

  

```yaml

# /etc/prometheus/prometheus.yml

  

global:

  scrape_interval: 30s

  evaluation_interval: 30s

  

scrape_configs:

  - job_name: 'integrador-laravel'

    scrape_interval: 30s

    scrape_timeout: 10s

    metrics_path: '/metrics/prometheus'

    scheme: https

  

    static_configs:

      - targets: ['integrador.compulandia.com.py']

        labels:

          environment: 'production'

          application: 'integrador'

  

    # Autenticación Bearer Token

    authorization:

      type: Bearer

      credentials_file: /etc/prometheus/tokens/integrador.token

  

    # Alternativamente, Basic Auth

    # basic_auth:

    #   username: prometheus

    #   password_file: /etc/prometheus/passwords/integrador.pwd

```

  

### 9.2 Alertas Sugeridas (AlertManager)

  

```yaml

# /etc/prometheus/alerts/integrador.yml

  

groups:

  - name: integrador

    rules:

      - alert: HorizonDown

        expr: integrador_horizon_status != 1

        for: 5m

        labels:

          severity: critical

        annotations:

          summary: "Horizon no está activo"

          description: "Horizon ha estado inactivo por más de 5 minutos"

  

      - alert: HighQueueBacklog

        expr: integrador_queue_size{queue="sync"} > 500

        for: 10m

        labels:

          severity: warning

        annotations:

          summary: "Cola de sincronización con backlog alto"

          description: "La cola 'sync' tiene {{ $value }} jobs pendientes"

  

      - alert: HighFailedJobs

        expr: integrador_horizon_failed_jobs_per_hour > 50

        for: 5m

        labels:

          severity: warning

        annotations:

          summary: "Alta tasa de jobs fallidos"

          description: "{{ $value }} jobs han fallido en la última hora"

  

      - alert: LowSupplierStock

        expr: integrador_supplier_products_with_stock < 1000

        for: 1h

        labels:

          severity: warning

        annotations:

          summary: "Stock bajo de proveedor {{ $labels.supplier }}"

          description: "El proveedor {{ $labels.supplier }} tiene solo {{ $value }} productos con stock"

```

  

---

  

## 10. Diagrama de Arquitectura

  

```

┌─────────────────────────────────────────────────────────────────────┐

│                         PROMETHEUS SERVER                            │

│                                                                      │

│   ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐    │

│   │  Scraper    │───▶│   TSDB      │───▶│   AlertManager      │    │

│   │  (30s)      │    │  (Storage)  │    │   (Notifications)   │    │

│   └─────────────┘    └─────────────┘    └─────────────────────┘    │

│          │                  │                                        │

└──────────│──────────────────│────────────────────────────────────────┘

           │                  │

           │ HTTPS + Bearer   │

           │ Token            │

           ▼                  ▼

┌─────────────────────────────────────────────────────────────────────┐

│                    INTEGRADOR (Laravel App)                          │

│                                                                      │

│   ┌─────────────────────────────────────────────────────────────┐   │

│   │  /metrics/prometheus                                         │   │

│   │  ┌─────────────┐  ┌─────────────┐  ┌───────────────────┐   │   │

│   │  │ IP Whitelist│─▶│ Token Auth  │─▶│ Prometheus Route  │   │   │

│   │  │ Middleware  │  │ Middleware  │  │                   │   │   │

│   │  └─────────────┘  └─────────────┘  └───────────────────┘   │   │

│   └─────────────────────────────────────────────────────────────┘   │

│                              │                                       │

│                              ▼                                       │

│   ┌─────────────────────────────────────────────────────────────┐   │

│   │                    Collectors                                │   │

│   │  ┌─────────────┐  ┌─────────────┐  ┌───────────────────┐   │   │

│   │  │  Horizon    │  │   Queue     │  │    Custom         │   │   │

│   │  │  Metrics    │  │   Metrics   │  │    Metrics        │   │   │

│   │  │  (7 tipos)  │  │  (3 tipos)  │  │  (Inventario,     │   │   │

│   │  │             │  │             │  │   Syncs, etc.)    │   │   │

│   │  └─────────────┘  └─────────────┘  └───────────────────┘   │   │

│   └─────────────────────────────────────────────────────────────┘   │

│                                                                      │

│   ┌──────────────────┐  ┌──────────────────┐                        │

│   │     Horizon      │  │     Database     │                        │

│   │     (Redis)      │  │    (MariaDB)     │                        │

│   └──────────────────┘  └──────────────────┘                        │

└─────────────────────────────────────────────────────────────────────┘

```

  

---

  

## 11. Consideraciones de Performance

  

### 11.1 Impacto en la Aplicación

  

| Aspecto | Consideración |

|---------|---------------|

| **Frecuencia de scraping** | 30 segundos (configurable) |

| **Queries por scrape** | ~10-15 queries simples |

| **Tiempo de respuesta** | < 500ms esperado |

| **Caching** | Considerar cache de 15-30s para métricas costosas |

  

### 11.2 Optimizaciones Futuras

  

- Implementar caching para métricas que requieran queries pesados

- Usar vistas materializadas para conteos frecuentes

- Considerar métricas asíncronas para datos históricos

  

---

  

## 12. Roadmap de Versiones Futuras

  

### v1.1 (Siguiente)

- [ ] Métricas de latencia de APIs externas (Compulandia, Fastrax, etc.)

- [ ] Histogramas de tiempo de procesamiento de jobs

  

### v1.2

- [ ] Métricas de errores por categoría

- [ ] Tracking de sincronizaciones exitosas/fallidas por partner

  

### v2.0

- [ ] Dashboard Grafana pre-configurado

- [ ] Alertas personalizadas por proveedor

- [ ] Métricas de negocio (ventas, conversiones)

  

---

  

## 13. Referencias

  

- [spatie/laravel-prometheus - GitHub](https://github.com/spatie/laravel-prometheus)

- [Documentación Laravel Prometheus - Spatie](https://spatie.be/docs/laravel-prometheus/v1/introduction)

- [Prometheus Exposition Format](https://prometheus.io/docs/instrumenting/exposition_formats/)

- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)

  

---

  

**Documento preparado por**: Claude AI

**Revisado por**: [Pendiente]

**Aprobado por**: [Pendiente]