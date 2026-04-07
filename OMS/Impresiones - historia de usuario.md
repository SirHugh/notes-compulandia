# Documentación de Historia de Usuario

---

## 1. Información General
- **ID:** US-001
- **Título:** Impresión de Documentos
- **Épica asociada:** Módulo de Impresiones
- **Proyecto / App:** OMS Backend
- **Fecha de creación:** 2025-03-23
- **Última actualización:** 2025-03-23
- **Autor:** -
- **Estado:** `[x] Backlog` `[ ] En progreso` `[ ] En revisión` `[ ] Completado`

---

## 2. Historia de Usuario
> **Como** usuario del sistema (vendedor, técnico, administrativo),
> **quiero** imprimir un documento (pedido, factura) presionando un botón de imprimir. con posibilidad de cambiar el layout y la impresora,
> **para** entregar una copia física al cliente o para archivo.

---

## 3. Descripción
El usuario necesita imprimir documentos del sistema (pedidos, facturas) de forma directa. Cada documento tiene un layout asignado por defecto, pero el usuario puede cambiarlo al momento de imprimir. El usuario también debe seleccionar la impresora destino y la cantidad de copias.

### Integración con Servicios Externos

El sistema se integra con dos servicios externos para completar el flujo de impresión:

#### SAP API Gateway (Puerto 60000)
- **Función:** Genera el PDF del documento usando Crystal Reports
- **Autenticación:** Sesión con credenciales de sistema (usuario/contraseña)
- **Output:** PDF codificado en **base64**
- **Endpoint:** `POST /rs/v1/ExportPDFData`

#### PrintNode
- **Función:** Envía el PDF a impresoras físicas remotas
- **Autenticación:** API Key
- **Input:** PDF en formato **base64** (compatible directo con el output del API Gateway)
- **Content-Type:** `pdf_base64`
- **Endpoint:** `POST https://api.printnode.com/printjobs`

### Flujo de Integración

```
┌──────────┐     ┌──────────────┐     ┌─────────────────┐     ┌───────────┐     ┌───────────┐
│ Frontend │────▶│ OMS Backend  │────▶│ SAP API Gateway │     │ PrintNode │────▶│ Impresora │
└──────────┘     └──────────────┘     └─────────────────┘     └───────────┘     └───────────┘
                        │                      │                    ▲
                        │   1. Request PDF     │                    │
                        │─────────────────────▶│                    │
                        │                      │                    │
                        │   2. PDF (base64)    │                    │
                        │◀─────────────────────│                    │
                        │                                           │
                        │   3. Enviar PDF (base64) + printerId      │
                        │──────────────────────────────────────────▶│
                        │                                           │
                        │   4. printJobId                           │
                        │◀──────────────────────────────────────────│
```

---

## 4. Actores Involucrados
| Actor | Rol |
|-------|-----|
| Usuario | Vendedor, técnico o administrativo que necesita imprimir documentos |
| Sistema OMS | Orquesta la generación del PDF y el envío a impresora |
| SAP API Gateway | Genera el PDF con Crystal Reports |
| PrintNode | Servicio de impresión en la nube |

---

## 5. Criterios de Aceptación

- [ ] **CA-01:** El usuario puede acceder a la opción de imprimir desde un documento (pedido o factura)
- [ ] **CA-02:** El sistema muestra el layout asignado por defecto para ese documento específico
- [ ] **CA-03:** ~~El usuario puede seleccionar un layout diferente de la lista de layouts disponibles~~
- [ ] **CA-04:** El usuario puede ver la lista de impresoras disponibles
- [ ] **CA-05:** El usuario puede seleccionar la impresora destino
- [ ] **CA-06:** ~~El usuario puede indicar la cantidad de copias a imprimir~~
- [ ] **CA-07:** Al confirmar, el sistema envía el documento a la impresora seleccionada
- [ ] **CA-08:** El usuario recibe confirmación del envío con el ID del trabajo de impresión
- [ ] **CA-09:** Si ocurre un error, el sistema muestra un mensaje descriptivo

---

## 6. Escenarios (Gherkin / BDD)

```gherkin
Escenario: Imprimir documento con layout por defecto e impresora preseleccionada
  Dado que el usuario está viendo un documento (pedido o factura)
  Y el documento tiene un layout asignado por defecto
  Y el documento tiene una impresora asignada por defecto en la succursal
  Cuando el usuario selecciona "Imprimir"
  ~~Y selecciona una impresora~~
  ~~Y confirma la impresión~~
  Entonces el sistema genera el PDF con el layout por defecto
  Y envía el documento a la impresora seleccionada
  Y muestra el mensaje "Documento enviado a imprimir" con el ID del trabajo
```

