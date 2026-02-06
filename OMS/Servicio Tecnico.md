# Servicio Técnico - Guía de Implementación Pragmática

**Versión:** 1.0
**Fecha:** 4 de Febrero de 2026
**Basado en:** UML-Servicio-Tecnico.md + Patrones de Inventory Kanban

---

## 1. Event Storming - Eventos de Dominio

### 1.1 Flujo Principal de Eventos

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           EVENT STORMING - SERVICIO TÉCNICO                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

Tiempo ────────────────────────────────────────────────────────────────────────────────►

┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  CLIENTE    │    │  EQUIPO     │    │ SERVICE     │    │  TÉCNICO    │
│  BUSCADO    │───►│ REGISTRADO  │───►│ CALL        │───►│ ASIGNADO    │
│             │    │             │    │ CREADO      │    │             │
│ 🟧 Orange   │    │ 🟧 Orange   │    │ 🟧 Orange   │    │ 🟦 Blue     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                            │
        ┌───────────────────────────────────┘
        │
        ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ DIAGNÓSTICO │    │ PRESUPUESTO │    │ PRESUPUESTO │    │ REPARACIÓN  │
│ REGISTRADO  │───►│ CREADO      │───►│ APROBADO    │───►│ INICIADA    │
│             │    │             │    │             │    │             │
│ 🟦 Blue     │    │ 🟪 Purple   │    │ 🟪 Purple   │    │ 🟩 Green    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                          │                                     │
                          ▼                                     ▼
                   ┌─────────────┐                       ┌─────────────┐
                   │ PRESUPUESTO │                       │ REPUESTOS   │
                   │ RECHAZADO   │                       │ SOLICITADOS │
                   │             │                       │             │
                   │ 🟥 Red      │                       │ 🟨 Yellow   │
                   └─────────────┘                       └─────────────┘
                          │                                     │
                          │                                     ▼
                          │                              ┌─────────────┐
                          │                              │ REPUESTOS   │
                          │                              │ RECIBIDOS   │
                          │                              │             │
                          │                              │ 🟨 Yellow   │
                          │                              └─────────────┘
                          │                                     │
                          ▼                                     ▼
                   ┌─────────────┐                       ┌─────────────┐
                   │ CASO        │◄──────────────────────│ REPARACIÓN  │
                   │ CERRADO     │                       │ COMPLETADA  │
                   │             │                       │             │
                   │ ⬛ Gray     │                       │ 🟩 Green    │
                   └─────────────┘                       └─────────────┘
                          ▲                                     │
                          │                                     ▼
                          │                              ┌─────────────┐
                          │                              │ EQUIPO      │
                          └──────────────────────────────│ ENTREGADO   │
                                                         │             │
                                                         │ 🟩 Green    │
                                                         └─────────────┘
```

### 1.2 Comandos y Eventos

| Comando (Acción)     | Evento Resultante     | Actor          | Agregado        |
| -------------------- | --------------------- | -------------- | --------------- |
| BuscarCliente        | ClienteBuscado        | Recepción      | BusinessPartner |
| RegistrarEquipo      | EquipoRegistrado      | Recepción      | EquipmentCard   |
| CrearServiceCall     | ServiceCallCreado     | Recepción      | ServiceCall     |
| AsignarTécnico       | TécnicoAsignado       | Jefe Taller    | ServiceCall     |
| RegistrarDiagnóstico | DiagnósticoRegistrado | Técnico        | ServiceCall     |
| CrearPresupuesto     | PresupuestoCreado     | Jefe Taller    | ServiceCall     |
| AprobarPresupuesto   | PresupuestoAprobado   | Jefe Taller    | ServiceCall     |
| RechazarPresupuesto  | PresupuestoRechazado  | Jefe Taller    | ServiceCall     |
| IniciarReparación    | ReparaciónIniciada    | Técnico        | ServiceCall     |
| SolicitarRepuestos   | RepuestosSolicitados  | Técnico        | ServiceCall     |
| RecibirRepuestos     | RepuestosRecibidos    | Sistema        | ServiceCall     |
| CompletarReparación  | ReparaciónCompletada  | Técnico        | ServiceCall     |
| EntregarEquipo       | EquipoEntregado       | Recepción      | ServiceCall     |
| CerrarCaso           | CasoCerrado           | Cajero/Sistema | ServiceCall     |
| AgregarActividad     | ActividadAgregada     | Cualquiera     | Activity        |

### 1.3 Read Models (Vistas)

| Read Model               | Propósito                                    | Consumidor        |
| ------------------------ | -------------------------------------------- | ----------------- |
| ServiceKanbanView        | Vista Kanban por estados                     | Todos los roles   |
| ServiceCallDetailView    | Detalle completo del caso                    | Todos los roles   |
| TechnicianWorkloadView   | Carga de trabajo por técnico                 | Jefe Taller       |
| CustomerServiceHistory   | Historial de servicio del cliente            | Recepción         |
| EquipmentServiceHistory  | Historial de servicio del equipo             | Técnico           |
| PendingBudgetsView       | Presupuestos pendientes de aprobación        | Jefe Taller       |
| PendingDeliveriesView    | Equipos listos para entregar                 | Recepción, Cajero |

---

## 2. State Machine (Compatible con XState)

### 2.1 Definición Formal de Estados

```typescript
// types/service/service-call-states.ts

export const SERVICE_CALL_STATES = {
  ABIERTO: -3,
  DIAGNOSTICO: 2,
  PRESUPUESTO: 3,
  EJECUCION: 4,
  ESPERANDO_REPUESTOS: 5,
  TERMINADO: 1,
  CERRADO: -1,
} as const;

export type ServiceCallStateValue = typeof SERVICE_CALL_STATES[keyof typeof SERVICE_CALL_STATES];

export const SERVICE_CALL_STATE_LABELS: Record<ServiceCallStateValue, string> = {
  [-3]: 'Abierto',
  [2]: 'En Diagnóstico',
  [3]: 'Presupuesto',
  [4]: 'En Ejecución',
  [5]: 'Esperando Repuestos',
  [1]: 'Terminado',
  [-1]: 'Cerrado',
};

export const SERVICE_CALL_STATE_COLORS: Record<ServiceCallStateValue, string> = {
  [-3]: '#f59e0b', // amber - Abierto
  [2]: '#3b82f6',  // blue - Diagnóstico
  [3]: '#8b5cf6',  // purple - Presupuesto
  [4]: '#10b981',  // emerald - Ejecución
  [5]: '#eab308',  // yellow - Esperando Repuestos
  [1]: '#22c55e',  // green - Terminado
  [-1]: '#6b7280', // gray - Cerrado
};
```

### 2.2 Máquina de Estados XState

```typescript
// machines/serviceCallMachine.ts
import { createMachine, assign } from 'xstate';

