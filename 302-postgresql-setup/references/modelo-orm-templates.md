# Templates: Modelos ORM para PostgreSQL

## Modelo Basico (Integer PK)

```python
from datetime import UTC, datetime

from sqlalchemy import String, func
from sqlalchemy.orm import Mapped, mapped_column

from ..database import Base


class Item(Base):
    __tablename__ = "items"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    description: Mapped[str | None] = mapped_column(String(1000))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(UTC),
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(UTC),
        server_default=func.now(),
        onupdate=lambda: datetime.now(UTC),
    )

    def __repr__(self) -> str:
        return f"<Item id={self.id} name={self.name!r}>"
```

## Modelo con UUID (recomendado para PostgreSQL)

```python
import uuid
from datetime import UTC, datetime

from sqlalchemy import String, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from ..database import Base


class Item(Base):
    __tablename__ = "items"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
    )
    name: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    description: Mapped[str | None] = mapped_column(String(1000))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(UTC),
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(UTC),
        server_default=func.now(),
        onupdate=lambda: datetime.now(UTC),
    )

    def __repr__(self) -> str:
        return f"<Item id={self.id} name={self.name!r}>"
```

## Modelo con FK y JSONB

```python
import uuid
from datetime import UTC, datetime

from sqlalchemy import ForeignKey, String, func
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship

from ..database import Base


class Category(Base):
    __tablename__ = "categories"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    items: Mapped[list["Item"]] = relationship("Item", back_populates="category")


class Item(Base):
    __tablename__ = "items"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    metadata_: Mapped[dict | None] = mapped_column("metadata", JSONB)  # JSONB indexable
    category_id: Mapped[uuid.UUID | None] = mapped_column(
        ForeignKey("categories.id", ondelete="SET NULL"), nullable=True
    )
    category: Mapped["Category | None"] = relationship("Category", back_populates="items")
    created_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(UTC),
        server_default=func.now(),
    )

    def __repr__(self) -> str:
        return f"<Item id={self.id} name={self.name!r}>"
```

## Tipos PostgreSQL Nativos Utiles

```python
from sqlalchemy.dialects.postgresql import (
    ARRAY,      # arrays: Mapped[list[str]] = mapped_column(ARRAY(String))
    JSONB,      # JSON indexable: Mapped[dict] = mapped_column(JSONB)
    UUID,       # UUIDs nativos
    INET,       # IPs: Mapped[str] = mapped_column(INET)
    ENUM,       # Enums: Mapped[str] = mapped_column(ENUM("a","b", name="myenum"))
    TSVECTOR,   # Full-text search
)
```

## Enum con Python

```python
import enum
from sqlalchemy import Enum as SAEnum

class Status(str, enum.Enum):
    active = "active"
    inactive = "inactive"
    pending = "pending"

class Item(Base):
    __tablename__ = "items"
    # ...
    status: Mapped[Status] = mapped_column(
        SAEnum(Status, name="item_status"),
        default=Status.pending,
    )
```
