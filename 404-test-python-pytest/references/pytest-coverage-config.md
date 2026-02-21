# Configuración pytest + pytest-cov + mutmut

## pyproject.toml — configuración recomendada

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

[tool.coverage.run]
source = ["src"]
omit = [
    "src/main.py",           # entrypoint
    "src/api/migrate_data.py",  # script de migración
    "*/tests/*",
    "*/__pycache__/*",
]

[tool.coverage.report]
fail_under = 80
show_missing = true
exclude_lines = [
    "pragma: no cover",
    "if __name__ == .__main__.:",
    "raise NotImplementedError",
    "pass",
]

[tool.mutmut]
paths_to_mutate = "src/"
backup = false
runner = "pytest"
tests_dir = "tests/"
dict_synonyms = "Struct, NamedStruct"
```

## Instalar dependencias

```bash
# Testing async
pip install pytest pytest-asyncio httpx

# Cobertura
pip install pytest-cov

# Mutation testing
pip install mutmut
```

## requirements-dev.txt mínimo

```
pytest>=8.0
pytest-asyncio>=0.23
httpx>=0.27
aiosqlite>=0.20
pytest-cov>=5.0
mutmut>=2.4
```

## Comandos de ejecución

```bash
# Ejecutar todos los tests
pytest tests/ -v

# Con cobertura (terminal)
pytest tests/ --cov=src --cov-report=term-missing

# Con cobertura HTML
pytest tests/ --cov=src --cov-report=html
open htmlcov/index.html

# Falla si cobertura < 80%
pytest tests/ --cov=src --cov-fail-under=80

# Solo un archivo
pytest tests/api/test_tasks.py -v

# Solo una clase o test
pytest tests/api/test_tasks.py::TestTasksEndpoints::test_create_task -v

# Mutation testing (tarda varios minutos)
mutmut run
mutmut results           # ver resumen
mutmut show <id>         # ver mutante específico
mutmut html              # generar reporte HTML
```

## Estructura de directorios esperada

```
tests/
├── conftest.py           # fixtures compartidas (test_db, async_client)
├── api/
│   ├── test_tasks.py
│   ├── test_projects.py
│   └── test_subtasks.py
└── utils/
    └── test_helpers.py
```
