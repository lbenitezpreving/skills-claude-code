# Postman Environment - Template

## Estructura JSON completa

```json
{
  "id": "<uuid-v4>",
  "name": "<Nombre del proyecto> - Development",
  "values": [
    { "key": "base_url", "value": "http://localhost:8000", "enabled": true, "type": "default" },
    { "key": "project_id", "value": "", "enabled": true, "type": "default" },
    { "key": "task_id", "value": "", "enabled": true, "type": "default" }
  ],
  "_postman_variable_scope": "environment",
  "info": {
    "name": "<Nombre del proyecto> - Development",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/environment.json"
  }
}
```

## Notas

- Agregar una variable por cada tipo de ID que aparezca en los endpoints (project_id, task_id, subtask_id...) con valor vacio para que se rellene dinamicamente con los tests post-response.
- El `base_url` debe ser el unico valor pre-rellenado.
- Usar `"type": "default"` y `"enabled": true` para todas las variables.
