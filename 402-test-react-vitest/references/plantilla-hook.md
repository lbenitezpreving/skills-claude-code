# Plantilla: Test de Custom Hook

Usa `renderHook` de Testing Library para testear hooks en aislamiento.

## Hook síncrono básico

```tsx
import { describe, it, expect, vi } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('inicializa con el valor por defecto', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('inicializa con el valor proporcionado', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('incrementa el contador', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('decrementa el contador', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(4);
  });

  it('resetea al valor inicial', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(5);  // valor inicial, no 0
  });
});
```

## Hook con fetch de datos (async)

```tsx
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { renderHook, waitFor } from '@testing-library/react';
import { useTasks } from './useTasks';

// Mock del módulo de API
vi.mock('../api/tasksApi', () => ({
  fetchTasks: vi.fn(),
}));

import { fetchTasks } from '../api/tasksApi';
const mockFetchTasks = vi.mocked(fetchTasks);

describe('useTasks', () => {
  const mockTasks = [
    { id: 1, name: 'Tarea 1' },
    { id: 2, name: 'Tarea 2' },
  ];

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('inicia en estado de carga', () => {
    mockFetchTasks.mockResolvedValue(mockTasks);
    const { result } = renderHook(() => useTasks());

    expect(result.current.loading).toBe(true);
    expect(result.current.tasks).toEqual([]);
    expect(result.current.error).toBeNull();
  });

  it('carga las tareas correctamente', async () => {
    mockFetchTasks.mockResolvedValue(mockTasks);
    const { result } = renderHook(() => useTasks());

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.tasks).toHaveLength(2);  // exacto — mata mutantes
    expect(result.current.tasks[0].name).toBe('Tarea 1');
    expect(result.current.error).toBeNull();
  });

  it('maneja errores de red', async () => {
    mockFetchTasks.mockRejectedValue(new Error('Network error'));
    const { result } = renderHook(() => useTasks());

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.error).toBe('Network error');
    expect(result.current.tasks).toHaveLength(0);
  });

  it('re-fetch al llamar a refetch()', async () => {
    mockFetchTasks.mockResolvedValue(mockTasks);
    const { result } = renderHook(() => useTasks());

    await waitFor(() => expect(result.current.loading).toBe(false));

    // Segunda llamada
    mockFetchTasks.mockResolvedValue([{ id: 3, name: 'Nueva' }]);
    act(() => { result.current.refetch(); });

    await waitFor(() => expect(result.current.loading).toBe(false));

    expect(result.current.tasks).toHaveLength(1);
    expect(mockFetchTasks).toHaveBeenCalledTimes(2);
  });
});
```

## Hook con Context como dependencia

```tsx
import { describe, it, expect } from 'vitest';
import { renderHook } from '@testing-library/react';
import { ReactNode } from 'react';
import { TaskProvider } from '../context/TaskContext';
import { useTaskContext } from './useTaskContext';

// Wrapper con el Provider necesario
const wrapper = ({ children }: { children: ReactNode }) => (
  <TaskProvider>{children}</TaskProvider>
);

describe('useTaskContext', () => {
  it('retorna las tareas del contexto', () => {
    const { result } = renderHook(() => useTaskContext(), { wrapper });
    expect(result.current.tasks).toHaveLength(0);
  });

  it('lanza error cuando se usa fuera del Provider', () => {
    // Sin wrapper — debe lanzar error
    expect(() => {
      renderHook(() => useTaskContext());
    }).toThrow(/debe usarse dentro de TaskProvider/i);
  });
});
```

## Hook con efectos de tiempo (vi.useFakeTimers)

```tsx
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useDebounce } from './useDebounce';

describe('useDebounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('retorna el valor inmediatamente', () => {
    const { result } = renderHook(() => useDebounce('hola', 500));
    expect(result.current).toBe('hola');
  });

  it('no actualiza antes del delay', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'inicial' } }
    );

    rerender({ value: 'nuevo' });
    vi.advanceTimersByTime(300);  // menos del delay

    expect(result.current).toBe('inicial');
  });

  it('actualiza después del delay', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'inicial' } }
    );

    rerender({ value: 'nuevo' });
    act(() => { vi.advanceTimersByTime(500); });

    expect(result.current).toBe('nuevo');
  });
});
```
