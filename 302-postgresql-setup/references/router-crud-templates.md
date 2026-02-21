# Templates: Router CRUD con PostgreSQL

## Router Completo (UUID + JSONB)

```python
import uuid
from typing import Sequence

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from ..database import get_db
from ..models.item import Item
from ..schemas.item import ItemCreate, ItemRead, ItemUpdate

router = APIRouter(prefix="/items", tags=["items"])


@router.get("/", response_model=list[ItemRead])
async def list_items(
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_db),
) -> Sequence[Item]:
    result = await db.execute(select(Item).offset(skip).limit(limit))
    return result.scalars().all()


@router.get("/{item_id}", response_model=ItemRead)
async def get_item(item_id: uuid.UUID, db: AsyncSession = Depends(get_db)) -> Item:
    result = await db.execute(select(Item).where(Item.id == item_id))
    item = result.scalar_one_or_none()
    if not item:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Item no encontrado")
    return item


@router.post("/", response_model=ItemRead, status_code=status.HTTP_201_CREATED)
async def create_item(payload: ItemCreate, db: AsyncSession = Depends(get_db)) -> Item:
    item = Item(**payload.model_dump())
    db.add(item)
    try:
        await db.flush()
        await db.refresh(item)
    except IntegrityError:
        await db.rollback()
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Ya existe un item con ese nombre",
        )
    return item


@router.put("/{item_id}", response_model=ItemRead)
async def update_item(
    item_id: uuid.UUID,
    payload: ItemUpdate,
    db: AsyncSession = Depends(get_db),
) -> Item:
    result = await db.execute(select(Item).where(Item.id == item_id))
    item = result.scalar_one_or_none()
    if not item:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Item no encontrado")

    for field, value in payload.model_dump(exclude_unset=True).items():
        setattr(item, field, value)

    await db.flush()
    await db.refresh(item)
    return item


@router.delete("/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: uuid.UUID, db: AsyncSession = Depends(get_db)) -> None:
    result = await db.execute(select(Item).where(Item.id == item_id))
    item = result.scalar_one_or_none()
    if not item:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Item no encontrado")
    await db.delete(item)
```

## Schemas Pydantic v2 Asociados

```python
# src/api/schemas/item.py
import uuid
from datetime import datetime

from pydantic import BaseModel, ConfigDict


class ItemBase(BaseModel):
    name: str
    description: str | None = None
    is_active: bool = True
    metadata_: dict | None = None


class ItemCreate(ItemBase):
    pass


class ItemUpdate(BaseModel):
    name: str | None = None
    description: str | None = None
    is_active: bool | None = None
    metadata_: dict | None = None


class ItemRead(ItemBase):
    model_config = ConfigDict(from_attributes=True)

    id: uuid.UUID
    created_at: datetime
    updated_at: datetime
```

## Query con Filtros y Ordenacion

```python
from sqlalchemy import asc, desc, or_

@router.get("/", response_model=list[ItemRead])
async def list_items(
    search: str | None = None,
    is_active: bool | None = None,
    order_by: str = "created_at",
    order: str = "desc",
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_db),
):
    stmt = select(Item)

    if search:
        stmt = stmt.where(
            or_(Item.name.ilike(f"%{search}%"), Item.description.ilike(f"%{search}%"))
        )
    if is_active is not None:
        stmt = stmt.where(Item.is_active == is_active)

    sort_col = getattr(Item, order_by, Item.created_at)
    stmt = stmt.order_by(desc(sort_col) if order == "desc" else asc(sort_col))
    stmt = stmt.offset(skip).limit(limit)

    result = await db.execute(stmt)
    return result.scalars().all()
```

## Busqueda JSONB (PostgreSQL nativo)

```python
from sqlalchemy import cast
from sqlalchemy.dialects.postgresql import JSONB

# Buscar items donde metadata->>'status' == 'active'
stmt = select(Item).where(
    Item.metadata_["status"].as_string() == "active"
)

# Buscar items donde metadata contiene clave 'priority'
stmt = select(Item).where(Item.metadata_.has_key("priority"))
```
