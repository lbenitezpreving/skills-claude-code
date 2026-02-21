# Templates: Tests Async para PostgreSQL

## Opcion A: BD PostgreSQL de Test (recomendado)

Requiere una BD PostgreSQL real corriendo localmente o en CI.

### conftest.py

```python
import asyncio
from collections.abc import AsyncGenerator

import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from src.api.database import Base, get_db
from src.main import app

# BD de test separada (definir en .env.test o hardcoded aqui)
TEST_DATABASE_URL = "postgresql+asyncpg://postgres:password@localhost:5432/myapp_test"

test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, expire_on_commit=False)


@pytest_asyncio.fixture(scope="session")
async def setup_db():
    """Crea y destruye las tablas una vez por sesion de tests."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await test_engine.dispose()


@pytest_asyncio.fixture
async def db_session(setup_db) -> AsyncGenerator[AsyncSession, None]:
    """Session con rollback automatico para aislar cada test."""
    async with test_engine.connect() as conn:
        await conn.begin()
        async with TestSessionLocal(bind=conn) as session:
            yield session
        await conn.rollback()


@pytest_asyncio.fixture
async def async_client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """Cliente HTTP con BD de test inyectada."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

## Opcion B: SQLite en Memoria (para unit tests rapidos)

Util cuando no quieres depender de PostgreSQL en CI o tests unitarios.
Limitacion: no soporta tipos nativos PostgreSQL (JSONB, ARRAY, UUID nativo).

```python
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from src.api.database import Base, get_db
from src.main import app

SQLITE_TEST_URL = "sqlite+aiosqlite:///:memory:"

@pytest_asyncio.fixture
async def db_session():
    engine = create_async_engine(SQLITE_TEST_URL, echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    Session = async_sessionmaker(engine, expire_on_commit=False)
    async with Session() as session:
        yield session

    await engine.dispose()
```

## Ejemplo de Tests

```python
# tests/api/test_items.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_create_item(async_client: AsyncClient):
    response = await async_client.post(
        "/items/",
        json={"name": "Test Item", "description": "Descripcion de test"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test Item"
    assert "id" in data
    assert "created_at" in data


@pytest.mark.asyncio
async def test_list_items_empty(async_client: AsyncClient):
    response = await async_client.get("/items/")
    assert response.status_code == 200
    assert response.json() == []


@pytest.mark.asyncio
async def test_get_item_not_found(async_client: AsyncClient):
    fake_uuid = "00000000-0000-0000-0000-000000000000"
    response = await async_client.get(f"/items/{fake_uuid}")
    assert response.status_code == 404


@pytest.mark.asyncio
async def test_update_item(async_client: AsyncClient):
    # Crear
    create_response = await async_client.post("/items/", json={"name": "Original"})
    item_id = create_response.json()["id"]

    # Actualizar
    update_response = await async_client.put(
        f"/items/{item_id}",
        json={"name": "Actualizado"},
    )
    assert update_response.status_code == 200
    assert update_response.json()["name"] == "Actualizado"


@pytest.mark.asyncio
async def test_delete_item(async_client: AsyncClient):
    create_response = await async_client.post("/items/", json={"name": "Para borrar"})
    item_id = create_response.json()["id"]

    delete_response = await async_client.delete(f"/items/{item_id}")
    assert delete_response.status_code == 204

    get_response = await async_client.get(f"/items/{item_id}")
    assert get_response.status_code == 404


@pytest.mark.asyncio
async def test_create_duplicate_fails(async_client: AsyncClient):
    await async_client.post("/items/", json={"name": "Duplicado"})
    response = await async_client.post("/items/", json={"name": "Duplicado"})
    assert response.status_code == 409
```

## Comandos de Tests

```bash
# Ejecutar todos los tests
pytest tests/ -v

# Solo tests de API
pytest tests/api/ -v

# Con coverage
pytest tests/ --cov=src/api --cov-report=term-missing

# Ejecutar un test especifico
pytest tests/api/test_items.py::test_create_item -v

# Ver output aunque pase
pytest tests/ -v -s
```
