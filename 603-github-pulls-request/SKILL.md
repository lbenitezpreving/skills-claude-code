---
name: 603-github-pulls
description: This skill should be used when the user asks to "create a pull request", "merge a PR", "list pull requests", "crear pull request", "mergear PR", "listar PRs", "aprobar PR", "revisar PR", or wants to manage GitHub Pull Requests using the gh CLI.
version: 1.0.0
---

# 603 — GitHub Pull Requests Manager

Gestiona Pull Requests de GitHub directamente desde Claude Code usando el CLI `gh`.

## Argumentos:
- `crear "título" [--body "desc"] [--reviewer usuario] [--draft] [--base rama]`
- `listar [--state open|closed|merged] [--author usuario]`
- `ver <número> [--comments]`
- `merge <número> [--squash|--rebase|--merge] [--delete-branch]`
- `aprobar <número> [--body "comentario"]`
- `cambios <número> --body "comentario requerido"`

## Proceso:

### 1. Verificar prerequisitos
- Ejecuta `gh auth status` para confirmar autenticación con GitHub
- Si no está autenticado, indica al usuario que ejecute `gh auth login` y termina
- Detecta repositorio y rama actual con `git branch --show-current`

### 2. Parsear la acción solicitada
Analiza `$ARGUMENTS` para identificar la acción:
- Primera palabra: `crear`, `listar`, `ver`, `merge`, `aprobar`, o `cambios`
- Extrae número de PR y opciones adicionales según la acción

### 3. Ejecutar la acción

#### Acción: `crear`
```bash
gh pr create --title "título" --body "descripción" [--reviewer usuario] [--draft] [--base main]
```
- Verifica que haya commits en la rama actual no presentes en la base
- Si no se especifica `--base`, usa la rama principal del repo
- Muestra URL del PR creado:
```
✅ Pull Request creado exitosamente
   Número: #15
   Título: título del PR
   Estado: Open (o Draft)
   URL: https://github.com/owner/repo/pull/15
```

#### Acción: `listar`
```bash
gh pr list [--state open] [--author usuario] --limit 20
```
- Por defecto muestra PRs abiertos
- Formato tabular: número, título, rama, autor, checks, fecha
- Incluye estado de CI checks si está disponible

#### Acción: `ver`
```bash
gh pr view <número> [--comments]
gh pr checks <número>
```
- Muestra título, estado, rama origen/destino, reviewers, descripción
- Incluye estado de checks de CI/CD
- Si se pasa `--comments`, incluye los comentarios de revisión

#### Acción: `merge`
```bash
gh pr merge <número> [--squash|--rebase|--merge] [--delete-branch]
```
- Por defecto usa `--squash` si no se especifica método
- Métodos disponibles:
  - `--merge`: Merge commit (preserva historial)
  - `--squash`: Squash and merge (un solo commit limpio)
  - `--rebase`: Rebase and merge (historial lineal)
- `--delete-branch` elimina la rama origen tras el merge
- Verifica que los checks pasen antes de mergear
- Formato de confirmación:
```
✅ PR #15 mergeado exitosamente
   Método: squash
   Rama eliminada: feature/mi-rama
```

#### Acción: `aprobar`
```bash
gh pr review <número> --approve [--body "comentario opcional"]
```
- Aprueba el PR con comentario opcional
- Muestra confirmación del review enviado

#### Acción: `cambios`
```bash
gh pr review <número> --request-changes --body "descripción de los cambios requeridos"
```
- `--body` es **obligatorio** para solicitar cambios (debe explicar qué corregir)
- Si no se proporciona `--body`, solicitar al usuario la descripción antes de continuar

### 4. Manejo de errores
- Si el PR no existe, mostrar mensaje claro con los PRs abiertos disponibles
- Si los checks no pasan al mergear, advertir al usuario y pedir confirmación
- Si no hay cambios en la rama para crear PR, indicar que no hay commits nuevos
- Si hay conflictos, indicar al usuario que debe resolverlos primero

## Ejemplos de uso:
```
/603-github-pulls crear "Fix bug en login"
/603-github-pulls crear "Nueva feature dark mode" --draft --reviewer compañero
/603-github-pulls listar
/603-github-pulls listar --state merged --author @me
/603-github-pulls ver 15
/603-github-pulls ver 15 --comments
/603-github-pulls merge 15 --squash --delete-branch
/603-github-pulls merge 15 --rebase
/603-github-pulls aprobar 15
/603-github-pulls aprobar 15 --body "LGTM! Gran trabajo"
/603-github-pulls cambios 15 --body "Por favor añade tests para el nuevo endpoint"
```

## Notas importantes:
- Requiere `gh` CLI instalado y autenticado (`gh auth login`)
- Para crear un PR debes estar en una rama diferente a la principal
- Los reviewers deben tener acceso al repositorio
- `--squash` es el método recomendado para mantener historial limpio
- Verifica siempre los checks antes de mergear en producción
