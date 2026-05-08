# CLAUDE.md — Memoria de desarrollo de FarmLabs

> Este archivo es la memoria persistente del proyecto para asistentes Claude.
> Léelo al inicio de cada sesión antes de proponer cambios. Mantenlo actualizado
> cuando la arquitectura, las decisiones o el plan de mejoras cambien.

---

## 1. Resumen del proyecto

**FarmLabs** es una SPA de e-commerce/marketplace (catálogo tipo cannabis store)
construida con Create React App y desplegada en GitHub Pages.

- **Repo / homepage:** `https://jonahsly.github.io/FarmLabs/`
- **Tipo:** SPA front-end pura, sin backend propio.
- **Datos:** catálogo estático desde `public/data/products.json`. El carrito vive
  100 % en memoria (no hay persistencia ni API).
- **Estado del proyecto:** funcional como demo de portafolio; auth y checkout son
  mocks (`console.log`).

## 2. Stack y versiones

- React `^18.2.0` + ReactDOM `^18.2.0`
- React Router DOM `^6.4.3`
- Axios `^1.1.3` ⚠️ versión con CVEs conocidas, ver §7
- react-scripts `5.0.1` ⚠️ CRA está deprecado oficialmente
- Testing: `@testing-library/react`, `@testing-library/jest-dom`, Jest (vía CRA)
- Deploy: `gh-pages ^4.0.0` (manual con `npm run deploy`)
- Sin TypeScript, sin librería de UI, sin gestor de estado externo
- Sin `.eslintrc` ni `.prettierrc` propios (solo el preset CRA en `package.json`)

## 3. Estructura de carpetas

```
src/
├── App.jsx                    # Router + AppContext.Provider
├── index.js                   # Entry point con StrictMode
├── App.test.js                # Tests de catálogo, carrito y 404
├── BasenameRouting.test.js    # Tests con PUBLIC_URL=/FarmLabs
├── reportWebVitals.js         # Llamado sin callback (no mide nada)
├── setupTests.js
├── index.css
├── assets/                    # logos, iconos, imagen fallback
├── components/                # piezas pequeñas reutilizables
│   ├── Header/Header.jsx
│   ├── Menu/Menu.jsx
│   ├── ProductItem/ProductItem.jsx
│   └── OrderItem/OrderItem.jsx
├── containers/                # composiciones de varios componentes
│   ├── NavLayout.jsx          # ⚠️ solo se aplica a Home
│   ├── ProductList/ProductList.jsx
│   └── MyOrder/MyOrder.jsx    # panel lateral del carrito
├── pages/                     # vistas a nivel de ruta
│   ├── Home.jsx
│   ├── Login/Login.jsx
│   ├── CreateAccount/CreateAccount.jsx
│   ├── PasswordRecovery/PasswordRecovery.jsx
│   ├── NewPassword/NewPassword.jsx
│   ├── SendEmail/SendEmail.jsx
│   ├── MyAccount/MyAccount.jsx
│   ├── Checkout/Checkout.jsx
│   ├── Orders/Orders.jsx
│   └── NotFound/NotFound.jsx
├── hooks/
│   ├── useInitialState.js     # estado del carrito (addToCart, removeFromCart)
│   └── useGetProducts.js      # fetch del catálogo con AbortController
├── contexts/
│   └── AppContext.js          # contexto vacío, valor inyectado en App
├── utils/
│   ├── normalizeProducts.js   # capa defensiva sobre datos del JSON
│   └── validation.js          # isValidEmail, isValidPassword
├── constants/
│   ├── routes.js              # APP_ROUTES (Object.freeze)
│   └── uiText.js              # UI_TEXT (todos los textos centralizados)
├── config/
│   ├── appConfig.js           # supportEmail
│   └── productsSource.js      # endpoint + timeout
└── styles/
    ├── global.css
    └── vars.css

scripts/                       # tooling local
├── codeQuality.js             # wrapper de eslint (lint, lint:fix, format)
├── preReleaseCheck.js         # solo corre `npm run build`
├── preReleaseTest.js
└── preReleaseRoutes.js

public/
├── index.html
├── favicon_farm_labs.svg
└── data/products.json         # catálogo estático
```

## 4. Decisiones arquitectónicas vigentes

- **Rutas centralizadas** en `constants/routes.js` (`APP_ROUTES`). Cualquier
  nueva ruta o `<Link>` debe usar esta constante, no strings literales.
- **Textos UI centralizados** en `constants/uiText.js` (`UI_TEXT`). Cualquier
  copy nuevo se añade aquí, no inline.
