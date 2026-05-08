---
name: frontend-performance
description: Patrones de performance para FarmLabs centrados en Web Vitals (LCP, CLS, INP), code splitting con React.lazy/Suspense, memoización (React.memo/useMemo/useCallback), optimización de imágenes (loading="lazy", width/height, onError, referrerpolicy) y reportWebVitals. Úsalo cuando una tarea de §10 toque rendimiento — especialmente Sprint 5 completo (5.1–5.10). Aporta los patrones; el flujo execute-sprint-task aplica los cambios.
---

# Skill: frontend-performance

Aporta los patrones de performance específicos del Sprint 5 y mejoras puntuales en otros sprints. **No actúa sola** — la invoca el agente junto con `execute-sprint-task`.

## Métricas que importan

| Métrica | Objetivo | Qué la afecta en este repo |
|---------|----------|----------------------------|
| **LCP** (Largest Contentful Paint) | < 2.5s | Imágenes hot-linked sin `loading`, sin dimensiones, sin preconnect |
| **CLS** (Cumulative Layout Shift) | < 0.1 | Imágenes sin `width/height`, fonts sin `font-display: swap` |
| **INP** (Interaction to Next Paint) | < 200ms | Re-renders de toda la grilla al añadir al carrito |
| **Bundle size** | < 200kb gzip inicial | Sin code splitting, todas las páginas en el bundle inicial |

Objetivo del Sprint 5: Lighthouse Performance ≥ 85.

## 1. Code splitting por ruta (Sprint 5.1) 🟠

```jsx
// src/App.jsx
import React, { Suspense } from 'react';

const Home = React.lazy(() => import('./pages/Home'));
const Login = React.lazy(() => import('./pages/Login/Login'));
// ... una import por página

// dentro del Provider:
<Suspense fallback={<p>{UI_TEXT.catalog.loading}</p>}>
  <Routes>
    <Route path={APP_ROUTES.HOME} element={<NavLayout><Home /></NavLayout>} />
    {/* ... */}
  </Routes>
</Suspense>
```

Reglas:
- Una page = un chunk. NO hagas lazy del Header o NavLayout (siempre presentes).
- El fallback debe usar `UI_TEXT`, no copy literal.
- Verifica que el routing siga funcionando bajo `/FarmLabs/` (ver `BasenameRouting.test.js`).

## 2. Memoización (Sprint 5.2 / 5.3) 🟡

### React.memo en items de lista

```jsx
// src/components/ProductItem/ProductItem.jsx
const ProductItem = ({ product }) => { /* ... */ };

// Igualdad por id; el objeto product entero puede cambiar pero el contenido no.
const arePropsEqual = (a, b) => a.product.id === b.product.id;
export default React.memo(ProductItem, arePropsEqual);
```

### useMemo para totales

```jsx
const sumTotal = useMemo(() => state.cart.reduce((acc, item) => {
  const qty = item.quantity || 1;
  const price = Number.isFinite(Number(item.price)) ? Number(item.price) : 0;
  return acc + (price * qty);
}, 0), [state.cart]);

const articleCount = useMemo(
  () => state.cart.reduce((acc, item) => acc + (item.quantity || 1), 0),
  [state.cart]
);
```

### useCallback solo si memoizas el consumer

`useCallback` sin un `React.memo` consumiendo la función es overhead sin beneficio. NO lo añadas preventivamente.

## 3. Imágenes (Sprint 5.4 / 5.5 / 5.6)

```jsx
<img
  src={productImage}
  alt={productTitle}
  loading="lazy"
  width={240}
  height={240}
  referrerPolicy="no-referrer"
  onError={(e) => { e.currentTarget.src = fallbackProductImage; }}
/>
```

Reglas:
- `loading="lazy"` en TODAS las imágenes excepto la "above the fold" (logo del Header). En esas usa `loading="eager"` y considera `fetchpriority="high"`.
- `width` y `height` literales (no porcentajes) para reservar layout box → previene CLS.
- `referrerPolicy="no-referrer"` en imágenes externas (este repo: `purecannastore.com`).
- `onError` cae a `fallbackProductImage` (`pexels-photo-260024.webp`).

## 4. reportWebVitals (Sprint 5.9)

`src/index.js` actualmente:
```js
reportWebVitals();  // ⚠️ no mide nada
```

Cambio mínimo:
```js
reportWebVitals(console.log);
```

Cambio recomendado para producción futura:
```js
reportWebVitals((metric) => {
  // navigator.sendBeacon('/analytics', JSON.stringify(metric));
  if (process.env.NODE_ENV === 'development') console.log(metric);
});
```

## 5. Hook deps estables (Sprint 5.10)

❌ Frágil:
```jsx
useEffect(() => { /* fetch */ }, [requestVersion, source]);  // source es objeto
```

✅ Estable:
```jsx
const { endpoint, timeoutMs } = source;
useEffect(() => { /* fetch */ }, [requestVersion, endpoint, timeoutMs]);
```

## 6. Bundle analysis

Después de `npm run build`, inspecciona:

```bash
du -sh build/static/js/*.js
ls -la build/static/js/*.js
```

Tras Sprint 5.1, deberías ver múltiples chunks. El chunk principal `main.*.js` debería caer significativamente.

Para análisis profundo (no requerido por §10):
```bash
npx source-map-explorer 'build/static/js/*.js'
```

## 7. Anti-patrones

- `useMemo` sobre cálculos baratos (`a + b`).
- `React.memo` sobre todo "por si acaso".
- `loading="lazy"` en el logo / imagen LCP.
- Animaciones en `width/height/top/left` (forzar layout) — usar `transform`/`opacity`.
- Importar lodash entero por una función (`import _ from 'lodash'` → usa imports puntuales).
- Inline styles que cambian en cada render.
- Listas grandes sin `key={item.id}` estable.

## 8. Verificación

Tras aplicar cambios del Sprint 5:

1. `npm run build` y revisa tamaño de chunks (`build/static/js/`).
2. Servir build con `npx serve -s build` y correr Lighthouse.
3. Performance ≥ 85, Accessibility ≥ 95 (combinado con `frontend-accessibility`).
4. Verifica que `npm test` siga pasando — `React.lazy` puede romper tests síncronos sin `Suspense` en setup.

## Referencias cruzadas

- Imágenes accesibles → `frontend-accessibility`.
- Tests con lazy components → `frontend-testing`.
- Convenciones React → `frontend-component-author`.
