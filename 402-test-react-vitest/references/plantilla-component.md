# Plantilla: Test de Componente Funcional React

Basado en el patrón canónico de `KanbanBoard.test.tsx`.

## Estructura base

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import MiComponente from './MiComponente';

// ─── Mock data tipada ────────────────────────────────────────────────────────

interface MiEntidad {
  id: number;
  name: string;
  // ... otros campos
}

const mockItems: MiEntidad[] = [
  { id: 1, name: 'Elemento 1' },
  { id: 2, name: 'Elemento 2' },
];

// ─── Tests ───────────────────────────────────────────────────────────────────

describe('MiComponente', () => {
  // Setup compartido
  let user: ReturnType<typeof userEvent.setup>;

  beforeEach(() => {
    user = userEvent.setup();
  });

  // ── Render básico ──────────────────────────────────────────────────────────

  it('renderiza sin errores con props mínimas', () => {
    render(<MiComponente items={mockItems} />);
    // Verificar elemento central por rol semántico
    expect(screen.getByRole('list')).toBeInTheDocument();
  });

  it('muestra todos los elementos de la lista', () => {
    render(<MiComponente items={mockItems} />);
    const listItems = screen.getAllByRole('listitem');
    expect(listItems).toHaveLength(2);
  });

  it('muestra el nombre de cada elemento', () => {
    render(<MiComponente items={mockItems} />);
    expect(screen.getByText('Elemento 1')).toBeInTheDocument();
    expect(screen.getByText('Elemento 2')).toBeInTheDocument();
  });

  // ── Estado vacío ───────────────────────────────────────────────────────────

  it('muestra mensaje de vacío cuando no hay elementos', () => {
    render(<MiComponente items={[]} />);
    expect(screen.getByText(/no hay elementos/i)).toBeInTheDocument();
    expect(screen.queryByRole('listitem')).not.toBeInTheDocument();
  });

  // ── Estado de carga ────────────────────────────────────────────────────────

  it('muestra spinner cuando loading es true', () => {
    render(<MiComponente items={[]} loading={true} />);
    expect(screen.getByRole('status')).toBeInTheDocument(); // o getByText(/cargando/i)
  });

  // ── Interacciones ──────────────────────────────────────────────────────────

  it('llama a onItemClick con el elemento correcto al hacer clic', async () => {
    const onItemClick = vi.fn();
    render(<MiComponente items={mockItems} onItemClick={onItemClick} />);

    const firstItem = screen.getByRole('button', { name: /elemento 1/i });
    await user.click(firstItem);

    expect(onItemClick).toHaveBeenCalledTimes(1);
    expect(onItemClick).toHaveBeenCalledWith(mockItems[0]);
  });

  it('no llama a onItemClick si no se provee el callback', async () => {
    render(<MiComponente items={mockItems} />);
    const firstItem = screen.getByRole('button', { name: /elemento 1/i });
    // No debe lanzar error aunque no haya callback
    await user.click(firstItem);
  });

  // ── Props condicionales ────────────────────────────────────────────────────

  it('muestra badge cuando el elemento tiene proyecto asignado', () => {
    const itemsConProyecto = [{ ...mockItems[0], project: 'Proyecto A' }];
    render(<MiComponente items={itemsConProyecto} />);
    expect(screen.getByText('Proyecto A')).toBeInTheDocument();
  });

  it('no muestra badge cuando el elemento no tiene proyecto', () => {
    render(<MiComponente items={[{ id: 1, name: 'Sin proyecto' }]} />);
    expect(screen.queryByText(/proyecto/i)).not.toBeInTheDocument();
  });

  // ── Renderizado condicional ────────────────────────────────────────────────

  it('oculta el panel de acciones cuando disabled es true', () => {
    render(<MiComponente items={mockItems} disabled={true} />);
    expect(screen.queryByRole('button', { name: /eliminar/i })).not.toBeInTheDocument();
  });
});
```

## Variante: con `describe` anidados por comportamiento

```tsx
describe('MiComponente', () => {
  describe('cuando la lista está vacía', () => {
    it('muestra mensaje de estado vacío', () => {
      render(<MiComponente items={[]} />);
      expect(screen.getByText(/no hay elementos/i)).toBeInTheDocument();
    });

    it('no muestra el botón de limpiar', () => {
      render(<MiComponente items={[]} />);
      expect(screen.queryByRole('button', { name: /limpiar/i })).not.toBeInTheDocument();
    });
  });

  describe('cuando hay elementos', () => {
    it('muestra todos los elementos', () => {
      render(<MiComponente items={mockItems} />);
      expect(screen.getAllByRole('listitem')).toHaveLength(mockItems.length);
    });

    it('muestra el botón de limpiar', () => {
      render(<MiComponente items={mockItems} />);
      expect(screen.getByRole('button', { name: /limpiar/i })).toBeInTheDocument();
    });
  });
});
```

## Patrones de query por jerarquía

```tsx
// 1. Por rol semántico (preferido)
screen.getByRole('heading', { name: 'Mi Título' })
screen.getByRole('button', { name: /eliminar/i })
screen.getByRole('checkbox', { name: /completado/i })

// 2. Por label (inputs)
screen.getByLabelText(/nombre del proyecto/i)

// 3. Por texto visible
screen.getByText('Texto exacto')
screen.getByText(/texto parcial/i)

// 4. Para verificar ausencia
screen.queryByText(/no existe/i)  // retorna null si no existe (no lanza)

// 5. Para elementos asíncronos
await screen.findByText(/cargado/i)  // espera hasta que aparece
```