export const serviceCallMachine = createMachine({
  id: 'serviceCall',
  initial: 'abierto',
  context: {
    serviceCallId: null,
    technicianCode: null,
    diagnostico: null,
    presupuesto: null,
    hasWarranty: false,
    transferRequestId: null,
  },
  states: {
    abierto: {
      on: {
        ASIGNAR_TECNICO: {
          target: 'diagnostico',
          actions: assign({ technicianCode: (_, event) => event.technicianCode }),
        },
        CANCELAR: 'cerrado',
      },
    },
    diagnostico: {
      on: {
        REGISTRAR_DIAGNOSTICO: [
          {
            target: 'ejecucion',
            cond: 'tieneGarantia',
            actions: assign({ diagnostico: (_, event) => event.diagnostico }),
          },
          {
            target: 'presupuesto',
            actions: assign({ diagnostico: (_, event) => event.diagnostico }),
          },
        ],
      },
    },
    presupuesto: {
      on: {
        APROBAR_PRESUPUESTO: {
          target: 'ejecucion',
          actions: assign({ presupuesto: (_, event) => event.presupuesto }),
        },
        RECHAZAR_PRESUPUESTO: 'cerrado',
      },
    },
    ejecucion: {
      on: {
        SOLICITAR_REPUESTOS: {
          target: 'esperandoRepuestos',
          actions: assign({ transferRequestId: (_, event) => event.transferRequestId }),
        },
        COMPLETAR_REPARACION: 'terminado',
      },
    },
    esperandoRepuestos: {
      on: {
        REPUESTOS_RECIBIDOS: 'ejecucion',
      },
    },
    terminado: {
      on: {
        ENTREGAR_EQUIPO: 'cerrado',
      },
    },
    cerrado: {
      type: 'final',
    },
  },
}, {
  guards: {
    tieneGarantia: (context) => context.hasWarranty,
  },
});
```

### 2.3 Transiciones Permitidas (Matriz)

```typescript
// constants/service-transitions.ts

export const VALID_TRANSITIONS: Record<ServiceCallStateValue, ServiceCallStateValue[]> = {
  [-3]: [2, -1],        // Abierto → Diagnóstico, Cerrado
  [2]: [3, 4],          // Diagnóstico → Presupuesto, Ejecución (garantía)
  [3]: [4, -1],         // Presupuesto → Ejecución (aprobado), Cerrado (rechazado)
  [4]: [5, 1],          // Ejecución → Esperando Repuestos, Terminado
  [5]: [4],             // Esperando Repuestos → Ejecución
  [1]: [-1],            // Terminado → Cerrado
  [-1]: [],             // Cerrado (final)
};

export function canTransition(from: ServiceCallStateValue, to: ServiceCallStateValue): boolean {
  return VALID_TRANSITIONS[from]?.includes(to) ?? false;
}
```

---

## 3. Desarrollo por Fases (MVP)

### Fase 1: MVP Core (2-3 semanas)

**Objetivo:** Flujo básico funcional de ingreso a cierre.

#### User Stories - Fase 1

```gherkin
# US-001: Crear Service Call
Como Recepción
Quiero registrar el ingreso de un equipo
Para iniciar el proceso de servicio técnico

Criterios de Aceptación:
- [ ] Puedo buscar cliente por nombre, RUC o teléfono
- [ ] Puedo seleccionar equipo existente o crear nuevo
- [ ] Puedo ingresar motivo, accesorios y contraseña
- [ ] Se crea ServiceCall en SAP con estado ABIERTO (-3)
- [ ] Se muestra en columna "Abiertos" del Kanban

# US-002: Ver Kanban de Service Calls
Como cualquier usuario autorizado
Quiero ver el tablero Kanban de casos
Para visualizar el estado de todos los casos

Criterios de Aceptación:
- [ ] Veo 6 columnas (Abierto, Diagnóstico, Presupuesto, Ejecución, Terminado, Cerrado)
- [ ] Cada tarjeta muestra: #Caso, Cliente, Equipo, Fecha, Técnico asignado
- [ ] Puedo filtrar por fecha, técnico, sucursal
- [ ] Click en tarjeta abre modal de detalle

# US-003: Asignar Técnico
Como Jefe de Taller
Quiero asignar un técnico a un caso abierto
Para que inicie el diagnóstico

Criterios de Aceptación:
- [ ] Puedo seleccionar técnico de lista de empleados activos
- [ ] El caso cambia a estado DIAGNÓSTICO (2)
- [ ] Se registra Activity con la asignación
- [ ] El técnico ve el caso en su lista de asignados

# US-004: Registrar Diagnóstico
Como Técnico
Quiero registrar el diagnóstico del equipo
Para informar qué tiene y qué necesita

Criterios de Aceptación:
- [ ] Puedo escribir descripción técnica del problema
- [ ] Puedo indicar si requiere repuestos
- [ ] Puedo indicar si tiene garantía vigente
- [ ] Se actualiza U_Diagnostico_Tecnico en SAP
- [ ] Caso avanza a PRESUPUESTO (3) o EJECUCIÓN (4) según garantía

# US-005: Cambiar Estado del Caso
Como usuario autorizado según mi rol
Quiero cambiar el estado del caso
Para reflejar el avance del trabajo

Criterios de Aceptación:
- [ ] Solo puedo cambiar a estados válidos según la máquina de estados
- [ ] Se validan permisos según mi rol
- [ ] Se registra Activity con el cambio
- [ ] La tarjeta se mueve a la columna correspondiente
```

#### Componentes a Crear - Fase 1

**Frontend:**
```
src/
├── app/(DashboardLayout)/service/
│   └── page.tsx                      # Página principal Kanban
│
├── components/service/
│   ├── ServiceKanbanBoard/
│   │   ├── index.tsx                 # Board principal (basado en InventoryKanbanBoard)
│   │   ├── ServiceColumn.tsx         # Columna del Kanban
│   │   └── ServiceCard.tsx           # Tarjeta de caso
│   │
│   └── ServiceCallModal/
│       ├── index.tsx                 # Modal principal
│       ├── CreateServiceCallForm.tsx # Formulario de creación
│       ├── ServiceCallDetail.tsx     # Vista de detalle
│       └── StatusActions.tsx         # Acciones según estado
│
├── context/
│   └── ServiceKanbanContext.tsx      # Estado global del Kanban
│
├── hooks/service/
│   ├── useServiceCalls.ts            # Lista de casos
│   ├── useServiceCall.ts             # Detalle de un caso
│   └── useServicePermissions.ts      # Permisos por rol
│
└── types/service/
    ├── service-call.types.ts
    └── service-kanban.types.ts
```

**Backend:**
```
src/modules/
└── service-calls/
    ├── service-calls.module.ts
    ├── service-calls.controller.ts
    ├── service-calls.service.ts
    ├── dto/
    │   ├── create-service-call.dto.ts
    │   ├── update-service-call.dto.ts
    │   └── change-status.dto.ts
    ├── interfaces/
    │   └── service-call.interface.ts
    └── constants/
        ├── status.constants.ts
        └── transitions.constants.ts
```

---

### Fase 2: Presupuestos y Actividades (1-2 semanas)

#### User Stories - Fase 2

```gherkin
# US-006: Crear Presupuesto
Como Jefe de Taller
Quiero crear un presupuesto para el cliente
Para que apruebe o rechace la reparación

Criterios de Aceptación:
- [ ] Puedo agregar líneas de repuestos con precios
- [ ] Puedo agregar costo de mano de obra
- [ ] Se calcula total automáticamente
- [ ] Se actualiza U_Presupuesto_Tentativo en SAP
- [ ] Se notifica al cliente (futuro)

# US-007: Aprobar/Rechazar Presupuesto
Como Jefe de Taller
Quiero registrar la respuesta del cliente al presupuesto
Para continuar o cerrar el caso

Criterios de Aceptación:
- [ ] Botón "Cliente Aprueba" → caso a EJECUCIÓN
- [ ] Botón "Cliente Rechaza" → caso a CERRADO
- [ ] Se registra Activity con la decisión
- [ ] Se registra fecha y hora de la respuesta

# US-008: Agregar Actividad/Seguimiento
Como usuario autorizado
Quiero registrar notas y seguimientos del caso
Para mantener historial completo

