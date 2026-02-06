# Implementacion del workflow en en integrador
Solo implementado para cambios de stock y precio del SupplierProduct. 

- Dos Workflows creados:
	- ProductSyncWokflow
	- SupplierProductChangeWorkflow

## ProductSyncWorkflow
Este workflows es encargador de gestionar y nomitoreas las sincronizaciones con los canales:
-  woocomerce
-  tienda naranja
-  contimarket
-  medusa
-  algolia

## SupplierProductChangeWorkflow:
Este workflow se encarga de gestionar los cambios en el modelo SupplierProduct con el propósito de determinar si es elegible para sincronización. 
### Funcionamiento:
Cuando un producto de proveedor recibe una actulizacion en precio y stock, estos se validad para confirmar que el cambio es efectivo. Los cambios detectados si corresponden son pasados al SupplierProductChangeWorkflow. Este gestiona, segun los cambios recibidos, y va pasando en forma secuencial las actividades(jobs) siguientes:
- CalculatePricesActivity: si hubo cambio de precios
- GenerateContentActivity: si hubo cambios de atributos
- EvaluateSelectedActivity: si hubo cambios de supplier o stock o precio. 
- ProductSyncWorkflow: si hubo cambio que justifica que se sincronice.

