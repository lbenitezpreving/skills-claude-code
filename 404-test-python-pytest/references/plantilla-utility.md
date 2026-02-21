# Plantilla: Test de Funciones Utilitarias

Para funciones puras sin efectos secundarios ni dependencias de BD o HTTP.

## Estructura base

```python
"""Tests para funciones utilitarias."""
import pytest
from src.api.utils import (
    format_date,
    slugify,
    calculate_progress,
    validate_color,
    group_by_status,
)


# ─── format_date ──────────────────────────────────────────────────────────────

class TestFormatDate:
    """Tests para format_date."""

    def test_formatea_fecha_iso_correctamente(self):
        """Formatea fecha ISO a formato legible."""
        result = format_date("2024-01-15T10:30:00Z")
        assert result == "15/01/2024"

    def test_retorna_cadena_vacia_para_none(self):
        """Devuelve '' para None."""
        result = format_date(None)
        assert result == ""

    def test_retorna_cadena_vacia_para_fecha_invalida(self):
        """Devuelve '' para string que no es fecha."""
        result = format_date("not-a-date")
        assert result == ""


# ─── slugify ──────────────────────────────────────────────────────────────────

class TestSlugify:
    """Tests para slugify."""

    def test_convierte_a_minusculas(self):
        result = slugify("MAYÚSCULAS")
        assert result == "mayusculas"

    def test_reemplaza_espacios_con_guiones(self):
        result = slugify("Hola Mundo")
        assert result == "hola-mundo"

    def test_elimina_acentos(self):
        result = slugify("Título Con Acentos")
        assert result == "titulo-con-acentos"

    def test_trim_espacios_extremos(self):
        result = slugify("  espacios extra  ")
        assert result == "espacios-extra"

    def test_colapsa_multiples_espacios(self):
        result = slugify("múltiples   espacios")
        assert result == "multiples-espacios"


# ─── calculate_progress ───────────────────────────────────────────────────────

class TestCalculateProgress:
    """Tests para calculate_progress."""

    def test_retorna_cero_sin_completadas(self):
        assert calculate_progress(completed=0, total=5) == 0

    def test_retorna_cien_todas_completadas(self):
        assert calculate_progress(completed=5, total=5) == 100

    def test_calcula_porcentaje_correctamente(self):
        assert calculate_progress(completed=2, total=4) == 50
        assert calculate_progress(completed=1, total=3) == 33

    def test_retorna_cero_sin_total(self):
        """Sin tareas, progreso es 0 (sin división por cero)."""
        assert calculate_progress(completed=0, total=0) == 0


# ─── validate_color ───────────────────────────────────────────────────────────

class TestValidateColor:
    """Tests para validate_color."""

    def test_acepta_hex_valido(self):
        assert validate_color("#3b82f6") is True
        assert validate_color("#fff") is True

    def test_rechaza_sin_hash(self):
        assert validate_color("3b82f6") is False

    def test_rechaza_longitud_invalida(self):
        assert validate_color("#gggg") is False

    def test_rechaza_cadena_vacia(self):
        assert validate_color("") is False

    def test_rechaza_none(self):
        assert validate_color(None) is False


# ─── Tests parametrizados con @pytest.mark.parametrize ───────────────────────

class TestGroupByStatus:
    """Tests parametrizados para group_by_status."""

    @pytest.mark.parametrize("tasks,expected_counts", [
        (
            [{"status": "backlog"}, {"status": "doing"}, {"status": "done"}],
            {"backlog": 1, "doing": 1, "done": 1}
        ),
        (
            [{"status": "backlog"}, {"status": "backlog"}],
            {"backlog": 2, "doing": 0, "done": 0}
        ),
        (
            [],
            {"backlog": 0, "doing": 0, "done": 0}
        ),
    ])
    def test_agrupa_por_status(self, tasks, expected_counts):
        """Agrupa correctamente independientemente de la distribución."""
        result = group_by_status(tasks)
        for status, count in expected_counts.items():
            assert len(result[status]) == count  # exacto — mata mutantes


class TestValidateEmailParametrize:
    """Ejemplo de parametrize para validaciones."""

    @pytest.mark.parametrize("email,is_valid", [
        ("usuario@ejemplo.com", True),
        ("nombre.apellido@empresa.org", True),
        ("noesvalido", False),
        ("falta@", False),
        ("@sindominio.com", False),
        ("", False),
    ])
    def test_validacion_email(self, email, is_valid):
        from src.api.utils import validate_email
        assert validate_email(email) is is_valid
```
