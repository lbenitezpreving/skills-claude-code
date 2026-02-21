# Templates: Modelos ORM

## Modelo Basico (src/api/models/$ARGUMENTS.py)

```python
"""Modelo ORM para {resource}."""
from datetime import datetime, UTC
from typing import Optional
from sqlalchemy import String, Integer, DateTime, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from ..database import Base


class {RESOURCE}(Base):
    """Modelo ORM para {resource}."""

    __tablename__ = "{resources}"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    description: Mapped[Optional[str]] = mapped_column(String(500), nullable=True)

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(UTC),
        nullable=False
    )
    updated_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True),
        onupdate=lambda: datetime.now(UTC),
        nullable=True
    )

    def __repr__(self) -> str:
        return f"<{RESOURCE}(id={self.id}, name='{self.name}')>"
```

## Modelo con Foreign Key

```python
class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)

    project_id: Mapped[Optional[int]] = mapped_column(
        Integer,
        ForeignKey("projects.id", ondelete="SET NULL"),
        nullable=True
    )

    project: Mapped[Optional["Project"]] = relationship(
        "Project",
        back_populates="tasks"
    )
```
