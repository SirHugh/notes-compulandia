# Flujo de Confirmación de Productos Pendientes

> **Estado:** Análisis completado
> **Fecha:** 2026-01-27
> **Versión:** 1.0

## Resumen Ejecutivo

Este documento especifica el diseño de un nuevo flujo de confirmación para productos pendientes (SupplierProducts sin match) mediante un modal multistep. El objetivo es superar las limitaciones actuales del sistema y proporcionar una experiencia guiada que permita:

1. Matchear a variantes existentes (flujo actual mejorado)
2. Crear nuevas variantes de productos existentes (nuevo)
3. Crear productos completamente nuevos con configuración completa (mejorado)

---

## Índice

1. [Contexto y Problema](#1-contexto-y-problema)
2. [Modelo de Datos Relevante](#2-modelo-de-datos-relevante)
3. [Arquitectura de la Solución](#3-arquitectura-de-la-solución)
4. [Casos de Uso](#4-casos-de-uso)
5. [Especificación de Pasos](#5-especificación-de-pasos)
6. [Flujos por Caso](#6-flujos-por-caso)
7. [Integraciones](#7-integraciones)
8. [Wireframes Conceptuales](#8-wireframes-conceptuales)
9. [Consideraciones Técnicas](#9-consideraciones-técnicas)
10. [Plan de Implementación](#10-plan-de-implementación)

---

## 1. Contexto y Problema

### 1.1 Estado Actual

Los productos pendientes son `SupplierProduct` que no están relacionados a ninguna variante (`id_product_item = NULL`). Actualmente el sistema permite:

- **Matchear** a una variante existente como proveedor
- **Crear** un nuevo producto + variante única

### 1.2 Limitaciones Identificadas

| # | Limitación | Impacto |
|---|------------|---------|
| 1 | No se puede crear una variante para un producto existente | Si un SupplierProduct representa una variante nueva de un producto ya existente, hay que crear un producto duplicado |
| 2 | Nombre de producto no configurable al crear | El nombre se copia directamente del SupplierProduct (suele ser largo y técnico) |
| 3 | Sin configuración de atributos en creación | Las variantes se crean sin especificar color, capacidad, etc. |
| 4 | Búsqueda limitada (LIKE SQL) | No detecta coincidencias parciales ni sinónimos |

### 1.3 Objetivo

Implementar un modal multistep que:

- Permita búsqueda inteligente via Algolia
- Guíe la decisión: ¿es supplier nuevo, variante nueva, o producto nuevo?
- Configure todos los aspectos necesarios según el caso
- Integre servicios existentes (AI, Perplexity, etc.)

---

## 2. Modelo de Datos Relevante

### 2.1 Entidades Principales

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│ SupplierProduct │       │    Product      │       │   ProductItem   │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id              │       │ id              │       │ id              │
│ sku             │       │ name            │       │ product_id (FK) │
│ name            │       │ description     │       │ sku             │
│ supplier_id     │       │ brand           │       │ seo_name        │
│ id_product_item │──────►│ model           │◄──────│ ean             │
│ regular_price   │       │                 │       │ short_description│
│ special_price   │       └─────────────────┘       │ active          │
│ total_stock     │               │                 └─────────────────┘
│ ean             │               │                         │
└─────────────────┘               ▼                         │
                         ┌─────────────────┐                │
                         │ProductConfigura-│                │
                         │     tion        │◄───────────────┘
                         ├─────────────────┤
                         │ product_id (FK) │
                         │ product_item_id │ (nullable)
                         │ term_id (FK)    │
                         │ is_shared       │
                         │ value           │
                         └─────────────────┘
                                 │
                                 ▼
                         ┌─────────────────┐
                         │      Term       │
                         ├─────────────────┤
                         │ id              │
                         │ attribute_id    │
                         │ name            │
                         │ slug            │
                         └─────────────────┘
```

### 2.2 Relaciones Clave

- **SupplierProduct** → pertenece a → **ProductItem** (cuando está confirmado)
- **Product** → tiene muchos → **ProductItem** (variantes)
- **Product** → tiene muchos → **ProductConfiguration** (atributos configurados)
- **ProductItem** → tiene muchos → **ProductConfiguration** (atributos específicos de variante)
- **ProductConfiguration** → referencia → **Term** (valor del atributo)

### 2.3 Lógica de Atributos y Variantes

Los atributos se configuran a nivel de **Product** mediante `ProductConfiguration`:

```
Producto: iPhone 15 Pro Max
Atributos configurados:
  - Color: [Black, Green] (Terms)
  - Almacenamiento: [12Gb, 24Gb] (Terms)

Variantes posibles: 2 × 2 = 4 combinaciones
Variantes existentes (ProductItems):
  - iPhone 15 Pro Max Green 12Gb
  - iPhone 15 Pro Max Black 12Gb

Combinaciones disponibles (sin ProductItem):
  - Green + 24Gb
  - Black + 24Gb
```

---

## 3. Arquitectura de la Solución

### 3.1 Concepto: Casos y Pasos

El modal implementa un flujo basado en **casos** que determinan qué **pasos** seguir:

| Caso | Descripción | Resultado |
|------|-------------|-----------|
| **A** | Match a variante existente | SupplierProduct → ProductItem existente |
| **B** | Nueva variante de producto existente | SupplierProduct → nuevo ProductItem bajo Product existente |
| **C** | Producto completamente nuevo | SupplierProduct → nuevo Product + nuevo ProductItem |

### 3.2 Pasos del Flujo

| Paso | Nombre | Descripción |
|------|--------|-------------|
| 1 | Recomendación y Búsqueda | Buscar en Algolia, decidir caso |
| 2 | Producto y su Configuración | Nombre, marca, descripción, categorías |
| 3 | Variante y su Configuración | SKU, nombre SEO, EAN, atributos |
| 4 | Imágenes | Seleccionar/buscar imágenes |
| 5 | Resumen y Confirmación | Preview final y confirmar |

### 3.3 Pasos por Caso

| Paso | Caso A | Caso B | Caso C |
|------|:------:|:------:|:------:|
| 1. Recomendación y Búsqueda | ✅ | ✅ | ✅ |
| 2. Producto y su Configuración | - | - | ✅ |
| 3. Variante y su Configuración | - | ✅ (precargado si eligió combinación) | ✅ |
| 4. Imágenes | - | ✅ | ✅ |
| 5. Resumen y Confirmación | ✅ | ✅ | ✅ |

**Nota sobre Caso B:** Si en el Paso 1 el usuario selecciona "agregar como nueva variante" y elige una combinación de atributos existente, el Paso 3 se precarga con esa selección.

---

## 4. Casos de Uso

### 4.1 Caso A: Match a Variante Existente

**Escenario:** El SupplierProduct es el mismo producto que ya existe como variante, solo se agrega como nuevo proveedor.

**Ejemplo:**
- SupplierProduct: "iPhone 15 Pro Max Green 12Gb" de Fastrax
- ProductItem existente: "iPhone 15 Pro Max Green 12Gb" (ya tiene proveedor Compulandia)
- Acción: Agregar Fastrax como proveedor alternativo

**Flujo:** Paso 1 → Paso 5

### 4.2 Caso B: Nueva Variante de Producto Existente

**Escenario:** El SupplierProduct es una variante que no existe de un producto que sí existe.

**Ejemplo:**
- SupplierProduct: "iPhone 15 Pro Max Green 24Gb"
- Product existente: "iPhone 15 Pro Max"
- Variantes existentes: Green 12Gb, Black 12Gb
- Acción: Crear nueva variante "Green 24Gb"

**Flujo:** Paso 1 → Paso 3 (precargado) → Paso 4 → Paso 5

**Nota:** Si en el Paso 1 se selecciona una combinación de atributos disponible (ej: Green + 24Gb), el Paso 3 se precarga con esa selección.

### 4.3 Caso C: Producto Completamente Nuevo

**Escenario:** El SupplierProduct representa un producto que no existe en el sistema.

**Ejemplo:**
- SupplierProduct: "Samsung Galaxy S25 Ultra Black 256Gb"
- No existe ningún producto Samsung Galaxy S25
- Acción: Crear producto nuevo con su primera variante

**Flujo:** Paso 1 → Paso 2 → Paso 3 → Paso 4 → Paso 5

---

## 5. Especificación de Pasos

### 5.1 Paso 1: Recomendación y Búsqueda

**Propósito:** Ayudar al operador a decidir si el SupplierProduct es nuevo o existente.

#### Estructura de Dos Momentos

El paso se divide en dos momentos de interacción:

| Momento         | Descripción                                              | Carga de datos        |
| --------------- | -------------------------------------------------------- | --------------------- |
| **1. Búsqueda** | Dropdown simple con resultados de Algolia                | Ligera (solo nombres) |
| **2. Detalle**  | Panel con información completa del producto seleccionado | Bajo demanda          |

#### Momento 1: Búsqueda con Dropdown

**Componentes:**
- Panel de información del SupplierProduct (nombre, SKU, imagen, proveedor, precio)
- Campo de búsqueda conectado a Algolia (índice ProductItemView)
- Dropdown de resultados simplificado

**Datos del dropdown (por hit de Algolia):**
```typescript
{
  productId: number       // ID del Product padre
  productName: string     // Nombre del producto
  brandName: string       // Marca
}
```

**Comportamiento:**
- Búsqueda con debounce (300ms)
- Resultados agrupados por `productId` (deduplicados)
- Máximo 10-15 resultados visibles
- Al hacer clic en un resultado → cargar Momento 2

#### Momento 2: Panel de Detalle del Producto

**Se activa al:** Seleccionar un producto del dropdown.

**Datos cargados (consulta a BD):**
```typescript
{
  product: {
    id: number
    name: string
    brand: string
  }
  existingVariants: Array<{
    id: number           // ProductItem.id
    seoName: string
    sku: string
    attributeValues: string[]  // ["Green", "12Gb"]
  }>
  availableCombinations: Array<{
    termIds: number[]
    labels: string[]     // ["Green", "24Gb"]
    key: string          // "green-24gb"
  }>
  hasUnconfiguredAttributes: boolean  // Si el producto tiene más combinaciones posibles
}
```

**Secciones del panel:**
1. **Encabezado:** Nombre del producto y marca
2. **Variantes existentes:** Lista con opción "Matchear como supplier" → **Caso A**
3. **Combinaciones disponibles:** Lista con opción "Crear esta variante" → **Caso B** (precarga)
4. **Acción adicional:** "Agregar variante con otros atributos" → **Caso B** (sin precargar)

**Acciones al pie:**
- Botón "Seleccionar otro producto" → volver a Momento 1
- Botón "Crear producto nuevo" → **Caso C**

#### Datos de entrada (generales del paso)

```typescript
{
  supplierProduct: {
    id: number
    sku: string
    name: string
    brand_name: string
    regular_price: number
    special_price: number | null
    images: string[]
    supplier: { id: number, name: string }
  }
}
```

#### Acciones y Casos

| Acción | Caso | Precarga |
|--------|------|----------|
| Clic en variante existente | A | productItemId |
| Clic en combinación disponible | B | productId + selectedCombination |
| Clic en "otros atributos" | B | productId (sin combinación) |
| Clic en "Crear producto nuevo" | C | - |

#### Salida del Paso

```typescript
{
  caso: 'A' | 'B' | 'C'
  productItemId?: number              // Solo Caso A
  productId?: number                  // Solo Caso B
  selectedCombination?: {             // Caso B si eligió combinación
    termIds: number[]
    labels: string[]                  // ["Green", "24Gb"]
  }
}

---

### 5.2 Paso 2: Producto y su Configuración (Solo Caso C)

**Propósito:** Configurar los datos del nuevo producto y sus categorías.

#### Sección: Datos del Producto

| Campo | Tipo | Pre-llenado | Editable | AI |
|-------|------|-------------|----------|-----|
| Nombre | text | SupplierProduct.name | ✅ | ✅ |
| Marca | select/text | SupplierProduct.brand_name | ✅ | - |
| Modelo | text | - | ✅ | - |
| Descripción | textarea | SupplierProduct.long_description | ✅ | ✅ |

**Integración AI:**
- Botón para generar nombre optimizado
- Botón para generar descripción
- Servicio: `AIContentGeneration` (Google Gemini)

#### Sección: Categorías

- Integrar modal/selector de categorías existente
- Mostrar categorías del SupplierProduct como referencia
- Sugerencias basadas en MatchCategory

**Subsecciones:**
- Categorías CL (obligatorio al menos 1)
- Categorías TN (opcional)

#### Sección: Atributos del Producto (Opcional)

- Selector de Attributes disponibles
- Por cada Attribute seleccionado, selector de Terms (valores)
- Opción de agregar múltiples valores por atributo
- Los valores marcados definen la primera variante

**Nota:** Si el operador no desea configurar atributos en este momento, puede omitir esta sección (producto sin variantes configuradas, se configura después).

**Validaciones:**
- Nombre producto requerido
- Nombre no duplicado (warning si existe similar)
- Al menos 1 categoría CL seleccionada

---

### 5.3 Paso 3: Variante y su Configuración (Casos B y C)

**Propósito:** Configurar los datos específicos de la variante y sus atributos.

#### Comportamiento según contexto:

| Contexto | Estado del paso |
|----------|-----------------|
| Caso B + combinación seleccionada en Paso 1 | Atributos precargados (solo visualización) |
| Caso B + "otros atributos" | Mostrar combinaciones disponibles para seleccionar |
| Caso C + atributos configurados en Paso 2 | Atributos precargados de selección en Paso 2 |
| Caso C + sin atributos en Paso 2 | Sin sección de atributos |

#### Sección: Atributos de la Variante (Caso B)

**Mostrar:**
1. Atributos configurados del Product (ej: Color: [Black, Green])
2. Variantes existentes (ej: Green 12Gb ✓, Black 12Gb ✓)
3. Combinaciones disponibles (ej: Green 24Gb, Black 24Gb)

**Acciones:**
- Seleccionar combinación disponible
- O indicar que necesita agregar nuevo valor de atributo (redirige o modal inline)

**Datos necesarios (Caso B):**
```typescript
{
  product: {
    id: number
    name: string
    configurations: Array<{
      attribute: { id: number, name: string }
      values: Array<{ termId: number, name: string }>
    }>
  }
  existingVariants: Array<{
    id: number
    seo_name: string
    sku: string
    attributeValues: string[]  // ["Green", "12Gb"]
  }>
  availableCombinations: Array<{
    values: Array<{ termId: number, name: string }>
    key: string  // "green-24gb"
  }>
}
```

#### Sección: Datos de la Variante

| Campo | Tipo | Pre-llenado | Editable | AI |
|-------|------|-------------|----------|-----|
| SKU | text | SupplierProduct.sku | ✅ | - |
| Nombre SEO | text | SupplierProduct.name | ✅ | ✅ |
| EAN | text | SupplierProduct.ean | ✅ | - |
| Descripción corta | textarea | SupplierProduct.short_description | ✅ | - |

**Validaciones:**
- SKU requerido
- SKU único en sistema
- Nombre SEO requerido
- Combinación de atributos seleccionada (si aplica)

---

### 5.4 Paso 4: Imágenes (Casos B y C)

**Propósito:** Seleccionar y ordenar imágenes para la variante.

**Secciones:**

#### 1. Imágenes del Proveedor
- Mostrar imágenes del SupplierProduct
- Checkbox para seleccionar/deseleccionar
- Pre-seleccionadas por defecto

#### 2. Búsqueda Perplexity
- Campo de búsqueda (pre-llenado con nombre del producto/variante)
- Botón buscar
- Grid de resultados con checkbox para seleccionar

#### 3. Imágenes Seleccionadas
- Área de drag & drop para ordenar
- Primera imagen = imagen principal
- Indicador visual de orden

**Integración:**
- Servicio Perplexity existente (galería de producto)

**Validaciones:**
- (Opcional) Al menos 1 imagen seleccionada

---

### 5.5 Paso 5: Resumen y Confirmación (Todos los casos)

**Propósito:** Mostrar resumen completo y confirmar la acción.

**Mostrar según caso:**

#### Caso A: Match a Variante Existente
```
MATCH A VARIANTE EXISTENTE

SupplierProduct: iPhone 15 Pro Max Green 12Gb (Fastrax)
        ↓
ProductItem: iPhone 15 Pro Max Green 12Gb (ID: 456)

Precios calculados:
  - Regular: $1,250,000
  - Especial: $1,125,000 (hasta 2026-02-15)

[Cancelar] [Confirmar Match]
```

#### Caso B: Nueva Variante
```
NUEVA VARIANTE

Producto: iPhone 15 Pro Max
Nueva variante: Green + 24Gb
  - SKU: IPH15PM-GRN-24
  - Nombre: iPhone 15 Pro Max Green 24Gb
  - Imágenes: 3 seleccionadas

Precios calculados:
  - Regular: $1,450,000
  - Especial: -

[Cancelar] [Atrás] [Confirmar Creación]
```

#### Caso C: Nuevo Producto
```
NUEVO PRODUCTO

Producto: Samsung Galaxy S25 Ultra
  - Marca: Samsung
  - Categorías: Celulares, Smartphones

Primera variante: Black + 256Gb
  - SKU: SGS25U-BLK-256
  - Nombre: Samsung Galaxy S25 Ultra Black 256Gb
  - Imágenes: 4 seleccionadas

Atributos configurados:
  - Color: Black
  - Almacenamiento: 256Gb

Precios calculados:
  - Regular: $2,850,000
  - Especial: -

[Cancelar] [Atrás] [Confirmar Creación]
```

---

## 6. Flujos por Caso

### 6.1 Diagrama de Flujo Unificado

```
                    ┌─────────────────┐
                    │     PASO 1      │
                    │  Recomendación  │
                    │   y Búsqueda    │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  │                  ▼
     ┌─────────┐             │             ┌─────────┐
     │ CASO A  │             │             │ CASO C  │
     │ Match   │             │             │ Nuevo   │
     │ Directo │             │             │ Producto│
     └────┬────┘             │             └────┬────┘
          │                  │                  │
          │                  │                  ▼
          │                  │        ┌─────────────────┐
          │                  │        │     PASO 2      │
          │                  │        │   Producto y    │
          │                  │        │ su Configuración│
          │                  │        └────────┬────────┘
          │                  │                 │
          │                  ▼                 │
          │             ┌─────────┐            │
          │             │ CASO B  │            │
          │             │ Nueva   │            │
          │             │ Variante│            │
          │             └────┬────┘            │
          │                  │                 │
          │                  └────────┬────────┘
          │                           │
          │                           ▼
          │                 ┌─────────────────┐
          │                 │     PASO 3      │
          │                 │   Variante y    │
          │                 │ su Configuración│
          │                 └────────┬────────┘
          │                          │
          │                          ▼
          │                 ┌─────────────────┐
          │                 │     PASO 4      │
          │                 │    Imágenes     │
          │                 └────────┬────────┘
          │                          │
          └──────────────────────────┤
                                     │
                                     ▼
                            ┌─────────────────┐
                            │     PASO 5      │
                            │   Resumen y     │
                            │  Confirmación   │
                            └─────────────────┘
```

### 6.2 Secuencia por Caso

```
CASO A: Match a variante existente
┌──────────────┐     ┌──────────────┐
│   Paso 1     │ ──► │   Paso 5     │
│ Recomendación│     │  Resumen y   │
│  y Búsqueda  │     │ Confirmación │
└──────────────┘     └──────────────┘
    Total: 2 pasos


CASO B: Nueva variante de producto existente
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Paso 1     │ ──► │   Paso 3     │ ──► │   Paso 4     │ ──► │   Paso 5     │
│ Recomendación│     │  Variante y  │     │   Imágenes   │     │  Resumen y   │
│  y Búsqueda  │     │   su Config  │     │              │     │ Confirmación │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
    Total: 4 pasos

    Nota: Si en Paso 1 se seleccionó combinación de atributos,
          el Paso 3 viene precargado con esa selección.


CASO C: Producto completamente nuevo
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Paso 1     │──►│   Paso 2     │──►│   Paso 3     │──►│   Paso 4     │──►│   Paso 5     │
│ Recomendación│   │  Producto y  │   │  Variante y  │   │   Imágenes   │   │  Resumen y   │
│  y Búsqueda  │   │   su Config  │   │   su Config  │   │              │   │ Confirmación │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
    Total: 5 pasos (flujo completo)
```

---

## 7. Integraciones

### 7.1 Servicios Existentes a Utilizar

| Funcionalidad | Servicio/Componente | Ubicación |
|---------------|---------------------|-----------|
| Búsqueda | Algolia Scout | ProductItemView (índice existente) |
| Generación AI nombres/descripciones | AIContentGeneration | `app/Services/AIContentGeneration/` |
| Selector de categorías | Modal existente | (identificar componente) |
| Búsqueda de imágenes | Perplexity | Galería de producto existente |
| Cálculo de precios | SupplierProduct | `calculateAndStorePrices()` |
| Gestión de atributos | ProductConfiguration | Modelo existente |
| Subida de imágenes | Cloudflare | `app/Services/Cloudflare/` |

### 7.2 Búsqueda Algolia

**Índice:** `product_items_view` (ProductItemView)

**Campos buscables sugeridos:**
- seo_name
- sku
- ean
- product.name
- product.brand

**Configuración Algolia:**
```javascript
{
  searchableAttributes: ['seo_name', 'sku', 'ean', 'product.name', 'product.brand'],
  typoTolerance: true,
  minWordSizefor1Typo: 4,
  hitsPerPage: 20
}
```

### 7.3 Generación AI

**Servicio:** `AIContentGeneration` (Google Gemini)

**Uso en el modal:**
- Paso 2: Generar nombre de producto optimizado
- Paso 2: Generar descripción de producto
- Paso 5: Generar nombre SEO de variante

### 7.4 Búsqueda de Imágenes Perplexity

Reutilizar la integración existente en la galería de productos para el Paso 6.

---

## 8. Wireframes Conceptuales

### 8.1 Paso 1: Recomendación y Búsqueda

#### Momento 1: Búsqueda con Dropdown

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONFIRMAR PRODUCTO PENDIENTE                              [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 1 de 5: Recomendación y Búsqueda                              │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ SUPPLIER PRODUCT                                                 │ │
│ │ ─────────────────                                                │ │
│ │ [IMG] iPhone 15 Pro Max Green 24Gb                              │ │
│ │       SKU: IPH15PM-GRN-24  |  Proveedor: Fastrax                │ │
│ │       Precio: $1,250,000  |  Stock: 15                          │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ BUSCAR PRODUCTO EXISTENTE                                          │
│ ─────────────────────────                                          │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 🔍 iphone 15 pro max                                            │ │
│ ├─────────────────────────────────────────────────────────────────┤ │
│ │ iPhone 15 Pro Max              Apple                        [>] │ │
│ │ iPhone 15 Pro                  Apple                        [>] │ │
│ │ iPhone 15                      Apple                        [>] │ │
│ │ Samsung Galaxy S24 Ultra       Samsung                      [>] │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ───────────────────────────────────────────────────────────────────│
│ ¿No encontraste el producto?                                        │
│ [Crear producto completamente nuevo]                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Notas del Momento 1:**
- El dropdown muestra solo nombre del producto y marca
- Máximo 10-15 resultados visibles
- Al hacer clic en un resultado, se carga el Momento 2
- El botón "Crear producto nuevo" siempre visible al pie

#### Momento 2: Panel de Detalle (después de seleccionar producto)

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONFIRMAR PRODUCTO PENDIENTE                              [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 1 de 5: Recomendación y Búsqueda                              │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ SUPPLIER PRODUCT                                                 │ │
│ │ ─────────────────                                                │ │
│ │ [IMG] iPhone 15 Pro Max Green 24Gb                              │ │
│ │       SKU: IPH15PM-GRN-24  |  Proveedor: Fastrax                │ │
│ │       Precio: $1,250,000  |  Stock: 15                          │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ PRODUCTO SELECCIONADO                           [Cambiar]       │ │
│ │ ─────────────────────                                           │ │
│ │ iPhone 15 Pro Max                              Apple            │ │
│ ├─────────────────────────────────────────────────────────────────┤ │
│ │                                                                  │ │
│ │ Variantes existentes:                                           │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ Green + 12Gb  →  iPhone 15 Pro Max Green 12Gb               │ │ │
│ │ │                                       [Matchear como supplier]│ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ Black + 12Gb  →  iPhone 15 Pro Max Black 12Gb               │ │ │
│ │ │                                       [Matchear como supplier]│ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │                                                                  │ │
│ │ Combinaciones disponibles (sin variante):                       │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ Green + 24Gb                          [Crear esta variante] │ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ Black + 24Gb                          [Crear esta variante] │ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │                                                                  │ │
│ │ [+ Agregar variante con otros atributos]                        │ │
│ │                                                                  │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ───────────────────────────────────────────────────────────────────│
│ ¿No es el producto correcto?                                        │
│ [Crear producto completamente nuevo]                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Notas del Momento 2:**
- Se carga al seleccionar un producto del dropdown
- Muestra variantes existentes con acción de match (Caso A)
- Muestra combinaciones disponibles con acción de crear (Caso B con precarga)
- Opción de agregar variante con otros atributos (Caso B sin precarga)
- Botón "Cambiar" para volver al Momento 1 y buscar otro producto
- Botón "Crear producto nuevo" sigue visible al pie (Caso C)

### 8.2 Paso 2: Producto y su Configuración (Caso C)

```
┌─────────────────────────────────────────────────────────────────────┐
│ NUEVO PRODUCTO                                            [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 2 de 5: Producto y su Configuración                           │
│                                                                     │
│ DATOS DEL PRODUCTO                                                 │
│ ──────────────────                                                 │
│ Nombre:      [Samsung Galaxy S25 Ultra_______________] [✨ AI]    │
│ Marca:       [Samsung ▼]                                          │
│ Modelo:      [S25 Ultra_____________________________]             │
│ Descripción: [______________________________________ ] [✨ AI]    │
│              [                                        ]            │
│                                                                     │
│ CATEGORÍAS                                                         │
│ ──────────                                                         │
│ Categorías CL: [Celulares ✓] [Smartphones ✓] [+ Agregar]         │
│ Categorías TN: [Telefonía] [+ Agregar]                            │
│                                                                     │
│ ATRIBUTOS DEL PRODUCTO (opcional)                                  │
│ ─────────────────────────────────                                  │
│ Configura los atributos que tendrán las variantes                  │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ Color                                                    [🗑️]  ││
│ │ Valores: [Black ✓] [White] [Blue] [+ agregar]                   ││
│ └─────────────────────────────────────────────────────────────────┘│
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ Almacenamiento                                           [🗑️]  ││
│ │ Valores: [256Gb ✓] [512Gb] [1Tb] [+ agregar]                    ││
│ └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│ [+ Agregar otro atributo]                                          │
│                                                                     │
│ Primera variante (basada en selección): Black + 256Gb              │
│ ℹ️ Puedes omitir atributos y configurarlos después                 │
│                                                                     │
│                                          [← Atrás] [Continuar →]  │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 Paso 3: Variante y su Configuración

#### Caso B (Nueva variante de producto existente)

```
┌─────────────────────────────────────────────────────────────────────┐
│ NUEVA VARIANTE DE: iPhone 15 Pro Max                      [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 3 de 4: Variante y su Configuración                           │
│                                                                     │
│ SELECCIÓN DE ATRIBUTOS                                             │
│ ──────────────────────                                             │
│ Atributos del producto: Color [Black, Green], Almacenamiento [12Gb, 24Gb]│
│                                                                     │
│ Variantes existentes:                                              │
│ ✓ Green + 12Gb  →  iPhone 15 Pro Max Green 12Gb                   │
│ ✓ Black + 12Gb  →  iPhone 15 Pro Max Black 12Gb                   │
│                                                                     │
│ Combinaciones disponibles:                                         │
│ ● Green + 24Gb   ← Seleccionado                                   │
│ ○ Black + 24Gb                                                     │
│                                                                     │
│ ⚠️ ¿Necesitas otro valor? [+ Agregar nuevo valor de atributo]     │
│                                                                     │
│ DATOS DE LA VARIANTE                                               │
│ ────────────────────                                               │
│ SKU:              [IPH15PM-GRN-24_______________________]         │
│ Nombre SEO:       [iPhone 15 Pro Max Green 24Gb________] [✨ AI]  │
│ EAN:              [1234567890123________________________]         │
│ Descripción corta:[____________________________________]          │
│                                                                     │
│                                          [← Atrás] [Continuar →]  │
└─────────────────────────────────────────────────────────────────────┘
```

#### Caso B (Precargado desde Paso 1)

```
┌─────────────────────────────────────────────────────────────────────┐
│ NUEVA VARIANTE DE: iPhone 15 Pro Max                      [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 3 de 4: Variante y su Configuración                           │
│                                                                     │
│ ATRIBUTOS SELECCIONADOS                                            │
│ ───────────────────────                                            │
│ ✓ Color: Green                                                     │
│ ✓ Almacenamiento: 24Gb                                            │
│ (Seleccionado en paso anterior)                                    │
│                                                                     │
│ DATOS DE LA VARIANTE                                               │
│ ────────────────────                                               │
│ SKU:              [IPH15PM-GRN-24_______________________]         │
│ Nombre SEO:       [iPhone 15 Pro Max Green 24Gb________] [✨ AI]  │
│ EAN:              [1234567890123________________________]         │
│ Descripción corta:[____________________________________]          │
│                                                                     │
│                                          [← Atrás] [Continuar →]  │
└─────────────────────────────────────────────────────────────────────┘
```

#### Caso C (Primera variante de producto nuevo)

```
┌─────────────────────────────────────────────────────────────────────┐
│ PRIMERA VARIANTE DE: Samsung Galaxy S25 Ultra             [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 3 de 5: Variante y su Configuración                           │
│                                                                     │
│ ATRIBUTOS DE LA VARIANTE                                           │
│ ────────────────────────                                           │
│ ✓ Color: Black (configurado en paso anterior)                      │
│ ✓ Almacenamiento: 256Gb (configurado en paso anterior)            │
│                                                                     │
│ DATOS DE LA VARIANTE                                               │
│ ────────────────────                                               │
│ SKU:              [SGS25U-BLK-256______________________]          │
│ Nombre SEO:       [Samsung Galaxy S25 Ultra Black 256Gb] [✨ AI]  │
│ EAN:              [_____________________________________]         │
│ Descripción corta:[____________________________________]          │
│                                                                     │
│                                          [← Atrás] [Continuar →]  │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.4 Paso 4: Imágenes

```
┌─────────────────────────────────────────────────────────────────────┐
│ IMÁGENES DE LA VARIANTE                                   [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 4 de 5: Imágenes                                              │
│                                                                     │
│ IMÁGENES DEL PROVEEDOR                                             │
│ ──────────────────────                                             │
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                                   │
│ │ [✓] │ │ [✓] │ │ [ ] │ │ [ ] │                                   │
│ │ img1│ │ img2│ │ img3│ │ img4│                                   │
│ └─────┘ └─────┘ └─────┘ └─────┘                                   │
│                                                                     │
│ BUSCAR IMÁGENES (Perplexity)                                       │
│ ────────────────────────────                                       │
│ [Samsung Galaxy S25 Ultra Black_____________] [🔍 Buscar]          │
│                                                                     │
│ Resultados:                                                        │
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                   │
│ │ [ ] │ │ [ ] │ │ [ ] │ │ [ ] │ │ [ ] │ │ [ ] │                   │
│ │ res1│ │ res2│ │ res3│ │ res4│ │ res5│ │ res6│                   │
│ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘                   │
│                                                                     │
│ IMÁGENES SELECCIONADAS (arrastra para ordenar)                     │
│ ──────────────────────────────────────────────                     │
│ ┌─────┐ ┌─────┐                                                    │
│ │ ① │ │ ② │    ← Primera = principal                            │
│ │ img1│ │ img2│                                                    │
│ └─────┘ └─────┘                                                    │
│                                                                     │
│                                          [← Atrás] [Continuar →]  │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.5 Paso 5: Resumen y Confirmación

```
┌─────────────────────────────────────────────────────────────────────┐
│ RESUMEN Y CONFIRMACIÓN                                    [X]      │
├─────────────────────────────────────────────────────────────────────┤
│ Paso 5 de 5: Resumen y Confirmación                                │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ NUEVO PRODUCTO                                                   ││
│ │ ──────────────                                                   ││
│ │ Nombre: Samsung Galaxy S25 Ultra                                ││
│ │ Marca: Samsung                                                  ││
│ │ Categorías: Celulares, Smartphones                              ││
│ └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ PRIMERA VARIANTE                                                 ││
│ │ ────────────────                                                 ││
│ │ Atributos: Black + 256Gb                                        ││
│ │ SKU: SGS25U-BLK-256                                             ││
│ │ Nombre: Samsung Galaxy S25 Ultra Black 256Gb                    ││
│ │ Imágenes: 4 seleccionadas                                       ││
│ └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ PRECIOS CALCULADOS                                               ││
│ │ ──────────────────                                               ││
│ │ Regular: $2,850,000                                             ││
│ │ Especial: -                                                     ││
│ └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│                                [Cancelar] [← Atrás] [✓ Confirmar] │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Consideraciones Técnicas

### 9.1 Tecnología Recomendada

**Componente Livewire 3** por las siguientes razones:
- Estado del modal persistente entre pasos
- Validación server-side en tiempo real
- Búsqueda Algolia desde PHP (Scout)
- Integración natural con modelos Eloquent
- Consistente con el stack actual del proyecto

### 9.2 Estructura de Componente Sugerida

```
app/Livewire/
└── PendingProducts/
    └── ConfirmationModal.php    # Componente principal

resources/views/livewire/
└── pending-products/
    └── confirmation-modal.blade.php
        # Incluye partials para cada paso o usa Alpine.js para tabs
```

### 9.3 Estado del Componente

```php
class ConfirmationModal extends Component
{
    // Identificación
    public ?int $supplierProductId = null;
    public ?SupplierProduct $supplierProduct = null;

    // Control de flujo
    public string $caso = '';           // 'A', 'B', 'C'
    public int $currentStep = 1;
    public array $completedSteps = [];

    // Paso 1: Recomendación y Búsqueda
    public string $searchQuery = '';
    public ?int $selectedProductItemId = null;  // Caso A: variante a matchear
    public ?int $selectedProductId = null;      // Caso B: producto padre
    public array $preselectedCombination = [];  // Caso B: combinación elegida en paso 1

    // Paso 2: Producto y su Configuración (Caso C)
    public string $productName = '';
    public string $productBrand = '';
    public string $productModel = '';
    public string $productDescription = '';
    public array $selectedCategoryIds = [];     // Categorías CL y TN
    public array $configuredAttributes = [];    // Atributos del producto

    // Paso 3: Variante y su Configuración (Casos B y C)
    public array $selectedCombination = [];     // Caso B: combinación seleccionada
    public string $variantSku = '';
    public string $variantSeoName = '';
    public string $variantEan = '';
    public string $variantShortDescription = '';

    // Paso 4: Imágenes (Casos B y C)
    public array $selectedImages = [];
    public array $imageOrder = [];

    // Datos calculados
    public array $calculatedPrices = [];

    // ...
}
```

### 9.4 Validaciones por Paso

| Paso | Validaciones |
|------|--------------|
| 1 | Caso seleccionado, ID correspondiente según caso |
| 2 | Nombre producto requerido, no duplicado (warning), al menos 1 categoría CL |
| 3 | SKU requerido y único, nombre SEO requerido, combinación válida (si Caso B) |
| 4 | (Opcional) Al menos 1 imagen |
| 5 | - (solo visualización) |

### 9.5 Transacciones de Base de Datos

La confirmación final (Paso 7) debe ejecutarse en una transacción:

```php
DB::transaction(function () {
    // Caso A: Solo actualizar SupplierProduct
    // Caso B: Crear ProductItem + ProductConfiguration + actualizar SupplierProduct
    // Caso C: Crear Product + ProductItem + ProductConfiguration + Categories + actualizar SupplierProduct

    // Siempre: Calcular precios, exportar imágenes (queue)
});
```

---

## 10. Plan de Implementación

### 10.1 Enfoque Recomendado

**Iterativo por casos**, empezando por el más simple:

| Fase | Alcance | Dependencias |
|------|---------|--------------|
| 1 | Esqueleto del modal + Paso 1 (búsqueda) + Caso A completo | Algolia |
| 2 | Caso B: Pasos 3, 4, 5 para nueva variante | Fase 1 |
| 3 | Caso C: Paso 2 completo para nuevo producto | Fase 2 |
| 4 | Integraciones AI (nombres, descripciones) | Fases 1-3 |
| 5 | Integración Perplexity (imágenes) | Fases 1-3 |
| 6 | Refinamientos y edge cases | Fases 1-5 |

### 10.2 Fase 1: Detalle

**Objetivo:** Modal funcional con búsqueda Algolia y Caso A (match directo).

**Tareas:**
1. Crear componente Livewire `ConfirmationModal`
2. Implementar apertura del modal desde lista de pendientes
3. Implementar Paso 1 con búsqueda Algolia (mostrar productos, variantes, combinaciones disponibles)
4. Implementar Paso 5 para Caso A (resumen y confirmación de match)
5. Integrar cálculo de precios existente
6. Testing del flujo Caso A

**Entregable:** Usuario puede buscar y matchear SupplierProduct a variante existente.

### 10.3 Fase 2: Detalle

**Objetivo:** Caso B completo (nueva variante de producto existente).

**Tareas:**
1. Extender Paso 1 para permitir selección de combinación disponible
2. Implementar Paso 3 (variante y su configuración) con:
   - Selección de combinación de atributos (si no se preseleccionó)
   - Configuración de datos de variante (SKU, nombre SEO, etc.)
3. Implementar Paso 4 (imágenes básico, sin Perplexity aún)
4. Extender Paso 5 para Caso B
5. Implementar creación de ProductItem + ProductConfiguration
6. Testing del flujo Caso B

**Entregable:** Usuario puede crear nueva variante de producto existente.

### 10.4 Fase 3: Detalle

**Objetivo:** Caso C completo (producto nuevo).

**Tareas:**
1. Implementar Paso 2 (producto y su configuración):
   - Datos del producto (nombre, marca, modelo, descripción)
   - Selector de categorías (integrar existente)
   - Configuración de atributos del producto
2. Adaptar Paso 3 para primera variante de producto nuevo
3. Extender Paso 5 para Caso C
4. Implementar creación de Product + ProductItem + configuraciones
5. Testing del flujo Caso C

**Entregable:** Usuario puede crear producto nuevo con variante inicial.

### 10.5 Fases 4-6: Detalle

**Fase 4 - AI:**
- Integrar AIContentGeneration en Paso 2 (nombre y descripción de producto)
- Integrar AIContentGeneration en Paso 3 (nombre SEO de variante)
- Botones de generación con loading states

**Fase 5 - Imágenes:**
- Integrar búsqueda Perplexity en Paso 4
- Drag & drop para ordenamiento de imágenes

**Fase 6 - Refinamientos:**
- Manejo de errores
- Edge cases (SKU duplicado, etc.)
- UX improvements
- Logs según estándar del proyecto

---

## Anexos

### A. Glosario

| Término | Definición |
|---------|------------|
| SupplierProduct | Producto raw importado de un proveedor |
| Product | Producto base que agrupa variantes |
| ProductItem | Variante específica de un producto (lo que se vende) |
| ProductConfiguration | Relación entre Product/ProductItem y Term (atributo) |
| Term | Valor de un atributo (ej: "Black", "256Gb") |
| Attribute | Tipo de atributo (ej: "Color", "Almacenamiento") |
| Match | Relacionar SupplierProduct con ProductItem existente |

### B. Archivos de Referencia

| Archivo | Propósito |
|---------|-----------|
| `app/Models/SupplierProduct.php` | Modelo principal, validaciones, processNewSupplierProduct() |
| `app/Models/Product.php` | Modelo de producto base |
| `app/Models/ProductItem.php` | Modelo de variante |
| `app/Models/ProductConfiguration.php` | Configuración de atributos |
| `app/Http/Controllers/PendingProductsController.php` | Controlador actual de pendientes |
| `app/Services/AIContentGeneration/` | Generación de contenido con Gemini |
| `resources/views/pending_products/index.blade.php` | Vista actual de lista de pendientes |

### C. Historial del Documento

| Versión | Fecha | Cambios |
|---------|-------|---------|
| 1.0 | 2026-01-27 | Documento inicial basado en análisis colaborativo |
| 1.1 | 2026-01-27 | Consolidación de 7 pasos a 5 pasos. Paso 2 incluye categorías. Paso 3 incluye atributos y datos de variante. |
| 1.2 | 2026-01-28 | Paso 1 reestructurado en dos momentos: (1) Dropdown simple con búsqueda Algolia mostrando solo nombre+marca, (2) Panel de detalle cargado bajo demanda al seleccionar producto. Wireframes actualizados. |
