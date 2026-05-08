---
name: frontend-testing
description: Patrones de testing con React Testing Library + Jest siguiendo el estilo de App.test.js y BasenameRouting.test.js. Cubre tests unit de utils, hooks (con axios mock), componentes y formularios. Define cuándo usar fireEvent vs userEvent, cómo mockear hooks, y la cobertura objetivo (≥ 70% en utils/ y hooks/). Úsalo en todo el Sprint 4 (4.1–4.8) y siempre que una tarea pida "añade tests" o cambios que rompan los tests existentes. Aporta los patrones; el flujo execute-sprint-task aplica los cambios.
---

# Skill: frontend-testing

Aporta los patrones de testing de FarmLabs. **No actúa sola** — la invoca el agente junto con `execute-sprint-task` cuando la tarea es del Sprint 4 o introduce/modifica tests.

## Stack y convenciones existentes

- **Runner:** Jest vía CRA (cambiará a Vitest en Sprint 8.6).
- **Librería:** `@testing-library/react`, `@testing-library/jest-dom`.
- **Setup:** `src/setupTests.js`.
- **Tests existentes:** `src/App.test.js`, `src/BasenameRouting.test.js`. Estúdialos antes de añadir tests nuevos — replican el estilo del repo.

## Estructura de un archivo de test

```javascript
import { fireEvent, render, screen } from '@testing-library/react';
import App from './App';
import useGetProducts from './hooks/useGetProducts';

jest.mock('./hooks/useGetProducts');

const mockProducts = [/* ... */];

const mockHookResponse = () => {
  useGetProducts.mockReturnValue({
    products: mockProducts,
    loading: false,
    error: '',
    reload: jest.fn(),
  });
};

describe('Feature X', () => {
  beforeEach(() => {
    mockHookResponse();
  });

  afterEach(() => {
    jest.clearAllMocks();
    window.history.pushState({}, '', '/');
  });

  test('does the thing', () => {
    render(<App />);
    expect(screen.getByText('something')).toBeInTheDocument();
  });
});
```

Reglas que se ven en el repo:
- `describe` agrupa por feature, no por archivo.
- `beforeEach` para preparar mocks; `afterEach` para limpiar.
- `window.history.pushState` para resetear ruta entre tests.
- Mockea hooks de fetch — NO hagas peticiones reales en tests.

## Tests del Sprint 4 — qué escribir y dónde

### 4.1 — `src/utils/validation.test.js`

```javascript
import { isValidEmail, isValidPassword, MIN_PASSWORD_LENGTH } from './validation';

describe('isValidEmail', () => {
  test.each([
    ['user@example.com', true],
    ['  user@example.com  ', true],   // trim
    ['user.tag+x@dom.co', true],
    ['nope', false],
    ['no@dot', false],
    ['@nodom.com', false],
    ['', false],
  ])('"%s" → %s', (input, expected) => {
    expect(isValidEmail(input)).toBe(expected);
  });
});

describe('isValidPassword', () => {
  test('rejects below MIN_PASSWORD_LENGTH', () => {
    expect(isValidPassword('1234567')).toBe(false);
    expect(isValidPassword('12345678')).toBe(true);
  });
  test('handles whitespace', () => {
    expect(isValidPassword('  short  ')).toBe(false);
  });
});
```

### 4.2 — `src/utils/normalizeProducts.test.js`

Casos a cubrir:
- Producto vacío → defaults (`title: 'Untitled product'`, `price: 0`, `images: []`).
- `images` no array → `[]`.
- `price` string numérico válido → number.
- `price` no numérico → 0.
- Lista `null` o `undefined` en `normalizeProducts` → `[]`.
- Lista vacía → `[]`.

### 4.3 — `src/hooks/useInitialState.test.js`

Usa `renderHook` de `@testing-library/react`:

```javascript
import { renderHook, act } from '@testing-library/react';
import useInitialState from './useInitialState';

test('addToCart increments quantity for same id', () => {
  const { result } = renderHook(() => useInitialState());
  const product = { id: 1, title: 'X', price: 10 };
  act(() => result.current.addToCart(product));
  act(() => result.current.addToCart(product));
  expect(result.current.state.cart).toHaveLength(1);
  expect(result.current.state.cart[0].quantity).toBe(2);
});

test('removeFromCart drops line at zero', () => {
  const { result } = renderHook(() => useInitialState());
  const product = { id: 1, title: 'X', price: 10 };
  act(() => result.current.addToCart(product));
  act(() => result.current.removeFromCart(product));
  expect(result.current.state.cart).toHaveLength(0);
});
```

