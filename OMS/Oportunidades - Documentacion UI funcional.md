# V2 — Simplificación UX de Oportunidades

> Propuesta de rediseño de la experiencia de usuario para la gestión de oportunidades.
> Cambios sobre la V1 documentada en `diagramas_oportunidades.md`.
> Mockup interactivo: [`mockups/v2-detalle-oportunidad.html`](mockups/v2-detalle-oportunidad.html)
> Fecha: marzo 2026

---

## Principios de diseño

### 1. Abstracción de SAP

**La interfaz abstrae la complejidad de SAP.** El vendedor no piensa en entidades (OOPR, OQUT, OACT); piensa en acciones de negocio: "tengo un cliente que quiere esto, necesito cotizarlo". El sistema crea las entidades por detrás como consecuencia de las acciones del usuario.

### 2. Modelo de UI: Record Show Page (estilo Twenty CRM)

La interfaz adopta el patrón de **Record Show Page** popularizado por [Twenty CRM](https://github.com/twentyhq/twenty), inspirado a su vez en Notion, Linear y Airtable:

| Principio                      | Descripción                                                                                                                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Inline editing**             | No existe "modo edición". Se hace clic en un campo y se edita en el lugar.                                                                                                                        |
| **Auto-save**                  | No hay botón "Guardar" ni "Actualizar". Cada cambio persiste automáticamente al perder foco del campo (on blur).                                                                                  |
| **Sin modales**                | En lugar de modales/popups, el contenido secundario se abre en un **panel lateral (split view)** dentro del área central. Ejemplo: ver detalle de un documento, una actividad o una conversación. |
| **Formulario = Vista**         | No hay separación entre "ver" y "editar". La vista de detalle ES el formulario. Los datos se completan progresivamente.                                                                           |
| **Acciones como consecuencia** | El usuario realiza acciones de negocio (agregar cliente, agregar producto, solicitar cotización). El sistema traduce esas acciones a operaciones SAP por detrás.                                  |

### Ejemplo del patrón aplicado

```
┌─────────────────────────────────────────────────────────────┐
│  ← Oportunidades  /  OPR-00452                    [Perdida] │
│  ● Lead ─── Cotización ─── Oferta ─── Confirmado ─── Cierre │
├────────────────────────────┬────────────────────────────────┤
│                            │                                │
│  Cliente                   │  (Panel lateral - split view)  │
│  [Electrónica del Sur ▾]   │                                │
│                            │  Se abre aquí el contenido     │
│  Productos                 │  secundario:                   │
│  ┌──────────────────────┐  │  - Detalle de documento        │
│  │ Monitor 27" 4K   x2  │  │  - Detalle de actividad        │
│  │ Teclado mecánico x5  │  │  - Conversación                │
│  └──────────────────────┘  │                                │
│  [+ Agregar producto]      │                                │
│                            │                                │
│  Actividades               │                                │
│  ○ Cotización #365 ─ ✓     │                                │
│  ○ Compra #372 ─ En proc.  │                                │
│                            │                                │
│  Timeline                  │                                │
│  ├ Hoy: COMP respondió     │                                │
│  ├ Ayer: VEND solicitó...   │                                │
│  └ 28/feb: Creada           │                                │
│                            │                                │
└────────────────────────────┴────────────────────────────────┘
```

---

## Cambios respecto a V1

### Lo que se mantiene

- Las 5 etapas (Lead → Cotización → Oferta → Confirmado → Cierre)
- Las transiciones automáticas por evento
- Los actores (VEND, COMP)
- El modelo de datos SAP (OOPR, OQUT, OACT, ORDR, etc.)
- La arquitectura de conversaciones en Firebase

## Flujo propuesto

### 1. Vista de oportunidades (VEND)

El vendedor accede a sus oportunidades en dos modos:

- **Lista**: tabla con filtros y ordenamiento (estado, fecha, cliente, monto)
- **Kanban**: columnas por etapa (Lead, Cotización, Oferta, Confirmado, Cierre)

Ambas vistas son del mismo dato, solo cambia la presentación. Sin cambios respecto a V1.

### 2. Nueva oportunidad = Abrir la vista de detalle vacía

No existe un formulario de "nueva oportunidad" separado. **Crear una oportunidad es abrir la vista de detalle en blanco** y empezar a completar campos. Cada dato que el usuario completa dispara la creación de la entidad correspondiente en SAP.

#### Secuencia de creación progresiva

| Paso | Acción del usuario                           | Efecto en SAP                                                         | Feedback visual                                           |
| ---- | -------------------------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------- |
| 1    | Selecciona cliente                           | `POST SalesOpportunity` → OOPR creada (Stage=Lead)                    | El registro obtiene número, la barra de etapas se activa  |
| 2    | Agrega productos de interés                  | `POST Quotation` draft (líneas `dslt_Text`) + link en OpportunityLine | Los productos aparecen en la lista, inline editables      |
| 3    | Clic en "Solicitar cotización"               | `POST Activity` (Subject="Cotizacion") + transición a Etapa 2         | Actividad aparece en la sección, se abre en panel lateral |
| 3'   | *(alternativa)* Clic en "Solicitar créditos" | `POST Activity` (Subject="Creditos")                                  | Idem                                                      |
| 4    | Escribe comentario en la actividad           | `POST Firebase comment` en `comments/OACT/{id}`                       | Mensaje aparece en el chat del panel lateral              |

**No hay botón "Guardar".** Cada campo persiste al perder foco (on blur). Si el usuario cierra a mitad de camino, lo que ya se creó queda persistido en SAP.

#### Resultado: creación en cascada

```
Vista de detalle (nueva)
  ├─ Cliente seleccionado (blur) ──→ POST SalesOpportunity (OOPR)
  ├─ Producto agregado (blur) ─────→ POST Quotation (OQUT) draft + link en OpportunityLine
  ├─ Acción de negocio (clic) ─────→ POST Activity (OACT)
  └─ Comentario enviado (enter) ───→ POST Firebase comment
```

### 3. Vista de detalle — Record Show Page

La vista de detalle es **la misma para registros nuevos y existentes**. La diferencia es solo el estado inicial (vacío vs. con datos).

#### Layout

```
┌──────────────────────────────────────────────────────────────┐
│  Header: nombre oportunidad (inline editable) + acciones     │
│  Barra de etapas: ● Lead ── Cotización ── Oferta ── ...     │
├─────────────── Área central ─────────────────────────────────┤
│                            │                                 │
│  CONTENIDO PRINCIPAL       │  PANEL LATERAL (split view)     │
│                            │  Se abre al hacer clic en un    │
│  Campos inline editables:  │  elemento del contenido ppal.   │
│  - Cliente                 │                                 │
│  - Vendedor                │  Muestra:                       │
│  - Fecha cierre estimada   │  - Detalle de actividad + chat  │
│                            │  - Detalle de documento (OQUT)  │
│  Productos (lista inline)  │  - Conversación con cliente     │
│  [+ Agregar]               │                                 │
│                            │  El panel se cierra con ✕ o     │
│  Actividades               │  Escape, volviendo al ancho     │
│  Documentos                │  completo del contenido ppal.   │
│                            │                                 │
│  Timeline / Historial      │                                 │
│                            │                                 │
└────────────────────────────┴─────────────────────────────────┘
```

#### Comportamiento de auto-save

| Tipo de campo             | Trigger de guardado     | Operación SAP                       |
| ------------------------- | ----------------------- | ----------------------------------- |
| Texto, select, fecha      | On blur (pierde foco)   | PATCH a la entidad correspondiente  |
| Línea de producto         | On blur de la fila      | POST/PATCH línea en OQUT            |
| Acción de negocio (botón) | On click                | POST nueva entidad (Activity, etc.) |
| Comentario/chat           | On enter / botón enviar | POST Firebase comment               |

#### Panel lateral (reemplazo de modales)

En lugar de abrir modales, al hacer clic en un elemento se abre un **panel lateral** que divide el área central:

| Elemento clickeado | Contenido del panel lateral |
|--------------------|---------------------------|
| Una actividad (Cotización/Compra) | Detalle de la actividad + chat VEND↔COMP |
| Un documento (OQUT) | Vista del documento con líneas y totales |
| "Conversación con cliente" | Chat VEND↔CLI (`comments/OQUT/{id}`) |

El panel lateral:
- Ocupa ~40-50% del ancho del área central
- Se cierra con ✕ o Escape
- Solo uno abierto a la vez (abrir otro reemplaza el actual)
- Tiene su propio scroll independiente

### 4. Gestión de actividades (COMP)

Mockups: [`v2-gestion-actividades.html`](mockups/v2-gestion-actividades.html), [`v2-detalle-actividad.html`](mockups/v2-detalle-actividad.html)

#### Lista de actividades

Vista de lista con filtros para que COMP gestione las actividades pendientes:

- **Filtros por estado**: Abierta, En Proceso, Cerrada
- **Filtros por tipo**: Cotización, Compra, Créditos
- **Filtro por asignación**: Mis actividades, Sin asignar
- **Auto-asignación**: el que toma la actividad hace clic en "Tomar" y se auto-asigna. La actividad pasa a "En Proceso".

#### Tipos de actividad

| Tipo       | Propósito                                               | Quién la inicia  | Quién la trabaja |
| ---------- | ------------------------------------------------------- | ---------------- | ---------------- |
| Cotización | Solicitar precio/disponibilidad a Compras               | VEND             | COMP             |
| Compra     | Solicitar abastecimiento (PO, recepción, formalización) | VEND (o sistema) | COMP             |
| Créditos   | Solicitar evaluación de crédito del cliente             | VEND             | COMP/Finanzas    |

En la oportunidad, los tipos disponibles se muestran como **botones explícitos** en la sección "Acciones". No se requiere haber agregado productos para solicitar una actividad.

#### Detalle de actividad (3 columnas)

```
┌──────────────────┬────────────────────────────────────────────┐
│ INFO LATERAL     │  CONVERSACIÓN                              │
│                  │                                            │
│ Estado: [select] │  [VEND] Necesito cotización para...        │
│ Asignado: LC     │  [COMP] Recibido, consulto proveedores     │
│                  │  [COMP] Monitor: $320, Teclado: $85        │
│ CLIENTE          │  [VEND] Perfecto, armen el draft           │
│ Electrónica...   │                                            │
│ C001             │                                            │
│ Roberto Salazar  │                                            │
│ +591 3 344-5678  │  ─────────────────────────────────────     │
│                  │  [role ▾] [Escribe un mensaje...] [Enviar] │
│ PRODUCTOS        │                                            │
│ ┌──────────────┐ │                                            │
│ │ Monitor x2   │ │                                            │
│ │ Teclado x5   │ │                                            │
│ │ [+ Agregar]  │ │                                            │
│ └──────────────┘ │                                            │
│ Total: $1,384.50 │                                            │
└──────────────────┴────────────────────────────────────────────┘
```

#### Reglas de estado

| Estado         | Descripción                                 |
| -------------- | ------------------------------------------- |
| **Abierta**    | Recién creada, nadie la ha tomado           |
| **En Proceso** | Alguien se auto-asignó o COMP respondió     |
| **Cerrada**    | COMP o VEND la cerraron (trabajo terminado) |

#### Reapertura automática

Si una actividad está **cerrada** y alguien envía un nuevo mensaje, la actividad se **reabre automáticamente** (vuelve a "En Proceso"). Esto permite re-negociación sin crear actividades nuevas.

Ejemplo: VEND cierra una cotización, luego el cliente pide un cambio. VEND vuelve a comentar en la misma actividad → se reabre.

---

## Implicaciones técnicas

### Backend (NestJS)

En lugar de un endpoint monolítico, el backend expone **operaciones granulares** que el frontend invoca progresivamente:

```
# Paso 1: Usuario selecciona cliente → frontend invoca:
POST /api/opportunities
Body: 
{
    "CardCode": "C32851",
    "OpportunityName": "Venta de Licencias",
    "SalesPerson": 100,
    "SalesOpportunitiesLines": [
        {
            "MaxLocalTotal": 0.1,
            "MaxSystemTotal": 0.1,
            "StageKey": 1
        }
    ]
}

→ Respuesta: { opportunityId: 123, status: "Lead" }

# Paso 2: Usuario agrega producto → frontend invoca:
POST /api/opportunities/123/quotation
Body: { lines: [{ description: "Monitor 27\" 4K", quantity: 2 }] }
→ Respuesta: { docEntry: 456, docNum: "COT-04862" }

# Paso 3: Usuario hace clic en "Solicitar cotización" → frontend invoca:
POST /api/opportunities/123/activities
Body: { type: "cotizacion", comment: "Necesito precio para..." }
→ Respuesta: { activityCode: 789 }

# Auto-save de campos individuales:
PATCH /api/opportunities/123
HEADER B1S-ReplaceCollectionsOnPatch = tue
Body: 
{
    "CardCode": "C32851",
    "OpportunityName": "Venta de Licencias",
    "SalesPerson": 100,
    "SalesOpportunitiesLines": [
        {
            "LineNum": 0,
            "MaxLocalTotal": 15000,
            "StageKey": 1,            
            "Status": "so_Open"
        }
    ]
}
```

Cada endpoint es atómico: si falla, el usuario ve el error en ese campo/acción específica y puede reintentar.

### Frontend (Next.js + Zustand)

- **Un solo componente de vista de detalle** sirve para nuevo y existente
- Estado local con Zustand: `opportunityStore` con el registro y sus entidades relacionadas
- Cada campo usa un hook `useAutoSave(field, value)` que hace PATCH on blur
- El panel lateral es un componente `<SidePanel>` que renderiza contenido según contexto
- Optimistic updates: el campo se actualiza visualmente de inmediato, rollback si el PATCH falla

### Manejo de errores (por campo)

| Escenario | Comportamiento |
|-----------|---------------|
| PATCH falla en un campo | El campo vuelve al valor anterior + indicador de error inline |
| POST falla al crear entidad | Botón de acción muestra error + permite reintentar |
| Pérdida de conexión | Indicador global "sin conexión", cola de cambios pendientes |

---

## Aclaraciones

1.  Los Solicitud de credito es un tipo de actividad con su propio tablero igual que cotizacion/compra
2.  El panel lateral es de un solo nivel. Solo muestra un detalle a la vez. 
3.  Al crear la Oportunidad los campo obligatorios son:
	   - CardCode: Codigo del cliente
	   - SalesPerson: Codigo del vendedor
	   - SalesOpportunitiesLines: Etapa de la oportunidad, en la creacion es el stage 1 y debe tener un monto. Se puede utilizar el valor 0.1 como valor predeterminado para cumplir con la restriccion. 
		   {
		   - MaxLocalTotal: 
		   - StageKey

4. En la sección de Artefactos de la oportunidad se muestran como botón los tipos de objetos que se pueden crear. Por ejemplo En la sección de actividades se listan los tipos de actividades que se pueden solicitar. En la sección de documentos se lista los tipos de documentos que se pueden agregar a la oportunidad, Cotización etc. 



---

## Resumen visual

```
V1: [Nueva oportunidad] → Formulario → Guardar → Ir a detalle → Crear OQUT → Crear OACT → Comentar
    (múltiples pantallas, botones de guardar, modales)

V2: [Nueva oportunidad] → Se abre detalle vacío → Selecciona cliente (auto-save → crea OOPR)
    → Agrega productos (auto-save → crea OQUT) → "Solicitar cotización" (crea OACT)
    → Escribe comentario en panel lateral (Firebase)
    (una sola pantalla, sin botón guardar, sin modales)
```

**El usuario completa datos naturalmente. El sistema persiste y crea entidades como consecuencia.**
