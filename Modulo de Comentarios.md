# Campos a guardar
- Entidad relacionada (código - nombre)
- id del objeto 
- usuario
- comentario
- adjunto (array)
- timestamp

| entidad   | id_objeto | usuario   | comentario           | adjunto           |
| --------- | --------- | --------- | -------------------- | ----------------- |
| actividad | 200       | mvaliente | averiguar presupueto | [ d6fg456s8h4a6 ] |
| actividad | 200       | hquintero | https://lsldkajfñ    | [ ]               |

Pasos:
## planeamiento
- cargar tareas en Jira
## backend
- definición de datos, dto.
- Integración con Firebase/Algolia/SQLite.
- Integración con cloudflare. 
- Servicio
- Api
## frontend
- componente para agregar comentarios.
- Interfaz de visualización

## historia
Usuario: Como usuario quiero poder agregar comentarios a actividades u otro tipo de entidad de sap. También quiero poder subir imágenes. Podría necesitar editar los comentarios. 

### criterios de aceptación.
- agregar comentario
- editar comentario
- agregar imágenes con comentario