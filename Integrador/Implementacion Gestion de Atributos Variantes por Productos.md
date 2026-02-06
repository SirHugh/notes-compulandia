# Análisis Técnico: Gestión de Atributos y Variantes de Productos
 
**Fecha**: 2025-01-21

**Versión**: 1.0

**Estado**: Propuesta Técnica
  
## 1. Resumen
 
Este documento describe el análisis y solución propuesta para implementar la gestión de atributos de productos con soporte para:

- Atributos variantes (específicos por variante de producto)

- Atributos compartidos (comunes a todas las variantes de un producto)
  
La solución propone modificar la estructura existente de `product_configurations` agregando campos para soportar ambos casos de uso sin crear tablas adicionales.
 
---
 
## 2. Contexto del Problema 
### 2.1 Situación Actual
 
El sistema cuenta con las siguientes entidades para gestión de productos:
 
| Entidad              | Tabla                    | Propósito                                         |
| -------------------- | ------------------------ | ------------------------------------------------- |
| Product              | `products`               | Producto padre que agrupa variantes               |
| ProductItem          | `product_items`          | Variante específica con SKU único                 |
| Attribute            | `attributes`             | Definición de atributos (Color, RAM, etc.)        |
| Term                 | `terms`                  | Valores posibles de un atributo (Rojo, 8GB, etc.) |
| ProductConfiguration | `product_configurations` | Vincula variante con valores de atributos         |

### 2.2 Limitaciones Identificadas
 
1. **Sin relación directa Product → ProductConfiguration**: La configuración actual solo vincula `product_item_id` con `term_id`, sin referencia directa al producto padre.
 
2. **No existe concepto de atributos compartidos**: Todos los atributos se asignan a nivel de variante, generando redundancia cuando un atributo aplica a todas las variantes.
 
3. **No hay definición previa de atributos variantes**: No existe mecanismo para definir qué atributos pueden variar antes de crear las variantes.
  
## 3. Requerimientos
 
### 3.1 Requerimientos Funcionales
 
| ID    | Requerimiento                                                                               |
| ----- | ------------------------------------------------------------------------------------------- |
| RF-01 | Un producto puede tener múltiples variantes                                                 |
| RF-02 | Un producto debe tener atributos y valores configurables antes de asignar variantes         |
| RF-03 | Un producto puede variar por más de un atributo                                             |
| RF-04 | Un producto puede tener atributos compartidos con todas sus variantes                       |
| RF-05 | La configuración de atributos variantes debe existir antes de agregar variantes al producto |
### 3.2 Ejemplo de Negocio

**Producto**: Apple iPhone 14

**Atributos Variantes** (diferentes por cada variante):
 
| Atributo       | Valores Posibles     |
| -------------- | -------------------- |
| Color          | Verde, Rojo, Negro   |
| Almacenamiento | 32 GB, 64 GB, 128 GB |
| Memoria RAM    | 8 GB, 16 GB          |
**Atributos Compartidos** (iguales para todas las variantes):

| Atributo | Valor      |
| -------- | ---------- |
| Pantalla | 4 Pulgadas |
| Wifi     | Sí         |
| Antena   | 4G, 5G     |

**Variantes Resultantes**:

| SKU   | Combinación              |
| ----- | ------------------------ |
| 06645 | Verde, 32 GB, 8 GB RAM   |
| 02566 | Negro, 128 GB, 16 GB RAM |
| 01256 | Verde, 64 GB, 8 GB RAM   |
   
---

## 4. Solución Propuesta

### 4.1 Enfoque

Modificar la tabla existente `product_configurations` agregando:

- Campo `product_id` para relacionar directamente con el producto padre

- Campo `is_shared` para distinguir entre atributos compartidos y variantes
### 4.2 Modelo de Datos Propuesto
 

```

┌─────────────────────────────────────────────────────────────────┐

│                     product_configurations                       │

├─────────────────────────────────────────────────────────────────┤

│ id                  BIGINT PK                                   │

│ product_id          BIGINT FK → products.id                     │

│ product_item_id     BIGINT FK → product_items.id (NULLABLE)     │

│ term_id             BIGINT FK → terms.id                        │

│ is_shared           BOOLEAN DEFAULT false                       │

│ created_at          TIMESTAMP                                   │

│ updated_at          TIMESTAMP                                   │

└─────────────────────────────────────────────────────────────────┘

```
 
### 4.3 Diagrama Entidad-Relación
 
