# Plantilla: Test de Endpoint FastAPI

Basado en el patrón canónico de `tests/api/test_tasks.py`.

## Estructura base completa

```python
"""Tests para los endpoints de {entidad} con SQLite en memoria."""
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from src.main import app
from src.api.database import get_db, Base


# ─── Configuración de BD de test ─────────────────────────────────────────────

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
test_async_session_maker = async_sessionmaker(
    test_engine, class_=AsyncSession, expire_on_commit=False
)


@pytest.fixture
async def test_db():
    """Crea tablas antes de cada test y las destruye al finalizar."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


async def override_get_db():
    """Override de get_db para usar BD en memoria durante tests."""
    async with test_async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


@pytest.fixture
async def async_client(test_db):
    """AsyncClient con BD de test inyectada via dependency_overrides."""
    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        yield client

    app.dependency_overrides.clear()  # SIEMPRE limpiar al finalizar


# ─── Tests de endpoints CRUD ──────────────────────────────────────────────────

class TestMiEntidadEndpoints:
    """Tests para los endpoints CRUD de {entidad}."""

    @pytest.mark.asyncio
    async def test_get_all_empty(self, async_client):
        """GET /{entidades}/ devuelve lista vacía."""
        response = await async_client.get("/{entidades}/")
        assert response.status_code == 200
        assert response.json() == []

    @pytest.mark.asyncio
    async def test_create(self, async_client):
        """POST /{entidades}/ crea una entidad y devuelve 201."""
        data = {"name": "Mi entidad", "description": "Descripción"}
        response = await async_client.post("/{entidades}/", json=data)

        assert response.status_code == 201
        result = response.json()
        assert result["name"] == "Mi entidad"
        assert result["description"] == "Descripción"
        assert "id" in result
        assert "created_at" in result

    @pytest.mark.asyncio
    async def test_create_minimal(self, async_client):
        """POST /{entidades}/ con solo los campos requeridos."""
        response = await async_client.post("/{entidades}/", json={"name": "Mínimo"})
        assert response.status_code == 201
        assert response.json()["name"] == "Mínimo"

    @pytest.mark.asyncio
    async def test_create_invalid_empty_name(self, async_client):
        """POST /{entidades}/ con nombre vacío devuelve 422."""
        response = await async_client.post("/{entidades}/", json={"name": ""})
        assert response.status_code == 422

    @pytest.mark.asyncio
    async def test_create_missing_required_field(self, async_client):
        """POST /{entidades}/ sin campo requerido devuelve 422."""
        response = await async_client.post("/{entidades}/", json={})
        assert response.status_code == 422

    @pytest.mark.asyncio
    async def test_get_by_id(self, async_client):
        """GET /{entidades}/{id} devuelve la entidad correcta."""
        create_response = await async_client.post("/{entidades}/", json={"name": "Test"})
        entity_id = create_response.json()["id"]

        response = await async_client.get(f"/{entidades}/{entity_id}")
        assert response.status_code == 200
        assert response.json()["name"] == "Test"
        assert response.json()["id"] == entity_id

    @pytest.mark.asyncio
    async def test_get_by_id_not_found(self, async_client):
        """GET /{entidades}/999 devuelve 404."""
        response = await async_client.get("/{entidades}/999")
        assert response.status_code == 404

    @pytest.mark.asyncio
    async def test_update(self, async_client):
        """PUT /{entidades}/{id} actualiza campos correctamente."""
        create_response = await async_client.post("/{entidades}/", json={"name": "Original"})
        entity_id = create_response.json()["id"]

        response = await async_client.put(
            f"/{entidades}/{entity_id}",
            json={"name": "Actualizado"}
        )

        assert response.status_code == 200
        assert response.json()["name"] == "Actualizado"
        assert response.json()["id"] == entity_id

    @pytest.mark.asyncio
    async def test_update_not_found(self, async_client):
        """PUT /{entidades}/999 devuelve 404."""
        response = await async_client.put("/{entidades}/999", json={"name": "X"})
        assert response.status_code == 404

    @pytest.mark.asyncio
    async def test_delete(self, async_client):
        """DELETE /{entidades}/{id} elimina la entidad."""
        create_response = await async_client.post("/{entidades}/", json={"name": "A eliminar"})
        entity_id = create_response.json()["id"]

        response = await async_client.delete(f"/{entidades}/{entity_id}")
        assert response.status_code == 204

        # Verificar que ya no existe
        get_response = await async_client.get(f"/{entidades}/{entity_id}")
        assert get_response.status_code == 404

    @pytest.mark.asyncio
    async def test_delete_not_found(self, async_client):
        """DELETE /{entidades}/999 devuelve 404."""
        response = await async_client.delete("/{entidades}/999")
        assert response.status_code == 404

    @pytest.mark.asyncio
    async def test_list_multiple(self, async_client):
        """GET /{entidades}/ devuelve todos los elementos creados."""
        await async_client.post("/{entidades}/", json={"name": "Entidad 1"})
        await async_client.post("/{entidades}/", json={"name": "Entidad 2"})
        await async_client.post("/{entidades}/", json={"name": "Entidad 3"})

        response = await async_client.get("/{entidades}/")
        assert response.status_code == 200
        result = response.json()
        assert len(result) == 3  # exacto — mata mutantes "lista vacía"
```

## Variante: soft delete (patrón del proyecto)

```python
@pytest.mark.asyncio
async def test_soft_delete(self, async_client):
    """DELETE con soft delete: entidad oculta pero recuperable."""
    create_response = await async_client.post("/{entidades}/", json={"name": "Soft delete"})
    entity_id = create_response.json()["id"]
    assert create_response.json()["deleted_at"] is None

    # Eliminar (soft delete)
    response = await async_client.delete(f"/{entidades}/{entity_id}")
    assert response.status_code == 204

    # Sin show_deleted: no visible
    get_response = await async_client.get(f"/{entidades}/{entity_id}")
    assert get_response.status_code == 404

    # Con show_deleted=true: visible con deleted_at
    get_response = await async_client.get(f"/{entidades}/{entity_id}?show_deleted=true")
    assert get_response.status_code == 200
    assert get_response.json()["deleted_at"] is not None
```
