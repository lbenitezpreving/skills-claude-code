# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repositorio

Colección de skills personalizadas para Claude Code. Cada skill es una carpeta numerada (`NNN-nombre-skill/`) con un `SKILL.md` principal y, opcionalmente, una carpeta `references/` con templates de código.

Las skills se instalan en:
- **Global** (`~/.claude/skills/`): disponibles en todos los proyectos
- **Proyecto** (`.claude/skills/`): solo en el proyecto activo

## Estructura de una skill

```
NNN-nombre-skill/
├── SKILL.md              # Archivo principal obligatorio
└── references/           # Templates de código (opcional, para skills grandes)
    └── plantilla-X.md
```

### Frontmatter obligatorio en SKILL.md

```markdown
---
name: NNN-nombre-skill
description: This skill should be used when the user asks to "...", "...", or wants to ...
version: 1.0.0
argument-hint: "[argumento]"   # opcional
allowed-tools: Read, Write, Edit, Bash, Grep, Glob   # opcional
---
```

El campo `description` es lo que Claude usa para activar la skill automáticamente. Debe incluir frases literales entre comillas que el usuario podría decir.

### Variable de argumentos

Usa `$ARGUMENTS` en el cuerpo del SKILL.md para referenciar lo que el usuario pasa al invocar la skill (p.ej. `/101-react-component MyButton`).

### Patrón lazy loading

Para skills grandes, extrae los templates a `references/*.md` y usa instrucciones como `Lee references/X.md` en cada paso. Esto reduce tokens cargados por defecto.

## Taxonomía de códigos NNN

| Prefijo | Categoría |
|---------|-----------|
| `0xx` | Documentación y especificación |
| `1xx` | Frontend |
| `2xx` | Backend |
| `3xx` | Base de datos |
| `4xx` | Revisión de código / tests |
| `5xx` | Swagger / Postman / Contratos API |
| `6xx` | Gestión con Git |
| `7xx` | Despliegues |

La lista completa de subcódigos está en `000-skill-creator/references/taxonomia.md`.

## Skills disponibles

| Skill | Descripción |
|-------|-------------|
| `000-skill-creator` | Crea nuevas skills siguiendo la nomenclatura NNN |
| `002-document-api` | Documenta endpoints FastAPI en `docs/API.md` |
| `003-document-component` | Documenta componentes React en `docs/COMPONENTS.md` |
| `101-react-component` | Scaffold componente React + CSS Modules + Vitest |
| `201-python-fastapi-endpoint` | Endpoint FastAPI completo con schemas Pydantic v2, paginación y tests |
| `301-sqlite-setup` | Persistencia SQLite con SQLAlchemy 2.0 async |
| `302-postgresql-setup` | Persistencia PostgreSQL con SQLAlchemy 2.0 async + Alembic |
| `402-test-react-vitest` | Tests React/TypeScript con Vitest y Testing Library |
| `403-test-spring-boot` | Tests Spring Boot con JUnit 5 y Mockito |
| `404-test-python-pytest` | Tests Python/FastAPI con pytest y httpx AsyncClient |
| `501-clean-code-review` | Revisión de calidad de código (naming, duplicación, complejidad) |
| `502-postman-collection` | Genera colección Postman desde endpoints FastAPI |
| `601-commit` | Commit semántico con co-autoría y push opcional |
| `602-github-issues` | Gestión de issues GitHub via `gh` CLI |
| `603-github-pulls-request` | Gestión de pull requests GitHub via `gh` CLI |

## Convenciones al crear o modificar skills

1. El código NNN es único dentro de su categoría; si está ocupado, usar el siguiente libre.
2. Nombre en kebab-case (preferir español).
3. El `description` empieza siempre por `This skill should be used when...`.
4. Al añadir una skill de proyecto, registrarla en la sección **Skills disponibles** de este CLAUDE.md.
5. Al modificar una skill existente, incrementar el campo `version` del frontmatter.