Criterios de Aceptación:
- [ ] Puedo agregar notas de tipo: Llamada, Reunión, Tarea, Nota, Otro
- [ ] Se muestra timeline de actividades en el modal
- [ ] Se crea Activity vinculada al ServiceCall en SAP
```

#### Componentes a Crear - Fase 2

```
src/components/service/
├── BudgetSection/
│   ├── index.tsx
│   ├── BudgetLineForm.tsx
│   └── BudgetSummary.tsx
│
└── ActivityTimeline/
    ├── index.tsx
    ├── ActivityItem.tsx
    └── AddActivityForm.tsx
```

---

### Fase 3: Integración con Inventario (1-2 semanas)

#### User Stories - Fase 3

```gherkin
# US-009: Solicitar Repuestos desde Caso
Como Técnico
Quiero solicitar repuestos para una reparación
Para que se gestione la transferencia desde almacén

Criterios de Aceptación:
- [ ] Desde el caso puedo abrir modal de transferencia
- [ ] El modal viene pre-llenado con referencia al ServiceCall
- [ ] Se crea borrador de transferencia vinculado al caso
- [ ] El caso cambia a estado ESPERANDO_REPUESTOS (5)
- [ ] Se registra en ServiceCallInventoryExpenses

# US-010: Ver Documentos Vinculados
Como usuario autorizado
Quiero ver todos los documentos relacionados al caso
Para tener contexto completo

Criterios de Aceptación:
- [ ] Veo lista de transferencias solicitadas
- [ ] Veo órdenes de compra (si aplica)
- [ ] Veo facturas emitidas
- [ ] Puedo navegar a cada documento
```

---

### Fase 4: Entrega y Facturación (1-2 semanas)

#### User Stories - Fase 4

```gherkin
# US-011: Registrar Entrega de Equipo
Como Recepción o Cajero
Quiero registrar la entrega del equipo al cliente
Para cerrar el caso

Criterios de Aceptación:
- [ ] Verifico que el caso esté en TERMINADO
- [ ] Registro firma/conformidad del cliente (futuro)
- [ ] El caso cambia a CERRADO (-1)
- [ ] Se genera comprobante de entrega (opcional)

# US-012: Generar Factura
Como Cajero
Quiero generar la factura del servicio
Para cobrar al cliente

Criterios de Aceptación:
- [ ] Puedo crear factura desde el caso
- [ ] Se incluyen repuestos y mano de obra del presupuesto
- [ ] Se vincula factura al ServiceCall (InventoryExpenses)
- [ ] Se puede usar flujo de cobro existente (QR, transferencia, etc.)
```

---

## 4. API Contract First (OpenAPI)

### 4.1 Endpoints Principales

```yaml
# openapi.yaml (extracto)
paths:
  /service-calls:
    get:
      summary: Listar Service Calls para Kanban
      parameters:
        - name: status
          in: query
          schema:
            type: integer
            enum: [-3, 2, 3, 4, 5, 1, -1]
        - name: assignee
          in: query
          schema:
            type: integer
        - name: branch
          in: query
          schema:
            type: string
        - name: dateFrom
          in: query
          schema:
            type: string
            format: date
        - name: dateTo
          in: query
          schema:
            type: string
            format: date
      responses:
        200:
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ServiceCallKanbanResponse'

    post:
      summary: Crear Service Call
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateServiceCallDto'
      responses:
        201:
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ServiceCall'

  /service-calls/{id}:
    get:
      summary: Obtener detalle de Service Call
      responses:
        200:
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ServiceCallDetail'

    patch:
      summary: Actualizar Service Call
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateServiceCallDto'

  /service-calls/{id}/status:
    patch:
      summary: Cambiar estado del Service Call
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [status]
              properties:
                status:
                  type: integer
                  enum: [-3, 2, 3, 4, 5, 1, -1]
                notes:
                  type: string
      responses:
        200:
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ServiceCall'
        400:
          description: Transición de estado inválida

  /service-calls/{id}/activities:
    get:
      summary: Listar actividades del caso
    post:
      summary: Agregar actividad al caso

components:
  schemas:
    ServiceCallKanbanResponse:
      type: object
      properties:
        abiertos:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'
        diagnostico:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'
        presupuesto:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'
        ejecucion:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'
        esperandoRepuestos:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'
        terminados:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'
        cerrados:
          type: array
          items:
            $ref: '#/components/schemas/ServiceCallCard'

    ServiceCallCard:
      type: object
      properties:
        ServiceCallID:
          type: integer
        CustomerCode:
          type: string
        CustomerName:
          type: string
        ItemCode:
          type: string
        ItemDescription:
          type: string
        Subject:
          type: string
        Status:
          type: integer
        Priority:
          type: string
          enum: [scp_Low, scp_Medium, scp_High]
        CreationDate:
          type: string
          format: date
        TechnicianCode:
          type: integer
        TechnicianName:
          type: string
        U_Sucursal:
          type: string

    CreateServiceCallDto:
      type: object
      required:
        - CustomerCode
        - Subject
        - U_Sucursal
      properties:
        CustomerCode:
          type: string
        InternalSerialNum:
          type: string
        ItemCode:
          type: string
        Subject:
          type: string
        Priority:
          type: string
          enum: [scp_Low, scp_Medium, scp_High]
          default: scp_Medium
        U_Sucursal:
          type: string
        U_Accesorios:
          type: string
        U_Contrasenha:
          type: string
        Remarks:
          type: string
```

---

## 5. Columnas del Kanban

### 5.1 Configuración de Columnas

```typescript
// types/service/service-kanban.types.ts

export const SERVICE_KANBAN_COLUMNS = [
  {
    id: 'abiertos',
    title: 'Abiertos',
    status: -3,
    color: '#f59e0b', // amber
    description: 'Casos recién ingresados, sin técnico asignado',
  },
  {
    id: 'diagnostico',
    title: 'En Diagnóstico',
    status: 2,
    color: '#3b82f6', // blue
    description: 'Técnico evaluando el equipo',
  },
  {
    id: 'presupuesto',
    title: 'Presupuesto',
    status: 3,
    color: '#8b5cf6', // purple
    description: 'Esperando aprobación del cliente',
  },
  {
    id: 'ejecucion',
    title: 'En Ejecución',
    status: 4,
    color: '#10b981', // emerald
    description: 'Reparación en proceso',
  },
  {
    id: 'esperandoRepuestos',
    title: 'Esperando Repuestos',
    status: 5,
    color: '#eab308', // yellow
    description: 'Aguardando transferencia de repuestos',
  },
  {
    id: 'terminados',
    title: 'Terminados',
    status: 1,
    color: '#22c55e', // green
    description: 'Listos para entregar al cliente',
  },
] as const;

