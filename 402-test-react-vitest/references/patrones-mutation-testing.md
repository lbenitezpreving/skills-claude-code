# Patrones para Mutation Testing (Stryker)

La cobertura de líneas (Vitest) mide si el código se ejecuta. La cobertura de mutaciones (Stryker) mide si los tests **detectan cambios** en el código. Un test débil puede ejecutar código sin verificar su resultado — Stryker lo descubre.

## Configuración de Stryker en stryker.config.mjs

```js
// @ts-check
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
const config = {
  testRunner: 'vitest',
  reporters: ['html', 'clear-text', 'progress'],
  coverageAnalysis: 'perTest',
  vitest: {
    configFile: 'vite.config.ts',
  },
  mutate: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/*.spec.{ts,tsx}',
    '!src/test/**',
    '!src/main.tsx',
    '!src/vite-env.d.ts',
  ],
  thresholds: {
    high: 80,
    low: 60,
    break: 50,          // falla el proceso si score < 50%
  },
};

export default config;
```

Ejecutar:
```bash
npx stryker run
# Reporte en: reports/mutation/html/index.html
```

---

## Técnica 1: `toHaveLength(N)` exacto en lugar de `toBeInTheDocument()`

Stryker genera mutantes que devuelven arrays vacíos. `toBeInTheDocument()` en uno solo no detecta si la lista tiene 0 o N elementos; `toHaveLength(N)` sí.

```tsx
// ❌ Débil — no mata el mutante "retorna array vacío"
const items = screen.getAllByRole('listitem');
expect(items[0]).toBeInTheDocument();

// ✅ Fuerte — mata el mutante
const items = screen.getAllByRole('listitem');
expect(items).toHaveLength(3);  // exacto
```

---

## Técnica 2: `toHaveBeenCalledWith` — verificar valores exactos

Stryker muta los argumentos pasados a las funciones. `toHaveBeenCalled()` no detecta si se pasaron los argumentos correctos; `toHaveBeenCalledWith(valorExacto)` sí.

```tsx
// ❌ Débil — no mata el mutante que cambia los argumentos
expect(onTaskEdit).toHaveBeenCalled();

// ✅ Fuerte — mata mutantes en los argumentos
expect(onTaskEdit).toHaveBeenCalledTimes(1);
expect(onTaskEdit).toHaveBeenCalledWith(mockTasks[0]);
// O con valores específicos:
expect(onTaskDelete).toHaveBeenCalledWith(1);  // ID exacto
```

---

## Técnica 3: Cubrir todas las ramas de condicionales

Stryker genera mutantes que eliminan condiciones o las invierten. Para cada `if (condición)` necesitas tests para `true` Y `false`.

```tsx
// Código de producción:
// if (loading) { return <Spinner />; }

// ❌ Incompleto — solo prueba loading=false
it('renderiza la lista', () => {
  render(<TaskList tasks={mockTasks} loading={false} />);
  expect(screen.getAllByRole('listitem')).toHaveLength(3);
});

// ✅ Completo — prueba AMBAS ramas
it('muestra spinner durante la carga', () => {
  render(<TaskList tasks={[]} loading={true} />);
  expect(screen.getByRole('status')).toBeInTheDocument();
  expect(screen.queryByRole('listitem')).not.toBeInTheDocument();
});

it('muestra la lista cuando no está cargando', () => {
  render(<TaskList tasks={mockTasks} loading={false} />);
  expect(screen.queryByRole('status')).not.toBeInTheDocument();
  expect(screen.getAllByRole('listitem')).toHaveLength(3);
});
```

---

## Técnica 4: `toEqual` exacto para objetos

Stryker muta literales. Verificar el objeto completo mata más mutantes.

```tsx
// ❌ Débil
expect(result.task).toBeTruthy();

// ✅ Fuerte
expect(result.task).toEqual({
  id: 1,
  name: 'Tarea',
  completed: false,
  status: 'backlog',
});
```

---

## Técnica 5: Verificar que callbacks NO se llaman cuando no deben

Stryker puede mover llamadas fuera de condicionales. Verificar que NO se llama en el caso negativo mata esos mutantes.

```tsx
// Ambas ramas del callback
it('llama a onDelete cuando se confirma', async () => {
  const user = userEvent.setup();
  const onDelete = vi.fn();
  render(<TaskCard task={mockTask} onDelete={onDelete} />);

  await user.click(screen.getByRole('button', { name: /eliminar/i }));
  await user.click(screen.getByRole('button', { name: /confirmar/i }));

  expect(onDelete).toHaveBeenCalledTimes(1);
  expect(onDelete).toHaveBeenCalledWith(mockTask.id);  // ID exacto
});

it('NO llama a onDelete cuando se cancela', async () => {
  const user = userEvent.setup();
  const onDelete = vi.fn();
  render(<TaskCard task={mockTask} onDelete={onDelete} />);

  await user.click(screen.getByRole('button', { name: /eliminar/i }));
  await user.click(screen.getByRole('button', { name: /cancelar/i }));

  expect(onDelete).not.toHaveBeenCalled();  // mata mutante "llamada incondicional"
});
```

---

## Indicadores de bajo Mutation Score y sus causas

| Síntoma en reporte Stryker | Causa probable | Solución |
|---------------------------|----------------|----------|
| Mutante "BlockStatement removed" sobrevive | Solo hay assertions de render, no de comportamiento | Añadir `toHaveBeenCalledWith(valorExacto)` |
| Mutante "StringLiteral" sobrevive | Se verifica con regex demasiado amplio | Usar regex más específico o texto exacto |
| Mutante "ConditionalExpression" sobrevive | Falta test del caso negativo del condicional | Añadir test con la condición en `false` |
| Mutante "ArrayDeclaration" sobrevive | Solo se verifica que el array existe | Añadir `toHaveLength(N)` exacto |
| Mutante "ArrowFunction" sobrevive | Callback verificado solo con `toHaveBeenCalled()` | Añadir `toHaveBeenCalledWith(args)` y `Times(N)` |
