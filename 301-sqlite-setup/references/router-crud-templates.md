# Templates: Router CRUD con SQLAlchemy

## GET all con paginacion

```python
@router.get("/", response_model=PaginatedResponse[{RESOURCE}Response])
async def get_all_{resources}(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=10, ge=1, le=100),
    db: AsyncSession = Depends(get_db)
):
    count_query = select(func.count()).select_from({RESOURCE})
    total_result = await db.execute(count_query)
    total = total_result.scalar_one()

    query = select({RESOURCE}).offset(skip).limit(limit)
    result = await db.execute(query)
    items = result.scalars().all()

    items_response = [{RESOURCE}Response.model_validate(item) for item in items]
    return PaginatedResponse(items=items_response, total=total, skip=skip, limit=limit)
```

## GET by ID

```python
@router.get("/{{{resource}_id}}", response_model={RESOURCE}Response)
async def get_{resource}({resource}_id: int, db: AsyncSession = Depends(get_db)):
    query = select({RESOURCE}).where({RESOURCE}.id == {resource}_id)
    result = await db.execute(query)
    item = result.scalar_one_or_none()

    if item is None:
        raise ResourceNotFoundException("{RESOURCE}", {resource}_id)

    return {RESOURCE}Response.model_validate(item)
```

## POST

```python
@router.post("/", response_model={RESOURCE}Response, status_code=201)
async def create_{resource}(data: {RESOURCE}Create, db: AsyncSession = Depends(get_db)):
    db_{resource} = {RESOURCE}(**data.model_dump())
    db.add(db_{resource})
    await db.flush()
    await db.refresh(db_{resource})
    return {RESOURCE}Response.model_validate(db_{resource})
```

## PUT

```python
@router.put("/{{{resource}_id}}", response_model={RESOURCE}Response)
async def update_{resource}({resource}_id: int, data: {RESOURCE}Update, db: AsyncSession = Depends(get_db)):
    query = select({RESOURCE}).where({RESOURCE}.id == {resource}_id)
    result = await db.execute(query)
    db_{resource} = result.scalar_one_or_none()

    if db_{resource} is None:
        raise ResourceNotFoundException("{RESOURCE}", {resource}_id)

    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(db_{resource}, field, value)

    db_{resource}.updated_at = datetime.now(UTC)
    await db.flush()
    await db.refresh(db_{resource})
    return {RESOURCE}Response.model_validate(db_{resource})
```

## DELETE

```python
@router.delete("/{{{resource}_id}}", status_code=204)
async def delete_{resource}({resource}_id: int, db: AsyncSession = Depends(get_db)):
    query = select({RESOURCE}).where({RESOURCE}.id == {resource}_id)
    result = await db.execute(query)
    db_{resource} = result.scalar_one_or_none()

    if db_{resource} is None:
        raise ResourceNotFoundException("{RESOURCE}", {resource}_id)

    await db.delete(db_{resource})
    return None
```