// Nota: CERRADO (-1) no se muestra como columna,
// se accede via filtro o historial
```

### 5.2 Datos de la Tarjeta

```typescript
export interface ServiceCallCard {
  ServiceCallID: number;
  CustomerCode: string;
  CustomerName: string;
  ItemCode?: string;
  ItemDescription?: string;
  InternalSerialNum?: string;
  Subject: string;
  Status: ServiceCallStateValue;
  Priority: 'scp_Low' | 'scp_Medium' | 'scp_High';
  CreationDate: string;
  TechnicianCode?: number;
  TechnicianName?: string;
  U_Sucursal: string;
  // Calculados
  DaysOpen: number;
  HasPendingTransfer: boolean;
}
```

---

## 6. Integración con SAP

### 6.1 Entidades SAP Involucradas

| Entidad SAP              | Endpoint Service Layer                | Uso                           |
| ------------------------ | ------------------------------------- | ----------------------------- |
| ServiceCalls             | `/ServiceCalls`                       | CRUD principal                |
| Activities               | `/Activities`                         | Seguimientos y notas          |
| CustomerEquipmentCards   | `/CustomerEquipmentCards`             | Tarjetas de equipo            |
| ServiceContracts         | `/ServiceContracts`                   | Verificar garantías           |
| BusinessPartners         | `/BusinessPartners`                   | Datos de clientes             |
| EmployeesInfo            | `/EmployeesInfo`                      | Lista de técnicos             |
| Items                    | `/Items`                              | Productos/repuestos           |
| StockTransferRequests    | `/InventoryTransferRequests`          | Solicitud de repuestos        |

### 6.2 Campos Personalizados (UDFs)

```typescript
// Campos que deben existir en SAP para ServiceCalls
const REQUIRED_UDFS = {
  U_Sucursal: 'string',           // Sucursal donde se ingresó
  U_Accesorios: 'string',         // Lista de accesorios entregados
  U_Contrasenha: 'string',        // Contraseña del equipo (si aplica)
  U_Diagnostico_Tecnico: 'string', // Diagnóstico técnico detallado
  U_Presupuesto_Tentativo: 'decimal', // Monto del presupuesto
};
```

---

## 7. Checklist de Implementación

### Fase 1 - MVP Core

**Backend:**
- [ ] Crear módulo `service-calls` en NestJS
- [ ] Implementar `GET /service-calls` (lista para Kanban)
- [ ] Implementar `POST /service-calls` (crear caso)
- [ ] Implementar `GET /service-calls/:id` (detalle)
- [ ] Implementar `PATCH /service-calls/:id/status` (cambiar estado)
- [ ] Validar transiciones de estado
- [ ] Implementar filtros por fecha, técnico, sucursal

**Frontend:**
- [ ] Crear página `/service`
- [ ] Crear `ServiceKanbanContext`
- [ ] Crear `ServiceKanbanBoard` (basado en InventoryKanbanBoard)
- [ ] Crear `ServiceCard` (tarjeta del Kanban)
- [ ] Crear `ServiceCallModal` (modal de detalle/creación)
- [ ] Crear `CreateServiceCallForm`
- [ ] Implementar `useServiceCalls` hook
- [ ] Implementar `useServicePermissions` hook

### Fase 2 - Presupuestos y Actividades

**Backend:**
- [ ] Implementar `GET /service-calls/:id/activities`
- [ ] Implementar `POST /service-calls/:id/activities`
- [ ] Agregar campo U_Presupuesto_Tentativo a updates

**Frontend:**
- [ ] Crear `ActivityTimeline` component
- [ ] Crear `AddActivityForm`
- [ ] Crear `BudgetSection` component
- [ ] Integrar en `ServiceCallModal`

### Fase 3 - Integración Inventario

**Backend:**
- [ ] Modificar transferencias para aceptar ServiceCallID
- [ ] Implementar vinculación en InventoryExpenses

**Frontend:**
- [ ] Integrar `TransferModal` desde casos
- [ ] Mostrar transferencias vinculadas en detalle

### Fase 4 - Entrega y Facturación

**Backend:**
- [ ] Implementar vinculación de facturas

**Frontend:**
- [ ] Crear flujo de entrega
- [ ] Integrar con módulo de facturación existente

---

## 8. Referencias

- [UML-Servicio-Tecnico.md](./UML-Servicio-Tecnico.md) - Documento UML completo
- [SAP Service Layer - ServiceCalls](https://help.sap.com/docs/SAP_BUSINESS_ONE) - Documentación oficial
- [XState Documentation](https://xstate.js.org/docs/) - State machines
- [InventoryKanbanBoard](../../src/app/components/apps/inventory-kanban/) - Referencia de implementación

---

## 9. Ejemplos de Respuestas SAP (Reales)

### 9.1 ServiceCall - Detalle Completo

```json
{
  "ServiceCallID": 70,
  "Subject": "Cliente manifiesta que el equipo no enciende",
  "CustomerCode": "C03200",
  "CustomerName": "MACHAIN DIETZE, MARIA GABRIELA",
  "InternalSerialNum": "NXAASAA00422117AAD3400",
  "ContractID": 1235,
  "ContractEndDate": "2024-02-07T00:00:00Z",
  "ItemCode": "01016",
  "ItemDescription": "NOTEBOOK ACER ASPIRE 5 A515-56-32DK (12GB, 384GB SSD)",
  "Status": -1,
  "Priority": "scp_Low",
  "CallType": 2,
  "ProblemType": 2,
  "AssigneeCode": 12,
  "Description": "Se verifico que el equipo tiene pantalla rota,",
  "TechnicianCode": 3,
  "Resolution": "Cambio de pantalla 1.000.000\rPresupuesto no aprobado",
  "CreationDate": "2023-07-21T00:00:00Z",
  "CreationTime": "13:37:00",
  "BPPhone1": "0981618932",
  "BPeMail": "",
  "U_Accesorios": "cargador con cable usb",
  "U_Contrasenha": "contraseña de prueba",
  "U_DescTarjeta": "NOTEBOOK ACER ASPIRE 5 A515-56-32DK (12GB, 384GB SSD)",
  "U_Sucursal": "1",
  "U_Diagnostico_Tecnico": null,
  "U_Presupuesto_Tentativo": null,
  "ServiceCallActivities": [],
  "ServiceCallInventoryExpenses": [
    {
      "LineNum": 0,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_Order",
      "DocumentPostingDate": "2023-07-21T00:00:00Z",
      "DocumentNumber": 824,
      "StockTransferDirection": null,
      "DocEntry": 3701
    }
  ],
  "ServiceCallSolutions": []
}
```

### 9.2 ServiceCallStatus - Estados Disponibles

```json
{
  "value": [
    { "StatusId": -3, "Name": "Abierto", "Description": "En tratamiento" },
    { "StatusId": -2, "Name": "Pendiente", "Description": "Pendiente" },
    { "StatusId": -1, "Name": "Cerrado", "Description": "Cerrado" },
    { "StatusId": 1, "Name": "Terminado", "Description": null },
    { "StatusId": 2, "Name": "Diagnóstico", "Description": null },
    { "StatusId": 3, "Name": "Presupuesto", "Description": null },
    { "StatusId": 4, "Name": "Ejecución", "Description": null },
    { "StatusId": 5, "Name": "Esperando Repuestos", "Description": null }
  ]
}
```

### 9.3 ServiceCallTypes - Tipos de Llamada

```json
{
  "value": [
    { "CallTypeID": 1, "Name": "Garantía" },
    { "CallTypeID": 2, "Name": "Reparación" },
    { "CallTypeID": 3, "Name": "Servicio Externo" }
  ]
}
```

### 9.4 ServiceCallOrigins - Orígenes

```json
{
  "value": [
    { "OriginID": -3, "Name": "Correo electrónico" },
    { "OriginID": -2, "Name": "Número de teléfono" },
    { "OriginID": -1, "Name": "Web" }
  ]
}
```

### 9.5 ServiceCallProblemTypes - Tipos de Problema

```json
{
  "value": [
    { "ProblemTypeID": 1, "Name": "Problema Pantalla" },
    { "ProblemTypeID": 2, "Name": "Mant. Completo" }
  ]
}
```

### 9.6 ServiceCallProblemSubTypes - Estado del Equipo

```json
{
  "value": [
    { "ProblemSubTypeID": 1, "Name": "Excelente", "Description": "Como nuevo, sin signos de desgaste" },
    { "ProblemSubTypeID": 2, "Name": "Muy bueno", "Description": "Bien cuidado, pocos signos de desgaste" },
    { "ProblemSubTypeID": 3, "Name": "Bueno", "Description": "Desgaste menor, funcional" },
    { "ProblemSubTypeID": 4, "Name": "Regular", "Description": "Daño visible pero funcional" },
    { "ProblemSubTypeID": 5, "Name": "No funcional", "Description": "No enciende o daños serios" },
    { "ProblemSubTypeID": 6, "Name": "Equipo Lento", "Description": null }
  ]
}
```

### 9.7 CustomerEquipmentCard - Tarjeta de Equipo

```json
{
  "EquipmentCardNum": 41014,
  "CustomerCode": "C08295",
  "CustomerName": "NELSON FERREIRA DE LIMA",
  "ManufacturerSerialNum": "",
  "InternalSerialNum": "50026B7283127B93",
  "ItemCode": "00786",
  "ItemDescription": "SSD M2 NVMe 1TB KINGSTON NV2",
  "InvoiceCode": 292346,
  "InvoiceNumber": 700713,
  "DeliveryDate": "2023-08-21T00:00:00Z",
  "ContactPhone": "+595 983 531018",
  "StatusOfSerialNumber": "sns_Active",
  "ServiceBPType": "et_Sales",
  "CustomerEquipmentCardBusinessPartners": [
    { "BPCode": "C01048" },
    { "BPCode": "C08295" }
  ]
}
```

### 9.8 ServiceContract - Contrato de Garantía

```json
{
  "ContractID": 80458,
  "CustomerCode": "C05383",
  "CustomerName": "ANIBA SA",
  "Status": "scs_Approved",
  "ContractTemplate": "3 meses",
  "ContractType": "ct_SerialNumber",
  "DurationOfCoverage": 3,
  "StartDate": "2023-09-18T00:00:00Z",
  "EndDate": "2023-12-18T00:00:00Z",
  "IncludeParts": "tYES",
  "IncludeLabor": "tYES",
  "ServiceType": "bst_Warranty",
  "Service_Contract_Lines": [
    {
      "LineNum": 1,
      "InternalSerialNum": "N3N0CV15977112E",
      "ItemCode": "02183",
      "ItemName": "NOTEBOOK ASUS VIVOBOOK F515EA-DH75",
      "StartDate": "2023-09-18T00:00:00Z",
      "EndDate": "2023-12-18T00:00:00Z"
    }
  ]
}
```

### 9.9 KnowledgeBaseSolutions - Base de Conocimiento

```json
{
  "value": [
    {
      "SolutionCode": 6,
      "Solution": "Cambio de HDD",
      "Symptom": "Equipo muy lento",
      "Cause": null
    },
    {
      "SolutionCode": 2,
      "Solution": "Validación del Office",
      "Symptom": "Office caducado",
      "Cause": null
    }
  ]
}
```

### 9.10 ServiceCall con Documentos Vinculados

```json
{
  "ServiceCallID": 6264,
  "ServiceCallInventoryExpenses": [
    {
      "LineNum": 0,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_Quotation",
      "DocumentNumber": 11,
      "DocEntry": 197
    },
    {
      "LineNum": 1,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_StockTransfer",
      "DocumentNumber": 20814,
      "DocEntry": 19207
    },
    {
      "LineNum": 3,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_Order",
      "DocumentNumber": 15573,
      "DocEntry": 32780
    },
    {
      "LineNum": 4,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_Invoice",
      "DocumentNumber": 835306,
      "DocEntry": 723901
    },
    {
      "LineNum": 6,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_Delivery",
      "DocumentNumber": 2593,
      "DocEntry": 5180
    }
  ],
  "ServiceCallSolutions": [
    { "LineNum": 0, "SolutionID": 9 },
    { "LineNum": 1, "SolutionID": 6 }
  ],
  "ServiceCallActivities": [
    { "LineNum": 0, "ActivityCode": 277 },
    { "LineNum": 1, "ActivityCode": 281 }
  ]
}
```

### 9.11 Tipos de Documentos Vinculables (DocumentType)

| Valor | Descripción |
|-------|-------------|
| `edt_Quotation` | Cotización |
| `edt_Order` | Orden de Venta |
| `edt_Delivery` | Nota de Entrega |
| `edt_Invoice` | Factura |
| `edt_StockTransfer` | Transferencia de Stock |
| `edt_PurchaseQuotation` | Cotización de Compra |
| `edt_PurchaseOrder` | Orden de Compra |

---

## 10. Queries SAP Útiles

### 10.1 Listar ServiceCalls para Kanban

```
GET /ServiceCalls?$top=100&$orderby=CreationDate desc
    &$select=ServiceCallID,Subject,CustomerCode,CustomerName,ItemCode,ItemDescription,
             Status,Priority,CreationDate,TechnicianCode,U_Sucursal
    &$filter=Status ne -1