- **Config centralizada** en `config/appConfig.js` y `config/productsSource.js`.
- **Normalización defensiva** en `utils/normalizeProducts.js`: todo lo que llega
  desde la red pasa por `normalizeProduct(s)` antes de tocar el render.
- **Validación reutilizable** en `utils/validation.js` (regex de email + min
  length de password). Importada por todas las páginas de auth.
- **Estado global mínimo:** `useInitialState` retorna `{ state, addToCart,
  removeFromCart }` y se inyecta en `AppContext.Provider`. Solo se gestiona
  el carrito.
- **Fetch seguro:** `useGetProducts` usa `AbortController` + flag `isMounted`
  para evitar `setState` después de unmount.
- **Routing con basename:** `<BrowserRouter basename={process.env.PUBLIC_URL}>`
  para soportar el subpath `/FarmLabs` de GitHub Pages.

## 5. Comandos

```bash
npm start              # dev server
npm run build          # build de producción (CRA)
npm test               # Jest en watch mode
npm run lint           # eslint sobre src/**/*.{js,jsx}
npm run lint:fix       # eslint --fix
npm run format         # alias de eslint --fix (no hay Prettier separado)
npm run pre-release:check   # corre `npm run build`
npm run deploy         # gh-pages -d build (predeploy ejecuta build)
```

## 6. Convenciones (ya en uso, mantenerlas)

- Cada componente vive en su propia carpeta con un `.jsx` y un `.css` hermano.
- Imports relativos (sin alias `@/`).
- `export default` al final del archivo.
- Comentarios "intent" cortos sobre el porqué de patrones defensivos.
- `Object.freeze` en objetos de constantes para evitar mutación accidental.

> ⚠️ Inconsistencia conocida: hay mezcla de **tabs** (Home, Login, Header,
> ProductItem) con **2 espacios** (NewPassword, PasswordRecovery, MyAccount).
> Pendiente unificar con Prettier.

## 7. Issues conocidos y deuda técnica

### 🔴 Crítico — corregir antes del próximo deploy
1. **`console.log({ password })`** en `Login.jsx`, `CreateAccount.jsx`,
   `NewPassword.jsx`. Fuga de credenciales en consola.
2. **PII hardcodeada** en `MyAccount.jsx` ("Jonah Sly" / "slyjonatan@gmail.com")
   versionada en git.
3. **`axios ^1.1.3`** afectado por CVE-2023-45857 y CVE-2024-39338.
   Subir a ≥ 1.7.4.
4. Verificar que **`build/`** esté en `.gitignore` (actualmente aparece
   versionada en el repo).

### 🟠 Alto impacto
5. **Auth es mock total.** No hay `AuthContext`, ni token, ni rutas protegidas.
   `MyAccount`, `Orders`, `Checkout` son públicas.
6. **Botones rotos:**
   - "Sign up" en Login no navega.
   - "Checkout" en `MyOrder` no navega a `/checkout`.
   - `<form action="/">` en `MyAccount` recarga la página al pulsar "Edit"
     (perdiendo el carrito).
   - Categorías del Header son `<Link to="/">` (placeholder).
   - El icono de menú (`imenu`) está en el DOM pero no tiene `onClick`; el
     toggle real está en el email — UX poco descubrible.
7. **Solo Home usa `NavLayout`.** `MyAccount`, `Orders`, `Checkout` no tienen
   header; el usuario queda atrapado sin navegación.
8. **CRA deprecado.** Migrar a Vite (rápido) o Next.js (robusto).

### 🟡 Mejora continua
9. **Accesibilidad:** imágenes clicables (`<img onClick>`) deben ser
   `<button>` (cart toggle, close en OrderItem, add-to-cart en ProductItem).
   Errores de formulario sin `aria-live`.
10. **Sin code splitting.** Todas las páginas se cargan en el bundle inicial.
    Convertir páginas a `React.lazy` + `Suspense`.
11. **Re-renders innecesarios.** `ProductItem` no está memoizado; `sumTotal`
    y `articleCount` se recalculan en cada render. Usar `React.memo` y
    `useMemo`.
12. **Estado del carrito sin persistencia.** Refresh = carrito vacío. Migrar
    a `useReducer` + `localStorage` (o Zustand/Jotai si crece).
13. **Tests escasos.** Solo `App.test.js` y `BasenameRouting.test.js`. Faltan
    unit tests de `validation.js`, `normalizeProducts.js`, `useInitialState`,
    `useGetProducts` (axios mock) y los formularios.
14. **Sin CI.** No hay workflows de GitHub Actions. Añadir uno que corra
    `lint + test + build` en cada PR.
