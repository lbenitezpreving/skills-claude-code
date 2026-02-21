# Template: Script de Migracion de Datos

## src/api/migrate_data.py

```python
"""Script para migrar datos iniciales a SQLite."""
import asyncio
from sqlalchemy import select
from src.api.database import async_session_maker, init_db
from src.api.models.project import Project


async def migrate_projects():
    """Migra proyectos iniciales."""
    initial_projects = [
        {"id": 1, "name": "Trabajo", "color": "#3498db"},
        {"id": 2, "name": "Personal", "color": "#2ecc71"},
        {"id": 3, "name": "Estudios", "color": "#9b59b6"},
        {"id": 4, "name": "Hogar", "color": "#e74c3c"},
    ]

    async with async_session_maker() as session:
        for proj_data in initial_projects:
            query = select(Project).where(Project.id == proj_data["id"])
            result = await session.execute(query)
            existing = result.scalar_one_or_none()

            if existing is None:
                project = Project(**proj_data)
                session.add(project)
                print(f"Proyecto creado: {proj_data['name']}")
            else:
                print(f"Proyecto ya existe: {proj_data['name']}")

        await session.commit()


async def main():
    """Ejecuta todas las migraciones."""
    print("Iniciando migracion de datos...")
    await init_db()
    print("Tablas creadas")
    await migrate_projects()
    print("Migracion completada")


if __name__ == "__main__":
    asyncio.run(main())
```

## Ejecutar migracion

```bash
python -m src.api.migrate_data
```