```

### 10.2 Filtrar por Estado

```
GET /ServiceCalls?$filter=Status eq -3
GET /ServiceCalls?$filter=Status eq 2
GET /ServiceCalls?$filter=Status in (-3, 2, 3, 4, 5, 1)
```

### 10.3 Filtrar por Técnico Asignado

```
GET /ServiceCalls?$filter=TechnicianCode eq 3 and Status ne -1
```

### 10.4 Filtrar por Rango de Fechas

```
GET /ServiceCalls?$filter=CreationDate ge '2026-01-01' and CreationDate le '2026-01-31'
```

### 10.5 Obtener Equipos de un Cliente

```
GET /CustomerEquipmentCards?$filter=CustomerCode eq 'C01048'
    &$select=EquipmentCardNum,InternalSerialNum,ItemCode,ItemDescription,StatusOfSerialNumber
```

### 10.6 Verificar Garantía Vigente

```
GET /ServiceContracts?$filter=CustomerCode eq 'C01048'
    and Status eq 'scs_Approved'
    and EndDate ge '2026-02-04'
```

### 10.7 Crear ServiceCall (POST)

```json
POST /ServiceCalls
{
  "CustomerCode": "C01048",
  "Subject": "Equipo no enciende",
  "InternalSerialNum": "ABC123",
  "ItemCode": "01016",
  "Priority": "scp_Medium",
  "U_Sucursal": "1",
  "U_Accesorios": "Cargador, mouse",
  "U_Contrasenha": "1234"
}
```

### 10.8 Actualizar Estado (PATCH)

```json
PATCH /ServiceCalls(70)
{
  "Status": 2,
  "TechnicianCode": 3
}
```

### 10.9 Agregar Diagnóstico (PATCH)

```json
PATCH /ServiceCalls(70)
{
  "Status": 3,
  "U_Diagnostico_Tecnico": "Placa madre dañada, requiere reemplazo",
  "U_Presupuesto_Tentativo": 500000
}
```

---

## 11. Gestión de CustomerEquipmentCard (Tarjetas de Equipo)

### 11.1 ¿Cuándo se crea una tarjeta de equipo?

| Escenario | ¿Se crea automáticamente? | Acción requerida |
|-----------|---------------------------|------------------|
| Venta de producto serializado | Sí (SAP lo crea al facturar) | Ninguna |
| Cliente trae equipo comprado en otro lado | No | Crear manualmente |
| Equipo sin serial (accesorios) | No aplica | No se crea tarjeta |

### 11.2 Buscar Equipos de un Cliente

```
GET /CustomerEquipmentCards?$filter=CustomerCode eq 'C01048'
    &$select=EquipmentCardNum,InternalSerialNum,ItemCode,ItemDescription,StatusOfSerialNumber
    &$orderby=EquipmentCardNum desc
