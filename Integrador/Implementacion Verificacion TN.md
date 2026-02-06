# Solución: Verificación de Sincronización con TiendaNaranja
## Problema Original
Se estaban duplicando productos en TiendaNaranja con SKUs como `COML-07012-1`, `COML-07012-2`, etc.
### Causa Raíz
Race condition: cuando un producto cambia múltiples veces antes de que TN procese la primera creación, cada job no encontraba el SKU en TN y creaba un duplicado.


```

Job #1 → crea producto → TN retorna process_id → producto en cola TN

Job #2 → getProductBySku() → no encuentra (TN aún procesa) → CREA DUPLICADO

```
### Problemas Identificados

1. **Race condition en creación**: Múltiples jobs intentaban crear el mismo producto
2. **No se usaba el `process_id`**: TN retorna un `process_id` que permite consultar el estado de la cola
3. **Reintentos peligrosos**: `scheduleRetry` por error de imágenes podía causar duplicados
4. **Dos flujos paralelos**: `scheduleRetry` y `VerifyTnProductSync` duplicaban lógica
5. **Verificación no subía imágenes**: El job de verificación solo verificaba datos

---
## Solución Implementada
### 1. Protección contra Race Condition en `ProductSyncService.syncProduct()`
Antes de decidir crear, verificar si hay un `process_id` pendiente:

```php

// 5.2) Verificar si hay un process_id pendiente (producto en cola de TN)

$hasPendingProcessId = ProductSyncLog::where('product_id', $productId)

    ->where('channel', SyncChannels::TN)

    ->where('response_message', 'EN_COLA_TN')

    ->where('attempted_at', '>', now()->subMinutes(15))

    ->exists();

  

if ($hasPendingProcessId && !$existsRemote) {

    Log::channel(LogChannels::TN)->info(

        "Producto tiene process_id pendiente en TN. Esperando que TN procese.",

        ['product_id' => $productId, 'sku_tn' => $skuTn]

    );

    return;

}

```

  ### 2. Marcar `EN_COLA_TN` Inmediatamente Después de Crear

En `createProductWithImages()`, guardar el `process_id` antes de intentar subir imágenes:

```php

$processId = $dataResponse[0]['process_id'] ?? null;

if ($processId) {

    ProductSyncLog::updateOrCreate(

        ['product_id' => $productId, 'channel' => SyncChannels::TN],

        [

            'status' => ProductSyncLog::STATUS_PENDING,

            'response_message' => 'EN_COLA_TN',

            'response_body' => $dataResponse,

            'attempted_at' => now(),

            'date' => now()->toDateString(),

        ]

    );

}

```
### 3. Job Unificado de Verificación: `VerifyTnProductSync`
Un solo job que verifica TODOS los productos pendientes en cola de TN.
#### Características

  

- **`ShouldBeUnique`**: Solo un job de verificación puede estar en cola a la vez

- **Verifica en batch**: Procesa todos los `EN_COLA_TN` pendientes en una sola ejecución

- **Re-encola si hay pendientes**: Si quedan productos con `status: "queued"`, programa otra verificación

- **Sube imágenes**: Si el producto se completó pero falta subir imágenes, las sube

  

#### Flujo del Job

  

```

┌─────────────────────────────────────────────────────────────────────────────┐

│                    VerifyTnProductSync.handle()                              │

└─────────────────────────────────────────────────────────────────────────────┘

                                      │

                                      ▼

                    ┌─────────────────────────────────┐

                    │ Buscar TODOS los logs con:      │

                    │ - response_message = EN_COLA_TN │

                    │ - status = PENDING              │

                    └─────────────────────────────────┘

                                      │

                                      ▼

                         ┌────────────────────┐

                         │ Para cada log:     │

                         │ GET /product-status│◄─────────────┐

                         │ /{process_id}      │              │

                         └────────────────────┘              │

                                   │                         │

                    ┌──────────────┼──────────────┐          │

                    │              │              │          │

                "queued"      "completed"      "error"       │

                    │              │              │          │

                    ▼              ▼              ▼          │

             ┌──────────┐   ┌──────────┐   ┌──────────┐     │

             │ Marcar   │   │ Subir    │   │ Marcar   │     │

             │ como     │   │ imágenes │   │ FAILED   │     │

             │ "sigue   │   │ si falta │   │ o retry  │     │

             │ pendiente│   │ → SUCCESS│   └──────────┘     │

             │          │   └──────────┘                    │

             │ $still   │                                   │

             │ Pending++│                                   │

             └──────────┘                                   │

                    │                                        │

                    └────────────────────────────────────────┘

                                      │

                                      ▼

                    ┌─────────────────────────────────┐

                    │ Al final:                       │

                    │ ¿$stillPending > 0?             │

                    └─────────────────────────────────┘

                              │              │

                             SÍ             NO

                              │              │

                              ▼              ▼

                    ┌───────────────┐ ┌───────────────┐

                    │ self::dispatch│ │ No re-encola  │

                    │ (delay 5 min) │ │ (todo listo)  │

                    └───────────────┘ └───────────────┘

```

  

---

  

## Endpoints de TN Utilizados

  

### 1. Crear/Actualizar Producto

  

```

POST /mpapi/sellers/me/addproduct

```

  

**Respuesta:**

