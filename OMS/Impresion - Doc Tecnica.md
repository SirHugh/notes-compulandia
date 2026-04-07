# Documentación Técnica: Módulo de Impresión

## Información General

| Campo | Valor |
|-------|-------|
| **Módulo** | `print` |
| **Ubicación** | `src/modules/print/` |
| **Propósito** | Impresión de documentos de SAP B1 mediante Crystal Reports y PrintNode |
| **Fecha** | 2026-03-24 |
| **Estado** | En desarrollo |

---

## Patrones de Diseño Utilizados

| Patrón | Propósito |
|--------|-----------|
| **Strategy** | Diferentes algoritmos de impresión por entidad |
| **Template Method** | Esqueleto común con pasos específicos por entidad |
| **Factory** | Crear la strategy correcta según entidad |
| **Observer** | Notificar eventos de impresión (logs, métricas) |
| **Facade** | Ocultar complejidad de SAP + PrintNode |

---

## Estructura de Carpetas

```
src/modules/print/
│
├── interfaces/
│   ├── index.ts
│   ├── crystal-param.interface.ts
│   ├── print-strategy.interface.ts
│   └── print-result.interface.ts
│
├── strategies/
│   ├── index.ts
│   ├── base-print.strategy.ts        # Template Method
│   ├── order-print.strategy.ts
│   └── invoice-print.strategy.ts
│
├── observers/
│   ├── index.ts
│   ├── print-observer.interface.ts
│   └── logging-print.observer.ts
│
├── dto/
│   ├── print-request.dto.ts
│   └── print-response.dto.ts
│
├── print-strategy.registry.ts        # Factory
├── print.service.ts                  # Facade + Observer Subject
├── print.controller.ts
└── print.module.ts
```

---

## Interfaces

### `CrystalParam`

Representa un parámetro para Crystal Reports de SAP.

| Propiedad | Tipo         | Descripción                                                |
| --------- | ------------ | ---------------------------------------------------------- |
| `name`    | `string`     | Nombre del parámetro (ej: "Dockey@")                       |
| `type`    | `string`     | Tipo de dato ("xsd:string" \| "xsd:decimal" \| "xsd:date") |
| `value`   | `string[][]` | Valor en formato SAP                                       |

### `PrintStrategy`

Contrato que todas las strategies de impresión deben cumplir.

**Propiedades:**

| Propiedad     | Tipo     | Descripción                              |
| ------------- | -------- | ---------------------------------------- |
| `entity`      | `string` | Identificador único ("order", "invoice") |
| `layoutCode`  | `string` | Código Crystal Report ("RCRI0026")       |
| ~~`displayName`~~ | ~~`string`~~ | ~~Nombre legible ("Pedido de Venta")~~       |

**Métodos:**

| Método                       | Retorno          | Descripción                  |
| ---------------------------- | ---------------- | ---------------------------- |
| `validate(params)`           | `void`           | Valida parámetros requeridos |
| `buildCrystalParams(params)` | `CrystalParam[]` | Construye params para SAP    |

### `PrintObserver`

Contrato para observadores de eventos de impresión.

| Método                            | Descripción                       |
| --------------------------------- | --------------------------------- |
| `onPrintStarted(entity, params)`  | Llamado al iniciar impresión      |
| `onPrintCompleted(entity, jobId)` | Llamado al completar exitosamente |
| `onPrintFailed(entity, error)`    | Llamado al fallar                 |

### `PrintResult`

Resultado de una operación de impresión.

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `success` | `boolean` | Si fue exitoso |
| `printJobId` | `number` | ID del trabajo en PrintNode |
| `message` | `string` | Mensaje descriptivo |

---

## Clases

### `BasePrintStrategy` (abstract)

| Campo | Valor |
|-------|-------|
| **Patrón** | Template Method |
| **Ubicación** | `strategies/base-print.strategy.ts` |
| **Propósito** | Define el esqueleto del algoritmo de preparación para impresión |

**Propiedades abstractas:**

- `entity: string`
- `layoutCode: string`
- `displayName: string`

**Template Method:**

```
prepareForPrint(params): CrystalParam[]
  │
  ├──► commonValidation(params)      [HOOK - puede sobrescribirse]
  ├──► validate(params)              [ABSTRACT - debe implementarse]
  ├──► transformParams(params)       [HOOK - puede sobrescribirse]
  └──► buildCrystalParams(params)    [ABSTRACT - debe implementarse]
```

**Helpers protegidos:**

