# Router Templates

## Imports y setup inicial

```python
from typing import Dict
from datetime import datetime, UTC
from fastapi import APIRouter, Query
from ..schemas.$ARGUMENTS import {RESOURCE}Response, {RESOURCE}Create, {RESOURCE}Update
from ..exceptions import ResourceNotFoundException
from ..pagination import PaginatedResponse

router = APIRouter(prefix="/{resources}", tags=["{resources}"])

_{resources}_db: Dict[int, dict] = {}
_next_id = 1
```

## GET all con paginacion

```python
@router.get(
    "/",
    response_model=PaginatedResponse[{RESOURCE}Response],
    summary="Obtener todos los {resources}",
    description="Retorna lista paginada de {resources}. Usa skip/limit para navegar.",
    response_description="Lista paginada de {resources}",
    responses={
        200: {
            "description": "{RESOURCES} recuperados exitosamente",
            "content": {
                "application/json": {
                    "example": {
                        "items": [{"id": 1, "name": "Ejemplo"}],
                        "total": 50, "skip": 0, "limit": 10
                    }
                }
            }
        }
    }
)
async def get_all_{resources}(
    skip: int = Query(default=0, ge=0, description="Numero de registros a saltar para paginacion"),
    limit: int = Query(default=10, ge=1, le=100, description="Maximo de registros a retornar (max: 100)")
):
    """
    Obtiene {resources} paginados.

    Args:
        skip: Offset para paginacion (default: 0)
        limit: Limite de resultados (default: 10, max: 100)

    Returns:
        PaginatedResponse: Respuesta paginada con items y metadata
    """
    all_items = list(_{resources}_db.values())
    total = len(all_items)
    items = all_items[skip:skip + limit]
    return PaginatedResponse(items=items, total=total, skip=skip, limit=limit)
```

## GET by ID

```python
@router.get(
    "/{{{resource}_id}}",
    response_model={RESOURCE}Response,
    summary="Obtener {resource} por ID",
    description="Obtiene un {resource} especifico por su identificador unico",
    response_description="{RESOURCE} solicitado",
    responses={
        200: {"description": "{RESOURCE} encontrado exitosamente"},
        404: {
            "description": "{RESOURCE} no encontrado",
            "content": {"application/json": {"example": {"detail": "{RESOURCE} with id 999 not found"}}}
        }
    }
)
async def get_{resource}({resource}_id: int):
    """
    Obtiene un {resource} por ID.

    Args:
        {resource}_id: ID unico del {resource}

    Returns:
        {RESOURCE}Response: {RESOURCE} solicitado

    Raises:
        ResourceNotFoundException: Si el {resource} no existe
    """
    if {resource}_id not in _{resources}_db:
        raise ResourceNotFoundException("{RESOURCE}", {resource}_id)
    return _{resources}_db[{resource}_id]
```

## POST

```python
@router.post(
    "/",
    response_model={RESOURCE}Response,
    status_code=201,
    summary="Crear nuevo {resource}",
    description="Crea un nuevo {resource} con los datos proporcionados",
    response_description="{RESOURCE} creado exitosamente",
    responses={
        201: {"description": "{RESOURCE} creado exitosamente"},
        422: {"description": "Error de validacion en los datos proporcionados"}
    }
)
async def create_{resource}({resource}: {RESOURCE}Create):
    """
    Crea un nuevo {resource}.

    Args:
        {resource}: Datos del {resource} a crear

    Returns:
        {RESOURCE}Response: {RESOURCE} creado con ID asignado
    """
    global _next_id
    {resource}_dict = {resource}.model_dump()
    {resource}_dict["id"] = _next_id
    {resource}_dict["created_at"] = datetime.now(UTC)
    _{resources}_db[_next_id] = {resource}_dict
    _next_id += 1
    return {resource}_dict
```
