---
name: 002-document-api
description: This skill should be used when the user asks to "document the API", "generate API documentation", "documentar la API", "documentar endpoints", "generar documentación de la API", or wants to create or update docs/API.md with FastAPI endpoint information.
version: 1.0.0
---

# 002 — Documentador de API FastAPI

Genera documentación completa para todos los endpoints FastAPI.

## Tareas:

1. Lee todos los routers en `src/api/routes/`
2. Extrae endpoints y schemas de cada archivo
3. Genera `docs/API.md` con:
   - Lista de todos los endpoints
   - Parámetros y tipos de cada endpoint
   - Ejemplos de request/response
   - Códigos de error posibles

## Formato de salida (docs/API.md):

```markdown
# API Documentation

## Endpoints

### Resource Name

#### `GET /resource`
Description

**Response (200):**
```json
[{ "id": 1, "name": "example" }]
```

#### `POST /resource`
Description

**Request:**
```json
{ "name": "example" }
```

**Response (201):**
```json
{ "id": 1, "name": "example" }
```
```

## Instrucciones:

1. Busca todos los archivos .py en src/api/routes/
2. Extrae la información de cada endpoint
3. Crea o actualiza docs/API.md