```gherkin
Escenario: Imprimir documento con layout diferente
  Dado que el usuario está viendo un documento
  Y existen múltiples layouts disponibles para ese tipo de documento
  Cuando el usuario selecciona "Imprimir"
  Y cambia el layout por uno diferente al default
  Y selecciona una impresora
  Y confirma la impresión
  Entonces el sistema genera el PDF con el layout seleccionado
  Y envía el documento a la impresora seleccionada
```

```gherkin
Escenario: Imprimir múltiples copias
  Dado que el usuario está en el diálogo de impresión
  Cuando indica una cantidad de copias mayor a 1
  Y confirma la impresión
  Entonces el sistema envía el trabajo con la cantidad de copias indicada
```

```gherkin
Escenario: No hay impresoras disponibles
  Dado que el usuario selecciona "Imprimir" en un documento
  Y no hay impresoras registradas en el sistema
  Cuando se carga el diálogo de impresión
  Entonces el sistema muestra el mensaje "No hay impresoras disponibles"
  Y deshabilita el botón de imprimir
```

---

## 7. Wireframes / Mockups

```
┌─────────────────────────────────────────────────────────────────┐
│                    Imprimir Documento                       [X] │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Documento: Pedido #15629                                       │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Layout:                                                    │ │
│  │ ┌────────────────────────────────────────────────────────┐ │ │
│  │ │ Pedido Estándar 2024                                 ▼ │ │ │
│  │ └────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Impresora:                                                 │ │
│  │ ┌────────────────────────────────────────────────────────┐ │ │
│  │ │ SUC-CENTRAL-LASER-01                                 ▼ │ │ │
│  │ └────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Copias:                                                    │ │
│  │ ┌──────────┐                                               │ │
│  │ │    1     │                                               │ │
│  │ └──────────┘                                               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                               [Cancelar]        [Imprimir]      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Flujo de Usuario

1. El usuario accede a un documento (pedido o factura)
2. El usuario selecciona la opción "Imprimir"
3. El sistema muestra el diálogo de impresión con:
   - El layout asignado por defecto para ese documento
   - La lista de layouts disponibles
   - La lista de impresoras disponibles
   - Campo para cantidad de copias (default: 1)
4. El usuario puede cambiar el layout si lo desea
5. El usuario selecciona la impresora destino
6. El usuario indica la cantidad de copias (opcional, default: 1)
7. El usuario presiona "Imprimir"
8. El sistema genera el PDF con el layout seleccionado
9. El sistema envía el PDF a PrintNode con la impresora y copias indicadas
10. El sistema muestra confirmación con el ID del trabajo de impresión

---

## 9. Reglas de Negocio

- **RN-01:** Cada documento tiene un layout asignado por defecto
- **RN-02:** El layout seleccionado al imprimir es solo para esa impresión, no modifica el default del documento
- **RN-03:** Las impresoras son globales; el nombre indica sucursal y tipo (ej: "SUC-CENTRAL-LASER-01")
- **RN-04:** La cantidad mínima de copias es 1
- **RN-05:** El usuario debe seleccionar una impresora para poder imprimir
- **RN-06:** Si no hay impresoras disponibles, no se puede imprimir

---

## 10. Requisitos Técnicos / Notas de Implementación

### Endpoints del Backend (OMS)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/reports/layouts/:documentType` | Lista layouts disponibles para un tipo de documento |
| GET | `/api/printers` | Lista impresoras disponibles en PrintNode |
| POST | `/api/print` | Envía documento a imprimir |

### Configuración Requerida (.env)

```env
# SAP Service Layer (ya existente)
SAP_SERVICE_LAYER_URL=https://sap-linux:50000/b1s/v2
SAP_SL_USER=usuario_sistema
SAP_SL_PASSWORD=password_sistema
SAP_COMPANY_DB=nombre_empresa

# Layouts por defecto (Crystal Reports)
REPORT_LAYOUT_ORDER=RCRI0026
REPORT_LAYOUT_INVOICE=RCRI0029

# PrintNode
PRINTNODE_API_KEY=tu_api_key_aqui
```

---

### SAP API Gateway - Generación de PDF

**URL Base:** Se deriva de `SAP_SERVICE_LAYER_URL` cambiando el puerto a 60000

**Autenticación:** Login con credenciales de sistema

```
POST https://sap-linux:60000/login
Content-Type: application/json

{
  "CompanyDB": "nombre_empresa",
  "UserName": "usuario_sistema",
  "Password": "password_sistema"
}
```

**Respuesta:** Cookies de sesión (`Session`, `ROUTEID`)

