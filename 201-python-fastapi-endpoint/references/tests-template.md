# Tests Template

```python
"""Tests para endpoints de {resources}."""
import pytest
from fastapi.testclient import TestClient
from src.main import app
from src.api.routes.{resources} import _{resources}_db, _next_id


@pytest.fixture
def client():
    return TestClient(app)


@pytest.fixture(autouse=True)
def reset_db():
    _{resources}_db.clear()
    global _next_id
    _next_id = 1
    yield
    _{resources}_db.clear()


def test_pagination_basic(client, reset_db):
    for i in range(25):
        client.post("/{resources}/", json={"name": f"Item {i}"})
    response = client.get("/{resources}/?skip=0&limit=10")
    assert response.status_code == 200
    data = response.json()
    assert len(data["items"]) == 10
    assert data["total"] == 25
    assert data["skip"] == 0
    assert data["limit"] == 10

    response = client.get("/{resources}/?skip=10&limit=10")
    data = response.json()
    assert len(data["items"]) == 10
    assert data["skip"] == 10


def test_pagination_limits(client, reset_db):
    response = client.get("/{resources}/?limit=100")
    assert response.status_code == 200
    response = client.get("/{resources}/?limit=101")
    assert response.status_code == 422


def test_custom_exception_format(client, reset_db):
    response = client.get("/{resources}/999")
    assert response.status_code == 404
    assert "{RESOURCE} with id 999 not found" in response.json()["detail"]


def test_create_{resource}(client, reset_db):
    response = client.post("/{resources}/", json={"name": "Test {resource}", "description": "Test description"})
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test {resource}"
    assert "id" in data
    assert "created_at" in data


def test_validation_errors(client, reset_db):
    response = client.post("/{resources}/", json={"name": ""})
    assert response.status_code == 422
    response = client.post("/{resources}/", json={"name": "x" * 101})
    assert response.status_code == 422
```
