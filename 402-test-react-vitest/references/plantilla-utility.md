# Plantilla: Test de Funciones Utilitarias

Para funciones puras sin efectos secundarios ni dependencias externas.

## Estructura base

```tsx
import { describe, it, expect } from 'vitest';
import {
  formatDate,
  truncateText,
  groupByStatus,
  calculateProgress,
} from './utils';

// ─── formatDate ───────────────────────────────────────────────────────────────

describe('formatDate', () => {
  it('formatea fecha ISO correctamente', () => {
    expect(formatDate('2024-01-15T10:30:00Z')).toBe('15/01/2024');
  });

  it('retorna cadena vacía para valor null', () => {
    expect(formatDate(null)).toBe('');
  });

  it('retorna cadena vacía para valor undefined', () => {
    expect(formatDate(undefined)).toBe('');
  });

  it('retorna cadena vacía para fecha inválida', () => {
    expect(formatDate('not-a-date')).toBe('');
  });
});

// ─── truncateText ─────────────────────────────────────────────────────────────

describe('truncateText', () => {
  it('devuelve el texto sin cambios si es más corto que el límite', () => {
    expect(truncateText('Hola', 10)).toBe('Hola');
  });

  it('devuelve el texto sin cambios si tiene exactamente el límite', () => {
    expect(truncateText('Hola mundo', 10)).toBe('Hola mundo');
  });

  it('trunca y añade "..." cuando el texto supera el límite', () => {
    expect(truncateText('Hola mundo cruel', 10)).toBe('Hola mundo...');
  });

  it('devuelve cadena vacía para entrada vacía', () => {
    expect(truncateText('', 10)).toBe('');
  });
});

// ─── groupByStatus ────────────────────────────────────────────────────────────

describe('groupByStatus', () => {
  const tasks = [
    { id: 1, name: 'A', status: 'backlog' as const },
    { id: 2, name: 'B', status: 'doing' as const },
    { id: 3, name: 'C', status: 'done' as const },
    { id: 4, name: 'D', status: 'backlog' as const },
  ];

  it('agrupa correctamente por status', () => {
    const result = groupByStatus(tasks);

    expect(result.backlog).toHaveLength(2);  // exacto — mata mutantes
    expect(result.doing).toHaveLength(1);
    expect(result.done).toHaveLength(1);
  });

  it('incluye los elementos correctos en cada grupo', () => {
    const result = groupByStatus(tasks);

    expect(result.backlog[0].name).toBe('A');
    expect(result.backlog[1].name).toBe('D');
    expect(result.doing[0].name).toBe('B');
    expect(result.done[0].name).toBe('C');
  });

  it('retorna grupos vacíos cuando no hay tareas', () => {
    const result = groupByStatus([]);

    expect(result.backlog).toHaveLength(0);
    expect(result.doing).toHaveLength(0);
    expect(result.done).toHaveLength(0);
  });
});

// ─── calculateProgress ────────────────────────────────────────────────────────

describe('calculateProgress', () => {
  it('retorna 0 cuando no hay tareas completadas', () => {
    expect(calculateProgress(0, 5)).toBe(0);
  });

  it('retorna 100 cuando todas las tareas están completadas', () => {
    expect(calculateProgress(5, 5)).toBe(100);
  });

  it('calcula el porcentaje correctamente', () => {
    expect(calculateProgress(2, 4)).toBe(50);
    expect(calculateProgress(1, 3)).toBe(33);
  });

  it('retorna 0 cuando no hay tareas en total', () => {
    expect(calculateProgress(0, 0)).toBe(0);
  });
});
```

## Tests parametrizados con `it.each`

```tsx
import { describe, it, expect } from 'vitest';
import { validateEmail, slugify } from './utils';

// ─── Tests parametrizados — múltiples inputs con una sola definición ──────────

describe('validateEmail', () => {
  it.each([
    ['usuario@ejemplo.com', true],
    ['nombre.apellido@empresa.org', true],
    ['usuario+tag@dominio.co', true],
  ])('valida "%s" como válido', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });

  it.each([
    ['noesvalido', false],
    ['falta@', false],
    ['@sindominio.com', false],
    ['espacios @x.com', false],
    ['', false],
  ])('rechaza "%s" como inválido', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });
});

describe('slugify', () => {
  it.each([
    ['Hola Mundo', 'hola-mundo'],
    ['Título Con Acentos', 'titulo-con-acentos'],
    ['  espacios extra  ', 'espacios-extra'],
    ['múltiples   espacios', 'multiples-espacios'],
    ['MAYÚSCULAS', 'mayusculas'],
  ])('convierte "%s" a "%s"', (input, expected) => {
    expect(slugify(input)).toBe(expected);
  });
});
```
