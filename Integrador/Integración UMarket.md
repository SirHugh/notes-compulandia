## Dar de alta al Partner: 
	-  Crear el partner
	-  migrar las categorias del partner a la base de datos, en la tabla category
-  **obs**: crear Seeder para las migración de datos iniciales.

## Relacionamiento de producto con categoría de UMarket 
## Category

| id  | nombre             | categorizable_Id | categoriazable_type | master_category_id |
| --- | ------------------ | ---------------- | ------------------- | ------------------ |
| 120 | Auricular          | 1                | supplier            |                    |
| 543 | Auricular          | 2                | partner             |                    |
| 564 | Auricular y sonido | 1                | partner             |                    |
| 896 | Auricular          | 4                | partner             | 543                |

## Match Category

|      |      |
| ---- | ---- |
| id_a | id_b |
| 120  | 543  |
| 120  | 564  |
| 120  | 896  |
## Product

| id  | nombre              |     |
| --- | ------------------- | --- |
| 1   | auricular jbl 20    |     |
| 5   | auricular xiomi xls |     |
## Product Partner Categories

| id_product | id_category |     |        |
| ---------- | ----------- | --- | ------ |
| 1          | 543         |     | cl     |
| 1          | 564         |     | tn     |
| 5          | 543         |     | cl     |
| 5          | 564         |     | tn     |
| ==1==      | ==896==     |     | ==um== |
| ==5==      | ==896==     |     | ==um== |

**Obs**: 
	Crear comando para inferir la relación entre categorias de los partner CL y UM. Con el propósito de encontrar la lista de productos relacionada a la categoría del partner CL y asignarlas a la categoría relacionada del partner UM. 
	Se puede hacer esto atreves de la relación que se tiene en match_category entre supplier CL y partners CL y UM . 
	Relacionar los productos que se encuentran relacionados con la categoría del partner CL, a la categoría del nuevo partner UM relacionada.  
	Solo se puede tener una categoría por producto por partner. 
	Tambien se debe relacionar la categoria del partner um con la categoria de 
## Formulario de confirmación
- En el formulario de confirmación de productos de la lista de Pendientes, agregar el campo de asignación y recomendación de categoría para el nuevo partner UMarket para el producto siguiendo la forma como se hizo para los partners Compulandia y Tienda Naranja. Esto es el el paso 2 del formulario. 
## Generación.
- Se quiere generar archivo xls con el formato definido por el Partner, un archivo por categoría con la lista de productos de la categoría. 
- La generación de la lista de productos por categoría, se hará sobre las categorías activas del partner UMarket. 
- Incluir en la ui en la sección de Utilidades un enlace a una pagina para descargar los .xlsx generados. 


---
## Problemas encontrados
-  No se relacionan todos los productos de la categoría Master a la categoría de umarket
-  Se agregan productos desactivados. 
-  Se agregan productos que tiene otro supplier seleccionado. 