15. **Imágenes sin `loading="lazy"` ni `width/height`** → CLS y LCP afectados.
    Existe un fallback (`pexels-photo-260024.webp`) que se podría usar también
    con `onError`.
16. **`reportWebVitals()` invocado sin callback** → no mide nada.
17. **`useGetProducts` depende de la referencia `source`** completa. Funciona
    porque `productsSource` es un singleton de módulo, pero es frágil. Mejor
    desestructurar `endpoint`/`timeoutMs` en las deps.
18. Imágenes externas hot-linked desde `purecannastore.com` sin
    `referrerpolicy="no-referrer"`.

### 🟢 Nice-to-have
- TypeScript progresivo (empezar por `utils/` y `hooks/`).
- `"engines": { "node": ">=18" }` en `package.json`.
- Husky + lint-staged en pre-commit.
- `.prettierrc` y `.eslintrc` propios para unificar estilo.

## 8. Reglas para Claude

Al trabajar en este repo:

1. **Antes de proponer cambios**, lee este archivo y los archivos relevantes
   en `src/`. La estructura es pequeña, no hace falta delegar a agentes salvo
   que la tarea cruce ≥ 5 archivos.
2. **Usa siempre `APP_ROUTES`** (no strings literales) para navegación.
3. **Usa siempre `UI_TEXT`** para copy visible al usuario.
4. **Pasa datos del catálogo** por `normalizeProduct(s)` antes de renderizar.
5. **Para forms de auth** reutiliza `isValidEmail` / `isValidPassword`.
6. **Nunca añadas `console.log` con credenciales** ni con PII.
7. **No introduzcas dependencias nuevas** sin necesidad. Si la tarea no lo
   exige, mantén el stack actual.
8. **No toques `node_modules/` ni `build/`** ni los versiones.
9. **Antes de "completar" una tarea** que toca código, corre mentalmente:
   `npm run lint`, `npm test`, `npm run build`. Si algo se rompería, repórtalo.
10. **Indentación:** respeta el estilo del archivo que estás editando hasta
    que se introduzca Prettier. No conviertas tabs a espacios (ni viceversa)
    en archivos no relacionados con tu cambio.
11. **PRs pequeños y enfocados.** Si la tarea pide algo grande (ej. migrar a
    Vite, añadir auth real), divide en pasos y confirma con el usuario antes
    de empezar.

## 9. Roadmap sugerido (orden recomendado)