```json

[

    {

        "error": 0,

        "message": "Producto en cola para procesamiento",

        "process_id": "mpapi_product_695e542f541fa2.21377885"

    }

]

```

  

### 2. Consultar Estado de Cola

  

```

GET /rest/V1/mpapi/sellers/me/product-status/{process_id}

```

  

**Respuesta cuando aún procesa:**

```json

[

    {

        "error": 0,

        "process_id": "mpapi_product_695e542f541fa2.21377885",

        "status": "queued",

        "timestamp": 1767789615,

        "data": {

            "seller_id": 235748,

            "sku": "N/A"

        }

    }

]

```

  

**Respuesta cuando ya procesó:**

```json

[

    {

        "error": 0,

        "process_id": "mpapi_product_695e542f541fa2.21377885",

        "status": "completed",

        "timestamp": 1767789668,

        "data": {

            "error": 0,

            "product_id": "1416195",

            "message": "Producto editado con éxito",

            "row_id": "1427394",

            "seller_id": 235748,

            "sku": "unknown"

        }

    }

]

```

  

---

  

## Flujo Completo de Sincronización

  

```

┌─────────────────────────────────────────────────────────────────────────────┐

│                         SYNC (CREATE o UPDATE)                               │

└─────────────────────────────────────────────────────────────────────────────┘

                                      │

                                      ▼

                    ┌─────────────────────────────────┐

                    │ ¿Hay process_id pendiente?      │

                    │ (EN_COLA_TN < 15 min)           │

                    └─────────────────────────────────┘

                              │              │

                             SÍ             NO

                              │              │

                              ▼              ▼

                    ┌───────────────┐ ┌───────────────────┐

                    │ return        │ │ Continuar sync    │

                    │ (esperar)     │ │ normal            │

                    └───────────────┘ └───────────────────┘

                                              │

                                              ▼

                              ┌───────────────────────────┐

                              │ saveProduct()             │

                              │ → TN retorna process_id   │

                              └───────────────────────────┘

                                              │

                                              ▼

                              ┌───────────────────────────┐

                              │ Guardar EN_COLA_TN        │

                              │ con process_id            │

                              └───────────────────────────┘

                                              │

                                              ▼

                              ┌───────────────────────────┐

                              │ Intentar subir imágenes   │

                              └───────────────────────────┘

                                       │           │

                                      OK        FALLO

                                       │           │

                                       ▼           ▼

                              ┌──────────┐ ┌───────────────┐

                              │ SUCCESS  │ │ Mantener      │

                              │          │ │ EN_COLA_TN    │

                              └──────────┘ │ (verify       │

                                           │ subirá imgs)  │

                                           └───────────────┘

                                              │

                                              ▼

                              ┌───────────────────────────┐

                              │ Despachar                 │

                              │ VerifyTnProductSync       │

                              │ (delay 5 min)             │

                              └───────────────────────────┘

  

═══════════════════════════════════════════════════════════════════════════════

                         5 MINUTOS DESPUÉS

═══════════════════════════════════════════════════════════════════════════════

  

┌─────────────────────────────────────────────────────────────────────────────┐

│                    VerifyTnProductSync                                       │

├─────────────────────────────────────────────────────────────────────────────┤

│  1. Buscar todos los EN_COLA_TN pendientes                                  │

│  2. Para cada uno, consultar GET /product-status/{process_id}               │

│  3. Si "completed" → subir imágenes si faltan → marcar SUCCESS              │

│  4. Si "queued" → dejar pendiente, contar                                   │

│  5. Si "error" → marcar FAILED o reintentar sync                            │

│  6. Si quedaron pendientes → re-encolar verify en 5 min                     │

└─────────────────────────────────────────────────────────────────────────────┘

```

  

---

  

## Archivos Modificados

  

| Archivo | Cambio |

|---------|--------|

| `app/Services/TiendaNaranja/ProductSyncService.php` | Agregar verificación de `hasPendingProcessId` y guardar `EN_COLA_TN` inmediatamente después de crear |

| `app/Services/TiendaNaranja/ProductApiServiceTN.php` | Agregar método `getProductQueueStatus($processId)` |

| `app/Jobs/VerifyTnProductSync.php` | Refactorizar para verificar en batch y usar endpoint de estado de cola |

| `app/Jobs/SyntToTiendaNaranjaJob.php` | Simplificar `handleSyncResult()` |

  

---

  

## Estados de ProductSyncLog

  

| response_message | status | Significado |

|------------------|--------|-------------|

| `EN_COLA_TN` | PENDING | Producto enviado a TN, esperando que procese |

| `OK` | SUCCESS | Sincronización completada exitosamente |

| `PRODUCT_NOT_READY_FOR_IMAGES` | FAILED | TN aún no indexó, imágenes pendientes |

| `SIN CATEGORIA RELACIONADA` | FAILED | Producto sin categoría TN mapeada |

| `SIN IMAGEN` | FAILED | Producto sin imágenes locales |

  

---

  

## Recomendaciones de TN

  

1. **No crear múltiples veces**: La regla de TN es que no existan SKUs duplicados

2. **Consultar estado de cola**: Usar `GET /product-status/{process_id}` antes de reintentar

3. **Ventana de 15 minutos**: TN recomienda esperar máximo 15 minutos para que procese