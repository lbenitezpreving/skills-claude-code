---
name: 602-github-issues
description: This skill should be used when the user asks to "create an issue", "list issues", "view issue", "crear issue en GitHub", "listar issues", "ver issues", "gestionar issues de GitHub", or wants to manage GitHub Issues using the gh CLI.
version: 1.0.0
---

# 602 — GitHub Issues Manager

Gestiona issues de GitHub directamente desde Claude Code usando el CLI `gh`.

## Argumentos:
- `crear "título" [--label etiqueta] [--assignee usuario] [--body "descripción"]`
- `listar [--state open|closed|all] [--label etiqueta] [--assignee usuario]`
- `ver <número> [--comments]`

## Proceso:

### 1. Verificar prerequisitos
- Ejecuta `gh auth status` para confirmar autenticación con GitHub
- Si no está autenticado, indica al usuario que ejecute `gh auth login` y termina
- Detecta el repositorio actual con `gh repo view --json nameWithOwner`

### 2. Parsear la acción solicitada
Analiza `$ARGUMENTS` para identificar la acción:
- Primera palabra: `crear`, `listar`, o `ver`
- Extrae opciones adicionales según la acción

### 3. Ejecutar la acción

#### Acción: `crear`
```bash
gh issue create --title "título" --body "descripción" [--label etiqueta] [--assignee usuario]
```
- Si no se proporciona `--body`, usa `--body ""` o pide al usuario una descripción breve
- Muestra la URL del issue creado al finalizar
- Formato de salida:
```
✅ Issue creado exitosamente
   Número: #42
   Título: título del issue
   URL: https://github.com/owner/repo/issues/42
```

#### Acción: `listar`
```bash
gh issue list [--state open] [--label etiqueta] [--assignee usuario] --limit 20
```
- Por defecto muestra issues abiertos (`--state open`)
- Formato tabular: número, título, labels, asignado, fecha
- Si no hay issues, indica "No se encontraron issues con los filtros aplicados"

#### Acción: `ver`
```bash
gh issue view <número> [--comments]
```
- Muestra título, estado, labels, asignados, cuerpo del issue
- Si se pasa `--comments`, incluye los comentarios
- Formato legible con secciones claras

### 4. Manejo de errores
- Si el repositorio no tiene issues habilitados, informar al usuario
- Si el número de issue no existe, mostrar mensaje claro
- Si hay error de permisos, sugerir verificar acceso al repositorio

## Ejemplos de uso:
```
/602-github-issues crear "Bug en el formulario de login" --label bug
/602-github-issues crear "Añadir dark mode" --label enhancement --assignee miusuario
/602-github-issues listar
/602-github-issues listar --state closed --label bug
/602-github-issues listar --assignee @me
/602-github-issues ver 42
/602-github-issues ver 42 --comments
```

## Notas importantes:
- Requiere `gh` CLI instalado y autenticado (`gh auth login`)
- Los labels deben existir en el repositorio
- `--assignee @me` asigna al usuario autenticado
- Para issues con cuerpo largo, considerar usar un archivo con `--body-file`
