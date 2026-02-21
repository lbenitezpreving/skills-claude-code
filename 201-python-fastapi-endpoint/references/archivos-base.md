# Archivos Base - Templates

## src/api/exceptions.py

```python
"""Excepciones personalizadas para la API."""
from fastapi import HTTPException, status


class ResourceNotFoundException(HTTPException):
    """Excepcion para recursos no encontrados."""
    def __init__(self, resource: str, resource_id: int):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"{resource} with id {resource_id} not found"
        )


class ResourceAlreadyExistsException(HTTPException):
    """Excepcion para recursos duplicados."""
    def __init__(self, resource: str, field: str, value: str):
        super().__init__(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"{resource} with {field}='{value}' already exists"
        )


class InvalidOperationException(HTTPException):
    """Excepcion para operaciones invalidas."""
    def __init__(self, message: str):
        super().__init__(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=message
        )
```

## src/api/pagination.py

```python
"""Utilidades para paginacion."""
from typing import Generic, List, TypeVar
from pydantic import BaseModel, Field, ConfigDict

T = TypeVar('T')


class PaginatedResponse(BaseModel, Generic[T]):
    """Respuesta paginada generica."""
    items: List[T] = Field(description="Items en la pagina actual")
    total: int = Field(description="Total de items en la coleccion")
    skip: int = Field(description="Offset aplicado")
    limit: int = Field(description="Limite por pagina")

    model_config = ConfigDict(
        json_schema_extra={
            "examples": [{
                "items": [{"id": 1, "name": "Item de ejemplo"}],
                "total": 50,
                "skip": 0,
                "limit": 10
            }]
        }
    )
```