```

┌──────────────┐       ┌──────────────────────┐       ┌───────────────┐

│   products   │       │ product_configurations│       │ product_items │

├──────────────┤       ├──────────────────────┤       ├───────────────┤

│ id           │──┐    │ id                   │    ┌──│ id            │

│ name         │  │    │ product_id      FK   │────┘  │ product_id FK │

│ description  │  └───▶│ product_item_id FK   │◀──────│ sku           │

│ brand        │       │ term_id         FK   │       │ seo_name      │

│ model        │       │ is_shared            │       └───────────────┘

└──────────────┘       └──────────────────────┘

                                │

                                ▼

                       ┌───────────────┐       ┌──────────────┐

                       │     terms     │       │  attributes  │

                       ├───────────────┤       ├──────────────┤

                       │ id            │       │ id           │

                       │ attribute_id  │──────▶│ name         │

                       │ value         │       │ type         │

                       └───────────────┘       │ unit_measure │

                                               └──────────────┘

```

  
### 4.4 Lógica del Campo `is_shared`

| is_shared | product_id | product_item_id | Comportamiento                                                 |
| --------- | ---------- | --------------- | -------------------------------------------------------------- |
| `true`    | Requerido  | NULL            | Atributo compartido: aplica a TODAS las variantes del producto |
| `false`   | Requerido  | Requerido       | Atributo variante: aplica solo a UNA variante específica       |

### 4.5 Regla de Integridad

Se debe garantizar mediante constraint que:

- Cuando `is_shared = true`, el campo `product_item_id` debe ser NULL

- Cuando `is_shared = false`, el campo `product_item_id` debe tener valor

 
---

## 5. Ejemplo de Datos

### 5.1 Atributos Compartidos (is_shared = true)

Para el producto iPhone 14 (product_id = 1):

| id  | product_id | product_item_id | term_id | is_shared | Descripción          |
| --- | ---------- | --------------- | ------- | --------- | -------------------- |
| 1   | 1          | NULL            | 50      | true      | Pantalla: 4 Pulgadas |
| 2   | 1          | NULL            | 51      | true      | Wifi: Sí             |
| 3   | 1          | NULL            | 52      | true      | Antena: 4G           |
| 4   | 1          | NULL            | 53      | true      | Antena: 5G           |
### 5.2 Atributos Variantes (is_shared = false)

| id  | product_id | product_item_id | term_id | is_shared | Descripción                     |
| --- | ---------- | --------------- | ------- | --------- | ------------------------------- |
| 5   | 1          | 101             | 34      | false     | SKU 06645: Color Verde          |
| 6   | 1          | 101             | 40      | false     | SKU 06645: Almacenamiento 32GB  |
| 7   | 1          | 101             | 45      | false     | SKU 06645: RAM 8GB              |
| 8   | 1          | 102             | 35      | false     | SKU 02566: Color Negro          |
| 9   | 1          | 102             | 42      | false     | SKU 02566: Almacenamiento 128GB |
| 10  | 1          | 102             | 46      | false     | SKU 02566: RAM 16GB             |
 
---
## 6. Plan de Migración de Datos

### 6.1 Problema

Al agregar el campo `product_id` a la tabla `product_configurations`, los registros existentes tendrán este campo vacío (NULL).
### 6.2 Estrategia de Migración
 
1. **Agregar campos nuevos**: Crear `product_id` e `is_shared` como campos nullable inicialmente.
 
2. **Poblar product_id**: Ejecutar migración de datos que obtenga el `product_id` a través de la relación existente:

   ```

   product_configurations.product_item_id

   → product_items.product_id

   → products.id

   ```
 
3. **Establecer is_shared**: Todos los registros existentes se marcarán como `is_shared = false` ya que son configuraciones de variantes.
 
4. **Aplicar constraints**: Una vez migrados los datos, aplicar:

   - NOT NULL en `product_id`

   - CHECK constraint para validar lógica de `is_shared`
 
### 6.3 Orden de Ejecución

   
```

Paso 1: ALTER TABLE - Agregar product_id (nullable)

Paso 2: ALTER TABLE - Agregar is_shared (default false)

Paso 3: UPDATE - Poblar product_id desde product_items

Paso 4: ALTER TABLE - Cambiar product_id a NOT NULL

Paso 5: ALTER TABLE - Cambiar product_item_id a nullable

Paso 6: ADD CONSTRAINT - Check para lógica is_shared

```
 

---
  
## 7. Impacto en el Sistema

### 7.1 Modelos Afectados


| Modelo               | Cambios Requeridos                                              |
| -------------------- | --------------------------------------------------------------- |
| ProductConfiguration | Agregar campos, casts, relaciones y scopes                      |
| Product              | Agregar relaciones para configuraciones compartidas y variantes |
| ProductItem          | Sin cambios estructurales                                       |
### 7.2 Servicios Potencialmente Afectados

- Servicios de sincronización con WooCommerce

- Servicios de sincronización con Medusa

- Servicios de sincronización con TiendaNaranja

- Servicios de sincronización con Contimarket

