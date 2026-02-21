# Patrones para Mutation Testing (mutmut)

La cobertura de líneas (pytest-cov) mide si el código se ejecuta. La cobertura de mutaciones (mutmut) mide si los tests **detectan cambios** en el código. Un test débil puede ejecutar código sin verificar su resultado — mutmut lo descubre.

## Configuración de mutmut en pyproject.toml

```toml
[tool.mutmut]
paths_to_mutate = "src/"
backup = false
runner = "pytest"
tests_dir = "tests/"

# Excluir archivos sin lógica de negocio
# (añadir a .mutmut-config si se necesita granularidad)
```

Ejecutar:
```bash
# Mutation testing completo
mutmut run

# Ver resultados
mutmut results

# Ver mutante específico
mutmut show <id>

# Generar reporte HTML
mutmut html
# Reporte en: html/index.html

# Solo mutaciones en un archivo específico
mutmut run --paths-to-mutate src/api/routes/tasks.py
```

---

## Técnica 1: `len(result) == N` exacto en lugar de `assert result`

mutmut genera mutantes que devuelven listas vacías. `assert result` no detecta si la lista tiene 0 o N elementos; `assert len(result) == N` sí.

```python
# ❌ Débil — no mata el mutante "retorna lista vacía"
result = await get_all_tasks(db_session)
assert result

# ✅ Fuerte — mata el mutante
result = await get_all_tasks(db_session)
assert len(result) == 3  # exacto

# En tests HTTP:
data = response.json()
assert len(data) == 3  # no solo: assert data != []
```

---

## Técnica 2: comparaciones exactas `== valor`

mutmut muta literales y constantes. Verificar valores concretos mata más mutantes que verificar presencia.

```python
# ❌ Débil — no mata mutante que cambia el status_code
assert response.status_code < 300

# ✅ Fuerte — mata mutante
assert response.status_code == 201

# ❌ Débil
assert result["name"] is not None

# ✅ Fuerte
assert result["name"] == "Mi tarea"

# ❌ Débil
assert result["completed_at"] is not None

# ✅ Fuerte (cuando puedes)
assert result["completed_at"].startswith("2024-06")
# O al menos: assert result["completed_at"] is not None AND result["completed"] is True
```

---

## Técnica 3: Cubrir todas las ramas de condicionales

mutmut genera mutantes que eliminan condiciones o las invierten. Para cada `if condición` necesitas tests para `True` Y `False`.

```python
# Código de producción:
# if task.completed:
#     task.completed_at = datetime.now()
# else:
#     task.completed_at = None

# ❌ Incompleto — solo prueba una rama
async def test_completed_true_sets_timestamp(self, async_client):
    # ...
    assert result["completed_at"] is not None

# ✅ Completo — prueba AMBAS ramas
async def test_completed_true_sets_timestamp(self, async_client):
    response = await async_client.post("/tasks/", json={"name": "Test"})
    task_id = response.json()["id"]

    # Rama TRUE
    response = await async_client.put(f"/tasks/{task_id}", json={"completed": True})
    assert response.json()["completed"] is True
    assert response.json()["completed_at"] is not None  # timestamp presente

async def test_completed_false_clears_timestamp(self, async_client):
    # Crear y completar primero
    response = await async_client.post("/tasks/", json={"name": "Test"})
    task_id = response.json()["id"]
    await async_client.put(f"/tasks/{task_id}", json={"completed": True})

    # Rama FALSE
    response = await async_client.put(f"/tasks/{task_id}", json={"completed": False})
    assert response.json()["completed"] is False
    assert response.json()["completed_at"] is None  # timestamp eliminado
```

---

## Técnica 4: Verificar valores booleanos explícitamente

mutmut muta `True` ↔ `False`. Usar `is True` / `is False` en lugar de `is not None`.

```python
# ❌ Débil — no mata mutante que invierte booleano
assert result["completed"]

# ✅ Fuerte — explícito
assert result["completed"] is True
assert result["completed"] is False  # en el caso negativo

# ❌ Débil
assert not result["completed"]

# ✅ Fuerte
assert result["completed"] is False
```

---

## Técnica 5: Verificar efectos secundarios explícitos

mutmut puede eliminar líneas de efecto secundario. Verificar que el efecto ocurrió mata esos mutantes.

```python
# Código de producción:
# if status == "done":
#     task.completed = True
#     task.completed_at = now()

# ❌ Débil — solo verifica el status, no el efecto secundario
async def test_status_done(self, async_client):
    # ...
    assert result["status"] == "done"

# ✅ Fuerte — verifica también los efectos secundarios
async def test_status_done_sincroniza_campos(self, async_client):
    response = await async_client.post("/tasks/", json={"name": "Test"})
    task_id = response.json()["id"]

    response = await async_client.put(f"/tasks/{task_id}", json={"status": "done"})
    result = response.json()

    assert result["status"] == "done"            # campo directo
    assert result["completed"] is True           # efecto secundario 1
    assert result["completed_at"] is not None    # efecto secundario 2
```

---

## Indicadores de bajo Mutation Score y sus causas

| Síntoma en reporte mutmut | Causa probable | Solución |
|---------------------------|----------------|----------|
| Mutante "== replaced with !=" sobrevive | Se verifica solo la presencia, no el valor | Usar `== valorExacto` |
| Mutante "True replaced with False" sobrevive | `assert campo` en lugar de `assert campo is True` | Usar `is True` / `is False` explícito |
| Mutante "return []" sobrevive | Solo `assert result`, no longitud | Añadir `assert len(result) == N` |
| Mutante "if removed" sobrevive | Falta test del caso donde la condición es False | Añadir test para la rama False |
| Mutante "+ replaced with -" sobrevive | Operaciones matemáticas sin verificar valor exacto | Comparar con `== valorEsperado` |
| Mutante "None replaced with value" sobrevive | `assert campo is not None` sin verificar valor | Añadir `assert campo == valorEsperado` |