### 4.4 — `src/hooks/useGetProducts.test.js`

Mock axios con `jest.mock('axios')`:

```javascript
import axios from 'axios';
import { renderHook, waitFor } from '@testing-library/react';
import useGetProducts from './useGetProducts';

jest.mock('axios');

const source = { endpoint: '/data/products.json', timeoutMs: 1000 };

test('returns products on success', async () => {
  axios.get.mockResolvedValue({ data: [{ id: 1, title: 'X', price: 5 }] });
  const { result } = renderHook(() => useGetProducts(source));
  await waitFor(() => expect(result.current.loading).toBe(false));
  expect(result.current.products).toHaveLength(1);
});

test('sets error on failure', async () => {
  axios.get.mockRejectedValue(new Error('500'));
  const { result } = renderHook(() => useGetProducts(source));
  await waitFor(() => expect(result.current.loading).toBe(false));
  expect(result.current.error).toBeTruthy();
});
```

### 4.5 — `src/pages/Login/Login.test.jsx`

```javascript
import { render, screen, fireEvent } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import Login from './Login';

const renderWithRouter = () => render(<MemoryRouter><Login /></MemoryRouter>);

test('shows error on invalid email', () => {
  renderWithRouter();
  fireEvent.change(screen.getByLabelText(/email/i), { target: { value: 'nope' } });
  fireEvent.change(screen.getByLabelText(/password/i), { target: { value: '12345678' } });
  fireEvent.click(screen.getByRole('button', { name: /log in/i }));
  expect(screen.getByText(/valid email/i)).toBeInTheDocument();
});
```

### 4.6 — `src/components/RequireAuth.test.jsx`

Tras Sprint 3.4. Ver `frontend-component-author` para el patrón de `RequireAuth`.

## Queries — orden de preferencia

1. `getByRole('button', { name: '...' })` — más cercano al usuario.
2. `getByLabelText(...)` — para inputs.
3. `getByText(...)` — para texto visible.
4. `getByAltText(...)` — solo si la imagen es la única señal (ya en uso).
5. `getByTestId(...)` — último recurso, solo si nada de lo anterior funciona.

## fireEvent vs userEvent

- `fireEvent` es lo que usa el repo actualmente — síncrono, suficiente para clicks simples.
- `userEvent` es más realista (eventos de teclado, focus). Para forms con typing prefiérelo:
  ```javascript
  import userEvent from '@testing-library/user-event';
  await userEvent.type(input, 'hello');
  await userEvent.click(button);
  ```
- No mezcles ambos en un mismo test.

## Cobertura

Sprint 4 objetivo:
- `src/utils/` ≥ 70%.
- `src/hooks/` ≥ 70%.
- `src/components/` y `src/pages/` cubiertos por tests de integración (App.test.js + tests específicos de auth).

Reporta con:
```bash
npm test -- --coverage --watchAll=false
```

## Anti-patrones

- Llamar a APIs reales (incluso `/data/products.json` local) — siempre mock.
- `expect(component).toMatchSnapshot()` — frágil, evita.
- Tests dependientes del orden (sin `beforeEach`/`afterEach` limpiando).
- `wait(() => ...)` con `setTimeout` en lugar de `waitFor`.
- Selectores por className (`container.querySelector('.x')`) — usa queries semánticas.
- `act()` ignorado — silencia warnings importantes.
- Tests sobre detalles de implementación (`useState` interno).

## Migración a Vitest (Sprint 8.6)

Cuando llegue Sprint 8, los tests siguen siendo casi idénticos. Cambios:
- `jest.mock` → `vi.mock`.
- `jest.fn()` → `vi.fn()`.
- `jest.clearAllMocks()` → `vi.clearAllMocks()`.
- Setup en `vitest.config.js` con `setupFiles: ['./src/setupTests.js']` y `globals: true`.

## Referencias cruzadas

- Convenciones React → `frontend-component-author`.
- A11y queries (`getByRole`, `getByLabelText`) → `frontend-accessibility`.
- Implementar la tarea de §10 → `execute-sprint-task`.
- Verificar coverage → `verify-quality-gates`.