- Vistas de administración de productos
 
### 7.3 Queries Afectados
 
Cualquier query que actualmente acceda a `product_configurations` deberá considerar:

- Filtrar por `is_shared` según el caso de uso

- Incluir la nueva relación con `product_id`
 
---
 
## 8. Consideraciones Técnicas
 
| Campo             | Índice     | Estado              |
| ----------------- | ---------- | ------------------- |
| `product_item_id` | Automático | Ya existe           |
| `term_id`         | Automático | Ya existe           |
| `product_id`      | Automático | Se creará con la FK |
#### Índice Compuesto Adicional

| Índice                  | Campos                    | Justificación                                             |
| ----------------------- | ------------------------- | --------------------------------------------------------- |
| `idx_pc_product_shared` | `(product_id, is_shared)` | Filtrar atributos compartidos vs variantes de un producto |

**Nota**: No se requiere índice compuesto `(product_item_id, is_shared)` porque la relación es determinística:

- Si `product_item_id` tiene valor → `is_shared = false` (siempre)

- Si `product_item_id` es NULL → `is_shared = true` (siempre)

#### Ejemplo de Uso del Índice Compuesto

Para obtener todos los atributos de una variante (ej: SKU 06645, product_item_id = 101):

1. **Atributos específicos de la variante**:

   ```

   WHERE product_item_id = 101

   ```

   → Usa índice FK de `product_item_id`

2. **Atributos compartidos del producto**:

   ```

   WHERE product_id = 1 AND is_shared = true

   ```

   → Usa índice compuesto `(product_id, is_shared)`

### 8.2 Validaciones de Negocio

1. No permitir duplicados de `term_id` para el mismo `product_id` cuando `is_shared = true`

2. No permitir duplicados de `term_id` para el mismo `product_item_id` cuando `is_shared = false`

3. Validar que el `product_item_id` pertenezca al `product_id` especificado
### 8.3 Compatibilidad con Medusa JS
 
La estructura propuesta es compatible con el modelo de Medusa que utiliza:

- `ProductOption` → equivalente a `attributes`

- `ProductOptionValue` → equivalente a `terms`

- `ProductVariant` → equivalente a `product_items`

  
---
## 9. Riesgos y Mitigación

| Riesgo                                    | Probabilidad | Impacto | Mitigación                                       |
| ----------------------------------------- | ------------ | ------- | ------------------------------------------------ |
| Inconsistencia de datos durante migración | Media        | Alto    | Ejecutar migración en transacción, backup previo |
| Queries existentes fallan                 | Media        | Medio   | Revisar y actualizar queries antes de deploy     |
| Performance degradada                     | Baja         | Medio   | Agregar índices apropiados                       |
| Sincronizaciones externas fallan          | Media        | Alto    | Actualizar servicios de sync antes de migración  |

---
## 10. Lista de Tareas (Jira)

### Epic: Gestión de Atributos Compartidos y Variantes de Productos

#### 10.1 Análisis y Diseño
 
| ID      | Tarea                                                         | Tipo     | Prioridad | Estimación |
| ------- | ------------------------------------------------------------- | -------- | --------- | ---------- |
| TASK-01 | Revisar y aprobar documento técnico de diseño                 | Análisis | Alta      | 2h         |
| TASK-02 | Identificar todos los servicios que usan ProductConfiguration | Análisis | Alta      | 4h         |
| TASK-03 | Documentar queries actuales que serán afectados               | Análisis | Alta      | 3h         |
#### 10.2 Base de Datos

| ID      | Tarea                                                              | Tipo    | Prioridad | Estimación |
| ------- | ------------------------------------------------------------------ | ------- | --------- | ---------- |
| TASK-04 | Crear migración: agregar campo product_id a product_configurations | Backend | Alta      | 1h         |
| TASK-05 | Crear migración: agregar campo is_shared a product_configurations  | Backend | Alta      | 1h         |
| TASK-06 | Crear migración: poblar product_id desde relación existente        | Backend | Alta      | 2h         |
| TASK-07 | Crear migración: aplicar constraint NOT NULL a product_id          | Backend | Alta      | 1h         |
| TASK-08 | Crear migración: hacer product_item_id nullable                    | Backend | Alta      | 1h         |
| TASK-09 | Crear migración: agregar CHECK constraint para lógica is_shared    | Backend | Media     | 1h         |
| TASK-10 | Crear migración: agregar índices de performance                    | Backend | Media     | 1h         |
#### 10.3 Modelos y Lógica de Negocio

