# Formato de SKILL.md

Template completo para crear una nueva skill:

```markdown
---
name: NNN-nombre-skill
description: This skill should be used when the user asks to "[frase trigger]", mentions "[keyword]", or wants to [contexto de uso]. [Descripcion de lo que hace].
version: 1.0.0
---

# NNN — Titulo Legible

[Parrafo introductorio breve]

## Cuando aplica esta skill

- [condicion 1]
- [condicion 2]

## Proceso

### 1. [Primer paso]
[detalle]

### 2. [Segundo paso]
[detalle]

## Output esperado

[Que produce, crea o valida la skill]

## Notas
- [Restricciones, dependencias, convenciones]
```

> **Clave**: el campo `description` del frontmatter es lo que Claude usa para decidir cuando activar la skill automaticamente. Debe incluir frases literales entre comillas que el usuario podria decir.

> **Patron lazy loading**: Para skills grandes con muchos templates, extrae el codigo a `references/` y usa instrucciones como "Lee `references/X.md`" en cada paso que lo necesite. Esto reduce tokens cargados por defecto.
