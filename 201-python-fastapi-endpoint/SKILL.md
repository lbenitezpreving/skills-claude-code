---
name: 201-python-endpoint
description: This skill should be used when the user asks to "create a FastAPI endpoint", "add an endpoint", "crear endpoint FastAPI", "nuevo endpoint Python", "generar endpoint", or wants to scaffold a complete FastAPI endpoint with Pydantic v2 schemas, pagination, custom exceptions and async tests.
version: 2.0.0
argument-hint: "[endpoint_name]"
---

# 201 — Generador de Endpoints FastAPI con Mejores Practicas

Genera un endpoint completo para: `$ARGUMENTS`

## Archivos Base Requeridos

Antes de generar el endpoint, verifica que existan:
- `src/api/exceptions.py` - Excepciones personalizadas
- `src/api/pagination.py` - Utilidades de paginacion

Si no existen, crealos. **Lee `references/archivos-base.md`** para los templates.

## Estructura a crear

```
src/api/routes/$ARGUMENTS.py      # Router con endpoints CRUD
src/api/schemas/$ARGUMENTS.py     # Schemas Pydantic v2
tests/api/test_$ARGUMENTS.py      # Tests pytest
```

## Templates de Referencia

| Archivo | Contenido | Referencia |
|---------|-----------|------------|
| `src/api/exceptions.py` | ResourceNotFoundException, etc. | `references/archivos-base.md` |
| `src/api/pagination.py` | PaginatedResponse generico | `references/archivos-base.md` |
| Router CRUD | GET all, GET by ID, POST + decoradores | `references/router-templates.md` |
| Schemas Pydantic v2 | Base, Create, Update, Response | `references/schemas-template.md` |
| Tests | Paginacion, excepciones, CRUD | `references/tests-template.md` |

## Router: src/api/routes/$ARGUMENTS.py

Lee `references/router-templates.md` para los templates completos.

**Checklist:**
- [ ] Prefix: `/$ARGUMENTS`
- [ ] Imports: ResourceNotFoundException, PaginatedResponse, Query
- [ ] GET all con paginacion: skip, limit (Query), PaginatedResponse
- [ ] GET by id con ResourceNotFoundException
- [ ] POST, PUT, DELETE con excepciones apropiadas
- [ ] Decoradores completos: summary, description, response_description, responses
- [ ] Docstrings: descripcion, Args, Returns, Raises
- [ ] Async en todos los endpoints
- [ ] DB simulada: diccionario `_{resources}_db: Dict[int, dict] = {}`

## Schemas: src/api/schemas/$ARGUMENTS.py

Lee `references/schemas-template.md` para el template completo.

**Checklist:**
- [ ] Herencia: Base -> Create/Update/Response
- [ ] json_schema_extra con examples en Create y Response
- [ ] Field() con description en todos los campos
- [ ] ConfigDict apropiado (from_attributes=True en Response)
- [ ] Validaciones: min_length, max_length segun corresponda

## Tests: tests/api/test_$ARGUMENTS.py

Lee `references/tests-template.md` para los templates.

**Checklist:**
- [ ] Fixture client con TestClient
- [ ] Fixture reset_db (autouse=True) para limpiar DB
- [ ] Test paginacion: skip, limit, limite maximo 100
- [ ] Test excepciones personalizadas: 404 con formato correcto
- [ ] Test validaciones Pydantic: 422
- [ ] Test CRUD completo
- [ ] Coverage objetivo: >85%

## Instrucciones de Ejecucion

1. Verificar archivos base (exceptions.py, pagination.py)
2. Crear directorios necesarios si no existen
3. Generar router con todos los endpoints
4. Generar schemas con ejemplos y validaciones
5. Generar tests completos
6. Registrar router en `src/main.py` si es necesario
7. Ejecutar: `pytest tests/api/test_{resources}.py -v --cov`
8. Verificar cobertura >85%

## Notas de Nomenclatura

- `{RESOURCE}` -> PascalCase (ej: `UserProfile`)
- `{resources}` -> plural snake_case (ej: `user_profiles`)
- `{resource}` -> singular snake_case (ej: `user_profile`)
- Usar `datetime.now(UTC)` e importar `from datetime import datetime, UTC`