| Método | Retorno | Descripción |
|--------|---------|-------------|
| `createStringParam(name, value)` | `CrystalParam` | Crea param tipo string |
| `createDateParam(name, value)` | `CrystalParam` | Crea param tipo date |
| `createDecimalParam(name, value)` | `CrystalParam` | Crea param tipo decimal |

---

### `OrderPrintStrategy`

| Campo | Valor |
|-------|-------|
| **Extiende** | `BasePrintStrategy` |
| **Ubicación** | `strategies/order-print.strategy.ts` |
| **Propósito** | Impresión de Pedidos de Venta |

**Configuración:**

| Propiedad     | Valor                        |
| ------------- | ---------------------------- |
| `entity`      | `"order"`                    |
| `layoutCode`  | `REPORT_LAYOUT_ORDER` (.env) |
| `displayName` | `"Pedido de Venta"`          |
|               |                              |

**Parámetros requeridos:**

| Parámetro  | Tipo     | Obligatorio |
| ---------- | -------- | ----------- |
| `docEntry` | `number` | Sí          |
| `docNum`   | `number` | Sí          |

**Parámetros Crystal generados:**

| Crystal Param | Origen |
|---------------|--------|
| `Dockey@` | `params.docEntry` |
| `FolioNum@` | `params.docNum` |

---

### `InvoicePrintStrategy`

| Campo | Valor |
|-------|-------|
| **Extiende** | `BasePrintStrategy` |
| **Ubicación** | `strategies/invoice-print.strategy.ts` |
| **Propósito** | Impresión de Facturas |

**Configuración:**

| Propiedad | Valor |
|-----------|-------|
| `entity` | `"invoice"` |
| `layoutCode` | `REPORT_LAYOUT_INVOICE` (.env) |
| `displayName` | `"Factura de Venta"` |

**Parámetros requeridos:**

| Parámetro | Tipo | Obligatorio |
|-----------|------|-------------|
| `docEntry` | `number` | Sí |
| `docNum` | `number` | Sí |

**Parámetros Crystal generados:**

| Crystal Param | Origen |
|---------------|--------|
| `Dockey@` | `params.docEntry` |
| `FolioNum@` | `params.docNum` |

---

### `PrintStrategyRegistry`

| Campo | Valor |
|-------|-------|
| **Patrón** | Factory |
| **Ubicación** | `print-strategy.registry.ts` |
| **Propósito** | Mapea entity → Strategy y las instancia |

**Constructor:**

- Recibe todas las strategies por inyección de dependencias
- Las registra en un `Map<string, PrintStrategy>` interno

**Métodos:**

| Método                   | Retorno         | Descripción                    |
| ------------------------ | --------------- | ------------------------------ |
| `get(entity)`            | `PrintStrategy` | Retorna strategy o lanza error |
| `getAvailableEntities()` | `string[]`      | Lista entidades soportadas     |

---

### `LoggingPrintObserver`

| Campo | Valor |
|-------|-------|
| **Implementa** | `PrintObserver` |
| **Ubicación** | `observers/logging-print.observer.ts` |
| **Propósito** | Registra eventos de impresión en el log |

**Comportamiento:**

| Método | Log generado |
|--------|--------------|
| `onPrintStarted` | `[START] entity: params` |
| `onPrintCompleted` | `[OK] entity: JobId=xxx` |
| `onPrintFailed` | `[FAIL] entity: error` |

---

### `PrintService`

| Campo | Valor |
|-------|-------|
| **Patrón** | Facade + Observer Subject |
| **Ubicación** | `print.service.ts` |
| **Propósito** | Orquesta todo el proceso de impresión |

**Dependencias:**

| Dependencia             | Propósito                       |
| ----------------------- | ------------------------------- |
| `PrintStrategyRegistry` | Obtener strategy según entidad  |
| `ReportsService`        | Generar PDF con SAP API Gateway |
| `PrintNodeService`      | Enviar a impresora física       |

**Métodos:**

| Método | Retorno | Descripción |
|--------|---------|-------------|
| `print(entity, params, printerId)` | `Promise<PrintResult>` | Ejecuta impresión completa |
| `addObserver(observer)` | `void` | Registrar observer |
| `removeObserver(observer)` | `void` | Remover observer |
| `getAvailableEntities()` | `string[]` | Lista entidades |

**Flujo de `print()`:**

```
1. Notificar observers: onPrintStarted()
2. registry.get(entity)           → obtener strategy
3. strategy.validate(params)      → validar
4. strategy.buildCrystalParams()  → construir params
5. reportsService.exportToPdf()   → generar PDF
6. printNodeService.print()       → enviar a impresora
7. Notificar observers: onPrintCompleted() | onPrintFailed()
8. return PrintResult
```