1. Limpieza de seguridad inmediata (§7 críticos 1–4).
2. Reparar botones rotos y aplicar `NavLayout` a las páginas restantes (§7 #6, #7).
3. Implementar `AuthContext` mock + `<RequireAuth>` (§7 #5).
4. Code splitting con `React.lazy` (§7 #10).
5. Persistir el carrito en `localStorage` con `useReducer` (§7 #12).
6. Ampliar cobertura de tests (§7 #13).
7. Configurar CI con GitHub Actions (§7 #14).
8. Migrar a Vite (§7 #8).
9. Introducir TypeScript de forma incremental.

## 10. Plan de implementación por sprints

> Cada tarea trae **nombre de commit sugerido** (Conventional Commits). Los
> sprints están pensados para 1 dev a tiempo parcial; ajusta la duración si
> trabajan más personas. Cada sprint debería terminar con `lint + test + build`
> en verde y un deploy verificado.

---

### Sprint 1 — Hotfix de seguridad (1–2 días) 🔴

**Objetivo:** cerrar fugas obvias antes del próximo deploy. Bloqueante.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 1.1 | Quitar log de credenciales | Borrar `console.log({ password })` / `console.log(data)` en `Login.jsx`, `CreateAccount.jsx`, `NewPassword.jsx`. Reemplazar por un `// TODO: wire to auth service`. | `fix(security): remove credential logging from auth forms` |
| 1.2 | Limpiar PII hardcodeada | Sustituir "Jonah Sly" / "slyjonatan@gmail.com" en `MyAccount.jsx` por placeholders neutros o por datos derivados del (futuro) `AuthContext`. | `fix(security): remove hardcoded PII from MyAccount` |
| 1.3 | Bump de axios | `npm install axios@^1.7.4` y verificar que `useGetProducts` siga funcionando. Correr `npm audit` y revisar reporte. | `chore(deps): bump axios to ^1.7.4 to patch CVEs` |
| 1.4 | `.gitignore` y purga de `build/` | Confirmar que `build/` está ignorado; si está versionada, `git rm -r --cached build/` y commitear. | `chore: ignore build artifacts and untrack build/` |
| 1.5 | `npm audit fix` | Resolver vulnerabilidades adicionales que se puedan arreglar sin major bumps. | `chore(deps): npm audit fix for non-breaking advisories` |

**Definition of Done:** ningún `console.log` con credenciales, `npm audit` solo reporta avisos transitivos no resolubles (documentados), `build/` no aparece en `git status`.

---

### Sprint 2 — Reparar UX rota (3–5 días) 🟠

**Objetivo:** que ningún botón visible quede muerto y todas las páginas tengan navegación.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 2.1 | Botón "Sign up" funcional | En `Login.jsx`, envolver el botón en `<Link to={APP_ROUTES.CREATE_ACCOUNT}>` o usar `useNavigate`. | `fix(login): wire sign up button to create-account route` |
| 2.2 | Botón "Checkout" del carrito | En `MyOrder.jsx`, `onClick` con `useNavigate(APP_ROUTES.CHECKOUT)`. | `fix(cart): navigate to checkout from MyOrder button` |
| 2.3 | Form de MyAccount sin reload | Sustituir `<form action="/">` por `<form onSubmit>` con `event.preventDefault()`; el botón "Edit" debe abrir un modo edición o navegar. | `fix(account): prevent default submit in MyAccount form` |
| 2.4 | Toggle de menú descubrible | Mover el `onClick` del email al icono `imenu` (o ambos). Añadir `aria-expanded` y `aria-controls`. | `fix(header): make menu icon the primary toggle` |
| 2.5 | Decidir destino de categorías | Opción A: implementar filtro vía query string (`?category=Flowers`) en `Home`. Opción B: ocultar las categorías hasta tener routing real. | `feat(catalog): filter products by category from header` *o* `chore(header): hide non-functional category links` |
| 2.6 | Layout consistente | Envolver `MyAccount`, `Orders`, `Checkout` en `<NavLayout>`. Considerar un `<AuthLayout>` separado para Login/CreateAccount/PasswordRecovery. | `refactor(layout): apply NavLayout to account, orders, checkout` |

**DoD:** todos los botones del Header, MyOrder y MyAccount producen una acción observable; el usuario nunca queda atrapado sin Header en páginas internas.

---

### Sprint 3 — Auth mock + rutas protegidas (1 semana) 🟠

**Objetivo:** simular autenticación real (sin backend) para poder proteger rutas privadas y eliminar la sensación de mock total.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 3.1 | `AuthContext` | Crear `src/contexts/AuthContext.js` con `{ user, login, logout, isAuthenticated }`. Persistir el "mock token" en `sessionStorage` (no `localStorage`, para reducir XSS). | `feat(auth): add AuthContext with mock login/logout` |
| 3.2 | Hook `useAuth` | `src/hooks/useAuth.js` que envuelva `useContext(AuthContext)` y lance error si se usa fuera del Provider. | `feat(auth): expose useAuth hook with provider guard` |
| 3.3 | Conectar `Login` y `CreateAccount` | Reemplazar `console.log(data)` por `auth.login(email, password)`; en éxito, `navigate(APP_ROUTES.HOME)`. | `feat(auth): wire login and create-account forms to AuthContext` |
| 3.4 | `<RequireAuth>` wrapper | Componente que redirige a `/login` si `!isAuthenticated`, preservando `from` en `state` para volver tras login. | `feat(auth): add RequireAuth route guard with return url` |
| 3.5 | Proteger rutas privadas | Envolver `MY_ACCOUNT`, `ORDERS`, `CHECKOUT` en `<RequireAuth>` dentro de `App.jsx`. | `feat(auth): protect private routes with RequireAuth` |
| 3.6 | Sign out funcional | El "Sign out" del `Menu` llama a `auth.logout()` y navega a `/login`. | `feat(auth): wire sign out menu item to logout` |
| 3.7 | Mostrar usuario real en MyAccount | Sustituir placeholders del Sprint 1 por `user.name` / `user.email`. | `feat(account): render authenticated user data in MyAccount` |

**DoD:** un usuario sin sesión es redirigido a `/login` al pedir `/my-account`, `/orders` o `/checkout`; tras login vuelve a la ruta solicitada.

---

### Sprint 4 — Tests + CI (1 semana) 🟡

**Objetivo:** elevar la cobertura y bloquear regresiones automáticamente.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 4.1 | Tests unit de `validation.js` | Casos: emails válidos/ inválidos, password < 8, password con espacios. | `test(utils): cover isValidEmail and isValidPassword` |
| 4.2 | Tests unit de `normalizeProducts.js` | Producto sin id/title/price, `images` no array, lista vacía, `null`. | `test(utils): cover normalizeProduct(s) edge cases` |
| 4.3 | Tests de `useInitialState` | Add → quantity++, remove → quantity-- y elimina al llegar a 0, ítems independientes por id. | `test(hooks): cover cart add/remove logic` |
| 4.4 | Tests de `useGetProducts` | Mock de `axios`, casos: éxito, 500, abort, timeout. | `test(hooks): cover useGetProducts success and error paths` |
| 4.5 | Tests de formularios de auth | RTL: input inválido muestra error, submit válido invoca `auth.login` (mock). | `test(auth): cover Login and CreateAccount form behavior` |
| 4.6 | Tests de `RequireAuth` | Usuario no auth → redirect; auth → renderiza children. | `test(auth): cover RequireAuth redirect behavior` |
| 4.7 | Workflow CI | `.github/workflows/ci.yml`: matrix Node 18/20, pasos `npm ci → lint → test --watchAll=false --coverage → build`. | `ci: add GitHub Actions workflow for lint, test and build` |
| 4.8 | Badge de CI en README | Añadir badge del workflow en el README del repo. | `docs(readme): add CI status badge` |

**DoD:** coverage ≥ 70% en `utils/` y `hooks/`; el workflow corre en cada PR y bloquea merges en rojo.

---

### Sprint 5 — Performance + accesibilidad (1 semana) 🟡

**Objetivo:** mejorar Web Vitals y hacer la app utilizable con teclado/lector de pantalla.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 5.1 | Code splitting por ruta | Convertir las páginas en `React.lazy` y envolver `<Routes>` en `<Suspense fallback>`. | `perf(app): code-split routes with React.lazy and Suspense` |
| 5.2 | Memoizar `ProductItem` | Envolver con `React.memo`; verificar igualdad por `product.id`. | `perf(catalog): memoize ProductItem to avoid re-renders` |
| 5.3 | `useMemo` en totales | `sumTotal`, `articleCount`, `cartItemsCount` derivados con `useMemo` sobre `state.cart`. | `perf(cart): memoize cart totals and counts` |
| 5.4 | Imágenes con `loading="lazy"` | Añadir a `ProductItem`, `OrderItem` y assets externas. Definir `width/height` para evitar CLS. | `perf(images): add lazy loading and intrinsic size` |
| 5.5 | `onError` con fallback | Imágenes externas caen a `pexels-photo-260024.webp` si fallan. | `fix(images): swap to fallback image on load error` |
| 5.6 | `referrerpolicy` en hot-links | `referrerpolicy="no-referrer"` en `<img>` que apuntan a `purecannastore.com`. | `fix(images): set no-referrer policy on external images` |
| 5.7 | Botones accesibles | Sustituir `<img onClick>` clicables por `<button type="button">` con icono dentro (cart, close, add-to-cart, menu). | `a11y(ui): replace clickable images with buttons` |
| 5.8 | `aria-live` en errores | `<p className="form-error" role="alert" aria-live="polite">` en todos los formularios. | `a11y(forms): announce form errors with aria-live` |
| 5.9 | `reportWebVitals` útil | `reportWebVitals(console.log)` o un endpoint mock. | `chore(perf): wire reportWebVitals to console in dev` |
| 5.10 | Fix deps de `useGetProducts` | Desestructurar `endpoint`/`timeoutMs` en las deps del `useEffect`. | `refactor(hooks): use stable dependencies in useGetProducts` |

**DoD:** Lighthouse Performance ≥ 85 y Accessibility ≥ 95 en build de producción.

---

### Sprint 6 — Persistencia del carrito (3–5 días) 🟡

**Objetivo:** que el carrito sobreviva a un refresh.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 6.1 | Migrar a `useReducer` | Reescribir `useInitialState` con un reducer (`ADD_TO_CART`, `REMOVE_FROM_CART`, `CLEAR_CART`, `HYDRATE`). | `refactor(cart): migrate state to useReducer` |
| 6.2 | Hook `useLocalStorage` | Pequeño hook genérico `[value, setValue]` con JSON parse/stringify y try/catch. | `feat(hooks): add useLocalStorage helper` |
| 6.3 | Hidratar carrito al montar | Al inicio, despachar `HYDRATE` con lo guardado; suscribir un `useEffect` que persista cambios. | `feat(cart): persist cart in localStorage` |
| 6.4 | Acción `CLEAR_CART` tras checkout | Limpiar carrito al confirmar pedido (cuando exista flujo). | `feat(checkout): clear cart on order confirmation` |
| 6.5 | Tests de persistencia | RTL + `localStorage` mock: hidratación inicial, persistencia tras add/remove. | `test(cart): cover localStorage hydration and persistence` |

**DoD:** un refresh con productos en el carrito no pierde el estado; el `localStorage` solo guarda el carrito (no PII).

---

### Sprint 7 — Tooling, formato y consistencia (3–5 días) 🟢

**Objetivo:** unificar estilo y prevenir regresiones de formato.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 7.1 | `.prettierrc` | Añadir Prettier con `printWidth: 100`, `singleQuote: true`, `trailingComma: 'es5'`, `tabWidth: 2`, `useTabs: false`. | `chore(tooling): add Prettier configuration` |
| 7.2 | `.eslintrc` propio | Extender de `react-app` + `plugin:jsx-a11y/recommended` + `prettier`. | `chore(tooling): add standalone eslint configuration` |
| 7.3 | Reformatear todo | Un commit aislado de `prettier --write src/` para no contaminar diffs futuros. Ignorar en `git blame` con `.git-blame-ignore-revs`. | `style: apply Prettier across src/` |
| 7.4 | Husky + lint-staged | Pre-commit que corre `eslint --fix` + `prettier --write` sobre archivos staged. | `chore(tooling): add Husky and lint-staged pre-commit hook` |
| 7.5 | `engines` en package.json | `"engines": { "node": ">=18" }` y `.nvmrc`. | `chore: declare Node engine and add .nvmrc` |

**DoD:** `npm run format` deja `git status` limpio; el pre-commit bloquea código no formateado.

---

### Sprint 8 — Migración a Vite (1 semana) 🟠

**Objetivo:** salir de CRA (deprecado) sin romper el deploy a GH Pages.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 8.1 | Branch de migración | Trabajar en `feat/vite-migration`; mantener `main` deployable. | (rama, sin commit) |
| 8.2 | Instalar Vite | `npm install -D vite @vitejs/plugin-react`; eliminar `react-scripts`. | `build: replace CRA with Vite and React plugin` |
| 8.3 | `vite.config.js` | `base: '/FarmLabs/'`, plugin React, alias si se desea. | `build(vite): configure base path for GitHub Pages` |
| 8.4 | Mover `index.html` | A la raíz, con `<script type="module" src="/src/index.jsx">`; renombrar `index.js` → `index.jsx`. | `build(vite): move index.html to project root` |
| 8.5 | Variables de entorno | Reemplazar `process.env.PUBLIC_URL` por `import.meta.env.BASE_URL` en `productsSource.js` y `App.jsx`. | `refactor(config): replace PUBLIC_URL with import.meta.env.BASE_URL` |
| 8.6 | Migrar tests | Cambiar Jest por **Vitest** (`npm install -D vitest jsdom @testing-library/jest-dom`); ajustar `setupTests` y mocks. | `test(vitest): migrate test runner from Jest to Vitest` |
| 8.7 | Scripts `package.json` | `dev: vite`, `build: vite build`, `preview: vite preview`, `test: vitest`. | `chore(scripts): update npm scripts for Vite` |
| 8.8 | Verificar deploy | `npm run build` → `gh-pages -d dist` (no `build`); ajustar script `deploy`. | `build(deploy): point gh-pages to Vite dist folder` |
| 8.9 | Smoke test post-deploy | Verificar todas las rutas en `https://jonahsly.github.io/FarmLabs/` (Home, Login, /unknown). | `docs: post-migration smoke test checklist` |

**DoD:** `npm run dev` arranca en < 2s, el bundle de producción se sirve correctamente bajo `/FarmLabs/`, todas las rutas (incluida 404) funcionan.

---

### Sprint 9 — TypeScript incremental (2 semanas) 🟢

**Objetivo:** introducir tipos sin big-bang. Cada paso debe seguir compilando.

| # | Tarea | Qué hacer | Commit |
|---|-------|-----------|--------|
| 9.1 | `tsconfig.json` permisivo | `allowJs: true`, `strict: false`, `jsx: react-jsx`. Instalar `typescript`, `@types/react`, `@types/react-dom`, `@types/node`. | `chore(ts): add TypeScript with allowJs configuration` |
| 9.2 | Migrar `utils/` | `validation.ts`, `normalizeProducts.ts` con tipos `Product` exportados. | `refactor(ts): convert utils to TypeScript` |
| 9.3 | Migrar `constants/` y `config/` | `routes.ts` con `as const`, `uiText.ts` tipado, `appConfig.ts`, `productsSource.ts`. | `refactor(ts): convert constants and config to TypeScript` |
| 9.4 | Migrar `hooks/` | `useInitialState.ts`, `useGetProducts.ts`, `useAuth.ts`, `useLocalStorage.ts`. | `refactor(ts): convert hooks to TypeScript` |
| 9.5 | Migrar `contexts/` | `AppContext.ts`, `AuthContext.ts` con tipos del valor por defecto. | `refactor(ts): convert contexts to TypeScript` |
| 9.6 | Migrar `components/` | Componentes pequeños: `ProductItem.tsx`, `OrderItem.tsx`, `Header.tsx`, `Menu.tsx`. | `refactor(ts): convert presentational components to TypeScript` |
| 9.7 | Migrar `containers/` | `NavLayout.tsx`, `MyOrder.tsx`, `ProductList.tsx`. | `refactor(ts): convert containers to TypeScript` |
| 9.8 | Migrar `pages/` | Una por commit pequeño para revisar bien props y handlers. | `refactor(ts): convert <Page> to TypeScript` (× N) |
| 9.9 | Endurecer `tsconfig` | Activar `strict: true` y arreglar lo que salte. | `chore(ts): enable strict mode and fix type errors` |

**DoD:** `tsc --noEmit` pasa en CI; no quedan archivos `.jsx` o `.js` en `src/` (excepto setup de tests si Vitest lo requiere).

---

### Resumen visual del plan

| Sprint | Tema | Duración | Riesgo si no se hace |
|--------|------|----------|----------------------|
| 1 | Hotfix de seguridad | 1–2 días | Fugas de credenciales/PII en producción |
| 2 | Reparar UX rota | 3–5 días | Usuarios atrapados, botones muertos |
| 3 | Auth mock + protected routes | 1 sem | Páginas privadas accesibles a cualquiera |
| 4 | Tests + CI | 1 sem | Regresiones invisibles |
| 5 | Performance + a11y | 1 sem | Bundle pesado, app inutilizable con teclado |
| 6 | Persistencia del carrito | 3–5 días | UX pobre tras refresh |
| 7 | Tooling y formato | 3–5 días | Diff noise, drift de estilo |
| 8 | Migración a Vite | 1 sem | Stack deprecado, audits crónicos |
| 9 | TypeScript incremental | 2 sem | Bugs por tipos, refactors caros |

**Total estimado:** ~7–9 semanas a tiempo parcial. Los sprints 1–3 son
imprescindibles antes de mostrar el portafolio; 4–6 lo elevan a producto;
7–9 son inversión de mantenibilidad a largo plazo.

## 11. Agente y skills del repositorio

Este repo expone **un único agente** (`.claude/agents/farmlabs.md`) que actúa como
punto de entrada para todo el trabajo. La especialización vive en **skills**
(`.claude/skills/<nombre>/SKILL.md`) que el agente invoca según el tipo de
petición.

### Agente

| Nombre | Rol | Tools |
|--------|-----|-------|
| `farmlabs` | Punto de entrada único. Lee `CLAUDE.md`, clasifica la petición y delega en la skill correcta. Verifica el DoD antes de declarar done. | `Read, Edit, Write, Grep, Glob, Bash, Skill` |

### Skills

Las skills se dividen en **workflow** (acción) y **domain** (conocimiento). El
agente combina ambas: un workflow para qué hacer + una o más domain skills para
cómo hacerlo en este proyecto.

#### Workflow skills

| Skill | Cuándo se invoca | Qué hace |
|-------|------------------|----------|
| `execute-sprint-task` | "ejecuta 1.3", "haz la tarea 2.4", "implementa el sprint 1", "arregla X" donde X está en §10 | Localiza la fila de §10, edita los archivos respetando convenciones, devuelve resumen + commit message exacto. |
| `audit-security` | "audita el repo", "busca fugas", "revisa CVEs", "antes de deploy" | Recorre el checklist de 10 puntos (credenciales, PII, CVEs, XSS, tokens, hot-linking, secrets, .gitignore…) y emite un reporte clasificado por severidad. **Solo lee.** |
| `verify-quality-gates` | Tras `execute-sprint-task`, antes de declarar done, o por petición explícita del usuario | Corre `npm run lint`, `npm test`, `npm run build` y reporta veredicto (🟢/🟡/🔴). |

#### Domain skills — frontend

| Skill | Cuándo consultarla | Qué cubre |
|-------|---------------------|-----------|
| `frontend-component-author` | Cualquier tarea que toque `src/components`, `src/containers`, `src/pages`, `src/hooks`, `src/contexts` | Convenciones del repo (`APP_ROUTES`, `UI_TEXT`, `normalizeProduct`), patrón de fetch con `AbortController`, patrón de form de auth, memoización, lazy loading, anti-patrones |
| `frontend-accessibility` | Sprint 2 (UX rota), Sprint 5.7/5.8 (a11y), o petición explícita "haz esto accesible" | Botones reales vs `<img onClick>`, `aria-live` en errores, `aria-expanded` en toggles, etiquetas, navegación por teclado, contraste, semántica HTML |
| `frontend-performance` | Sprint 5 completo (5.1–5.10) | Code splitting con `React.lazy` + `Suspense`, `React.memo`/`useMemo`/`useCallback`, imágenes (`loading="lazy"`, `width/height`, `referrerPolicy`, `onError`), `reportWebVitals`, hook deps estables |
| `frontend-testing` | Sprint 4 completo (4.1–4.8); también si una tarea rompe los tests existentes | Patrones de RTL/Jest siguiendo `App.test.js` (mock de hooks, queries semánticas, cobertura), tests para utils/hooks/forms, migración a Vitest en Sprint 8.6 |

#### Domain skills — backend (forward-looking)

> El repo aún no tiene backend. Estas skills se activan cuando se vaya más allá
> del mock de Sprint 3 o se reemplace `public/data/products.json` por una API
> real. **No invocar preventivamente.**

| Skill | Cuándo consultarla | Qué cubre |
|-------|---------------------|-----------|
| `backend-api-design` | "diseña el endpoint X", planificar reemplazo del JSON estático, evolución de auth a backend real | Convenciones REST, status codes, shape de respuesta, paginación/filtros, idempotencia, validación con `zod`, OpenAPI, CORS, rate limiting, endpoints sugeridos para FarmLabs |
| `backend-auth` | Cuando el `AuthContext` mock de Sprint 3 se conecte a un backend real | Access + refresh tokens, cookies `HttpOnly` + `SameSite` + `Secure`, hashing con `argon2id`, CSRF, password reset, OAuth, qué meter (y qué no) en JWT |
| `backend-data` | Modelar schema, migraciones, queries, índices (mover catálogo a DB, persistir órdenes/usuarios) | Postgres vs SQLite, ORMs (Drizzle/Prisma), schema mínimo para FarmLabs (users, sessions, products, orders), migraciones reversibles, seed data, prevención de N+1, transacciones, backups |

### Mapeo sprint → domain skills

| Sprint | Domain skills relevantes |
|--------|--------------------------|
| 1. Hotfix seguridad | `audit-security` para verificar antes de cerrar |
| 2. UX rota | `frontend-component-author` + `frontend-accessibility` |
| 3. Auth mock | `frontend-component-author`. Si evoluciona a backend real → `backend-auth` |
| 4. Tests + CI | `frontend-testing` |
| 5. Perf + a11y | `frontend-performance` + `frontend-accessibility` |
| 6. Carrito persistente | `frontend-component-author` (state management) |
| 7. Tooling/Prettier | — |
| 8. Vite | `frontend-testing` (ajustes para Vitest) |
| 9. TypeScript | `frontend-component-author` (props/types) |

### Flujo recomendado

```
Usuario → farmlabs → clasifica intención
                 │
                 ├─ "implementa 5.1"  → execute-sprint-task
                 │                       ↳ consulta frontend-performance
                 │                       ↳ consulta frontend-accessibility (si aplica)
                 │                     → verify-quality-gates
                 │                     → reporte
                 │
                 ├─ "audita el repo"  → audit-security
                 │                     → reporte
                 │
                 ├─ "verifica"        → verify-quality-gates
                 │                     → reporte
                 │
                 └─ "diseña la API"   → execute-sprint-task (si está en §10)
                                        ↳ consulta backend-api-design
                                        ↳ consulta backend-auth y/o backend-data
                                      → reporte
```

### Cómo invocar al agente

- **Implícita:** cualquier petición sobre el código del repo entra por `farmlabs` automáticamente (Claude Code matchea por descripción).
- **Explícita:** "@farmlabs ejecuta la tarea 1.3" o "usa el agente farmlabs para auditar".

### Reglas para extender este sistema

1. Una skill nueva se añade como carpeta en `.claude/skills/<nombre>/SKILL.md`
   con su propio frontmatter `name` + `description` (descriptor que sirve de
   trigger).
2. Decide si es **workflow** (acción end-to-end) o **domain** (conocimiento
   reutilizable). Frontend y backend van en domain.
3. El agente `farmlabs` se actualiza para incluir la skill en su tabla de
   clasificación y en el mapeo sprint → domain.
4. La nueva skill se documenta en esta sección §11.
5. **No se crean agentes adicionales.** Si la complejidad pide separación,
   se separa por skills, no por agentes.
6. Las domain skills **no editan archivos por sí solas**: aportan patrones que
   `execute-sprint-task` aplica. Esto mantiene el flujo predecible.

---

_Última actualización: 2026-05-08. Mantén esta fecha al editar el archivo._