```

**Respuesta:**
```json
{
  "value": [
    {
      "EquipmentCardNum": 41014,
      "InternalSerialNum": "50026B7283127B93",
      "ItemCode": "00786",
      "ItemDescription": "SSD M2 NVMe 1TB KINGSTON NV2",
      "StatusOfSerialNumber": "sns_Active"
    }
  ]
}
```

### 11.3 Buscar por Serial (cuando cliente no recuerda dónde compró)

```
GET /CustomerEquipmentCards?$filter=InternalSerialNum eq 'ABC123' or ManufacturerSerialNum eq 'ABC123'
```

### 11.4 Crear Tarjeta de Equipo Manualmente (POST)

**Endpoint:** `POST /CustomerEquipmentCards`

**Payload:**
```json
{
  "CustomerCode": "C01048",
  "ItemCode": "01016",
  "InternalSerialNum": "SERIAL123456",
  "ManufacturerSerialNum": "MFG-2024-001",
  "ServiceBPType": "et_Customer",
  "StatusOfSerialNumber": "sns_Active"
}
```

**Campos obligatorios:**
| Campo | Tipo | Descripción |
|-------|------|-------------|
| CustomerCode | string | Código del cliente dueño |
| ItemCode | string | Código del producto |

**Campos opcionales importantes:**
| Campo | Tipo | Descripción |
|-------|------|-------------|
| InternalSerialNum | string | Serial interno (si no existe, SAP genera uno) |
| ManufacturerSerialNum | string | Serial del fabricante |
| ServiceBPType | enum | `et_Customer` (cliente) o `et_Sales` (vendido) |
| StatusOfSerialNumber | enum | `sns_Active`, `sns_Terminated`, `sns_Loaned` |

**Respuesta exitosa (201):**
```json
{
  "EquipmentCardNum": 41015,
  "CustomerCode": "C01048",
  "ItemCode": "01016",
  "InternalSerialNum": "SERIAL123456",
  "StatusOfSerialNumber": "sns_Active"
}
```

### 11.5 Flujo en el Frontend

```typescript
// hooks/service/useEquipmentCard.ts

export function useCreateEquipmentCard() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: CreateEquipmentCardDto) => {
      return serviceApi.createEquipmentCard(data);
    },
    onSuccess: (_, variables) => {
      // Invalidar cache de equipos del cliente
      queryClient.invalidateQueries({
        queryKey: ['equipment', variables.CustomerCode]
      });
    }
  });
}
```

---

## 12. Verificación de Garantía (ServiceContracts)

### 12.1 ¿Tiene garantía vigente?

```
GET /ServiceContracts?$filter=CustomerCode eq 'C01048'
    and Status eq 'scs_Approved'
    and EndDate ge '2026-02-05'
    &$expand=Service_Contract_Lines
```

### 12.2 Verificar cobertura de un serial específico

```
GET /ServiceContracts?$filter=Service_Contract_Lines/any(l: l/InternalSerialNum eq 'SERIAL123')
    and Status eq 'scs_Approved'
    and EndDate ge '2026-02-05'
```

### 12.3 Respuesta con garantía vigente

```json
{
  "value": [
    {
      "ContractID": 80458,
      "CustomerCode": "C05383",
      "Status": "scs_Approved",
      "ContractTemplate": "12 meses",
      "StartDate": "2025-09-18T00:00:00Z",
      "EndDate": "2026-09-18T00:00:00Z",
      "IncludeParts": "tYES",
      "IncludeLabor": "tYES",
      "ServiceType": "bst_Warranty",
      "Service_Contract_Lines": [
        {
          "InternalSerialNum": "SERIAL123",
          "ItemCode": "01016",
          "EndDate": "2026-09-18T00:00:00Z"
        }
      ]
    }
  ]
}
```

### 12.4 Función de verificación en Backend

```typescript
// service-calls.service.ts

async checkWarranty(customerCode: string, serialNum?: string): Promise<WarrantyCheckResult> {
  const today = new Date().toISOString().split('T')[0];

  let filter = `CustomerCode eq '${customerCode}' and Status eq 'scs_Approved' and EndDate ge '${today}'`;

  if (serialNum) {
    filter = `Service_Contract_Lines/any(l: l/InternalSerialNum eq '${serialNum}') and Status eq 'scs_Approved' and EndDate ge '${today}'`;
  }

  const contracts = await this.sapClient.get('/ServiceContracts', {
    $filter: filter,
    $expand: 'Service_Contract_Lines',
    $top: 1
  });

  if (contracts.value.length === 0) {
    return { hasWarranty: false };
  }

  const contract = contracts.value[0];
  return {
    hasWarranty: true,
    contractId: contract.ContractID,
    endDate: contract.EndDate,
    includesParts: contract.IncludeParts === 'tYES',
    includesLabor: contract.IncludeLabor === 'tYES',
  };
}
```

### 12.5 Endpoint de verificación

```yaml
GET /service-calls/check-warranty?customer=C01048&serial=ABC123

