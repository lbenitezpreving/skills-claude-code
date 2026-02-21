# Patrones avanzados de testing React/Vitest

## 1. userEvent avanzado — keyboard, tab, select

```tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import TaskForm from './TaskForm';

describe('TaskForm — interacciones avanzadas', () => {
  it('navega entre campos con Tab', async () => {
    const user = userEvent.setup();
    render(<TaskForm />);

    await user.tab();
    expect(screen.getByLabelText(/nombre/i)).toHaveFocus();

    await user.tab();
    expect(screen.getByLabelText(/descripción/i)).toHaveFocus();
  });

  it('selecciona opción en select', async () => {
    const user = userEvent.setup();
    render(<TaskForm />);

    await user.selectOptions(
      screen.getByRole('combobox', { name: /estado/i }),
      'doing'
    );

    expect(screen.getByRole('combobox', { name: /estado/i })).toHaveValue('doing');
  });

  it('envía formulario con Enter', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<TaskForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/nombre/i), 'Mi tarea');
    await user.keyboard('{Enter}');

    expect(onSubmit).toHaveBeenCalledWith({ name: 'Mi tarea' });
  });

  it('borra texto con teclado', async () => {
    const user = userEvent.setup();
    render(<TaskForm />);

    const input = screen.getByLabelText(/nombre/i);
    await user.type(input, 'texto a borrar');
    await user.clear(input);

    expect(input).toHaveValue('');
  });
});
```

## 2. waitFor y findByX — elementos asíncronos

```tsx
import { render, screen, waitFor } from '@testing-library/react';

// Esperar hasta que aparezca un elemento
const element = await screen.findByText(/cargado/i);

// Esperar hasta que desaparezca
await waitFor(() => {
  expect(screen.queryByRole('status')).not.toBeInTheDocument();
});

// Esperar múltiples condiciones
await waitFor(() => {
  expect(screen.getByText('Tarea 1')).toBeInTheDocument();
  expect(screen.getByText('Tarea 2')).toBeInTheDocument();
  expect(screen.queryByRole('status')).not.toBeInTheDocument();
});

// waitFor con timeout personalizado (evitar si es posible)
await waitFor(
  () => expect(screen.getByText(/muy lento/i)).toBeInTheDocument(),
  { timeout: 5000 }
);
```

## 3. Timers falsos (vi.useFakeTimers)

```tsx
import { vi } from 'vitest';
import { render, screen, act } from '@testing-library/react';
import AutoSave from './AutoSave';

describe('AutoSave — debounce', () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it('no guarda inmediatamente', () => {
    const onSave = vi.fn();
    render(<AutoSave value="texto" onSave={onSave} delay={1000} />);

    vi.advanceTimersByTime(500);  // menos del delay
    expect(onSave).not.toHaveBeenCalled();
  });

  it('guarda después del delay', () => {
    const onSave = vi.fn();
    render(<AutoSave value="texto" onSave={onSave} delay={1000} />);

    act(() => { vi.advanceTimersByTime(1000); });
    expect(onSave).toHaveBeenCalledTimes(1);
    expect(onSave).toHaveBeenCalledWith('texto');
  });

  it('resetea el timer si el valor cambia', () => {
    const onSave = vi.fn();
    const { rerender } = render(<AutoSave value="inicial" onSave={onSave} delay={1000} />);

    vi.advanceTimersByTime(500);
    rerender(<AutoSave value="actualizado" onSave={onSave} delay={1000} />);
    vi.advanceTimersByTime(500);  // 500ms después del nuevo render

    expect(onSave).not.toHaveBeenCalled();  // timer reiniciado
  });
});
```

## 4. rerender — probar cambios de props

```tsx
import { render, screen } from '@testing-library/react';
import StatusBadge from './StatusBadge';

describe('StatusBadge — rerender', () => {
  it('actualiza el texto cuando cambia el status', () => {
    const { rerender } = render(<StatusBadge status="backlog" />);
    expect(screen.getByText('Pendiente')).toBeInTheDocument();

    rerender(<StatusBadge status="doing" />);
    expect(screen.getByText('En progreso')).toBeInTheDocument();

    rerender(<StatusBadge status="done" />);
    expect(screen.getByText('Completado')).toBeInTheDocument();
  });
});
```

## 5. Spy de módulos — verificar llamadas a funciones externas

```tsx
import { vi } from 'vitest';
import * as router from '../router';

describe('NavButton', () => {
  it('navega a la ruta correcta al hacer clic', async () => {
    const navigateSpy = vi.spyOn(router, 'navigate').mockImplementation(() => {});
    const user = userEvent.setup();

    render(<NavButton to="/tasks">Tareas</NavButton>);
    await user.click(screen.getByRole('button', { name: 'Tareas' }));

    expect(navigateSpy).toHaveBeenCalledWith('/tasks');
    navigateSpy.mockRestore();
  });
});
```

## 6. Drag & drop — solo con fireEvent (no soportado por userEvent)

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import KanbanBoard from './KanbanBoard';

it('mueve tarea a otra columna vía drag & drop', () => {
  const onStatusChange = vi.fn();
  render(<KanbanBoard tasks={mockTasks} onTaskStatusChange={onStatusChange} />);

  const card = screen.getByRole('heading', { name: 'Tarea en Backlog' })
    .closest('div[draggable="true"]');

  // Simular drag start
  const dragStartEvent = new Event('dragstart', { bubbles: true });
  Object.defineProperty(dragStartEvent, 'dataTransfer', {
    value: { effectAllowed: '', dropEffect: '' },
  });
  fireEvent(card!, dragStartEvent);

  // Simular drop en columna destino
  const doingColumn = screen.getByText(/En Progreso/i).closest('[class*="column"]');
  const dropEvent = new Event('drop', { bubbles: true });
  Object.defineProperty(dropEvent, 'dataTransfer', {
    value: { dropEffect: '' },
  });
  fireEvent(doingColumn!, dropEvent);

  expect(onStatusChange).toHaveBeenCalledWith(1, 'doing');
});
```

## 7. Assertions avanzadas de DOM

```tsx
// Verificar atributos
expect(button).toBeDisabled();
expect(input).toBeRequired();
expect(link).toHaveAttribute('href', '/tasks');
expect(img).toHaveAttribute('alt', 'Descripción');

// Verificar clases CSS
expect(element).toHaveClass('active');
expect(element).not.toHaveClass('error');

// Verificar estilos inline
expect(element).toHaveStyle({ color: 'red', display: 'none' });

// Verificar texto
expect(element).toHaveTextContent('Texto exacto');
expect(element).toHaveTextContent(/parcial/i);

// Verificar valor de inputs
expect(input).toHaveValue('texto');
expect(checkbox).toBeChecked();
expect(select).toHaveValue('opcion');
```
