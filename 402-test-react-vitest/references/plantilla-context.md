# Plantilla: Test de Context / Provider

## Estructura base

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, act } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TaskProvider, useTaskContext } from './TaskContext';

// ─── Componente auxiliar para testear el contexto ─────────────────────────────

const TestConsumer = () => {
  const { tasks, addTask, removeTask, toggleTask } = useTaskContext();

  return (
    <div>
      <span data-testid="task-count">{tasks.length}</span>
      <ul>
        {tasks.map((task) => (
          <li key={task.id}>
            <span>{task.name}</span>
            <span>{task.completed ? 'completada' : 'pendiente'}</span>
            <button onClick={() => toggleTask(task.id)}>Toggle</button>
            <button onClick={() => removeTask(task.id)}>Eliminar</button>
          </li>
        ))}
      </ul>
      <button onClick={() => addTask({ name: 'Nueva tarea' })}>Añadir</button>
    </div>
  );
};

// ─── Tests ────────────────────────────────────────────────────────────────────

describe('TaskContext', () => {
  const renderWithProvider = (ui = <TestConsumer />) =>
    render(<TaskProvider>{ui}</TaskProvider>);

  it('provee estado inicial vacío', () => {
    renderWithProvider();
    expect(screen.getByTestId('task-count')).toHaveTextContent('0');
  });

  it('añade una tarea correctamente', async () => {
    const user = userEvent.setup();
    renderWithProvider();

    await user.click(screen.getByRole('button', { name: 'Añadir' }));

    expect(screen.getByTestId('task-count')).toHaveTextContent('1');
    expect(screen.getByText('Nueva tarea')).toBeInTheDocument();
    expect(screen.getByText('pendiente')).toBeInTheDocument();
  });

  it('elimina una tarea correctamente', async () => {
    const user = userEvent.setup();
    renderWithProvider();

    await user.click(screen.getByRole('button', { name: 'Añadir' }));
    expect(screen.getByTestId('task-count')).toHaveTextContent('1');

    await user.click(screen.getByRole('button', { name: 'Eliminar' }));
    expect(screen.getByTestId('task-count')).toHaveTextContent('0');
  });

  it('alterna el estado de completado', async () => {
    const user = userEvent.setup();
    renderWithProvider();

    await user.click(screen.getByRole('button', { name: 'Añadir' }));
    expect(screen.getByText('pendiente')).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: 'Toggle' }));
    expect(screen.getByText('completada')).toBeInTheDocument();

    // Volver a pendiente
    await user.click(screen.getByRole('button', { name: 'Toggle' }));
    expect(screen.getByText('pendiente')).toBeInTheDocument();
  });

  it('lanza error cuando se usa useTaskContext fuera del Provider', () => {
    // Silenciar el error de consola esperado
    const consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {});

    expect(() => {
      render(<TestConsumer />);
    }).toThrow(/debe usarse dentro de TaskProvider/i);

    consoleSpy.mockRestore();
  });
});
```

## Variante: Context con estado asíncrono (carga de datos)

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import { TaskProvider } from './TaskContext';

// Mock del módulo de API
vi.mock('../api/tasksApi', () => ({
  fetchTasks: vi.fn().mockResolvedValue([
    { id: 1, name: 'Tarea cargada', completed: false },
  ]),
}));

describe('TaskContext con carga async', () => {
  it('muestra spinner durante la carga', () => {
    render(
      <TaskProvider>
        <TestConsumer />
      </TaskProvider>
    );

    expect(screen.getByRole('status')).toBeInTheDocument();  // spinner
  });

  it('muestra los datos una vez cargados', async () => {
    render(
      <TaskProvider>
        <TestConsumer />
      </TaskProvider>
    );

    await waitFor(() => {
      expect(screen.queryByRole('status')).not.toBeInTheDocument();
    });

    expect(screen.getByText('Tarea cargada')).toBeInTheDocument();
    expect(screen.getByTestId('task-count')).toHaveTextContent('1');
  });
});
```

## Variante: múltiples Providers anidados

```tsx
const AllProviders = ({ children }: { children: ReactNode }) => (
  <AuthProvider>
    <TaskProvider>
      <ThemeProvider>
        {children}
      </ThemeProvider>
    </TaskProvider>
  </AuthProvider>
);

// Reutilizable en todos los tests del módulo
const renderWithAllProviders = (ui: ReactNode) =>
  render(ui, { wrapper: AllProviders });
```
