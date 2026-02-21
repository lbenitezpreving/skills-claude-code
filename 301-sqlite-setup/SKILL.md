---
name: 301-sqlite-setup
description: This skill should be used when the user asks to "setup SQLite", "configure database", "configurar SQLite", "configurar base de datos", "setup SQLAlchemy", "migrar a SQLite", or wants to set up SQLite persistence with SQLAlchemy 2.0 async, ORM models and in-memory async tests.
version: 2.0.0
argument-hint: "[model_name]"
---

# 301 — Setup de SQLite con SQLAlchemy 2.0 Async

Configura persistencia con SQLite para el modelo: `$ARGUMENTS`

## Dependencias Requeridas

```txt
sqlalchemy>=2.0.25
aiosqlite>=0.19.0
```

```bash
pip install sqlalchemy>=2.0.25 aiosqlite>=0.19.0
```

## Templates de Referencia

| Paso | Template | Fichero |
|------|----------|---------|
| Configurar BD | `database.py` completo + `main.py` lifespan | `references/database-template.md` |
| Crear modelo ORM | Modelo basico + Modelo con FK | `references/modelo-orm-templates.md` |
| Actualizar router | GET/POST/PUT/DELETE con SQLAlchemy | `references/router-crud-templates.md` |
| Migrar datos | Script migrate_data.py | `references/migrate-data-template.md` |
| Adaptar tests | Fixtures async + test de ejemplo | `references/test-fixtures-async.md` |

## Paso 1: Configurar database.py

Lee `references/database-template.md` para los templates.

**Checklist:**
- [ ] AsyncEngine con aiosqlite
- [ ] async_sessionmaker configurado
- [ ] Dependency get_db() para FastAPI
- [ ] init_db() para crear tablas
- [ ] Base declarativa para modelos
- [ ] Lifespan en main.py

## Paso 2: Modelo ORM (src/api/models/$ARGUMENTS.py)

Lee `references/modelo-orm-templates.md` para los templates.

**Checklist:**
- [ ] Heredar de Base (desde database.py)
- [ ] Mapped types (SQLAlchemy 2.0 style)
- [ ] __tablename__ en snake_case
- [ ] Primary key con mapped_column(primary_key=True)
- [ ] Foreign keys con ForeignKey() y relationship() si aplica
- [ ] Timestamps: created_at, updated_at
- [ ] __repr__ para debugging

## Paso 3: Router con SQLAlchemy (src/api/routes/$ARGUMENTS.py)

Lee `references/router-crud-templates.md` para los templates.

**Checklist:**
- [ ] Imports: select, func, AsyncSession, Depends
- [ ] Import del modelo: from ..models.$ARGUMENTS import {RESOURCE}
- [ ] Dependency: db: AsyncSession = Depends(get_db)
- [ ] Reemplazar diccionarios globales por queries SQL
- [ ] flush() + refresh() para obtener IDs sin commit completo

## Paso 4: Migracion de Datos (src/api/migrate_data.py)

Lee `references/migrate-data-template.md` para el script completo.

## Paso 5: Tests Async (tests/api/test_$ARGUMENTS.py)

Lee `references/test-fixtures-async.md` para los fixtures.

**Checklist:**
- [ ] SQLite en memoria (`:memory:`) para tests
- [ ] Fixture test_db crea/destruye tablas
- [ ] Fixture async_client con override de get_db
- [ ] Tests marcados con @pytest.mark.asyncio
- [ ] asyncio_mode = "auto" en pyproject.toml

## Mejores Practicas

- `async with` para sessions
- `expire_on_commit=False` para evitar lazy loading
- `flush()` + `refresh()` para obtener IDs
- `select()` en lugar de `Query` (SQLAlchemy 2.0)
- `scalar_one_or_none()` para un solo resultado
- `model_validate()` para ORM a Pydantic
- `datetime.now(UTC)` en lugar de `utcnow()`

## Checklist de Validacion Final

- [ ] BD creada: `ls -lh app.db`
- [ ] Tablas: `sqlite3 app.db ".schema"`
- [ ] Tests: `pytest tests/api/test_{resources}.py -v`
- [ ] Coverage: `pytest --cov=src/api --cov-report=term-missing`
- [ ] API: `uvicorn src.main:app --reload`

## Migracion a PostgreSQL (Futuro)

Cambiar `DATABASE_URL = "sqlite+aiosqlite:///./app.db"` por `"postgresql+asyncpg://user:pass@localhost/dbname"` e instalar `asyncpg`. Los modelos no requieren cambios.
