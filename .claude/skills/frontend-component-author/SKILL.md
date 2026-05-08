---
name: frontend-component-author
description: Patrones para escribir o modificar componentes React en FarmLabs. Aplica las convenciones específicas del proyecto (carpeta + .jsx + .css, APP_ROUTES, UI_TEXT, normalizeProduct, AppContext) y los patrones defensivos vigentes (Number.isFinite, fallbacks, AbortController). Úsalo como guía dominio cuando una tarea de §10 requiera crear o tocar componentes/hooks/contextos React. Triggers típicos: "crea un componente", "añade un hook", "modifica X.jsx", o cualquier tarea de los Sprints 2, 3, 5, 6 que toque src/components, src/containers, src/hooks o src/contexts. No edita archivos por sí sola: aporta patrones que el flujo execute-sprint-task aplica.
---

# Skill: frontend-component-author

Aporta los patrones React específicos de FarmLabs. **No actúa sola** — la invoca el agente `farmlabs` junto con `execute-sprint-task` cuando la tarea toca código React.

## Convenciones del proyecto (ver §3, §4, §6 de CLAUDE.md)

### Estructura de archivos

- Cada componente nuevo → `src/components/<Nombre>/<Nombre>.jsx` + `<Nombre>.css` hermano.
- Containers (composición de varios componentes) → `src/containers/<Nombre>/`.
- Páginas a nivel de ruta → `src/pages/<Nombre>/<Nombre>.jsx`.
- Hooks → `src/hooks/use<Nombre>.js`.
- Contextos → `src/contexts/<Nombre>Context.js`.

### Imports y exports

- Imports relativos (`../../`), agrupados: React → libs → utils/constants/config → componentes → assets/CSS.
- `export default` al final del archivo, nunca inline.
- Nombre del archivo = nombre del componente exportado.

### Constantes y copy

- **Toda ruta o `<Link>`** usa `APP_ROUTES` desde `src/constants/routes.js`. Nunca strings literales.
- **Todo copy visible** usa `UI_TEXT` desde `src/constants/uiText.js`. Si el copy es nuevo, añádelo a `UI_TEXT` antes de usarlo.
- **Datos del catálogo** pasan por `normalizeProduct(s)` desde `src/utils/normalizeProducts.js` antes del render.

### Estado y contexto

- Estado del carrito vive en `useInitialState()` y se inyecta vía `AppContext.Provider` en `App.jsx`. Cualquier componente que necesite el carrito hace `useContext(AppContext)`.
- Para forms, usa `useState` para el error y `useRef` o `event.currentTarget` para `FormData` — sigue el patrón que ya usan `Login.jsx` y `CreateAccount.jsx`.
- A partir del Sprint 6, el carrito migrará a `useReducer` + `localStorage`. Hasta entonces, NO añadas reducers nuevos al carrito.

## Patrones defensivos vigentes

Estos patrones ya existen en el repo, replícalos en código nuevo:

```jsx
// Numéricos seguros
const unitPrice = Number.isFinite(Number(item.price)) ? Number(item.price) : 0;

// Strings con fallback
const title = product.title || 'Untitled product';

// Arrays con fallback
const image = product.images && product.images.length > 0 ? product.images[0] : fallbackProductImage;

// Quantities con default
const quantity = item.quantity || 1;
```

## Patrón de fetch (useGetProducts)

Cualquier hook que haga peticiones HTTP:

```javascript
useEffect(() => {
  let isMounted = true;
  const controller = new AbortController();

  const run = async () => {
    try {
      const response = await axios.get(endpoint, {
        signal: controller.signal,
        timeout: timeoutMs,
      });
      if (!isMounted) return;
      setData(normalize(response.data));
    } catch (err) {
      if (!isMounted || controller.signal.aborted) return;
      setError(UI_TEXT.<scope>.loadError);
    }
  };
  run();

  return () => {
    isMounted = false;
    controller.abort();
  };
}, [/* deps estables */]);
```

⚠️ **Sprint 5.10:** desestructura `endpoint`/`timeoutMs` de `source` en las deps, no pases el objeto entero.

## Patrón de form de auth

```jsx
const form = useRef(null);
const [error, setError] = useState('');

const handleSubmit = (event) => {
  event.preventDefault();
  const formData = new FormData(form.current);
  const email = (formData.get('email') || '').toString().trim();
  const password = (formData.get('password') || '').toString();

  if (!isValidEmail(email)) { setError('...'); return; }
  if (!isValidPassword(password)) { setError('...'); return; }

  setError('');
  // Sprint 1.1: NO console.log({ password })
  // Sprint 3.3: auth.login(email, password)
};
```

## Cuándo memoizar (Sprint 5.2 / 5.3)

- `React.memo(Component)` cuando el componente está dentro de un `.map()` y las props son estables (ej. `ProductItem` por `product.id`).
- `useMemo(() => derived, [deps])` para totales/derivados costosos (`sumTotal`, `articleCount`, `cartItemsCount`).
- `useCallback(fn, [deps])` solo si la función se pasa como prop a un componente memoizado.

NO memoices preventivamente — solo cuando hay re-render observable o profilable.

## Lazy loading de rutas (Sprint 5.1)

```jsx
const Home = React.lazy(() => import('./pages/Home'));
// ...
<Suspense fallback={<p>{UI_TEXT.catalog.loading}</p>}>
  <Routes>...</Routes>
</Suspense>
```

## Anti-patrones

- Strings literales en `<Link to="...">` o `navigate('...')`.
- Copy hardcodeado en JSX que no esté en `UI_TEXT`.
- `<img onClick>` en lugar de `<button>` (ver `frontend-accessibility`).
- `console.log` con datos sensibles.
- Mutar estado directamente (`state.cart.push(...)`).
- `<form action="/">` sin `onSubmit` + `preventDefault`.
- Usar `localStorage` para tokens de auth (usa `sessionStorage` en mock).
- Crear hooks/contextos sin guard (`if (!ctx) throw new Error(...)` cuando aplique).

## Referencias cruzadas

- Para accesibilidad → `frontend-accessibility`.
- Para perf (lazy/memo/imágenes) → `frontend-performance`.
- Para tests → `frontend-testing`.
- Para implementar → siempre dentro de `execute-sprint-task`.
