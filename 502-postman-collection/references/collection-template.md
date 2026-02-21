# Postman Collection v2.1 - Template

## Estructura JSON completa

```json
{
  "info": {
    "name": "<Nombre del proyecto> API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json",
    "_postman_id": "<uuid-v4>",
    "description": "Coleccion generada automaticamente desde FastAPI"
  },
  "variable": [
    { "key": "base_url", "value": "http://localhost:8000", "type": "string" },
    { "key": "project_id", "value": "1", "type": "string" },
    { "key": "task_id", "value": "1", "type": "string" }
  ],
  "item": [
    {
      "name": "<Recurso, e.g. Projects>",
      "item": [
        {
          "name": "<Nombre del request, e.g. List Projects>",
          "request": {
            "method": "GET",
            "header": [
              { "key": "Content-Type", "value": "application/json" }
            ],
            "url": {
              "raw": "{{base_url}}/api/projects",
              "host": ["{{base_url}}"],
              "path": ["api", "projects"]
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 200', function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "const json = pm.response.json();",
                  "if (Array.isArray(json) && json.length > 0) {",
                  "    pm.collectionVariables.set('project_id', json[0].id);",
                  "}"
                ],
                "type": "text/javascript"
              }
            }
          ]
        }
      ]
    }
  ]
}
```

## Reglas de construccion

### Orden de requests dentro de cada carpeta
1. List (GET /)
2. Create (POST /)
3. Get by ID (GET /{id})
4. Update (PUT /{id} o PATCH /{id})
5. Delete (DELETE /{id})
6. Endpoints especiales (nested resources, acciones)

### URLs
- Siempre usar `{{base_url}}` como base
- Parametros de path como variables de coleccion: `{{project_id}}`, `{{task_id}}`

### Tests post-response
- Validar status code esperado: 200 para GET/PUT, 201 para POST, 204 para DELETE
- Para requests que devuelven un objeto con `id`, capturar ese ID en la variable de coleccion correspondiente con `pm.collectionVariables.set()`

### Bodies
- Generar JSON realista usando los campos del schema Pydantic correspondiente
- Usar valores de `json_schema_extra` si existen; si no, inventar valores representativos
- Solo incluir body en endpoints POST/PUT/PATCH
- Incluir header `Content-Type: application/json` en requests con body

### Paginacion
- Si hay parametros `skip`/`limit`, agregar como query params con defaults (skip=0, limit=10)
