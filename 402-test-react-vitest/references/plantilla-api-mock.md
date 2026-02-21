# Plantilla: Test de Servicio con Mock de API

Para componentes o hooks que hacen llamadas a APIs externas (fetch, axios).

## Opción A: vi.mock con módulos (más simple)

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import TaskList from './TaskList';

// Mock del módulo completo de API
vi.mock('../api/tasksApi', () => ({
  getTasks: vi.fn(),
  createTask: vi.fn(),
  deleteTask: vi.fn(),
  updateTask: vi.fn(),
}));

import * as tasksApi from '../api/tasksApi';

describe('TaskList', () => {
  const mockTasks = [
    { id: 1, name: 'Tarea 1', completed: false, status: 'backlog' },
    { id: 2, name: 'Tarea 2', completed: true, status: 'done' },
  ];

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('muestra spinner durante la carga', () => {
    vi.mocked(tasksApi.getTasks).mockReturnValue(new Promise(() => {})); // pending
    render(<TaskList />);
    expect(screen.getByRole('status')).toBeInTheDocument();
  });

  it('renderiza tareas una vez cargadas', async () => {
    vi.mocked(tasksApi.getTasks).mockResolvedValue(mockTasks);
    render(<TaskList />);

    await waitFor(() => {
      expect(screen.queryByRole('status')).not.toBeInTheDocument();
    });

    expect(screen.getAllByRole('listitem')).toHaveLength(2);
    expect(screen.getByText('Tarea 1')).toBeInTheDocument();
    expect(screen.getByText('Tarea 2')).toBeInTheDocument();
  });

  it('muestra error cuando la API falla', async () => {
    vi.mocked(tasksApi.getTasks).mockRejectedValue(new Error('API Error'));
    render(<TaskList />);

    await waitFor(() => {
      expect(screen.getByText(/error al cargar/i)).toBeInTheDocument();
    });
  });

  it('crea una tarea y actualiza la lista', async () => {
    const user = userEvent.setup();
    vi.mocked(tasksApi.getTasks).mockResolvedValue(mockTasks);
    const newTask = { id: 3, name: 'Nueva tarea', completed: false, status: 'backlog' };
    vi.mocked(tasksApi.createTask).mockResolvedValue(newTask);

    render(<TaskList />);
    await waitFor(() => screen.getByRole('textbox'));

    await user.type(screen.getByRole('textbox'), 'Nueva tarea');
    await user.click(screen.getByRole('button', { name: /crear/i }));

    expect(tasksApi.createTask).toHaveBeenCalledTimes(1);
    expect(tasksApi.createTask).toHaveBeenCalledWith({ name: 'Nueva tarea' });

    await waitFor(() => {
      expect(screen.getByText('Nueva tarea')).toBeInTheDocument();
    });
  });

  it('elimina una tarea y la remueve de la lista', async () => {
    const user = userEvent.setup();
    vi.mocked(tasksApi.getTasks).mockResolvedValue(mockTasks);
    vi.mocked(tasksApi.deleteTask).mockResolvedValue(undefined);

    render(<TaskList />);
    await waitFor(() => screen.getAllByRole('listitem'));

    const deleteButtons = screen.getAllByRole('button', { name: /eliminar/i });
    await user.click(deleteButtons[0]);

    expect(tasksApi.deleteTask).toHaveBeenCalledWith(1);
  });
});
```

## Opción B: Mock de fetch global (sin librería externa)

```tsx
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import TaskList from './TaskList';

describe('TaskList con fetch mock', () => {
  const mockTasks = [{ id: 1, name: 'Tarea', status: 'backlog' }];

  beforeEach(() => {
    global.fetch = vi.fn();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('carga y muestra tareas', async () => {
    vi.mocked(global.fetch).mockResolvedValueOnce({
      ok: true,
      json: async () => mockTasks,
    } as Response);

    render(<TaskList />);

    await waitFor(() => {
      expect(screen.getByText('Tarea')).toBeInTheDocument();
    });

    expect(global.fetch).toHaveBeenCalledWith('/tasks/');
  });

  it('maneja respuesta no-ok del servidor', async () => {
    vi.mocked(global.fetch).mockResolvedValueOnce({
      ok: false,
      status: 500,
      json: async () => ({ detail: 'Internal Server Error' }),
    } as Response);

    render(<TaskList />);

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

## Opción C: Mock Service Worker (MSW) — para tests más realistas

```tsx
// src/test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/tasks/', () => {
    return HttpResponse.json([
      { id: 1, name: 'Tarea MSW', status: 'backlog' }
    ]);
  }),

  http.post('/tasks/', async ({ request }) => {
    const body = await request.json() as { name: string };
    return HttpResponse.json(
      { id: 2, name: body.name, status: 'backlog' },
      { status: 201 }
    );
  }),

  http.delete('/tasks/:id', () => {
    return new HttpResponse(null, { status: 204 });
  }),
];

// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// src/test/setup.ts — añadir:
import { server } from './mocks/server';
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```
