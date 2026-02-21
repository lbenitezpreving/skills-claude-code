# Plantilla: Test de Lógica de Negocio (Service)

Para funciones de servicio que operan directamente sobre la BD (no vía HTTP).

## Estructura base

```python
"""Tests para la lógica de negocio de {entidad}."""
import pytest
from sqlalchemy.ext.asyncio import AsyncSession
from src.api.services.task_service import (
    create_task,
    get_task,
    get_all_tasks,
    update_task,
    delete_task,
)
from src.api.schemas.task import TaskCreate, TaskUpdate


class TestTaskService:
    """Tests para las funciones del servicio de tasks."""

    @pytest.mark.asyncio
    async def test_create_task_retorna_entidad(self, db_session: AsyncSession):
        """create_task devuelve la entidad creada con id asignado."""
        data = TaskCreate(name="Mi tarea", description="Descripción")
        result = await create_task(db_session, data)

        assert result.id is not None
        assert result.name == "Mi tarea"
        assert result.description == "Descripción"
        assert result.completed is False

    @pytest.mark.asyncio
    async def test_create_task_persiste_en_bd(self, db_session: AsyncSession):
        """La tarea creada es recuperable de la BD."""
        data = TaskCreate(name="Tarea persistida")
        created = await create_task(db_session, data)

        retrieved = await get_task(db_session, created.id)
        assert retrieved is not None
        assert retrieved.id == created.id
        assert retrieved.name == "Tarea persistida"

    @pytest.mark.asyncio
    async def test_get_task_inexistente_retorna_none(self, db_session: AsyncSession):
        """get_task devuelve None para ID que no existe."""
        result = await get_task(db_session, 999)
        assert result is None

    @pytest.mark.asyncio
    async def test_get_all_tasks_vacio(self, db_session: AsyncSession):
        """get_all_tasks devuelve lista vacía cuando no hay tareas."""
        result = await get_all_tasks(db_session)
        assert result == []

    @pytest.mark.asyncio
    async def test_get_all_tasks_retorna_todas(self, db_session: AsyncSession):
        """get_all_tasks devuelve exactamente las tareas creadas."""
        await create_task(db_session, TaskCreate(name="Tarea 1"))
        await create_task(db_session, TaskCreate(name="Tarea 2"))
        await create_task(db_session, TaskCreate(name="Tarea 3"))

        result = await get_all_tasks(db_session)
        assert len(result) == 3  # exacto — mata mutante "lista vacía"

    @pytest.mark.asyncio
    async def test_update_task_modifica_campos(self, db_session: AsyncSession):
        """update_task actualiza los campos correctamente."""
        created = await create_task(db_session, TaskCreate(name="Original"))

        update_data = TaskUpdate(name="Actualizado", completed=True)
        updated = await update_task(db_session, created.id, update_data)

        assert updated is not None
        assert updated.name == "Actualizado"
        assert updated.completed is True

    @pytest.mark.asyncio
    async def test_update_task_inexistente_retorna_none(self, db_session: AsyncSession):
        """update_task devuelve None para ID que no existe."""
        result = await update_task(db_session, 999, TaskUpdate(name="X"))
        assert result is None

    @pytest.mark.asyncio
    async def test_delete_task_elimina_correctamente(self, db_session: AsyncSession):
        """delete_task elimina la tarea y devuelve True."""
        created = await create_task(db_session, TaskCreate(name="A eliminar"))

        success = await delete_task(db_session, created.id)
        assert success is True

        # Verificar que ya no existe
        result = await get_task(db_session, created.id)
        assert result is None

    @pytest.mark.asyncio
    async def test_delete_task_inexistente_retorna_false(self, db_session: AsyncSession):
        """delete_task devuelve False para ID que no existe."""
        result = await delete_task(db_session, 999)
        assert result is False
```

## Variante: lógica de negocio con validaciones complejas

```python
class TestTaskStatusSync:
    """Tests para la sincronización bidireccional status ↔ completed."""

    @pytest.mark.asyncio
    async def test_status_done_sincroniza_completed_true(self, db_session: AsyncSession):
        """Al cambiar status a 'done', completed debe ser True."""
        created = await create_task(db_session, TaskCreate(name="Test"))
        assert created.completed is False  # estado inicial

        updated = await update_task(
            db_session, created.id, TaskUpdate(status="done")
        )
        assert updated.status == "done"
        assert updated.completed is True          # sincronizado
        assert updated.completed_at is not None   # timestamp registrado

    @pytest.mark.asyncio
    async def test_status_backlog_sincroniza_completed_false(self, db_session: AsyncSession):
        """Al cambiar status a 'backlog', completed debe ser False."""
        # Crear tarea ya completada
        created = await create_task(db_session, TaskCreate(name="Test"))
        await update_task(db_session, created.id, TaskUpdate(status="done"))

        # Volver a backlog
        updated = await update_task(
            db_session, created.id, TaskUpdate(status="backlog")
        )
        assert updated.status == "backlog"
        assert updated.completed is False
        assert updated.completed_at is None

    @pytest.mark.asyncio
    async def test_completed_true_sincroniza_status_done(self, db_session: AsyncSession):
        """Al poner completed=True, status debe ser 'done'."""
        created = await create_task(db_session, TaskCreate(name="Test"))

        updated = await update_task(
            db_session, created.id, TaskUpdate(completed=True)
        )
        assert updated.completed is True
        assert updated.status == "done"
```
