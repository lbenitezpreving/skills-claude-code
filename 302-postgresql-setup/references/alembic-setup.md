# Template: Alembic para PostgreSQL Async

## 1. Inicializar Alembic

```bash
alembic init alembic
```

Estructura resultante:
```
alembic/
├── env.py
├── script.py.mako
└── versions/
alembic.ini
```

## 2. alembic.ini

Cambiar la linea `sqlalchemy.url` para leer desde variable de entorno:

```ini
# Busca: sqlalchemy.url = driver://user:pass@localhost/dbname
# Reemplaza por:
sqlalchemy.url = %(DATABASE_URL)s
```

> O dejarla vacia y manejarla todo en env.py (ver abajo).

## 3. alembic/env.py (async completo)

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine

# Importar Base y Settings del proyecto
from src.api.database import Base
from src.api.config import settings

# Necesario para que Alembic detecte los modelos
import src.api.models  # noqa: F401  <- importar todos los modelos aqui

config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    """Migraciones sin conexion activa (genera SQL)."""
    url = settings.database_url
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Migraciones con motor async."""
    engine = create_async_engine(settings.database_url)
    async with engine.begin() as conn:
        await conn.run_sync(do_run_migrations)
    await engine.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

## 4. src/api/models/__init__.py

```python
# Importar todos los modelos para que Alembic los detecte
from .item import Item
from .category import Category

__all__ = ["Item", "Category"]
```

## 5. Comandos Alembic del dia a dia

```bash
# Crear migracion automatica (tras cambiar modelos)
alembic revision --autogenerate -m "descripcion del cambio"

# Aplicar todas las migraciones pendientes
alembic upgrade head

# Ver estado actual
alembic current

# Ver historial
alembic history --verbose

# Revertir una migracion
alembic downgrade -1

# Revertir todo
alembic downgrade base

# Ver SQL que generaria (sin ejecutar)
alembic upgrade head --sql
```

## 6. Ejemplo de migracion generada

```python
# alembic/versions/001_init.py

"""init

Revision ID: abc123
Revises:
Create Date: 2025-01-01 00:00:00.000000
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import UUID, JSONB

revision: str = "abc123"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.create_table(
        "items",
        sa.Column("id", UUID(as_uuid=True), nullable=False),
        sa.Column("name", sa.String(255), nullable=False),
        sa.Column("metadata", JSONB(), nullable=True),
        sa.Column("created_at", sa.DateTime(), server_default=sa.text("now()"), nullable=False),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_items_name", "items", ["name"])


def downgrade() -> None:
    op.drop_index("ix_items_name", table_name="items")
    op.drop_table("items")
```

## 7. Ejecutar Alembic en produccion (CI/CD)

```bash
# En Dockerfile o script de deploy:
alembic upgrade head && uvicorn src.main:app --host 0.0.0.0 --port 8000
```

## Notas Importantes

- Siempre revisar la migracion autogenerada antes de aplicar (puede no detectar cambios de nombre)
- No editar migraciones ya aplicadas en produccion
- El `env.py` debe importar TODOS los modelos ORM para que `--autogenerate` funcione
- En FastAPI, NO usar `create_all()` en produccion, solo `alembic upgrade head`
