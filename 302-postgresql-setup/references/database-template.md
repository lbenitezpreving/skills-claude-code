# Template: database.py para PostgreSQL

## src/api/database.py

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

from .config import settings

engine = create_async_engine(
    settings.database_url,
    pool_size=settings.database_pool_size,
    max_overflow=settings.database_max_overflow,
    pool_pre_ping=True,  # detecta conexiones muertas
    echo=settings.is_development,  # logs SQL en dev
)

AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    expire_on_commit=False,
    autoflush=False,
    autocommit=False,
)


class Base(DeclarativeBase):
    pass


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


async def init_db() -> None:
    """Solo para desarrollo. En produccion usar Alembic."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


async def drop_db() -> None:
    """Util para tests."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
```

## src/main.py (lifespan)

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from .api.database import engine, init_db
from .api.config import settings


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    if settings.is_development:
        await init_db()  # en produccion, usa alembic upgrade head
    yield
    # Shutdown
    await engine.dispose()


app = FastAPI(title="Mi API", lifespan=lifespan)
```

## Alternativa con psycopg3

```bash
pip install psycopg[binary,pool]>=3.1
```

```python
# Cambiar URL:
DATABASE_URL = "postgresql+psycopg://user:pass@localhost:5432/dbname"
# o async:
DATABASE_URL = "postgresql+psycopg_async://user:pass@localhost:5432/dbname"
```

> psycopg3 vs asyncpg: psycopg3 soporta COPY y mas tipos nativos.
> asyncpg es mas rapido pero menos compatible.
