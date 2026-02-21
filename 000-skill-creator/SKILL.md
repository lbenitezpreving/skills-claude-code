---
name: 000-skill-creator
description: This skill should be used when the user asks to "create a new skill", "add a skill", "crear una skill", "nueva skill", "añadir un comando", mentions creating a numbered NNN skill, wants to follow the personal skill numbering convention, or asks what skills exist or how to organize them.
version: 2.0.0
---

# 000 — Creador de Skills

Guia para crear nuevas skills siguiendo la nomenclatura numerica personal.
Funciona tanto a nivel global (`~/.claude/skills/`) como a nivel de proyecto (`.claude/skills/`).

## Taxonomia de Codigos

Lee `references/taxonomia.md` para ver la lista completa de categorias y codigos disponibles (0xx-7xx).

**Resumen de categorias:**
| Prefijo | Categoria |
|---------|-----------|
| `0xx` | Documentacion y especificacion |
| `1xx` | Frontend |
| `2xx` | Backend |
| `3xx` | Base de datos |
| `4xx` | Revision de codigo |
| `5xx` | Swagger / Postman / Contratos API |
| `6xx` | Gestion con Git |
| `7xx` | Despliegues |

## Estructura de cada skill

```
<raiz>/.claude/skills/
└── NNN-nombre-skill/
    ├── SKILL.md              <- archivo principal (obligatorio)
    └── references/           <- templates de codigo (opcional, para skills grandes)
        └── plantilla-X.md
```

Para el formato de SKILL.md, lee `references/formato-skill.md`.

## Proceso para crear una nueva skill

### 1. Parsear la peticion del usuario

Identifica:
- **Codigo NNN**: coherente con la taxonomia (lee `references/taxonomia.md`)
- **Nombre slug**: kebab-case descriptivo
- **Proposito**: que hara la skill, cuando se activara

### 2. Determinar el ambito

- **Global** (`~/.claude/skills/`): disponible en todos los proyectos
- **Proyecto** (`.claude/skills/`): solo en el proyecto activo
- Si el usuario no especifica y hay proyecto activo, preguntar

### 3. Verificar que no existe

Comprobar que no haya ya una carpeta `NNN-nombre/` en la ubicacion elegida.
Si existe, informar y detener.

### 4. Crear la carpeta y el SKILL.md

Crear:
```
<ubicacion>/NNN-nombre-skill/
└── SKILL.md
```

Redactar el `description` del frontmatter con frases trigger realistas. Lee `references/formato-skill.md` para el formato completo.

### 5. Registrar en CLAUDE.md (solo si es skill de proyecto)

Si existe `CLAUDE.md`, anadir entry en la seccion **Skills Disponibles**:
```markdown
- `NNN-nombre-skill`: Descripcion breve de cuando se activa
```

### 6. Confirmar al usuario

```
Skill creada: NNN-nombre-skill
   Ruta: <ruta>/NNN-nombre-skill/SKILL.md
   Categoria: [nombre de categoria]
   Ambito: [Global | Proyecto]
```

## Reglas de nomenclatura

1. El codigo NNN es unico y coherente con la taxonomia
2. Nombre slug en kebab-case (preferir espanol)
3. El `description` del frontmatter empieza por `This skill should be used when...`
4. Incluir siempre frases literales entre comillas en el `description`
5. Si el codigo exacto esta ocupado, usar el siguiente libre de la misma categoria
