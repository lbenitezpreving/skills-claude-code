---
name: 502-postman-collection
description: >
  This skill should be used when the user asks to "generate postman collection",
  "create postman", "export to postman", "generar colección postman",
  "crear colección postman", "exportar API a postman", or wants to create
  API testing collections from FastAPI endpoints.
version: 2.0.0
---

# Skill: Generar Coleccion Postman desde FastAPI

Tu tarea es generar una coleccion Postman v2.1 completa a partir de los endpoints del proyecto.

## Proceso

### Paso 1: Descubrir endpoints

Lee los siguientes archivos para mapear todos los endpoints disponibles:

1. `src/main.py` — identifica routers registrados y sus prefijos
2. Todos los archivos en `src/api/routes/` — extrae:
   - Metodo HTTP (`@router.get/post/put/delete/patch`)
   - Path del endpoint
   - Nombre de la funcion y docstring
   - Parametros de path, query y body

Construye una tabla de todos los endpoints con metodo, URL completa y schema.

Si la tecnología no es python, analiza los ficheros para buscar los endpoints disponibles.

### Paso 2: Leer schemas Pydantic

Lee todos los archivos en `src/api/schemas/` y extrae:
- Campos con sus tipos, opcionalidad y valores por defecto
- Ejemplos definidos en `json_schema_extra` o `model_config`.

Si no es python, busca la misma información.

### Paso 3: Generar la coleccion Postman

Crea `docs/postman/collection.json` con formato **Postman Collection v2.1**.

Lee `references/collection-template.md` para la estructura JSON completa y las reglas de construccion.

**Reglas clave:**
- Agrupa endpoints por recurso (prefijo del router -> nombre de carpeta)
- URLs: usa siempre `{{base_url}}` y variables de coleccion para IDs
- Bodies: genera JSON realista desde los schemas Pydantic
- Tests post-response: valida status code y captura IDs dinamicamente
- Si hay paginacion (skip/limit), agrega como query params con defaults

### Paso 4: Generar el archivo de environment

Crea `docs/postman/environment.json`.

Lee `references/environment-template.md` para la estructura JSON completa.

Incluye una variable por cada tipo de ID presente en los endpoints.

### Paso 5: Guardar y reportar

1. Asegurate de que `docs/postman/` existe (crealo si no)
2. Guarda `collection.json` y `environment.json`
3. Reporta:
   - Total de endpoints incluidos, agrupados por recurso
   - Rutas de los archivos generados
   - Instrucciones de importacion en Postman:
     1. Import -> seleccionar `collection.json`
     2. Environments -> Import -> seleccionar `environment.json`
     3. Activar environment y ejecutar con "Run Collection"
   - Advertencia si algun endpoint no pudo mapearse

## Notas importantes

- Genera UUIDs v4 validos para `_postman_id` e `id` del environment
- Si hay autenticacion (Bearer/API key), detecta headers y agrega `auth_token` como variable de coleccion con pre-request script en la carpeta raiz
- El JSON final debe ser valido y parseable
