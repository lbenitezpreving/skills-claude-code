# Template: database.py + main.py lifespan

## src/api/database.py

```python
"""Configuracion de base de datos SQLite con SQLAlchemy 2.0 async."""
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "sqlite+aiosqlite:///./app.db"

engine = create_async_engine(
    DATABASE_URL,
    echo=False,
    future=True,
)

async_session_maker = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


class Base(DeclarativeBase):
    """Base class para todos los modelos ORM."""
    pass


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """
    Dependency para obtener sesion de base de datos.

    Yields:
        AsyncSession: Sesion async de SQLAlchemy
    """
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()


async def init_db() -> None:
    """
    Inicializa la base de datos creando todas las tablas.

    Se debe llamar en el evento startup de FastAPI.
    """
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

## Actualizar src/main.py (lifespan)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from src.api.database import init_db


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Gestiona el ciclo de vida de la aplicacion."""
    await init_db()
    yield


app = FastAPI(lifespan=lifespan)
```