Response:
{
  "hasWarranty": true,
  "contractId": 80458,
  "endDate": "2026-09-18",
  "includesParts": true,
  "includesLabor": true
}
```

---

## 13. Crear Activities (Seguimientos)

### 13.1 Tipos de Actividad

| Valor | Nombre | Uso típico |
|-------|--------|------------|
| `cn_Conversation` | Llamada | Contacto telefónico con cliente |
| `cn_Meeting` | Reunión | Visita o reunión presencial |
| `cn_Task` | Tarea | Tarea interna asignada |
| `cn_Note` | Nota | Observación o comentario |
| `cn_Other` | Otro | Cualquier otro tipo |

### 13.2 Crear Activity vinculada a ServiceCall (POST)

**Endpoint:** `POST /Activities`

**Payload:**
```json
{
  "ActivityType": "cn_Note",
  "CardCode": "C01048",
  "Notes": "Se realizó diagnóstico. Equipo presenta falla en placa madre.",
  "Details": "Diagnóstico técnico completado",
  "ActivityDate": "2026-02-05",
  "ActivityTime": "14:30:00",
  "HandledBy": 3,
  "ServiceCall": 70
}
```

**Campos importantes:**
| Campo | Tipo | Descripción |
|-------|------|-------------|
| ActivityType | enum | Tipo de actividad (ver tabla arriba) |
| CardCode | string | Cliente relacionado |
| Notes | string | Contenido/descripción de la actividad |
| HandledBy | number | EmployeeID que realizó la actividad |
| ServiceCall | number | **ServiceCallID** para vincular |
| ActivityDate | date | Fecha de la actividad |

### 13.3 Respuesta exitosa

```json
{
  "ActivityCode": 285,
  "ActivityType": "cn_Note",
  "CardCode": "C01048",
  "Notes": "Se realizó diagnóstico. Equipo presenta falla en placa madre.",
  "ServiceCall": 70,
  "ActivityDate": "2026-02-05T00:00:00Z",
  "Closed": "tNO"
}
```

### 13.4 Listar Activities de un ServiceCall

```
GET /Activities?$filter=ServiceCall eq 70&$orderby=ActivityDate desc
```

---

## 14. Vincular Documentos (ServiceCallInventoryExpenses)

### 14.1 Tipos de documentos vinculables

| DocumentType | Descripción | Cuándo se vincula |
|--------------|-------------|-------------------|
| `edt_Quotation` | Cotización | Al crear presupuesto |
| `edt_Order` | Orden de Venta | Al confirmar venta de repuestos |
| `edt_Delivery` | Nota de Entrega | Al entregar equipo |
| `edt_Invoice` | Factura | Al facturar servicio |
| `edt_StockTransfer` | Transferencia | Al recibir repuestos |
| `edt_PurchaseOrder` | Orden de Compra | Al pedir repuestos a proveedor |

### 14.2 Vincular documento a ServiceCall (PATCH)

**Endpoint:** `PATCH /ServiceCalls(70)`

**Payload:**
```json
{
  "ServiceCallInventoryExpenses": [
    {
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_StockTransfer",
      "DocumentNumber": 20814,
      "DocEntry": 19207
    }
  ]
}
```

**Nota:** SAP agrega a la lista existente, no la reemplaza.

### 14.3 Obtener documentos vinculados

```
GET /ServiceCalls(70)?$select=ServiceCallID,ServiceCallInventoryExpenses
```

**Respuesta:**
```json
{
  "ServiceCallID": 70,
  "ServiceCallInventoryExpenses": [
    {
      "LineNum": 0,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_StockTransfer",
      "DocumentNumber": 20814,
      "DocEntry": 19207
    },
    {
      "LineNum": 1,
      "PartType": "sep_NonInventory",
      "DocumentType": "edt_Invoice",
      "DocumentNumber": 835306,
      "DocEntry": 723901
    }
  ]
}
```

---

## 15. WebSocket Events (Tiempo Real)

### 15.1 Eventos a implementar

| Evento | Payload | Cuándo se emite |
|--------|---------|-----------------|
| `service_call_created` | ServiceCallCard | Al crear nuevo caso |
| `service_call_updated` | ServiceCallCard | Al actualizar cualquier campo |
| `service_call_status_changed` | `{ id, oldStatus, newStatus }` | Al cambiar estado |
| `service_call_assigned` | `{ id, technicianCode, technicianName }` | Al asignar técnico |
| `service_call_closed` | `{ id }` | Al cerrar caso |

### 15.2 Salas (Rooms)

```typescript
// Similar a Transferencias
const SERVICE_ROOMS = {
  ALL_SERVICES: 'service:all',           // Jefe taller, Admin
  BRANCH_PREFIX: 'service:branch:',      // service:branch:SUC01
  TECHNICIAN_PREFIX: 'service:tech:',    // service:tech:3
};
```

### 15.3 Suscripción desde Frontend

```typescript
// context/ServiceKanbanContext.tsx

useEffect(() => {
  if (!socket || !user || !isOnServicePage) return;

  const subscriptionPayload = {
    userId: user.employeeId,
    roles: user.roles,
    branch: user.branch,
  };

  socket.emit('subscribeToServiceCalls', subscriptionPayload);

  return () => {
    socket.emit('unsubscribeFromServiceCalls', subscriptionPayload);
  };
}, [socket, user, isOnServicePage]);
```

### 15.4 Handlers en el Context

```typescript
// Registrar listeners
socket.on('service_call_created', handleServiceCallCreated);
socket.on('service_call_updated', handleServiceCallUpdated);
socket.on('service_call_status_changed', handleStatusChanged);
socket.on('service_call_closed', handleServiceCallClosed);

// Handler ejemplo
const handleStatusChanged = (payload: { id: number; oldStatus: number; newStatus: number }) => {
  setStatusColumns(prev => {
    // Remover de columna anterior
    const withoutCard = prev.map(col => ({
      ...col,
      cards: col.cards.filter(c => c.ServiceCallID !== payload.id)
    }));

    // Si el nuevo estado tiene columna, mover la card
    // (requiere fetch del card actualizado o tenerlo en el payload)
    return withoutCard;
  });
};
```

### 15.5 Emisión desde Backend

```typescript
// service-calls.service.ts

async changeStatus(id: number, newStatus: number, userId: number): Promise<ServiceCall> {
  const serviceCall = await this.findOne(id);
  const oldStatus = serviceCall.Status;

  // Validar transición
  if (!canTransition(oldStatus, newStatus)) {
    throw new BadRequestException(`Transición ${oldStatus} → ${newStatus} no permitida`);
  }

  // Actualizar en SAP
  const updated = await this.sapClient.patch(`/ServiceCalls(${id})`, {
    Status: newStatus
  });

  // Emitir evento WebSocket
  this.wsGateway.server.to(`service:branch:${serviceCall.U_Sucursal}`).emit(
    'service_call_status_changed',
    { id, oldStatus, newStatus, updatedBy: userId }
  );

  // Si tiene técnico asignado, notificarle también
  if (serviceCall.TechnicianCode) {
    this.wsGateway.server.to(`service:tech:${serviceCall.TechnicianCode}`).emit(
      'service_call_status_changed',
      { id, oldStatus, newStatus }
    );
  }

  return updated;
}
```

---

## 16. Configuración del Kanban

### 16.1 Archivo de configuración

```typescript
// config/service.kanban.config.ts

import { KanbanConfig } from '@/types/kanban.config';
import { ServiceCallCard } from '@/types/service/service-call.types';
import { serviceCallsApi } from '@/utils/omsBackend/omsBackend';

export interface ServiceKanbanExtra {
  callTypes: Array<{ CallTypeID: number; Name: string }>;
  problemTypes: Array<{ ProblemTypeID: number; Name: string }>;
  technicians: Array<{ EmployeeID: number; Name: string }>;
}

export const serviceKanbanConfig: KanbanConfig<ServiceCallCard, ServiceKanbanExtra> = {
  moduleName: 'service-calls',

  // Fetch de configuración (estados, tipos, técnicos)
  fetchConfig: async () => {
    const [statuses, callTypes, problemTypes, technicians] = await Promise.all([
      serviceCallsApi.getStatuses(),
      serviceCallsApi.getCallTypes(),
      serviceCallsApi.getProblemTypes(),
      serviceCallsApi.getTechnicians(),
    ]);

    return {
      statuses: statuses.filter(s => s.StatusId !== -1), // Excluir "Cerrado" del Kanban
      extra: { callTypes, problemTypes, technicians },
      defaultStatusId: -3, // Abierto
    };
  },

  // Fetch de entidades (casos)
  fetchEntities: async () => {
    return serviceCallsApi.getKanban();
  },

  // Obtener ID único de la entidad
  getEntityId: (card) => String(card.ServiceCallID),

  // Campos para ordenamiento
  sortFields: {
    priorityField: 'Priority',
    dateField: 'CreationDate',
  },

  // Función para actualizar entidad
  updateEntity: async (id, data) => {
    return serviceCallsApi.update(Number(id), data);
  },

  // Callback al mover tarjeta entre columnas
  onMoveTask: (card, destinationStatus) => ({
    Status: destinationStatus.StatusId,
  }),

  // Eventos WebSocket
  socketEvents: {
    created: 'service_call_created',
    updated: 'service_call_updated',
    deleted: 'service_call_closed',
    closed: 'service_call_closed',
  },
};
```

### 16.2 Context específico para Service

```typescript
// context/ServiceKanbanContext/index.tsx

"use client";

import React from "react";
import { ServiceCallCard } from "@/types/service/service-call.types";
import {
  serviceKanbanConfig,
  ServiceKanbanExtra,
} from "@/config/service.kanban.config";
import {
  KanbanDataProvider,
  useKanbanContext,
  KanbanStatus,
} from "@/app/context/Kanbancontext";
import { StatusColumn } from "@/types/kanban.types";

