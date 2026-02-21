---
name: 501-clean-code-review
description: This skill should be used when the user asks to "clean code review", "analyze code quality", "revisar código limpio", "buscar code smells", "analizar calidad del código", "detectar duplicación de código", or wants to find clean code issues like naming problems, long functions, duplication or high complexity.
version: 1.0.0
---

# 501 — Analizador de Código Limpio

Analiza el código del proyecto en busca de oportunidades de mejora según los principios de Clean Code. Al finalizar, muestra un resumen en consola con conteo por severidad y ejemplos de refactorización.

## Proceso de análisis

### 1. Explorar el proyecto
Usa Glob para localizar los archivos fuente del proyecto:
- Python: `**/*.py` (excluye `**/migrations/**`, `**/__pycache__/**`, `**/venv/**`, `**/.venv/**`)
- TypeScript/React: `**/*.ts`, `**/*.tsx` (excluye `**/node_modules/**`, `**/dist/**`, `**/*.d.ts`, `**/*.test.*`, `**/*.spec.*`)

Si el proyecto usa otro lenguaje, adapta los patrones. Excluye siempre archivos generados, dependencias externas y tests.

### 2. Leer y analizar cada archivo relevante
Para cada archivo encontrado, léelo y aplica los criterios de la sección siguiente. Acumula todos los hallazgos.

### 3. Evaluar según estos principios

#### A) Nombres descriptivos
**Crítico:**
- Variables de una sola letra fuera de contextos válidos (loops `i`, `j`, `k` son aceptables)
- Nombres genéricos: `data`, `info`, `temp`, `obj`, `val`, `res`, `result`, `aux`, `flag`, `thing`
- Funciones cuyo nombre no describe lo que hacen: `do_stuff`, `process`, `handle`, `manage`, `run`
- Clases o componentes con nombres vagos: `Manager`, `Handler`, `Processor`, `Helper`, `Utils` (sin contexto)
- Abreviaciones innecesarias que reducen legibilidad: `usr`, `btn`, `cnt`, `idx` (cuando el contexto no es inmediatamente obvio)

**Mejora:**
- Nombres que podrían ser más específicos para el dominio del negocio
- Booleanos sin prefijo `is_`, `has_`, `can_`, `should_`
- Funciones que retornan booleano pero no tienen nombre de predicado

#### B) Funciones pequeñas y con una responsabilidad
**Crítico:**
- Funciones/métodos de más de 40 líneas
- Funciones con más de 4 parámetros
- Funciones que hacen I/O Y también transforman datos Y también validan (múltiples responsabilidades claramente separables)
- Componentes React con más de 150 líneas (incluyendo JSX)

**Mejora:**
- Funciones de 25-40 líneas que podrían dividirse
- Funciones con comentarios internos que separan "secciones" (señal de que deberían ser funciones separadas)
- Funciones que devuelven datos Y tienen efectos secundarios (excepto cuando sea intencional)

#### C) DRY — No te repitas
**Crítico:**
- Bloques de código de 5+ líneas que aparecen duplicados (exactos o casi idénticos) en el mismo archivo o en archivos del mismo módulo
- Lógica de validación duplicada en múltiples lugares

**Mejora:**
- Constantes literales (strings o números) usadas en más de 2 lugares sin estar definidas como constante
- Patrones de código similares (no idénticos) que podrían unificarse con un parámetro

#### D) Complejidad ciclomática
**Crítico:**
- Funciones con más de 4 niveles de indentación anidados
- Funciones con más de 6 ramas lógicas (`if`/`elif`/`else`/`case`/`catch`)
- Condiciones booleanas compuestas con más de 3 operadores `and`/`or`/`&&`/`||`

**Mejora:**
- Funciones con 4-6 ramas lógicas que podrían simplificarse con early returns o tabla de despacho
- Condiciones negativas dobles (`if not not x`, `if (!(!flag))`)
- Switch/if-elif largo que podría reemplazarse con un diccionario/mapa

### 4. Reglas específicas por tecnología

#### Python / FastAPI
- Funciones síncronas en rutas FastAPI (deberían ser `async def`)
- `except Exception` sin especificar el tipo de excepción
- F-strings concatenadas donde una sola f-string bastaría
- Listas por comprensión vs. loops cuando una es claramente más legible
- Uso de `Optional[X]` en vez de `X | None` (Python 3.10+)

#### React / TypeScript
- Uso de `any` como tipo (siempre crítico)
- Props sin tipar o con `object` genérico
- `useEffect` sin array de dependencias o con dependencias faltantes
- Estado derivado que podría ser una variable calculada en render
- Componentes con múltiples `useState` que podrían agruparse en `useReducer`
- Inline styles en JSX (usar clases CSS en su lugar)

### 5. Qué NO marcar como problema
- Código en archivos de tests o specs
- Comentarios explicativos de lógica compleja (son bienvenidos)
- Parámetros `_unused` o variables prefijadas con `_` (convención intencional)
- `i`, `j`, `k` como índices en loops
- `e` como nombre de excepción o evento en handlers cortos
- Tipos genéricos de TypeScript (`T`, `K`, `V`)

---

## Formato de salida

Al finalizar el análisis, muestra el resultado en este formato:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ANÁLISIS DE CÓDIGO LIMPIO
  Archivos analizados: X  |  Líneas de código: ~X
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✗ CRÍTICOS (N)
──────────────
[Para cada hallazgo crítico:]
  archivo/ruta.py:línea — Descripción del problema

  Código actual:
    [fragmento relevante, máx. 8 líneas]

  Propuesta de mejora:
    [versión refactorizada]

⚠ MEJORAS SUGERIDAS (N)
────────────────────────
[Para cada mejora, igual formato pero más breve]

✓ BUENAS PRÁCTICAS DETECTADAS (N)
───────────────────────────────────
[Lista de 3-5 patrones correctos encontrados, para dar contexto positivo]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  RESUMEN: X críticos · X mejoras · X buenas prácticas
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Importante:**
- Los ejemplos de refactorización deben ser código real del proyecto, no genérico
- Si un archivo no tiene ningún problema, no lo menciones
- Limita el análisis a un máximo de 15 hallazgos totales (prioriza los más impactantes)
- Si el proyecto es grande (+50 archivos), analiza los que parezcan más complejos primero
- No menciones problemas de estilo puramente cosméticos (espacios, punto y coma) salvo que sean inconsistentes de forma notable
