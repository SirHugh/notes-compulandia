# Documento de Diseño UML

## Módulo de Servicio Técnico

**Versión:** 1.0
**Fecha:** 30 de Enero de 2026
**Autor:** Equipo de Desarrollo
**Estado:** Pendiente de Aprobación

---

## Tabla de Contenidos

1. [Introducción](#1-introducción)
2. [Alcance del Módulo](#2-alcance-del-módulo)
3. [Diagrama de Casos de Uso](#3-diagrama-de-casos-de-uso)
4. [Diagrama de Clases (Modelo de Datos)](#4-diagrama-de-clases-modelo-de-datos)
5. [Diagrama de Estados](#5-diagrama-de-estados)
6. [Diagramas de Secuencia](#6-diagramas-de-secuencia)
7. [Diagrama de Componentes](#7-diagrama-de-componentes)
8. [Matriz de Permisos por Rol](#8-matriz-de-permisos-por-rol)
9. [Endpoints de API](#9-endpoints-de-api)
10. [Glosario](#10-glosario)

---

## 1. Introducción

### 1.1 Propósito

Este documento describe el diseño técnico del Módulo de Servicio Técnico utilizando notación UML estándar. El objetivo es proporcionar una visión clara y completa del sistema antes de iniciar la implementación.

### 1.2 Contexto

El módulo gestionará el ciclo de vida completo de las reparaciones de equipos, desde el ingreso hasta la entrega al cliente. Se integrará con SAP Business One mediante la entidad `ServiceCalls` y sus relaciones.

### 1.3 Tecnologías

| Capa          | Tecnología                                        |
| ------------- | ------------------------------------------------- |
| Frontend      | Next.js 14, React 18, Material UI, TanStack Query |
| Backend       | NestJS, TypeScript                                |
| Base de Datos | SAP Business One (Service Layer API)              |
| Comunicación  | REST API, WebSockets (tiempo real)                |

---

## 2. Alcance del Módulo

### 2.1 Funcionalidades Incluidas

- Registro de ingreso de equipos
- Gestión de clientes y tarjetas de equipo
- Flujo de estados (Abierto → Diagnóstico → Presupuesto → Ejecución → Terminado → Cerrado)
- Asignación de técnicos
- Registro de diagnósticos y presupuestos
- Gestión de actividades/seguimientos
- Solicitud de repuestos (integración con módulo de Traslados)
- Facturación (integración con SAP)
- Vista Kanban para gestión visual
- Notificaciones de estado

### 2.2 Funcionalidades Excluidas (Fase 1)

- Aplicación móvil para técnicos
- Portal de cliente para seguimiento
- Integración con sistemas de mensajería (WhatsApp)
- Reportes avanzados y analytics

---

## 3. Diagrama de Casos de Uso

### 3.1 Actores del Sistema

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ACTORES                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐         │
│   │           │    │           │    │           │    │           │         │
│   │ Recepción │    │  Técnico  │    │   Jefe    │    │  Cajero   │         │
│   │           │    │           │    │  Taller   │    │           │         │
│   └───────────┘    └───────────┘    └───────────┘    └───────────┘         │
│        │                │                │                │                 │
│        │                │                │                │                 │
│   Ingreso de       Diagnóstico      Supervisión       Facturación          │
│   equipos          y reparación     y asignación      y cobro              │
│                                                                             │
│                        ┌───────────┐                                        │
│                        │           │                                        │
│                        │  Cliente  │  (Actor externo - no usa el sistema)  │
│                        │           │                                        │
│                        └───────────┘                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Casos de Uso por Actor

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CASOS DE USO - SERVICIO TÉCNICO                          │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────────────────┐
                              │    SISTEMA DE SERVICIO      │
                              │         TÉCNICO             │
                              │                             │
 ┌──────────┐                 │  ┌───────────────────────┐  │
 │          │                 │  │ CU-01: Registrar      │  │
 │Recepción │─────────────────┼─►│ Ingreso de Equipo     │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-02: Buscar/Crear   │  │
 │          │                 │  │ Cliente               │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-03: Registrar      │  │
 │          │                 │  │ Equipo (Tarjeta)      │  │
 └──────────┘                 │  └───────────────────────┘  │
                              │                             │
 ┌──────────┐                 │  ┌───────────────────────┐  │
 │          │                 │  │ CU-04: Ver Casos      │  │
 │ Técnico  │─────────────────┼─►│ Asignados             │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-05: Registrar      │  │
 │          │                 │  │ Diagnóstico           │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-06: Actualizar     │  │
 │          │                 │  │ Estado del Caso       │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-07: Registrar      │  │
 │          │                 │  │ Actividad             │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-08: Solicitar      │  │
 └──────────┘                 │  │ Repuestos             │  │
                              │  └───────────────────────┘  │
 ┌──────────┐                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │   Jefe   │─────────────────┼─►│ CU-09: Asignar        │  │
 │  Taller  │                 │  │ Técnico               │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-10: Crear          │  │
 │          │                 │  │ Presupuesto           │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-11: Ver Dashboard  │  │
 │          │                 │  │ Kanban                │  │
 └──────────┘                 │  └───────────────────────┘  │
                              │                             │
 ┌──────────┐                 │  ┌───────────────────────┐  │
 │          │                 │  │ CU-12: Generar        │  │
 │  Cajero  │─────────────────┼─►│ Factura               │  │
 │          │                 │  └───────────────────────┘  │
 │          │                 │                             │
 │          │                 │  ┌───────────────────────┐  │
 │          │─────────────────┼─►│ CU-13: Registrar      │  │
 │          │                 │  │ Entrega               │  │
 └──────────┘                 │  └───────────────────────┘  │
                              │                             │
                              └─────────────────────────────┘
```

### 3.3 Descripción de Casos de Uso Principales

#### CU-01: Registrar Ingreso de Equipo

| Campo                 | Descripción                                                                                                                                                                                                                                                                                                                            |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**                | CU-01                                                                                                                                                                                                                                                                                                                                  |
| **Nombre**            | Registrar Ingreso de Equipo                                                                                                                                                                                                                                                                                                            |
| **Actor Principal**   | Recepción                                                                                                                                                                                                                                                                                                                              |
| **Precondiciones**    | Usuario autenticado con rol de Recepción                                                                                                                                                                                                                                                                                               |
| **Flujo Principal**   | 1. Usuario busca cliente por nombre/documento<br>2. Sistema muestra datos del cliente<br>3. Usuario selecciona o registra equipo<br>4. Usuario ingresa motivo de servicio<br>5. Usuario registra accesorios y contraseña (si aplica)<br>6. Sistema crea ServiceCall con estado "Abierto"<br>7. Sistema genera comprobante de recepción |
| **Flujo Alternativo** | 2a. Cliente no existe → Usuario crea nuevo cliente                                                                                                                                                                                                                                                                                     |
| **Postcondiciones**   | ServiceCall creado, cliente notificado                                                                                                                                                                                                                                                                                                 |

#### CU-05: Registrar Diagnóstico

| Campo               | Descripción                                                                                                                                                                                                                                                 |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**              | CU-05                                                                                                                                                                                                                                                       |
| **Nombre**          | Registrar Diagnóstico                                                                                                                                                                                                                                       |
| **Actor Principal** | Técnico                                                                                                                                                                                                                                                     |
| **Precondiciones**  | Caso asignado al técnico, estado = "Diagnóstico"                                                                                                                                                                                                            |
| **Flujo Principal** | 1. Técnico selecciona caso asignado<br>2. Técnico evalúa el equipo<br>3. Técnico registra diagnóstico técnico<br>4. Técnico indica si requiere repuestos<br>5. Sistema actualiza campo U_Diagnostico_Tecnico<br>6. Sistema crea Activity con el diagnóstico |
| **Postcondiciones** | Diagnóstico registrado, caso listo para presupuesto                                                                                                                                                                                                         |

#### CU-10: Crear Presupuesto

| Campo               | Descripción                                                                                                                                                                                                                                                                      |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ID**              | CU-10                                                                                                                                                                                                                                                                            |
| **Nombre**          | Crear Presupuesto                                                                                                                                                                                                                                                                |
| **Actor Principal** | Jefe de Taller                                                                                                                                                                                                                                                                   |
| **Precondiciones**  | Diagnóstico completado                                                                                                                                                                                                                                                           |
| **Flujo Principal** | 1. Usuario revisa diagnóstico<br>2. Usuario calcula costo de repuestos<br>3. Usuario calcula mano de obra<br>4. Usuario registra presupuesto total<br>5. Sistema actualiza U_Presupuesto_Tentativo<br>6. Sistema cambia estado a "Presupuesto"<br>7. Sistema notifica al cliente |
| **Postcondiciones** | Presupuesto registrado, esperando aprobación                                                                                                                                                                                                                                     |

---

## 4. Diagrama de Clases (Modelo de Datos)

### 4.1 Modelo Conceptual

``` 
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MODELO DE CLASES - SAP B1                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────┐       ┌───────────────────────────────────┐
│         BusinessPartner           │       │      CustomerEquipmentCard        │
├───────────────────────────────────┤       ├───────────────────────────────────┤
│ + CardCode: string {PK}           │       │ + EquipmentCardNum: number {PK}   │
│ + CardName: string                │       │ + InternalSerialNum: string       │
│ + CardType: "cCustomer"|"cLead"   │       │ + ManufacturerSerialNum: string   │
│ + Phone1: string                  │       │ + ItemCode: string                │
│ + Phone2: string                  │       │ + ItemDescription: string         │
│ + E_Mail: string                  │       │ + CustomerCode: string {FK}       │
│ + Address: string                 │       │ + StatusOfSerialNumber: string    │
│ + FederalTaxID: string            │       │ + InstallLocation: string         │
├───────────────────────────────────┤       ├───────────────────────────────────┤
│ + getFullAddress(): string        │       │ + isActive(): boolean             │
└───────────────────────────────────┘       └───────────────────────────────────┘
              │                                           │
              │ 1                                         │ 1
              │                                           │
              ▼ *                                         ▼ *
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ServiceCall                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│ + ServiceCallID: number {PK}                                                │
│ + CustomerCode: string {FK}                                                 │
│ + InternalSerialNum: string {FK}                                            │
│ + ItemCode: string                                                          │
│ + ContractID: number {FK}                                                   │
│ + Subject: string                                                           │
│ + Status: ServiceCallStatus                                                 │
│ + Priority: "scp_Low" | "scp_Medium" | "scp_High"                           │
│ + CallType: number                                                          │
│ + Origin: string                                                            │
│ + AssigneeCode: number {FK}                                                 │
│ + TechnicianCode: number                                                    │
│ + Resolution: string                                                        │
│ + CreationDate: Date                                                        │
│ + CreationTime: Time                                                        │
│ + ResponseByTime: Time                                                      │
│ + ResolutionOnDate: Date                                                    │
│ + Remarks: string                                                           │
│ + U_Sucursal: string                      ◄── Campo personalizado           │
│ + U_Accesorios: string                    ◄── Campo personalizado           │
│ + U_Contrasenha: string                   ◄── Campo personalizado           │
│ + U_Diagnostico_Tecnico: string           ◄── Campo personalizado           │
│ + U_Presupuesto_Tentativo: decimal        ◄── Campo personalizado           │
├─────────────────────────────────────────────────────────────────────────────┤
│ + ServiceCallActivities: ServiceCallActivity[]                              │
│ + ServiceCallInventoryExpenses: ServiceCallInventoryExpense[]               │
│ + ServiceCallSolutions: ServiceCallSolution[]                               │
├─────────────────────────────────────────────────────────────────────────────┤
│ + changeStatus(newStatus: ServiceCallStatus): void                          │
│ + addActivity(activity: Activity): void                                     │
│ + addExpense(expense: InventoryExpense): void                               │
│ + canTransitionTo(status: ServiceCallStatus): boolean                       │
└─────────────────────────────────────────────────────────────────────────────┘
              │                    │                           │
              │ *                  │ *                         │ *
              ▼                    ▼                           ▼
┌────────────────────┐  ┌─────────────────────────┐  ┌────────────────────────┐
│ServiceCallActivity │  │ServiceCallInventory     │  │ ServiceCallSolution    │
├────────────────────┤  │Expense                  │  ├────────────────────────┤
│ + LineNum: number  │  ├─────────────────────────┤  │ + LineNum: number      │
│ + ActivityCode: int│  │ + LineNum: number       │  │ + SolutionCode: number │
└────────────────────┘  │ + PartType: string      │  └────────────────────────┘
         │              │ + DocumentType: string  │              │
         │ 1            │ + DocEntry: number      │              │ 1
         ▼              │ + DocNum: number        │              ▼
┌────────────────────┐  └─────────────────────────┘  ┌────────────────────────┐
│     Activity       │              │                │KnowledgeBaseSolution   │
├────────────────────┤              │                ├────────────────────────┤
│ + ActivityCode: int│              │                │ + SolutionCode: number │
│ + ActivityType: str│              ▼                │ + ItemCode: string     │
│ + CardCode: string │  ┌─────────────────────────┐  │ + Symptom: string      │
│ + Notes: string    │  │  Documentos Vinculados  │  │ + Cause: string        │
│ + ActivityDate:Date│  ├─────────────────────────┤  │ + Solution: string     │
│ + HandledBy: int   │  │ - Sales Order           │  └────────────────────────┘
│ + Closed: string   │  │ - Delivery              │
└────────────────────┘  │ - A/R Invoice           │
                        │ - Stock Transfer        │
                        │ - Purchase Order        │
                        └─────────────────────────┘


┌───────────────────────────────────┐
│          EmployeesInfo            │
├───────────────────────────────────┤
│ + EmployeeID: number {PK}         │
│ + FirstName: string               │
│ + LastName: string                │
│ + JobTitle: string                │
│ + Department: number              │
│ + Active: string                  │
└───────────────────────────────────┘
```

### 4.2 Enumeraciones

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ENUMERACIONES                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────┐   ┌───────────────────────────────┐
│     ServiceCallStatus         │   │      ActivityType             │
├───────────────────────────────┤   ├───────────────────────────────┤
│ ABIERTO = -3                  │   │ CONVERSATION = "cn_Conversation"
│ DIAGNOSTICO = 2               │   │ MEETING = "cn_Meeting"        │
│ PRESUPUESTO = 3               │   │ TASK = "cn_Task"              │
│ EJECUCION = 4                 │   │ NOTE = "cn_Note"              │
│ ESPERANDO_REPUESTOS = 5       │   │ OTHER = "cn_Other"            │
│ TERMINADO = 1                 │   └───────────────────────────────┘
│ CERRADO = -1                  │
└───────────────────────────────┘   ┌───────────────────────────────┐
                                    │     ExpenseDocumentType       │
┌───────────────────────────────┐   ├───────────────────────────────┤
│       Priority                │   │ ORDER = "edt_Order"           │
├───────────────────────────────┤   │ DELIVERY = "edt_Delivery"     │
│ LOW = "scp_Low"               │   │ INVOICE = "edt_Invoice"       │
│ MEDIUM = "scp_Medium"         │   │ STOCK_TRANSFER = "edt_StockTransfer"
│ HIGH = "scp_High"             │   │ PURCHASE_ORDER = "edt_PurchaseOrder"
└───────────────────────────────┘   └───────────────────────────────┘
```

---

## 5. Diagrama de Estados

### 5.1 Máquina de Estados del ServiceCall

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DIAGRAMA DE ESTADOS - ServiceCall                        │
└─────────────────────────────────────────────────────────────────────────────┘

                                    ┌─────────┐
                                    │ INICIO  │
                                    └────┬────┘
                                         │
                                         │ [crear caso]
                                         ▼
                              ┌─────────────────────┐
                              │      ABIERTO        │
                              │     (Status=-3)     │
                              │                     │
                              │ • Caso recién       │
                              │   ingresado         │
                              │ • Sin técnico       │
                              │   asignado          │
                              └──────────┬──────────┘
                                         │
                                         │ [asignar técnico]
                                         ▼
                              ┌─────────────────────┐
                              │    DIAGNÓSTICO      │
                              │     (Status=2)      │
                              │                     │
                              │ • Técnico evalúa    │
                              │ • Registra hallazgos│
                              └──────────┬──────────┘
                                         │
                        ┌────────────────┴────────────────┐
                        │                                 │
                        │ [tiene garantía]                │ [sin garantía]
                        ▼                                 ▼
              ┌─────────────────────┐          ┌─────────────────────┐
              │     EJECUCIÓN       │          │    PRESUPUESTO      │
              │     (Status=4)      │          │     (Status=3)      │
              │                     │          │                     │
              │ • Reparación        │◄─────────│ • Esperando         │
              │   en proceso        │ [acepta] │   aprobación        │
              │                     │          │   del cliente       │
              └──────────┬──────────┘          └──────────┬──────────┘
                         │                                │
         ┌───────────────┼───────────────┐                │ [rechaza]
         │               │               │                │
         │ [falta        │ [reparación   │                ▼
         │  repuesto]    │  exitosa]     │     ┌─────────────────────┐
         ▼               │               │     │      CERRADO        │
┌─────────────────────┐  │               │     │     (Status=-1)     │
│ ESPERANDO REPUESTOS │  │               │     │                     │
│     (Status=5)      │  │               │     │ • Sin reparar       │
│                     │  │               │     │ • Cliente rechazó   │
│ • Solicitud de      │  │               │     └─────────────────────┘
│   transferencia     │  │               │                ▲
│ • Orden de compra   │  │               │                │
└──────────┬──────────┘  │               │                │
           │             │               │                │
           │ [repuestos  │               │                │
           │  llegaron]  │               │                │
           └─────────────┼───────────────┘                │
                         │                                │
                         ▼                                │
              ┌─────────────────────┐                     │
              │     TERMINADO       │                     │
              │     (Status=1)      │                     │
              │                     │                     │
              │ • Reparación        │                     │
              │   completada        │                     │
              │ • Listo para        │                     │
              │   entrega           │                     │
              └──────────┬──────────┘                     │
                         │                                │
                         │ [entregar al cliente]          │
                         │ [generar factura]              │
                         ▼                                │
              ┌─────────────────────┐                     │
              │      CERRADO        │◄────────────────────┘
              │     (Status=-1)     │
              │                     │
              │ • Caso finalizado   │
              │ • Solo lectura      │
              └─────────────────────┘
```

### 5.2 Tabla de Transiciones Válidas

| Estado Actual           | Estados Posibles        | Condición                          | Actor Permitido |
| ----------------------- | ----------------------- | ---------------------------------- | --------------- |
| ABIERTO (-3)            | DIAGNÓSTICO (2)         | Asignar técnico                    | Jefe Taller     |
| ABIERTO (-3)            | CERRADO (-1)            | Cancelar caso                      | Jefe Taller     |
| DIAGNÓSTICO (2)         | PRESUPUESTO (3)         | Diagnóstico completo, sin garantía | Técnico, Jefe   |
| DIAGNÓSTICO (2)         | EJECUCIÓN (4)           | Diagnóstico completo, con garantía | Técnico, Jefe   |
| PRESUPUESTO (3)         | EJECUCIÓN (4)           | Cliente acepta                     | Jefe Taller     |
| PRESUPUESTO (3)         | CERRADO (-1)            | Cliente rechaza                    | Jefe Taller     |
| EJECUCIÓN (4)           | ESPERANDO_REPUESTOS (5) | Faltan piezas                      | Técnico         |
| EJECUCIÓN (4)           | TERMINADO (1)           | Reparación exitosa                 | Técnico         |
| ESPERANDO_REPUESTOS (5) | EJECUCIÓN (4)           | Repuestos recibidos                | Técnico         |
| TERMINADO (1)           | CERRADO (-1)            | Entrega completada                 | Jefe Taller     |

---

## 6. Diagramas de Secuencia

### 6.1 Secuencia: Registrar Ingreso de Equipo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              SECUENCIA: Registrar Ingreso de Equipo                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│Recepción │      │ Frontend │      │ Backend  │      │   SAP    │      │  Impres. │
│          │      │ (Next.js)│      │ (NestJS) │      │  B1 API  │      │          │
└────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘
     │                 │                 │                 │                 │
     │ 1. Buscar       │                 │                 │                 │
     │    cliente      │                 │                 │                 │
     │────────────────►│                 │                 │                 │
     │                 │                 │                 │                 │
     │                 │ 2. GET /api/    │                 │                 │
     │                 │    customers    │                 │                 │
     │                 │    ?search=...  │                 │                 │
     │                 │────────────────►│                 │                 │
     │                 │                 │                 │                 │
     │                 │                 │ 3. GET Business │                 │
     │                 │                 │    Partners     │                 │
     │                 │                 │────────────────►│                 │
     │                 │                 │                 │                 │
     │                 │                 │◄────────────────│                 │
     │                 │                 │ 4. Lista de     │                 │
     │                 │                 │    clientes     │                 │
     │                 │                 │                 │                 │
     │                 │◄────────────────│                 │                 │
     │◄────────────────│ 5. Mostrar     │                 │                 │
     │  Seleccionar    │    resultados   │                 │                 │
     │  cliente        │                 │                 │                 │
     │                 │                 │                 │                 │
     │ 6. Ver equipos  │                 │                 │                 │
     │    del cliente  │                 │                 │                 │
     │────────────────►│                 │                 │                 │
     │                 │                 │                 │                 │
     │                 │ 7. GET /api/    │                 │                 │
     │                 │    equipment    │                 │                 │
     │                 │    ?customer=...│                 │                 │
     │                 │────────────────►│                 │                 │
     │                 │                 │                 │                 │
     │                 │                 │ 8. GET Customer │                 │
     │                 │                 │    Equipment    │                 │
     │                 │                 │    Cards        │                 │
     │                 │                 │────────────────►│                 │
     │                 │                 │                 │                 │
     │                 │                 │◄────────────────│                 │
     │                 │                 │                 │                 │
     │                 │◄────────────────│                 │                 │
     │◄────────────────│                 │                 │                 │
     │                 │                 │                 │                 │
     │ 9. Completar    │                 │                 │                 │
     │    formulario   │                 │                 │                 │
     │    (Subject,    │                 │                 │                 │
     │     Accesorios, │                 │                 │                 │
     │     Contraseña) │                 │                 │                 │
     │────────────────►│                 │                 │                 │
     │                 │                 │                 │                 │
     │                 │ 10. POST /api/  │                 │                 │
     │                 │     service-    │                 │                 │
     │                 │     calls       │                 │                 │
     │                 │────────────────►│                 │                 │
     │                 │                 │                 │                 │
     │                 │                 │ 11. Validar     │                 │
     │                 │                 │     datos       │                 │
     │                 │                 │────┐            │                 │
     │                 │                 │    │            │                 │
     │                 │                 │◄───┘            │                 │
     │                 │                 │                 │                 │
     │                 │                 │ 12. POST        │                 │
     │                 │                 │     ServiceCalls│                 │
     │                 │                 │────────────────►│                 │
     │                 │                 │                 │                 │
     │                 │                 │◄────────────────│                 │
     │                 │                 │ 13. ServiceCall │                 │
     │                 │                 │     creado      │                 │
     │                 │                 │     (ID=12345)  │                 │
     │                 │                 │                 │                 │
     │                 │                 │ 14. Generar     │                 │
     │                 │                 │     comprobante │                 │
     │                 │                 │─────────────────────────────────►│
     │                 │                 │                 │                 │
     │                 │                 │◄─────────────────────────────────│
     │                 │                 │                 │                 │
     │                 │◄────────────────│                 │                 │
     │◄────────────────│ 15. Éxito +    │                 │                 │
     │                 │     imprimir   │                 │                 │
     │                 │     ticket     │                 │                 │
     │                 │                 │                 │                 │
```

### 6.2 Secuencia: Flujo de Diagnóstico a Presupuesto

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              SECUENCIA: Diagnóstico y Presupuesto                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ Técnico  │      │ Frontend │      │ Backend  │      │   SAP    │
│          │      │          │      │          │      │  B1 API  │
└────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘
     │                 │                 │                 │
     │ 1. Ver casos    │                 │                 │
     │    asignados    │                 │                 │
     │────────────────►│                 │                 │
     │                 │ 2. GET /api/service-calls        │
     │                 │    ?assignee=me&status=2         │
     │                 │────────────────►│                 │
     │                 │                 │ 3. GET ServiceCalls
     │                 │                 │────────────────►│
     │                 │                 │◄────────────────│
     │                 │◄────────────────│                 │
     │◄────────────────│                 │                 │
     │                 │                 │                 │
     │ 4. Abrir caso   │                 │                 │
     │    #12345       │                 │                 │
     │────────────────►│                 │                 │
     │                 │ 5. GET /api/service-calls/12345  │
     │                 │────────────────►│                 │
     │                 │                 │ 6. GET ServiceCalls(12345)
     │                 │                 │    + $expand    │
     │                 │                 │────────────────►│
     │                 │                 │◄────────────────│
     │                 │◄────────────────│                 │
     │◄────────────────│ 7. Detalle     │                 │
     │                 │    completo     │                 │
     │                 │                 │                 │
     │ 8. Registrar    │                 │                 │
     │    diagnóstico  │                 │                 │
     │   "Placa madre  │                 │                 │
     │    dañada..."   │                 │                 │
     │────────────────►│                 │                 │
     │                 │ 9. PATCH /api/service-calls/12345
     │                 │    {                              │
     │                 │      U_Diagnostico_Tecnico: "...",
     │                 │      Status: 3                    │
     │                 │    }            │                 │
     │                 │────────────────►│                 │
     │                 │                 │                 │
     │                 │                 │ 10. Validar     │
     │                 │                 │     transición  │
     │                 │                 │     2 → 3       │
     │                 │                 │────┐            │
     │                 │                 │    │ OK         │
     │                 │                 │◄───┘            │
     │                 │                 │                 │
     │                 │                 │ 11. PATCH       │
     │                 │                 │     ServiceCalls│
     │                 │                 │     (12345)     │
     │                 │                 │────────────────►│
     │                 │                 │                 │
     │                 │                 │◄────────────────│
     │                 │                 │                 │
     │                 │                 │ 12. POST        │
     │                 │                 │     Activities  │
     │                 │                 │    (registrar   │
     │                 │                 │     diagnóstico)│
     │                 │                 │────────────────►│
     │                 │                 │                 │
     │                 │                 │◄────────────────│
     │                 │                 │                 │
     │                 │◄────────────────│                 │
     │◄────────────────│ 13. Éxito      │                 │
     │                 │                 │                 │
```

### 6.3 Secuencia: Solicitar Repuestos

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              SECUENCIA: Solicitar Repuestos (Transferencia)                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
│ Técnico  │      │ Frontend │      │ Backend  │      │   SAP    │
│          │      │          │      │          │      │  B1 API  │
└────┬─────┘      └────┬─────┘      └────┬─────┘      └────┬─────┘
     │                 │                 │                 │
     │ 1. Solicitar    │                 │                 │
     │    repuesto     │                 │                 │
     │    desde caso   │                 │                 │
     │    #12345       │                 │                 │
     │────────────────►│                 │                 │
     │                 │                 │                 │
     │                 │ 2. Abrir modal  │                 │
     │                 │    de traslado  │                 │
     │                 │    (pre-llenado │                 │
     │                 │    con datos    │                 │
     │                 │    del caso)    │                 │
     │◄────────────────│                 │                 │
     │                 │                 │                 │
     │ 3. Seleccionar  │                 │                 │
     │    productos,   │                 │                 │
     │    almacén      │                 │                 │
     │    destino      │                 │                 │
     │────────────────►│                 │                 │
     │                 │                 │                 │
     │                 │ 4. POST /api/inventory/           │
     │                 │    transfer-drafts                │
     │                 │    {                              │
     │                 │      FromWarehouse: "ALM01",      │
     │                 │      ToWarehouse: "TALLER",       │
     │                 │      Lines: [...],                │
     │                 │      ServiceCallID: 12345 ◄───────│── Nuevo campo
     │                 │    }            │                 │
     │                 │────────────────►│                 │
     │                 │                 │                 │
     │                 │                 │ 5. POST Stock   │
     │                 │                 │    Transfer     │
     │                 │                 │    Request      │
     │                 │                 │────────────────►│
     │                 │                 │                 │
     │                 │                 │◄────────────────│
     │                 │                 │ Draft creado    │
     │                 │                 │                 │
     │                 │                 │ 6. PATCH        │
     │                 │                 │    ServiceCall  │
     │                 │                 │    Status = 5   │
     │                 │                 │────────────────►│
     │                 │                 │                 │
     │                 │                 │◄────────────────│
     │                 │                 │                 │
     │                 │                 │ 7. POST        │
     │                 │                 │    Agregar a    │
     │                 │                 │    ServiceCall  │
     │                 │                 │    Inventory    │
     │                 │                 │    Expenses     │
     │                 │                 │────────────────►│
     │                 │                 │                 │
     │                 │                 │◄────────────────│
     │                 │                 │                 │
     │                 │◄────────────────│                 │
     │◄────────────────│ 8. Traslado    │                 │
     │                 │    solicitado   │                 │
     │                 │                 │                 │
```

---

## 7. Diagrama de Componentes

### 7.1 Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ARQUITECTURA DE COMPONENTES                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (Next.js)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │     Pages       │  │   Components    │  │     Hooks       │             │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤             │
│  │ /service        │  │ ServiceKanban   │  │useServiceCalls  │             │
│  │ /service/[id]   │  │   Board         │  │useServiceCall   │             │
│  │                 │  │ ServiceCallModal│  │useActivities    │             │
│  │                 │  │ CustomerSearch  │  │useEquipment     │             │
│  │                 │  │ EquipmentForm   │  │usePermissions   │             │
│  │                 │  │ DiagnosticForm  │  │                 │             │
│  │                 │  │ BudgetForm      │  │                 │             │
│  │                 │  │ ActivityTimeline│  │                 │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│           │                   │                    │                        │
│           └───────────────────┼────────────────────┘                        │
│                               │                                             │
│                               ▼                                             │
│                    ┌─────────────────────┐                                  │
│                    │   TanStack Query    │                                  │
│                    │   (Cache + Estado)  │                                  │
│                    └──────────┬──────────┘                                  │
│                               │                                             │
└───────────────────────────────┼─────────────────────────────────────────────┘
                                │
                                │ HTTP/REST
                                │ WebSocket (futuro)
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BACKEND (NestJS)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         API Gateway Layer                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │   │
│  │  │  Auth Guard     │  │  Role Guard     │  │  Validation     │      │   │
│  │  │  (JWT)          │  │                 │  │  Pipe           │      │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                               │                                             │
│  ┌────────────────────────────┼────────────────────────────────────────┐   │
│  │                    Controller Layer                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │   │
│  │  │ ServiceCalls    │  │ Activities      │  │ Equipment       │      │   │
│  │  │ Controller      │  │ Controller      │  │ Controller      │      │   │
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘      │   │
│  └───────────┼────────────────────┼────────────────────┼───────────────┘   │
│              │                    │                    │                    │
│  ┌───────────┼────────────────────┼────────────────────┼───────────────┐   │
│  │           │            Service Layer                │               │   │
│  │  ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐      │   │
│  │  │ ServiceCalls    │  │ Activities      │  │ Equipment       │      │   │
│  │  │ Service         │  │ Service         │  │ Service         │      │   │
│  │  │                 │  │                 │  │                 │      │   │
│  │  │ • CRUD          │  │ • CRUD          │  │ • CRUD          │      │   │
│  │  │ • State Machine │  │ • Vinculación   │  │ • Búsqueda      │      │   │
│  │  │ • Validaciones  │  │   a ServiceCall │  │ • Crear tarjeta │      │   │
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘      │   │
│  └───────────┼────────────────────┼────────────────────┼───────────────┘   │
│              │                    │                    │                    │
│  ┌───────────┼────────────────────┼────────────────────┼───────────────┐   │
│  │           │          Integration Layer              │               │   │
│  │  ┌────────▼─────────────────────────────────────────▼────────┐      │   │
│  │  │                    SAP Service Layer Client                │      │   │
│  │  │                                                            │      │   │
│  │  │  • Autenticación con SAP                                   │      │   │
│  │  │  • Mapeo de entidades                                      │      │   │
│  │  │  • Manejo de errores SAP                                   │      │   │
│  │  │  • Retry logic                                             │      │   │
│  │  └────────────────────────────┬───────────────────────────────┘      │   │
│  └───────────────────────────────┼──────────────────────────────────────┘   │
│                                  │                                          │
└──────────────────────────────────┼──────────────────────────────────────────┘
                                   │
                                   │ HTTP/HTTPS
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SAP BUSINESS ONE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Service Layer API                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ServiceCalls │ │ Activities  │ │CustomerEquip│ │ Service     │          │
│  │             │ │             │ │ mentCards   │ │ Contracts   │          │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │Business     │ │  Items      │ │ Employees   │ │StockTransfer│          │
│  │Partners     │ │             │ │ Info        │ │Requests     │          │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Estructura de Carpetas

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ESTRUCTURA DE CARPETAS                               │
└─────────────────────────────────────────────────────────────────────────────┘

BACKEND (oms-ventas-backend)
────────────────────────────
src/
├── modules/
│   ├── service-calls/
│   │   ├── service-calls.module.ts
│   │   ├── service-calls.controller.ts
│   │   ├── service-calls.service.ts
│   │   ├── dto/
│   │   │   ├── create-service-call.dto.ts
│   │   │   ├── update-service-call.dto.ts
│   │   │   ├── change-status.dto.ts
│   │   │   └── filter-service-calls.dto.ts
│   │   ├── interfaces/
│   │   │   ├── service-call.interface.ts
│   │   │   └── service-call-response.interface.ts
│   │   └── constants/
│   │       ├── status.constants.ts
│   │       └── transitions.constants.ts
│   │
│   ├── activities/
│   │   ├── activities.module.ts
│   │   ├── activities.controller.ts
│   │   ├── activities.service.ts
│   │   └── dto/
│   │       └── create-activity.dto.ts
│   │
│   ├── equipment/
│   │   ├── equipment.module.ts
│   │   ├── equipment.controller.ts
│   │   ├── equipment.service.ts
│   │   └── dto/
│   │       ├── create-equipment.dto.ts
│   │       └── filter-equipment.dto.ts
│   │
│   └── service-contracts/
│       ├── service-contracts.module.ts
│       ├── service-contracts.controller.ts
│       └── service-contracts.service.ts


FRONTEND (OmsModernize)
───────────────────────
src/
├── app/
│   └── (DashboardLayout)/
│       └── service/
│           ├── page.tsx                     # Vista Kanban
│           ├── [id]/
│           │   └── page.tsx                 # Detalle del caso
│           └── layout.tsx
│
├── components/
│   └── service/
│       ├── ServiceKanbanBoard/
│       │   ├── index.tsx
│       │   ├── KanbanColumn.tsx
│       │   ├── ServiceCallCard.tsx
│       │   └── styles.ts
│       │
│       ├── ServiceCallModal/
│       │   ├── index.tsx
│       │   ├── HeaderSection.tsx
│       │   ├── CustomerSection.tsx
│       │   ├── EquipmentSection.tsx
│       │   ├── DiagnosticSection.tsx
│       │   ├── BudgetSection.tsx
│       │   ├── ActionsSection.tsx
│       │   └── tabs/
│       │       ├── InfoTab.tsx
│       │       ├── ActivitiesTab.tsx
│       │       └── DocumentsTab.tsx
│       │
│       ├── CustomerSearch/
│       │   ├── index.tsx
│       │   └── CustomerCard.tsx
│       │
│       ├── EquipmentForm/
│       │   ├── index.tsx
│       │   ├── EquipmentSelector.tsx
│       │   └── NewEquipmentForm.tsx
│       │
│       ├── ActivityTimeline/
│       │   ├── index.tsx
│       │   ├── ActivityItem.tsx
│       │   └── AddActivityForm.tsx
│       │
│       └── common/
│           ├── StatusBadge.tsx
│           ├── PriorityIndicator.tsx
│           └── TechnicianAvatar.tsx
│
├── hooks/
│   └── service/
│       ├── useServiceCalls.ts
│       ├── useServiceCall.ts
│       ├── useActivities.ts
│       ├── useEquipment.ts
│       ├── useServiceContracts.ts
│       └── useServicePermissions.ts
│
├── context/
│   └── ServiceContext/
│       ├── index.tsx
│       └── types.ts
│
├── types/
│   └── service/
│       ├── service-call.types.ts
│       ├── activity.types.ts
│       ├── equipment.types.ts
│       └── contract.types.ts
│
└── utils/
    └── omsBackend/
        └── serviceApi.ts                    # API client para servicio
```

---

## 8. Matriz de Permisos por Rol

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MATRIZ DE PERMISOS POR ROL                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────┬───────────┬──────────┬──────────┬─────────┬───────┐
│        ACCIÓN          │ Recepción │ Técnico  │   Jefe   │ Cajero  │ Admin │
│                        │           │          │  Taller  │         │       │
├────────────────────────┼───────────┼──────────┼──────────┼─────────┼───────┤
│ Ver lista de casos     │    ✓*     │    ✓*    │    ✓     │    ✓    │   ✓   │
│ Ver detalle de caso    │    ✓*     │    ✓*    │    ✓     │    ✓    │   ✓   │
│ Crear caso (ingreso)   │    ✓      │    ✗     │    ✓     │    ✗    │   ✓   │
│ Buscar/crear cliente   │    ✓      │    ✗     │    ✓     │    ✓    │   ✓   │
│ Registrar equipo       │    ✓      │    ✗     │    ✓     │    ✗    │   ✓   │
│ Asignar técnico        │    ✗      │    ✗     │    ✓     │    ✗    │   ✓   │
│ Registrar diagnóstico  │    ✗      │    ✓     │    ✓     │    ✗    │   ✓   │
│ Crear presupuesto      │    ✗      │    ✗     │    ✓     │    ✗    │   ✓   │
│ Cambiar estado         │    ✗      │    ✓**   │    ✓     │    ✓*** │   ✓   │
│ Agregar actividad      │    ✓      │    ✓     │    ✓     │    ✓    │   ✓   │
│ Solicitar repuestos    │    ✗      │    ✓     │    ✓     │    ✗    │   ✓   │
│ Generar factura        │    ✗      │    ✗     │    ✓     │    ✓    │   ✓   │
│ Registrar entrega      │    ✓      │    ✗     │    ✓     │    ✓    │   ✓   │
│ Cancelar caso          │    ✗      │    ✗     │    ✓     │    ✗    │   ✓   │
│ Ver reportes           │    ✗      │    ✗     │    ✓     │    ✗    │   ✓   │
│ Configurar sistema     │    ✗      │    ✗     │    ✗     │    ✗    │   ✓   │
├────────────────────────┴───────────┴──────────┴──────────┴─────────┴───────┤
│ NOTAS:                                                                      │
│ * Solo casos de su sucursal                                                 │
│ ** Solo casos asignados a él                                                │
│ *** Solo para transición Terminado → Cerrado (al facturar/entregar)         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Endpoints de API

### 9.1 Service Calls

| Método | Endpoint                        | Descripción           | Body/Query                                          | Respuesta                                |
| ------ | ------------------------------- | --------------------- | --------------------------------------------------- | ---------------------------------------- |
| GET    | `/service-calls`                | Listar casos          | `?status=2&assignee=5&branch=SUC01&page=1&limit=20` | `{ data: ServiceCall[], total: number }` |
| GET    | `/service-calls/:id`            | Detalle completo      | -                                                   | `ServiceCall` con relaciones             |
| POST   | `/service-calls`                | Crear caso            | `CreateServiceCallDto`                              | `ServiceCall`                            |
| PATCH  | `/service-calls/:id`            | Actualizar caso       | `UpdateServiceCallDto`                              | `ServiceCall`                            |
| PATCH  | `/service-calls/:id/status`     | Cambiar estado        | `{ status: number, notes?: string }`                | `ServiceCall`                            |
| GET    | `/service-calls/:id/activities` | Actividades del caso  | -                                                   | `Activity[]`                             |
| POST   | `/service-calls/:id/activities` | Agregar actividad     | `CreateActivityDto`                                 | `Activity`                               |
| GET    | `/service-calls/:id/expenses`   | Documentos vinculados | -                                                   | `InventoryExpense[]`                     |
| POST   | `/service-calls/:id/expenses`   | Vincular documento    | `{ docType, docEntry }`                             | `InventoryExpense`                       |

### 9.2 Equipment (Tarjetas de Equipo)

| Método | Endpoint                 | Descripción           | Body/Query                  | Respuesta                 |
| ------ | ------------------------ | --------------------- | --------------------------- | ------------------------- |
| GET    | `/equipment`             | Buscar equipos        | `?customer=C001&serial=ABC` | `CustomerEquipmentCard[]` |
| GET    | `/equipment/:id`         | Detalle de equipo     | -                           | `CustomerEquipmentCard`   |
| POST   | `/equipment`             | Registrar equipo      | `CreateEquipmentDto`        | `CustomerEquipmentCard`   |
| GET    | `/equipment/:id/history` | Historial de servicio | -                           | `ServiceCall[]`           |
|        |                          |                       |                             |                           |

### 9.3 Service Contracts

| Método | Endpoint                   | Descripción         | Body/Query                   | Respuesta                                          |
| ------ | -------------------------- | ------------------- | ---------------------------- | -------------------------------------------------- |
| GET    | `/service-contracts`       | Buscar contratos    | `?customer=C001&active=true` | `ServiceContract[]`                                |
| GET    | `/service-contracts/:id`   | Detalle de contrato | -                            | `ServiceContract`                                  |
| GET    | `/service-contracts/check` | Verificar cobertura | `?customer=C001&item=PROD01` | `{ covered: boolean, contract?: ServiceContract }` |

### 9.4 DTOs Principales

```typescript
// CreateServiceCallDto
{
  CustomerCode: string;           // Requerido
  InternalSerialNum?: string;     // Opcional si no tiene serial
  ItemCode?: string;              // Requerido si no hay serial
  Subject: string;                // Requerido
  Origin?: string;                // Ej: "Tienda", "Web"
  Priority?: "scp_Low" | "scp_Medium" | "scp_High";
  U_Sucursal: string;             // Requerido
  U_Accesorios?: string;
  U_Contrasenha?: string;
  Remarks?: string;
}

// UpdateServiceCallDto
{
  Subject?: string;
  Priority?: "scp_Low" | "scp_Medium" | "scp_High";
  AssigneeCode?: number;
  U_Diagnostico_Tecnico?: string;
  U_Presupuesto_Tentativo?: number;
  Resolution?: string;
  Remarks?: string;
}

// ChangeStatusDto
{
  status: -3 | -1 | 1 | 2 | 3 | 4 | 5;
  notes?: string;  // Obligatorio para ciertas transiciones
}

// CreateActivityDto
{
  ActivityType: "cn_Conversation" | "cn_Meeting" | "cn_Task" | "cn_Note" | "cn_Other";
  Notes: string;
  ActivityDate?: string;  // Default: hoy
}

// CreateEquipmentDto
{
  CustomerCode: string;
  ItemCode: string;
  ManufacturerSerialNum?: string;
  InstallLocation?: string;
}
```

---

## 10. Glosario

| Término                   | Definición                                                                    |
| ------------------------- | ----------------------------------------------------------------------------- |
| **ServiceCall**           | Entidad principal de SAP B1 que representa un caso de servicio técnico        |
| **CustomerEquipmentCard** | Tarjeta de equipo que registra un activo/dispositivo propiedad de un cliente  |
| **Activity**              | Registro de una interacción o seguimiento (llamada, nota, reunión)            |
| **ServiceContract**       | Contrato de servicio o garantía que cubre reparaciones                        |
| **InventoryExpense**      | Vínculo entre un ServiceCall y documentos transaccionales (órdenes, facturas) |
| **UDF**                   | User Defined Field - Campo personalizado en SAP B1                            |
| **State Machine**         | Patrón de diseño que controla las transiciones válidas entre estados          |
| **Kanban**                | Metodología visual de gestión de trabajo en columnas por estado               |

---

## Aprobaciones

| Rol             | Nombre             | Firma              | Fecha        |
| --------------- | ------------------ | ------------------ | ------------ |
| Líder de Equipo | ********\_******** | ********\_******** | **_/_**/2026 |
| Product Owner   | ********\_******** | ********\_******** | **_/_**/2026 |
| Tech Lead       | ********\_******** | ********\_******** | **_/_**/2026 |

---

## Historial de Versiones

| Versión | Fecha      | Autor             | Cambios         |
| ------- | ---------- | ----------------- | --------------- |
| 1.0     | 30/01/2026 | Equipo Desarrollo | Versión inicial |

---

_Documento generado como parte del análisis de sistemas para el Módulo de Servicio Técnico_

