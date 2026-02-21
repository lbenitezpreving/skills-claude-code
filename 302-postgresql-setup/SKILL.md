---
name: 302-postgresql-setup
description: This skill should be used when the user asks to "setup PostgreSQL", "configure PostgreSQL", "configurar PostgreSQL", "setup asyncpg", "migrar a PostgreSQL", "setup Alembic", "configurar Alembic", "añadir migraciones", "configurar base de datos PostgreSQL", or wants to set up PostgreSQL persistence with SQLAlchemy 2.0 async, asyncpg, Alembic migrations, ORM models and async tests.
version: 1.0.0
argument-hint: "[model_name]"
---

# 302 — Setup de PostgreSQL con SQLAlchemy 2.0 Async + Alembic

Configura persistencia con PostgreSQL para el modelo: `$ARGUMENTS`

## Dependencias Requeridas

```txt
sqlalchemy>=2.0.25
asyncpg>=0.29.0
alembic>=1.13.0
python-dotenv>=1.0.0
```

```bash
pip install sqlalchemy>=2.0.25 asyncpg>=0.29.0 alembic>=1.13.0 python-dotenv>=1.0.0
```

> Para tests: `pip install pytest-asyncio>=0.23.0 pytest>=7.4.0`
> Alternativa a asyncpg: `psycopg[binary,pool]>=3.1` (psycopg3)

## Templates de Referencia

| Paso | Template | Fichero |
|------|----------|---------|
| Variables de entorno | `.env` + `config.py` | `references/config-template.md` |
| Configurar BD | `database.py` completo + lifespan | `references/database-template.md` |
| Crear modelo ORM | Modelos con tipos PostgreSQL | `references/modelo-orm-templates.md` |
| Alembic: migraciones | `alembic.ini` + `env.py` + migraciones | `references/alembic-setup.md` |
| Router CRUD | GET/POST/PUT/DELETE async | `references/router-crud-templates.md` |
| Tests async | Fixtures con BD de test | `references/test-fixtures-async.md` |

## Paso 1: Variables de Entorno

Lee `references/config-template.md` para el template.

**Checklist:**
- [ ] `.env` con `DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/dbname`
- [ ] `.env.test` con BD de test separada (misma instancia, distinta BD)
- [ ] `config.py` con `Settings` via `pydantic-settings`
- [ ] `.env` en `.gitignore`

## Paso 2: Configurar database.py

Lee `references/database-template.md` para los templates.

**Checklist:**
- [ ] AsyncEngine con asyncpg
- [ ] Connection pool configurado (pool_size, max_overflow, pool_pre_ping)
- [ ] async_sessionmaker con expire_on_commit=False
- [ ] Dependency get_db() para FastAPI
- [ ] Base declarativa para modelos
- [ ] Lifespan en main.py (create_all solo en dev, Alembic en prod)

## Paso 3: Modelo ORM (src/api/models/$ARGUMENTS.py)

Lee `references/modelo-orm-templates.md` para los templates.

**Checklist:**
- [ ] Heredar de Base (desde database.py)
- [ ] Mapped types (SQLAlchemy 2.0 style)
- [ ] UUID como primary key (opcional, recomendado)
- [ ] Tipos PostgreSQL nativos si aplica (JSONB, ARRAY, UUID)
- [ ] Timestamps con timezone: created_at, updated_at
- [ ] __repr__ para debugging

## Paso 4: Alembic (Migraciones)

Lee `references/alembic-setup.md` para la configuracion completa.

**Checklist:**
- [ ] `alembic init alembic` ejecutado
- [ ] `alembic.ini`: `sqlalchemy.url` apunta a variable de entorno
- [ ] `alembic/env.py`: importa `Base` y usa `run_async_migrations()`
- [ ] Primera migracion: `alembic revision --autogenerate -m "init"`
- [ ] Aplicar: `alembic upgrade head`

## Paso 5: Router CRUD (src/api/routes/$ARGUMENTS.py)

Lee `references/router-crud-templates.md` para los templates.

**Checklist:**
- [ ] Imports: select, func, AsyncSession, Depends
- [ ] Dependency: db: AsyncSession = Depends(get_db)
- [ ] Manejo de IntegrityError para duplicados
- [ ] flush() + refresh() para obtener IDs/UUIDs sin commit completo

## Paso 6: Tests Async (tests/api/test_$ARGUMENTS.py)

Lee `references/test-fixtures-async.md` para los fixtures.

**Checklist:**
- [ ] BD de test en PostgreSQL separada (o SQLite en memoria para unit tests)
- [ ] Fixture que crea/destruye tablas por sesion de test
- [ ] Fixture async_client con override de get_db
- [ ] Tests marcados con @pytest.mark.asyncio
- [ ] asyncio_mode = "auto" en pyproject.toml

## Mejores Practicas

- `async with` para sessions (nunca dejar sesion abierta)
- `expire_on_commit=False` para evitar lazy loading post-commit
- `flush()` + `refresh()` para obtener IDs antes del commit
- `select()` en lugar de `Query` (SQLAlchemy 2.0)
- `scalar_one_or_none()` para un solo resultado
- `pool_pre_ping=True` para detectar conexiones muertas
- Nunca hardcodear credenciales: siempre desde `.env`
- Usar `UUID` como PK en lugar de enteros secuenciales
- JSONB para datos semiestructurados (mejor que JSON en PostgreSQL)
- `datetime.now(UTC)` con timezone en timestamps

## Checklist de Validacion Final

- [ ] Conexion: `psql $DATABASE_URL -c "SELECT 1"`
- [ ] Tablas: `alembic current` y `alembic history`
- [ ] Migracion aplicada: `alembic upgrade head`
- [ ] Tests: `pytest tests/api/test_$ARGUMENTS.py -v`
- [ ] Coverage: `pytest --cov=src/api --cov-report=term-missing`
- [ ] API: `uvicorn src.main:app --reload`

## Diferencias Clave vs SQLite (301)

| Aspecto | SQLite (301) | PostgreSQL (302) |
|---------|-------------|-----------------|
| Driver | aiosqlite | asyncpg |
| URL | `sqlite+aiosqlite:///./app.db` | `postgresql+asyncpg://user:pass@host/db` |
| Migraciones | create_all() directo | Alembic obligatorio |
| Tests | `:memory:` in-process | BD de test separada |
| PK | Integer autoincrement | UUID (recomendado) |
| JSON | JSON (texto) | JSONB (indexable) |
| Pool | N/A | pool_size + max_overflow |
| Tipos extra | - | ARRAY, JSONB, INET, CIDR, ENUM |