// Tipos específicos para Service
export interface ServiceKanbanContextType {
  statusColumns: StatusColumn<ServiceCallCard>[];
  setStatusColumns: React.Dispatch<React.SetStateAction<StatusColumn<ServiceCallCard>[]>>;
  deleteTask: (taskId: string) => void;
  sortTasks: (tasks: ServiceCallCard[]) => ServiceCallCard[];
  setError: React.Dispatch<React.SetStateAction<string | null>>;
  moveTask: (
    taskId: string,
    sourceStatusId: number,
    destinationStatusId: number,
    sourceIndex: number,
    destinationIndex: number
  ) => void;
  statuses: KanbanStatus[];
  extra: ServiceKanbanExtra | null;
  loadingConfig: boolean;
  isRefreshing: boolean;
  configError: string | null;
  defaultStatusId?: number;
  refresh: (silent?: boolean) => Promise<void>;
  // Extras específicos de Service
  callTypes: Array<{ CallTypeID: number; Name: string }>;
  problemTypes: Array<{ ProblemTypeID: number; Name: string }>;
  technicians: Array<{ EmployeeID: number; Name: string }>;
}

// Provider
interface ServiceKanbanProviderProps {
  children: React.ReactNode;
}

export function ServiceKanbanProvider({ children }: ServiceKanbanProviderProps) {
  return (
    <KanbanDataProvider config={serviceKanbanConfig}>
      {children}
    </KanbanDataProvider>
  );
}

// Hook
export function useServiceKanban(): ServiceKanbanContextType {
  const ctx = useKanbanContext<ServiceCallCard, ServiceKanbanExtra>();

  return {
    ...ctx,
    callTypes: ctx.extra?.callTypes || [],
    problemTypes: ctx.extra?.problemTypes || [],
    technicians: ctx.extra?.technicians || [],
  };
}

export type { KanbanStatus };
```

---

## 17. Manejo de Errores SAP

### 17.1 Errores comunes y cómo manejarlos

| Código SAP | Mensaje | Causa | Solución en código |
|------------|---------|-------|-------------------|
| -1 | "No matching records found" | ID no existe | Mostrar "Caso no encontrado" |
| -2 | "Field 'X' is required" | Campo obligatorio vacío | Validar antes de enviar |
| -5002 | "Invalid status transition" | Transición no permitida | Validar con `canTransition()` |
| -10 | "Session expired" | Token SAP expirado | Re-autenticar automáticamente |
| -2028 | "Customer code not found" | Cliente no existe | Validar cliente antes |

### 17.2 Interceptor de errores en Backend

```typescript
// common/interceptors/sap-error.interceptor.ts

@Injectable()
export class SapErrorInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError((error) => {
        if (error.response?.data?.error) {
          const sapError = error.response.data.error;

          // Mapear errores SAP a HTTP
          const errorMap: Record<number, HttpStatus> = {
            [-1]: HttpStatus.NOT_FOUND,
            [-2]: HttpStatus.BAD_REQUEST,
            [-5002]: HttpStatus.CONFLICT,
            [-10]: HttpStatus.UNAUTHORIZED,
          };

          const status = errorMap[sapError.code] || HttpStatus.INTERNAL_SERVER_ERROR;

          throw new HttpException({
            statusCode: status,
            message: sapError.message.value,
            sapCode: sapError.code,
          }, status);
        }

        throw error;
      }),
    );
  }
}
```

### 17.3 Manejo en Frontend

```typescript
// hooks/service/useServiceCall.ts

const mutation = useMutation({
  mutationFn: updateServiceCall,
  onError: (error: any) => {
    const sapCode = error.response?.data?.sapCode;

    switch (sapCode) {
      case -5002:
        toast.error('No se puede cambiar a ese estado desde el estado actual');
        break;
      case -1:
        toast.error('El caso ya no existe o fue eliminado');
        break;
      default:
        toast.error(error.response?.data?.message || 'Error al actualizar');
    }
  }
});
```

---

## 18. Checklist Final de Implementación

### Pre-requisitos
- [ ] Verificar que UDFs existen en SAP (U_Sucursal, U_Accesorios, U_Contrasenha, U_Diagnostico_Tecnico, U_Presupuesto_Tentativo)
- [ ] Probar queries básicas en Postman contra SAP real
- [ ] Confirmar lista de técnicos en EmployeesInfo

### Fase 1 - MVP Core
**Backend:**
- [ ] Crear módulo `service-calls` en NestJS
- [ ] Implementar `GET /service-calls` (lista para Kanban)
- [ ] Implementar `POST /service-calls` (crear caso)
- [ ] Implementar `GET /service-calls/:id` (detalle)
- [ ] Implementar `PATCH /service-calls/:id/status` (cambiar estado)
- [ ] Implementar validación de transiciones
- [ ] Implementar filtros por fecha, técnico, sucursal
- [ ] Agregar WebSocket gateway para service-calls

**Frontend:**
- [ ] Crear `service.kanban.config.ts`
- [ ] Crear `ServiceKanbanContext`
- [ ] Crear página `/service`
- [ ] Crear `ServiceKanbanBoard`
- [ ] Crear `ServiceCard`
- [ ] Crear `ServiceCallModal` (básico)
- [ ] Crear `CreateServiceCallForm`
- [ ] Implementar `useServiceCalls` hook

### Fase 2 - Equipos y Garantías
**Backend:**
- [ ] Crear módulo `equipment`
- [ ] Implementar `GET /equipment?customer=X`
- [ ] Implementar `POST /equipment` (crear tarjeta)
- [ ] Implementar `GET /service-calls/check-warranty`

**Frontend:**
- [ ] Crear `EquipmentSelector` component
- [ ] Crear `NewEquipmentForm`
- [ ] Mostrar badge de garantía en tarjetas
- [ ] Integrar verificación de garantía en flujo de diagnóstico

### Fase 3 - Activities y Presupuestos
**Backend:**
- [ ] Implementar `GET /service-calls/:id/activities`
- [ ] Implementar `POST /service-calls/:id/activities`
- [ ] Agregar campo U_Presupuesto_Tentativo a updates

**Frontend:**
- [ ] Crear `ActivityTimeline` component
- [ ] Crear `AddActivityForm`
- [ ] Crear `BudgetSection` component

### Fase 4 - Integración Inventario
**Backend:**
- [ ] Modificar transferencias para aceptar ServiceCallID
- [ ] Implementar vinculación en InventoryExpenses
- [ ] Emitir evento cuando transferencia se completa

**Frontend:**
- [ ] Integrar `TransferModal` desde casos
- [ ] Mostrar transferencias vinculadas
- [ ] Auto-cambiar estado a ESPERANDO_REPUESTOS

### Fase 5 - Entrega y Facturación
**Backend:**
- [ ] Implementar vinculación de facturas

**Frontend:**
- [ ] Crear flujo de entrega
- [ ] Integrar con módulo de facturación existente

---

## 19. Referencias

- [UML-Servicio-Tecnico.md](./UML-Servicio-Tecnico.md) - Documento UML completo (permisos, diagramas de secuencia)
- [SAP Service Layer - ServiceCalls](https://help.sap.com/docs/SAP_BUSINESS_ONE) - Documentación oficial
- [XState Documentation](https://xstate.js.org/docs/) - State machines
- [InventoryKanbanBoard](../../src/app/components/apps/inventory-kanban/) - Referencia de implementación
- [KanbanDataProvider](../../src/app/context/Kanbancontext/) - Context genérico reutilizable

---

**Documento completado.** Este documento junto con el UML proveen toda la información necesaria para implementar el módulo de Servicio Técnico.
