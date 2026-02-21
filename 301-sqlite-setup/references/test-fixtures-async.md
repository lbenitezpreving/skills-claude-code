# Template: Fixtures Async para Tests

## tests/api/test_$ARGUMENTS.py

```python
"""Tests para {resources} con SQLite."""
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from src.main import app
from src.api.database import get_db, Base

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
test_async_session_maker = async_sessionmaker(
    test_engine, class_=AsyncSession, expire_on_commit=False
)


@pytest.fixture
async def test_db():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


async def override_get_db():
    async with test_async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


@pytest.fixture
async def async_client(test_db):
    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        yield client
    app.dependency_overrides.clear()


@pytest.mark.asyncio
async def test_create_{resource}(async_client):
    response = await async_client.post(
        "/{resources}/",
        json={"name": "Test", "description": "Test desc"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test"
    assert "id" in data
```

## Configuracion pytest-asyncio en pyproject.toml

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```
