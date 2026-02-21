# Plantilla: Test de Schema Pydantic

Para validar que los schemas Pydantic aceptan los datos correctos y rechazan los incorrectos.

## Estructura base

```python
"""Tests para los schemas Pydantic de {entidad}."""
import pytest
from pydantic import ValidationError
from src.api.schemas.task import (
    TaskCreate,
    TaskUpdate,
    TaskResponse,
    TaskStatus,
)


class TestTaskCreate:
    """Tests para el schema TaskCreate."""

    def test_crea_con_campo_requerido(self):
        """Acepta name como campo mínimo requerido."""
        schema = TaskCreate(name="Mi tarea")
        assert schema.name == "Mi tarea"

    def test_crea_con_todos_los_campos(self):
        """Acepta todos los campos opcionales."""
        schema = TaskCreate(
            name="Tarea completa",
            description="Una descripción",
            project_id=1,
            status="doing"
        )
        assert schema.name == "Tarea completa"
        assert schema.description == "Una descripción"
        assert schema.project_id == 1
        assert schema.status == "doing"

    def test_falla_sin_nombre(self):
        """Falla si no se proporciona name."""
        with pytest.raises(ValidationError) as exc_info:
            TaskCreate()
        assert "name" in str(exc_info.value)

    def test_falla_con_nombre_vacio(self):
        """Falla si name es cadena vacía."""
        with pytest.raises(ValidationError) as exc_info:
            TaskCreate(name="")
        assert "name" in str(exc_info.value)

    def test_falla_con_nombre_muy_largo(self):
        """Falla si name supera el límite de caracteres."""
        nombre_largo = "x" * 300  # asumiendo max_length=255
        with pytest.raises(ValidationError):
            TaskCreate(name=nombre_largo)

    def test_valores_por_defecto(self):
        """Verifica los valores por defecto de los campos opcionales."""
        schema = TaskCreate(name="Tarea")
        assert schema.description is None
        assert schema.project_id is None
        assert schema.status == "backlog"  # valor por defecto


class TestTaskUpdate:
    """Tests para el schema TaskUpdate (todos los campos opcionales)."""

    def test_acepta_sin_campos(self):
        """TaskUpdate es completamente opcional."""
        schema = TaskUpdate()
        assert schema.name is None
        assert schema.completed is None

    def test_acepta_solo_nombre(self):
        schema = TaskUpdate(name="Nuevo nombre")
        assert schema.name == "Nuevo nombre"

    def test_acepta_completed_true(self):
        schema = TaskUpdate(completed=True)
        assert schema.completed is True

    def test_acepta_completed_false(self):
        """Ambas ramas booleanas son válidas."""
        schema = TaskUpdate(completed=False)
        assert schema.completed is False

    def test_acepta_project_id_none(self):
        """project_id puede ser None explícito (desasignar proyecto)."""
        schema = TaskUpdate(project_id=None)
        assert schema.project_id is None

    def test_falla_con_nombre_vacio(self):
        """Si se proporciona name, no puede ser vacío."""
        with pytest.raises(ValidationError):
            TaskUpdate(name="")


class TestTaskStatus:
    """Tests para el enum TaskStatus."""

    @pytest.mark.parametrize("status", ["backlog", "doing", "done"])
    def test_acepta_valores_validos(self, status: str):
        """Acepta todos los valores del enum."""
        schema = TaskCreate(name="Test", status=status)
        assert schema.status == status

    def test_rechaza_status_invalido(self):
        """Rechaza valores fuera del enum."""
        with pytest.raises(ValidationError) as exc_info:
            TaskCreate(name="Test", status="invalid")
        assert "status" in str(exc_info.value)

    def test_rechaza_status_en_mayusculas(self):
        """El enum es case-sensitive."""
        with pytest.raises(ValidationError):
            TaskCreate(name="Test", status="BACKLOG")


class TestTaskResponse:
    """Tests para el schema de respuesta."""

    def test_serializa_correctamente(self):
        """Verifica que model_dump produce los campos esperados."""
        data = {
            "id": 1,
            "name": "Tarea",
            "description": None,
            "completed": False,
            "status": "backlog",
            "project_id": None,
            "created_at": "2024-01-01T10:00:00",
            "completed_at": None,
            "deleted_at": None,
        }
        schema = TaskResponse(**data)
        result = schema.model_dump()

        assert result["id"] == 1
        assert result["name"] == "Tarea"
        assert result["completed"] is False
        assert result["status"] == "backlog"

    def test_campo_id_requerido(self):
        """TaskResponse requiere id (no es opcional como en Create)."""
        with pytest.raises(ValidationError):
            TaskResponse(name="Sin id")  # falta id
```
