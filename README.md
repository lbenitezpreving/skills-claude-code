# skills-claude-code

Skills personalizadas de Claude Code, organizadas por categoría numérica. Se invocan con `/NNN-nombre-skill [argumentos]` o se activan automáticamente según el contexto.

---

## 0xx — Documentación y especificación

### `000-skill-creator`
Crea nuevas skills siguiendo la nomenclatura NNN. Determina el código correcto según la taxonomía, genera la carpeta `NNN-nombre/SKILL.md` y la registra en `CLAUDE.md` si es una skill de proyecto.

### `002-document-api`
Lee todos los routers de `src/api/routes/` y genera o actualiza `docs/API.md` con la lista completa de endpoints, parámetros, ejemplos de request/response y códigos de error.

### `003-document-component`
Documenta un componente React leyendo su interfaz de props y añade una entrada en `docs/COMPONENTS.md` con descripción, tabla de props y ejemplo de uso.

---

## 1xx — Frontend

### `101-react-component`
Scaffold completo de un componente React. Genera `ComponentName.tsx`, `ComponentName.module.css`, `ComponentName.test.tsx` e `index.ts` bajo `src/components/ComponentName/`.

---

## 2xx — Backend

### `201-python-fastapi-endpoint`
Genera un endpoint FastAPI completo: router con CRUD paginado, schemas Pydantic v2 (Base/Create/Update/Response con ejemplos), excepciones personalizadas (`ResourceNotFoundException`) y tests pytest con cobertura objetivo >85%.

---

## 3xx — Base de datos

### `301-sqlite-setup`
Configura persistencia SQLite con SQLAlchemy 2.0 async: `database.py` con `AsyncEngine` + `async_sessionmaker`, modelo ORM con Mapped types, router con queries SQL y tests con BD en memoria (`:memory:`).

### `302-postgresql-setup`
Igual que 301 pero con PostgreSQL: driver `asyncpg`, connection pool, migraciones con Alembic, soporte a tipos nativos (JSONB, UUID, ARRAY), variables de entorno en `.env` y BD de test separada.

---

## 4xx — Revisión de código y tests

### `402-test-react-vitest`
Genera tests para componentes React, hooks, contexts y utilidades con Vitest + Testing Library. Prioriza queries por rol semántico (`getByRole`), usa `userEvent` sobre `fireEvent` y verifica callbacks con valores exactos. Objetivo: ≥80% cobertura y ≥80% mutation score (Stryker).

### `403-test-spring-boot`
Genera tests unitarios para Spring Boot con JUnit 5 + Mockito + AssertJ. Nunca usa `@SpringBootTest`; usa `@ExtendWith(MockitoExtension.class)` para services y `standaloneSetup`/`@WebMvcTest` para controllers. Genera informe JaCoCo y Pitest. Objetivo: ≥80% en ambas métricas.

### `404-test-python-pytest`
Genera tests para endpoints FastAPI y servicios Python con pytest + httpx `AsyncClient`. BD siempre en memoria (`sqlite+aiosqlite:///:memory:`), fixtures async, `dependency_overrides` para inyectar BD de test y patrón AAA. Objetivo: ≥80% cobertura y ≥80% mutation score (mutmut).

### `501-clean-code-review`
Analiza el código fuente del proyecto en busca de code smells: nombres genéricos, funciones largas (>40 líneas) o con muchos parámetros (>4), duplicación DRY, complejidad ciclomática alta (>4 niveles de anidación) y patrones específicos de Python/FastAPI y React/TypeScript. Produce un informe con hallazgos críticos, mejoras sugeridas y buenas prácticas detectadas.

---

## 5xx — Contratos y colecciones API

### `502-postman-collection`
Lee los routers de `src/api/routes/` y los schemas Pydantic, y genera `docs/postman/collection.json` (Postman v2.1) y `docs/postman/environment.json` con variables `{{base_url}}` e IDs dinámicos. Agrupa los endpoints por recurso e incluye tests post-response.

---

## 6xx — Gestión con Git

### `601-commit`
Analiza los cambios con `git diff`, sugiere un mensaje semántico (`feat/fix/refactor/...`), confirma con el usuario, hace `git add` de archivos específicos, commit con co-autoría de Claude y push opcional. Soporta `--force` para omitir confirmaciones.

### `602-github-issues`
Gestiona issues de GitHub vía `gh` CLI. Acciones disponibles: `crear`, `listar` (con filtros de estado/label/asignado) y `ver` (con comentarios opcionales).

### `603-github-pulls-request`
Gestiona Pull Requests vía `gh` CLI. Acciones: `crear` (con draft/reviewer/base), `listar`, `ver`, `merge` (squash/rebase/merge), `aprobar` y `cambios` (request changes con comentario obligatorio).
