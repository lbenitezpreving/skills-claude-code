---
name: 402-test-react-vitest
description: >
  Generates comprehensive tests for React/TypeScript components and hooks using Vitest and Testing Library.
  Follows Testing Library best practices, avoids implementation details, and generates coverage reports.
  Use when the user says "create tests", "write unit tests", "add tests for", "generate tests",
  "test coverage", or when reviewing untested React/TypeScript code.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
argument-hint: "[path/to/Component.tsx]"
---

# Testing React/TypeScript — Vitest + Testing Library

## Proceso (seguir en orden)

**Paso 1 — Analizar el archivo objetivo**
- Identifica el tipo: componente funcional, custom hook, Context/Provider, función utilitaria, servicio con fetch
- Lee las props, estados internos y callbacks del componente
- Detecta llamadas a API, efectos secundarios o contextos externos que deban mockearse
- Lista comportamientos: renders, interacciones de usuario, estados condicionales, edge cases
- Comprueba si ya existen tests para no duplicarlos

**Paso 2 — Verificar configuración Vitest**
Lee `references/vitest-config.md` para la config de `vite.config.ts`, `@vitest/coverage-v8` y Stryker.

**Paso 3 — Escribir los tests**
Selecciona la plantilla según el tipo de archivo:

| Tipo | Plantilla |
|------|-----------|
| Componente funcional React | `references/plantilla-component.md` |
| Custom Hook (`use*`) | `references/plantilla-hook.md` |
| Context / Provider | `references/plantilla-context.md` |
| Función utilitaria pura | `references/plantilla-utility.md` |
| Servicio con fetch/axios | `references/plantilla-api-mock.md` |
| `userEvent`, `waitFor`, timers, rerender | `references/patrones-avanzados.md` |
| Mutation testing (Stryker) | `references/patrones-mutation-testing.md` |

**Paso 4 — Ejecutar y verificar cobertura**
```bash
npx vitest run --coverage               # tests + reporte de cobertura
npx vitest run --coverage --reporter=html  # reporte HTML en coverage/
npx stryker run                         # cobertura de mutaciones con Stryker
```
Reportes:
- Vitest coverage: `coverage/index.html`
- Stryker:         `reports/mutation/html/index.html`

---

## Reglas de oro (NO NEGOCIABLES)

### ❌ NUNCA en tests de componentes
```tsx
// NUNCA testear detalles de implementación
component.state         // estado interno — no visible para el usuario
component.setState      // implementación — puede cambiar sin romper comportamiento

// NUNCA snapshots como única verificación
expect(container).toMatchSnapshot()  // oculta regresiones, no describe comportamiento

// NUNCA queries frágiles como primera opción
screen.getByTestId('mi-elemento')    // depende de atributo artificial, no del comportamiento
```

### ✅ SIEMPRE: jerarquía de queries
```tsx
// Orden de preferencia (de mejor a peor):
screen.getByRole('button', { name: /enviar/i })    // 1. Por rol semántico (ARIA)
screen.getByLabelText(/nombre/i)                   // 2. Por label (inputs)
screen.getByText(/texto visible/i)                 // 3. Por texto visible al usuario
screen.getByTestId('mi-elemento')                  // 4. Último recurso (evitar)
```

### ✅ SIEMPRE: `userEvent` sobre `fireEvent`
```tsx
import userEvent from '@testing-library/user-event';

// ✅ Correcto — simula comportamiento real del usuario (dispara todos los eventos)
const user = userEvent.setup();
await user.click(button);
await user.type(input, 'hola mundo');
await user.keyboard('{Enter}');

// ❌ Solo para drag-and-drop o eventos no soportados por userEvent
fireEvent.dragStart(card);
```

### ✅ SIEMPRE: verificar callbacks con exactitud
```tsx
const onSubmit = vi.fn();
await user.click(submitButton);
expect(onSubmit).toHaveBeenCalledTimes(1);
expect(onSubmit).toHaveBeenCalledWith({ name: 'Juan', email: 'juan@test.com' });
```

---

## Checklist de calidad

**Estructura general**
- [ ] Archivo de test colocado junto al componente (`ComponentName.test.tsx`)
- [ ] Usa `describe` para agrupar tests del mismo componente/hook
- [ ] Al menos un test por comportamiento observable (render, interacción, estado)
- [ ] Mock data tipada con la misma interface del componente
- [ ] No hay lógica de negocio en el test — solo setup y assertions

**Queries y assertions**
- [ ] Usa `getByRole` como primera opción
- [ ] No usa `getByTestId` salvo que sea imposible de otro modo
- [ ] Verifica texto exacto con `toHaveTextContent` o regex `/texto/i`
- [ ] Usa `queryByX` (no `getByX`) cuando verifica que algo NO existe
- [ ] Usa `findByX` (asíncrono) cuando el elemento aparece tras una operación async

**Interacciones**
- [ ] Usa `userEvent.setup()` + `await user.click/type/...` (no `fireEvent`)
- [ ] Tests asíncronos usan `await` + `waitFor` / `findByX`
- [ ] Callbacks verificados con `toHaveBeenCalledWith(valorExacto)`
- [ ] Callbacks verificados con `toHaveBeenCalledTimes(N)` exacto

**Cobertura y mutaciones**
- [ ] `@vitest/coverage-v8` muestra ≥ 80% de líneas cubiertas
- [ ] Se usa `toEqual(valorExacto)` en lugar de `toBeTruthy()` / `not.toBeNull()`
- [ ] Se usa `toHaveLength(N)` exacto en listas, no solo `toBeInTheDocument()`
- [ ] Hay tests para ambas ramas de cada condicional importante
- [ ] Stryker muestra ≥ 80% de mutation score

---

## Convención de nombres

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Archivo de test | `{Componente}.test.tsx` | `KanbanBoard.test.tsx` |
| Mock de datos | `mock{Tipo}` o `test{Tipo}` | `mockTasks`, `testUser` |
| Mock de función | `on{Acción}` o `mock{Fn}` | `onTaskEdit`, `mockFetch` |
| Describe principal | Nombre del componente | `describe('KanbanBoard', ...)` |
| Describe anidado | Comportamiento/estado | `describe('cuando está vacío', ...)` |
| It / test | Descripción observable | `it('muestra el mensaje de error cuando...', ...)` |