| ID      | Tarea                                                             | Tipo    | Prioridad | Estimación |
| ------- | ----------------------------------------------------------------- | ------- | --------- | ---------- |
| TASK-11 | Actualizar modelo ProductConfiguration: campos, casts, relaciones | Backend | Alta      | 2h         |
| TASK-12 | Agregar scopes shared() y variant() a ProductConfiguration        | Backend | Alta      | 1h         |
| TASK-13 | Actualizar modelo Product: agregar relaciones de configuraciones  | Backend | Alta      | 2h         |
| TASK-14 | Crear validaciones de negocio para asignación de atributos        | Backend | Alta      | 3h         |
#### 10.4 Servicios de Sincronización

| ID      | Tarea                                                                | Tipo    | Prioridad | Estimación |
| ------- | -------------------------------------------------------------------- | ------- | --------- | ---------- |
| TASK-15 | Actualizar servicio de sync WooCommerce para usar nueva estructura   | Backend | Alta      | 4h         |
| TASK-16 | Actualizar servicio de sync Medusa para usar nueva estructura        | Backend | Alta      | 4h         |
| TASK-17 | Actualizar servicio de sync TiendaNaranja para usar nueva estructura | Backend | Media     | 3h         |
| TASK-18 | Actualizar servicio de sync Contimarket para usar nueva estructura   | Backend | Media     | 3h         |
#### 10.5 Interfaz de Usuario

| ID      | Tarea                                                            | Tipo     | Prioridad | Estimación |
| ------- | ---------------------------------------------------------------- | -------- | --------- | ---------- |
| TASK-19 | Diseñar UI para gestión de atributos compartidos vs variantes    | UI/UX    | Alta      | 4h         |
| TASK-20 | Implementar componente para asignar atributos compartidos        | Frontend | Alta      | 6h         |
| TASK-21 | Implementar componente para asignar atributos de variante        | Frontend | Alta      | 6h         |
| TASK-22 | Actualizar vista de detalle de producto para mostrar ambos tipos | Frontend | Media     | 3h         |
#### 10.6 Testing
  
| ID      | Tarea                                                        | Tipo    | Prioridad | Estimación |
| ------- | ------------------------------------------------------------ | ------- | --------- | ---------- |
| TASK-23 | Escribir tests unitarios para modelo ProductConfiguration    | Testing | Alta      | 3h         |
| TASK-24 | Escribir tests de integración para migraciones               | Testing | Alta      | 2h         |
| TASK-25 | Escribir tests para servicios de sincronización actualizados | Testing | Alta      | 4h         |
| TASK-26 | Ejecutar pruebas de regresión en ambiente de staging         | Testing | Alta      | 4h         |
#### 10.7 Despliegue

| ID      | Tarea                                                    | Tipo   | Prioridad | Estimación |
| ------- | -------------------------------------------------------- | ------ | --------- | ---------- |
| TASK-27 | Crear backup de base de datos de producción              | DevOps | Alta      | 1h         |
| TASK-28 | Ejecutar migraciones en ambiente de staging              | DevOps | Alta      | 1h         |
| TASK-29 | Validar integridad de datos post-migración en staging    | QA     | Alta      | 2h         |
| TASK-30 | Ejecutar migraciones en producción                       | DevOps | Alta      | 1h         |
| TASK-31 | Validar integridad de datos post-migración en producción | QA     | Alta      | 2h         |
| TASK-32 | Monitorear logs de sincronización post-deploy            | DevOps | Alta      | 4h         |

---

## 11. Criterios de Aceptación

1. ✅ Todos los registros existentes en `product_configurations` tienen `product_id` poblado
2. ✅ El sistema permite crear atributos compartidos a nivel de producto
3. ✅ El sistema permite crear atributos de variante a nivel de product_item
4. ✅ Las sincronizaciones con plataformas externas funcionan correctamente
5. ✅ No hay pérdida de datos durante la migración
6. ✅ Los queries de consulta de atributos funcionan correctamente con la nueva estructura

---
## 12. Apéndice

### 12.1 Glosario

| Término             | Definición                                                        |
| ------------------- | ----------------------------------------------------------------- |
| Product             | Entidad padre que agrupa variantes de un mismo producto           |
| ProductItem         | Variante específica de un producto con SKU único                  |
| Attribute           | Característica configurable (ej: Color, Tamaño, RAM)              |
| Term                | Valor específico de un atributo (ej: Rojo, Grande, 8GB)           |
| Atributo Compartido | Valor de atributo que aplica a todas las variantes de un producto |
| Atributo Variante   | Valor de atributo específico de una variante                      |
  

### 12.2 Referencias

- [[Gestion de Atributos de Productos]]  
- Documentación Medusa JS: https://docs.medusajs.com/resources/commerce-modules/product
- Estructura actual de migraciones: `database/migrations/39/`, `database/migrations/48/`, `database/migrations/49/`
---

**Documento preparado por**: Claude AI
**Revisado por**: [Pendiente]
**Aprobado por**: [Pendiente]