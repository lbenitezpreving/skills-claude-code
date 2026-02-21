# Schemas Pydantic v2 Template

```python
"""Schemas para {resources}."""
from datetime import datetime, UTC
from typing import Optional
from pydantic import BaseModel, Field, ConfigDict


class {RESOURCE}Base(BaseModel):
    """Schema base para {resource}."""
    name: str = Field(min_length=1, max_length=100, description="Nombre del {resource}")
    description: Optional[str] = Field(default=None, max_length=500, description="Descripcion opcional del {resource}")


class {RESOURCE}Create({RESOURCE}Base):
    """Schema para crear un {resource}."""
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [{"name": "Ejemplo de {resource}", "description": "Descripcion detallada del {resource}"}]
        }
    )


class {RESOURCE}Update(BaseModel):
    """Schema para actualizar un {resource}."""
    name: Optional[str] = Field(default=None, min_length=1, max_length=100, description="Nombre del {resource}")
    description: Optional[str] = Field(default=None, max_length=500, description="Descripcion del {resource}")


class {RESOURCE}Response({RESOURCE}Base):
    """Schema de respuesta con campos adicionales."""
    model_config = ConfigDict(
        from_attributes=True,
        json_schema_extra={
            "examples": [{"id": 1, "name": "Ejemplo de {resource}", "description": "Descripcion detallada", "created_at": "2025-01-15T10:30:00Z"}]
        }
    )
    id: int = Field(description="ID unico del {resource}")
    created_at: datetime = Field(default_factory=lambda: datetime.now(UTC), description="Fecha de creacion")
```