---

### `PrintController`

| Campo | Valor |
|-------|-------|
| **Ubicación** | `print.controller.ts` |
| **Propósito** | Expone endpoints REST |

**Endpoints:**

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/api/print` | Imprimir documento |
| `GET` | `/api/print/entities` | Listar entidades disponibles |
| `GET` | `/api/print/printers` | Listar impresoras PrintNode |

---

## DTOs

### `PrintRequestDto`

Validación de entrada para `POST /api/print`

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `entity` | `string` | Sí | "order" \| "invoice" \| ... |
| `params` | `object` | Sí | { docEntry, docNum, ... } |
| `printerType` | `string` | Sí | "ticket" \| "paper" \| "label" |
| `printerId` | `number` | No | ID de PrintNode |

### `PrintResponseDto`

Respuesta de `POST /api/print`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `success` | `boolean` | Si fue exitoso |
| `printJobId` | `number` | ID del trabajo |
| `message` | `string` | Mensaje descriptivo |
| `printerName` | `string?` | Nombre de la impresora |

---

## Flujo de Ejecución

```
┌─────────────────────────────────────────────────────────────────────┐
│                         POST /api/print                              │
│         { entity: "order", params: { docEntry, docNum } }           │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                        PrintController                             │
│                     print(@Body() dto)                             │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                         PrintService                               │
│                                                                    │
│  1. Notificar observers: onPrintStarted()                         │
│                                │                                   │
│  2. registry.get("order") ─────┼──────────────────────┐           │
│                                │                      ▼           │
│                                │         ┌─────────────────────┐  │
│                                │         │ PrintStrategyRegistry│  │
│                                │         │   (Factory Pattern)  │  │
│                                │         └──────────┬──────────┘  │
│                                │                    │             │
│                                │         ┌──────────▼──────────┐  │
│                                │         │ OrderPrintStrategy  │  │
│                                │         │ (Template Method)   │  │
│  3. strategy.validate() ◄──────┼─────────┤                     │  │
│                                │         │ - validate()        │  │
│  4. strategy.buildCrystal() ◄──┼─────────┤ - buildCrystalParams│  │
│                                │         └─────────────────────┘  │
│                                │                                   │
│  5. reportsService.exportPdf(layoutCode, crystalParams)           │
│                                │                                   │
│                       ┌────────▼────────┐                         │
│                       │  SAP API Gateway │                         │
│                       │   (Puerto 60000) │                         │
│                       └────────┬────────┘                         │
│                                │                                   │
│                          PDF base64                                │
│                                │                                   │
│  6. printNodeService.print(printerId, pdf)                        │
│                                │                                   │
│                       ┌────────▼────────┐                         │
│                       │    PrintNode    │                         │
│                       └────────┬────────┘                         │
│                                │                                   │
│                           printJobId                               │
│                                │                                   │
│  7. Notificar observers: onPrintCompleted()                       │
│                                │                                   │
│  8. return { success, printJobId, message }                       │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
                        Response al cliente
```

---

## Configuración Requerida (.env)

```env
# Layouts de Crystal Reports
REPORT_LAYOUT_ORDER=RCRI0026
REPORT_LAYOUT_INVOICE=RCRI0029

# PrintNode
PRINTNODE_API_KEY=xxxxx
```

---

## Cómo Agregar Nueva Entidad

### Paso 1: Crear Strategy

Crear archivo en `strategies/nueva-entidad-print.strategy.ts`:

- Extender `BasePrintStrategy`
- Definir `entity`, `layoutCode`, `displayName`
- Implementar `validate()` y `buildCrystalParams()`

### Paso 2: Registrar en Registry

En `print-strategy.registry.ts`:

- Inyectar nueva strategy en constructor
- Agregar `this.strategies.set(strategy.entity, strategy)`

### Paso 3: Configurar layout

En `.env`:

```env
REPORT_LAYOUT_NUEVA_ENTIDAD=RCRI00XX
```

---

## Dependencias Externas

| Servicio | Puerto | Función |
|----------|--------|---------|
| SAP Service Layer | 50000 | Config de usuario (impresoras) |
| SAP API Gateway | 60000 | Generar PDF con Crystal |
| PrintNode | HTTPS | Enviar PDF a impresora física |

---

## Historial de Cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| 2026-03-24 | - | Documentación inicial |
