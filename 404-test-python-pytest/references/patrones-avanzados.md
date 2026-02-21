# Patrones avanzados de testing Python/pytest

## 1. monkeypatch — sustituir funciones y variables en tiempo de test

```python
import pytest
from src.api.utils import send_notification


class TestNotifications:
    """Tests con monkeypatch para sustituir dependencias."""

    @pytest.mark.asyncio
    async def test_create_task_envia_notificacion(
        self,
        async_client,
        monkeypatch
    ):
        """Verifica que se envía notificación al crear tarea."""
        notificaciones = []

        async def mock_send(message: str):
            notificaciones.append(message)

        monkeypatch.setattr("src.api.routes.tasks.send_notification", mock_send)

        response = await async_client.post("/tasks/", json={"name": "Test"})
        assert response.status_code == 201
        assert len(notificaciones) == 1  # exacto — mata mutantes
        assert "Test" in notificaciones[0]

    def test_config_desde_env(self, monkeypatch):
        """Verifica que la config se lee de variables de entorno."""
        monkeypatch.setenv("MAX_TASKS_PER_USER", "10")
        from src.api.config import get_max_tasks
        assert get_max_tasks() == 10  # exacto

    def test_config_default_sin_env(self, monkeypatch):
        """Verifica valor por defecto cuando env no está definida."""
        monkeypatch.delenv("MAX_TASKS_PER_USER", raising=False)
        from src.api.config import get_max_tasks
        assert get_max_tasks() == 100  # valor por defecto — mata mutante
```

## 2. caplog — verificar logs

```python
import pytest
import logging


class TestLogging:
    """Tests para verificar que el código genera los logs correctos."""

    @pytest.mark.asyncio
    async def test_error_queda_en_logs(self, async_client, caplog):
        """Verifica que errores 404 quedan registrados en el log."""
        with caplog.at_level(logging.WARNING, logger="src.api.routes"):
            response = await async_client.get("/tasks/999")

        assert response.status_code == 404
        assert "Task 999 not found" in caplog.text

    @pytest.mark.asyncio
    async def test_creacion_registrada(self, async_client, caplog):
        """Verifica log de info al crear una entidad."""
        with caplog.at_level(logging.INFO):
            response = await async_client.post("/tasks/", json={"name": "Test"})

        assert response.status_code == 201
        assert any("Task created" in record.message for record in caplog.records)
```

## 3. @pytest.mark.parametrize — múltiples casos con una sola definición

```python
import pytest
from httpx import AsyncClient


class TestTaskValidations:
    """Tests parametrizados para validaciones de entrada."""

    @pytest.mark.asyncio
    @pytest.mark.parametrize("invalid_data,expected_status", [
        ({"name": ""}, 422),           # nombre vacío
        ({}, 422),                      # sin nombre
        ({"name": "x" * 300}, 422),    # nombre muy largo
        ({"name": "ok", "status": "invalid"}, 422),  # status inválido
    ])
    async def test_validaciones_rechazan_datos_invalidos(
        self,
        async_client: AsyncClient,
        invalid_data: dict,
        expected_status: int
    ):
        """Verifica que todas las entradas inválidas devuelven el status correcto."""
        response = await async_client.post("/tasks/", json=invalid_data)
        assert response.status_code == expected_status

    @pytest.mark.asyncio
    @pytest.mark.parametrize("status,expected_completed", [
        ("backlog", False),
        ("doing", False),
        ("done", True),
    ])
    async def test_sincronizacion_status_completed(
        self,
        async_client: AsyncClient,
        status: str,
        expected_completed: bool
    ):
        """Verifica sincronización bidireccional para todos los estados."""
        create = await async_client.post("/tasks/", json={"name": "Test"})
        task_id = create.json()["id"]

        response = await async_client.put(f"/tasks/{task_id}", json={"status": status})
        assert response.status_code == 200
        assert response.json()["status"] == status
        assert response.json()["completed"] is expected_completed
```

## 4. freezegun — congelar tiempo en tests

```python
import pytest
from freezegun import freeze_time
from httpx import AsyncClient


class TestTimestamps:
    """Tests que verifican timestamps usando freezegun."""

    @pytest.mark.asyncio
    @freeze_time("2024-06-15 10:30:00")
    async def test_created_at_es_la_fecha_actual(self, async_client: AsyncClient):
        """created_at debe reflejar la fecha del momento de creación."""
        response = await async_client.post("/tasks/", json={"name": "Tarea con fecha"})

        assert response.status_code == 201
        assert response.json()["created_at"].startswith("2024-06-15")

    @pytest.mark.asyncio
    async def test_completed_at_registra_cuando_se_completa(self, async_client: AsyncClient):
        """completed_at se establece al completar y se borra al descompletar."""
        create = await async_client.post("/tasks/", json={"name": "Test"})
        task_id = create.json()["id"]
        assert create.json()["completed_at"] is None

        with freeze_time("2024-06-15 12:00:00"):
            response = await async_client.put(f"/tasks/{task_id}", json={"completed": True})
        assert response.json()["completed_at"].startswith("2024-06-15")

        # Al descompletar, completed_at vuelve a None
        response = await async_client.put(f"/tasks/{task_id}", json={"completed": False})
        assert response.json()["completed_at"] is None
```

## 5. Fixtures con scope — controlar ciclo de vida

```python
import pytest


# scope="session" — se crea una sola vez por sesión completa de pytest
@pytest.fixture(scope="session")
async def datos_base():
    """Datos base que no cambian entre tests (constantes de referencia)."""
    return {
        "proyectos_default": ["Trabajo", "Personal", "Estudios", "Hogar"],
        "status_validos": ["backlog", "doing", "done"],
    }

# scope="function" (default) — se crea en cada test
@pytest.fixture
async def task_nueva(async_client):
    """Tarea fresca para cada test."""
    response = await async_client.post("/tasks/", json={"name": "Tarea fresca"})
    return response.json()


# Uso combinado
class TestConFixturesEscaladas:
    @pytest.mark.asyncio
    async def test_status_son_validos(self, datos_base, task_nueva, async_client):
        task_id = task_nueva["id"]
        for status in datos_base["status_validos"]:
            response = await async_client.put(f"/tasks/{task_id}", json={"status": status})
            assert response.status_code == 200
            assert response.json()["status"] == status
```

## 6. tmpdir y archivos temporales

```python
import pytest
import json
from pathlib import Path


@pytest.mark.asyncio
async def test_exportar_json(async_client, tmp_path):
    """Verifica que la exportación genera un archivo JSON válido."""
    # Crear datos de prueba
    await async_client.post("/tasks/", json={"name": "Exportar"})

    # Solicitar exportación
    response = await async_client.get("/tasks/export")
    assert response.status_code == 200

    # Guardar y verificar el archivo
    export_file = tmp_path / "tasks.json"
    export_file.write_bytes(response.content)

    data = json.loads(export_file.read_text())
    assert len(data) == 1  # exacto — mata mutantes
    assert data[0]["name"] == "Exportar"
```