**Generación de PDF:**

```
POST https://sap-linux:60000/rs/v1/ExportPDFData?DocCode=RCRI0026
Content-Type: application/json
Cookie: Session=xxx; ROUTEID=.node1

[
  { "name": "Dockey@", "type": "xsd:string", "value": [["33008"]] },
  { "name": "FolioNum@", "type": "xsd:string", "value": [["15629"]] }
]
```

**Respuesta:** PDF en formato **base64** (string)

---

### PrintNode - Envío a Impresora

**URL Base:** `https://api.printnode.com`

**Autenticación:** Basic Auth con API Key

```
Authorization: Basic {base64(apiKey:)}
```

**Listar Impresoras:**

```
GET https://api.printnode.com/printers
Authorization: Basic {credentials}
```

**Respuesta:**
```json
[
  {
    "id": 12345,
    "name": "SUC-CENTRAL-LASER-01",
    "description": "HP LaserJet Pro",
    "state": "online"
  }
]
```

**Crear Trabajo de Impresión:**

```
POST https://api.printnode.com/printjobs
Authorization: Basic {credentials}
Content-Type: application/json

{
  "printerId": 12345,
  "title": "Pedido #15629",
  "contentType": "pdf_base64",
  "content": "JVBERi0xLjQK...",
  "source": "OMS Backend",
  "qty": 1
}
```

| Campo | Descripción |
|-------|-------------|
| `printerId` | ID de la impresora en PrintNode |
| `title` | Nombre del trabajo (visible en cola de impresión) |
| `contentType` | `pdf_base64` - indica que el contenido es PDF en base64 |
| `content` | El PDF en base64 (output directo del API Gateway) |
| `source` | Identificador de la aplicación origen |
| `qty` | Cantidad de copias |

**Respuesta Exitosa:**
```json
{
  "id": 789456
}
```

---

### Payload de Impresión (Frontend → Backend)

```json
{
  "documentType": "order",
  "docEntry": 33008,
  "docNum": 15629,
  "layoutCode": "RCRI0026",
  "printerId": 12345,
  "copies": 1
}
```

### Respuesta del Backend (Backend → Frontend)

```json
{
  "success": true,
  "printJobId": 789456,
  "message": "Documento enviado a imprimir"
}
```

---

### Flujo Interno del Backend

1. Recibe request del frontend con `docEntry`, `docNum`, `layoutCode`, `printerId`, `copies`
2. Obtiene sesión válida del API Gateway (login o cache)
3. Llama a `ExportPDFData` con el `layoutCode` y parámetros del documento
4. Recibe PDF en **base64** del API Gateway
5. Envía el PDF (base64) a PrintNode con `printerId` y `copies`
6. Recibe `printJobId` de PrintNode
7. Retorna confirmación al frontend

---

## 11. Dependencias

| ID Historia | Título | Tipo de dependencia |
|-------------|--------|----------------------|
| - | Configuración de PrintNode | Bloqueante |
| - | Registro de impresoras en PrintNode | Bloqueante |

---

## 12. Definición de Hecho (DoD)

- [ ] Código desarrollado y revisado (PR aprobado)
- [ ] Pruebas unitarias escritas y aprobadas
- [ ] Pruebas de integración pasadas
- [ ] Criterios de aceptación validados
- [ ] Documentación actualizada
- [ ] Desplegado en ambiente de QA

---

## 13. Estimación

| Métrica | Valor |
|---------|-------|
| Story Points | - |
| Esfuerzo estimado (horas) | - |
| Sprint asignado | - |

---

## 14. Historial de Cambios

| Fecha | Autor | Cambio realizado |
|-------|-------|-----------------|
| 2025-03-23 | - | Versión inicial |

```json
[
{

    "doc": "factura",

    "layout": "cri20002",
},
{

    "doc": "orden",

    "layout": "cri20002",
},
{

    "doc": "24",

    "layout": "cri20002",
},
{

    "doc": "nota de entrega",

    "layout": "cri20002",
},
{

    "doc": "pago",

    "layout": "cri20002",
},
{

    "doc": "nota de credito",

    "layout": "cri20002",
    
}
]
```

```yaml
  

    - entity: orders

      scrape_configs:

  

        # ── Servidor desarrollo ──────────────────────────────

        - layout: 'RVD20002'

          default: true

          printer_type: 'thermal'

          params:

            username: prometheus

            password: 'Compu2502##'

  

        # ── Servidor desarrollo ──────────────────────────────

        - layout: 'RVD20003'

          printer_type: 'thermal'

          params:

            username: prometheus

            password: 'Compu2502##'
```