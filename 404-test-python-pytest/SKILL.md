---
name: 404-test-python-pytest
description: >
  Generates comprehensive tests for Python/FastAPI projects using pytest and httpx AsyncClient.
  Ensures zero real database interaction, follows AAA pattern, uses async fixtures and dependency overrides.
  Use when the user says "create tests", "write unit tests", "add tests for", "generate tests",
  "test coverage", or when reviewing untested Python/FastAPI code.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
argument-hint: "[path/to/router_or_service.py]"
---

# Testing Python/FastAPI — pytest + httpx AsyncClient

## Proceso (seguir en orden)

**Paso 1 — Analizar el archivo objetivo**
- Identifica el tipo: endpoint FastAPI, servicio de lógica, schema Pydantic, función utilitaria
- Lee las dependencias inyectadas (get_db, auth, etc.) que deben sobreescribirse
- Detecta llamadas a BD, APIs externas u otro I/O que deba mockearse
- Lista endpoints/métodos públicos: happy path, validaciones, 404, 422, edge cases
- Comprueba si ya existen tests para no duplicarlos y si hay `conftest.py`

**Paso 2 — Verificar configuración pytest**
Lee `references/pytest-coverage-config.md` para la config de `pyproject.toml`, pytest-cov y mutmut.

**Paso 3 — Escribir los tests**
Selecciona la plantilla según el tipo de componente:

| Tipo | Plantilla |
|------|-----------|
| Endpoint FastAPI (router) | `references/plantilla-endpoint.md` |
| Lógica de negocio (service) | `references/plantilla-service.md` |
| Schema Pydantic (validación) | `references/plantilla-schema.md` |
| Función utilitaria pura | `references/plantilla-utility.md` |
| Fixtures compartidas | `references/plantilla-conftest.md` |
| monkeypatch, caplog, parametrize, freezegun | `references/patrones-avanzados.md` |
| Mutation testing (mutmut) | `references/patrones-mutation-testing.md` |

**Paso 4 — Ejecutar y verificar cobertura**
```bash
pytest tests/ -v                                   # ejecutar todos los tests
pytest tests/ --cov=src --cov-report=html          # con cobertura HTML
pytest tests/ --cov=src --cov-fail-under=80        # falla si < 80%
mutmut run                                         # mutation testing
mutmut results                                     # ver resultados
mutmut html                                        # reporte HTML
```
Reportes:
- pytest-cov: `htmlcov/index.html`
- mutmut:     `html/index.html`

---

## Reglas de oro (NO NEGOCIABLES)

### ❌ NUNCA en tests de FastAPI
```python
# NUNCA usar BD real en tests
engine = create_async_engine("sqlite+aiosqlite:///./app.db")  # BD de producción

# NUNCA tests síncronos para código async
def test_algo():          # falta async — el código async no se ejecuta correctamente
    result = await coro() # SyntaxError en función síncrona

# NUNCA setup procedural cuando se puede usar fixtures
class TestEndpoints:
    def setup_method(self):
        # crear cliente, BD, etc. manualmente — viola principio DRY
```

### ✅ SIEMPRE: BD en memoria para tests
```python
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
```

### ✅ SIEMPRE: AsyncClient con ASGITransport
```python
async with AsyncClient(
    transport=ASGITransport(app=app),
    base_url="http://test"
) as client:
    yield client
```

### ✅ SIEMPRE: dependency_overrides para inyectar BD de test
```python
app.dependency_overrides[get_db] = override_get_db
# ... tests ...
app.dependency_overrides.clear()  # limpiar SIEMPRE al finalizar
```

### ✅ SIEMPRE: fixtures async para setup/teardown
```python
@pytest.fixture
async def test_db():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
```

---

## Checklist de calidad

**Estructura general**
- [ ] Archivo de test en `tests/` espejando la estructura de `src/`
- [ ] Usa clases `TestXxx` para agrupar tests relacionados
- [ ] Al menos un test por endpoint: happy path + error (404/422)
- [ ] Usa fixtures en lugar de setup procedural
- [ ] `conftest.py` con fixtures compartidas (`test_db`, `async_client`)

**BD y async**
- [ ] Usa `sqlite+aiosqlite:///:memory:` — NUNCA BD real
- [ ] Todos los tests de async son `async def` con `@pytest.mark.asyncio`
- [ ] `dependency_overrides` limpiado con `.clear()` en el fixture
- [ ] Tablas creadas antes y destruidas después de cada test

**Assertions**
- [ ] Verifica `response.status_code` exacto (200, 201, 204, 404, 422)
- [ ] Verifica campos del JSON con `response.json()["campo"] == valor`
- [ ] Usa `assert len(result) == N` exacto, no solo `assert result`
- [ ] Tests de 404: endpoint con ID inexistente (999)
- [ ] Tests de 422: payload inválido (nombre vacío, tipo incorrecto)

**Cobertura y mutaciones**
- [ ] pytest-cov muestra ≥ 80% de líneas cubiertas
- [ ] Se usan comparaciones exactas `== valor`, no solo `is not None`
- [ ] Hay tests para ambas ramas de cada condicional importante
- [ ] mutmut muestra ≥ 80% de mutation score

---

## Convención de nombres

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Archivo de test | `test_{módulo}.py` | `test_tasks.py` |
| Clase de test | `Test{Entidad}{Tipo}` | `TestTasksEndpoints`, `TestTaskStatus` |
| Método de test | `test_{acción}_{condición}` | `test_create_task_invalid_empty_name` |
| Fixture de BD | `test_db` | fixture estándar del proyecto |
| Fixture de cliente | `async_client` | fixture estándar del proyecto |
| Datos de prueba | `{entidad}_data` o inline | `task_data = {"name": "Test"}` |
