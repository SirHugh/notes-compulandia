[[Implementacion Verificacion TN]]
## El problema: Race Condition.

 ```
 
 tienda naraja duplica los productos. esto sucede cuando un producto nuesvo es creado, este entra en cola de procesamiento. luego el producto sufre un cambio y se dispacha una sincronizacion antes que haya sido procesado el producto. este como no esta aun creado se vuelve a enviar y entra en cola de sincronizacion. luego se tiene la situacion de dos productos con el mismo sku en cola para ser procesado. ambos se crean con el mismo sku, pero el segundo con un sufijo '-1'.
 
 ```
 
## Solución
Tienda Naranja provee el siguiente endpoint para verificar el estado de procesamiento de un producto en cola.

~~~
{dominio}/rest/V1/mpapi/sellers/me/product-status/{process_id}
~~~
 
### Puntos de cambios
#### Job de sincronización de tienda naranja
Se agregara un proceso de verificación precio a la sincronización de un producto, este será encargado de ver si el producto ya fue creado. 

#### Job de verificación de estado de sincronización. 
Sera modificado para:
1.  bloquear otros intentos de encolar la misma instancia del job.
2.  será encargado de verificar todos los productos sincronizados con estado de EN_COLA_TN.
3.  cambiara el estado del registro de sincronización del producto. 
4.  Cuando se encuentre un producto con estado "queued" volverá a encolar la misma instancia del job para verificar si el producto esta procesado y actualizado.