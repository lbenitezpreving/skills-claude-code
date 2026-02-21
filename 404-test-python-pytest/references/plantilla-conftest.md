# Plantilla: conftest.py con Fixtures Compartidas

Extrae el patrón repetido de `test_engine`/`async_client`/`override_get_db`
a un `conftest.py` para evitar duplicación entre archivos de test.

## tests/conftest.py — fixtures del proyecto

```python
"""Fixtures compartidas para todos los tests del proyecto."""
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from src.main import app
from src.api.database import get_db, Base


# ─── Engine de test (una sola instancia por sesión de pytest) ─────────────────

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
test_async_session_maker = async_sessionmaker(
    test_engine, class_=AsyncSession, expire_on_commit=False
)


# ─── Fixtures ─────────────────────────────────────────────────────────────────

@pytest.fixture
async def test_db():
    """
    Crea todas las tablas antes de cada test y las destruye al finalizar.
    Garantiza aislamiento total entre tests.
    """
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield

    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


async def _override_get_db():
    """Reemplaza get_db con una sesión de BD en memoria."""
    async with test_async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


@pytest.fixture
async def async_client(test_db):
    """
    AsyncClient configurado con:
    - BD en memoria (via dependency_overrides)
    - ASGITransport para no necesitar servidor HTTP real

    Uso:
        async def test_algo(async_client):
            response = await async_client.get("/tasks/")
    """
    app.dependency_overrides[get_db] = _override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        yield client

    # CRÍTICO: limpiar siempre para no contaminar otros tests
    app.dependency_overrides.clear()


@pytest.fixture
async def db_session(test_db):
    """
    Sesión de BD directa para tests de servicios/utilidades
    que no pasan por HTTP.

    Uso:
        async def test_service(db_session):
            result = await my_service.get_all(db_session)
    """
    async with test_async_session_maker() as session:
        yield session
```

## tests/api/conftest.py — fixtures específicas de API

```python
"""Fixtures específicas para tests de API."""
import pytest
from httpx import AsyncClient


@pytest.fixture
async def task_en_backlog(async_client: AsyncClient) -> dict:
    """Crea una tarea en estado backlog lista para usar en tests."""
    response = await async_client.post("/tasks/", json={
        "name": "Tarea de prueba",
        "description": "Creada por fixture"
    })
    assert response.status_code == 201
    return response.json()


@pytest.fixture
async def task_completada(async_client: AsyncClient) -> dict:
    """Crea una tarea completada (status=done) lista para usar en tests."""
    create = await async_client.post("/tasks/", json={"name": "Tarea completada"})
    task_id = create.json()["id"]

    response = await async_client.put(f"/tasks/{task_id}", json={"completed": True})
    assert response.status_code == 200
    return response.json()


@pytest.fixture
async def proyecto(async_client: AsyncClient) -> dict:
    """Crea un proyecto listo para usar en tests."""
    response = await async_client.post("/projects/", json={
        "name": "Proyecto de prueba",
        "color": "#3b82f6"
    })
    assert response.status_code == 201
    return response.json()
```

## Uso de fixtures de conftest en los tests

```python
"""Test que usa fixtures del conftest."""
import pytest
from httpx import AsyncClient


class TestTasksConFixtures:
    """Tests usando fixtures compartidas del conftest."""

    @pytest.mark.asyncio
    async def test_lista_tareas_vacias(self, async_client: AsyncClient):
        """Usa async_client del conftest sin necesidad de definir nada."""
        response = await async_client.get("/tasks/")
        assert response.status_code == 200
        assert response.json() == []

    @pytest.mark.asyncio
    async def test_toggle_de_tarea_existente(
        self,
        async_client: AsyncClient,
        task_en_backlog: dict  # fixture específica de API
    ):
        """La tarea ya existe, no necesita crearla."""
        task_id = task_en_backlog["id"]

        response = await async_client.patch(f"/tasks/{task_id}/toggle")
        assert response.status_code == 200
        assert response.json()["completed"] is True

    @pytest.mark.asyncio
    async def test_tarea_con_proyecto(
        self,
        async_client: AsyncClient,
        proyecto: dict
    ):
        """Usa la fixture proyecto para obtener un ID real de proyecto."""
        project_id = proyecto["id"]

        response = await async_client.post("/tasks/", json={
            "name": "Tarea con proyecto",
            "project_id": project_id
        })
        assert response.status_code == 201
        assert response.json()["project_id"] == project_id
```

## pyproject.toml — configurar asyncio_mode

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"   # todas las fixtures/tests async se tratan como async automáticamente
```
