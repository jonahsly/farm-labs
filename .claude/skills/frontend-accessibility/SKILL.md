---
name: frontend-accessibility
description: Checklist y patrones de accesibilidad (WCAG 2.1 AA) aplicados a FarmLabs. Detecta y corrige imágenes clicables sin <button>, formularios sin aria-live, toggles sin aria-expanded, falta de etiquetas, navegación por teclado rota y contraste insuficiente. Úsalo cuando una tarea de §10 toque UI interactiva — especialmente Sprints 2 (UX rota), 5.7/5.8 (a11y), o cuando el usuario pida explícitamente "revisa accesibilidad", "haz esto accesible", "soporta teclado". Aporta patrones; el flujo execute-sprint-task aplica los cambios.
---

# Skill: frontend-accessibility

Aporta los patrones de accesibilidad para FarmLabs. **No actúa sola** — la invoca el agente `farmlabs` junto con `execute-sprint-task` cuando la tarea afecta UI interactiva.

## Checklist mínimo (WCAG 2.1 AA aplicado al repo)

### 1. Botones reales para acciones (Sprint 5.7) 🔴

❌ Mal:
```jsx
<img src={icart} onClick={toggleCart} alt="shopping cart" />
```

✅ Bien:
```jsx
<button
  type="button"
  className="navbar-shopping-cart"
  aria-label={UI_TEXT.header.cartLabel}
  onClick={toggleCart}
>
  <img src={icart} alt="" aria-hidden="true" />
  {cartItemsCount > 0 && <span>{cartItemsCount}</span>}
</button>
```

Aplica a: cart toggle, close en `OrderItem`, add-to-cart en `ProductItem`, menu icon en `Header`.

### 2. Form errors anunciados (Sprint 5.8) 🟠

❌ Mal:
```jsx
{error ? <p className="form-error">{error}</p> : null}
```

✅ Bien:
```jsx
<p
  className="form-error"
  role="alert"
  aria-live="polite"
>
  {error}
</p>
```

Renderiza el `<p>` siempre (vacío si no hay error) para que `aria-live` lo anuncie cuando aparezca texto. Aplica a: `Login`, `CreateAccount`, `NewPassword`, `PasswordRecovery`.

### 3. Toggles con estado expuesto (Sprint 2.4) 🟠

```jsx
<button
  type="button"
  aria-expanded={toggle}
  aria-controls="user-menu"
  onClick={handleToggle}
>
  <img src={imenu} alt="" aria-hidden="true" />
</button>
{toggle && <Menu id="user-menu" />}
```

### 4. Etiquetas de formulario

✅ Bien (ya en uso en el repo):
```jsx
<label htmlFor="email">{UI_TEXT.auth.emailAddressLabel}</label>
<input type="email" id="email" name="email" required />
```

NUNCA uses `placeholder` como sustituto de `<label>`.

### 5. Texto alternativo de imágenes

- Imágenes **decorativas** (icono dentro de un botón con label) → `alt=""` + `aria-hidden="true"`.
- Imágenes **informativas** (foto del producto) → `alt={productTitle}`.
- Imágenes **clicables** → mejor convertir el padre en `<button>` con `aria-label` y poner `alt=""` en el `<img>`.

### 6. Navegación por teclado

Verificar manualmente:
- `Tab` recorre todos los controles en orden visual.
- `Enter` y `Space` activan botones.
- `Esc` cierra menús/diálogos abiertos.
- `Tab` no entra en elementos `display: none` o `visibility: hidden` ✅ (lo asegura el navegador).
- Focus visible en todo elemento interactivo (no `outline: none` sin alternativa).

### 7. Contraste de color

- Texto normal: ratio ≥ 4.5:1.
- Texto grande (≥ 18pt o 14pt bold): ratio ≥ 3:1.
- Iconos accionables: ratio ≥ 3:1.

Revisa `src/styles/vars.css` y los CSS por componente. Si el repo introduce un `--text-on-dark` con #999 sobre #000, está roto.

### 8. Semántica HTML

- Una sola `<h1>` por página (la del `title` principal).
- `<nav>`, `<main>`, `<aside>`, `<section>` con encabezado.
- `<button>` para acciones; `<a>` (`<Link>`) para navegación.
- Listas con `<ul>`/`<ol>` cuando son listas, no `<div>` apilados.

### 9. Estados loading/empty/error

Anuncia con `role="status"` o `aria-live="polite"`:
```jsx
<p className="empty-state" role="status">{UI_TEXT.catalog.loading}</p>
```

### 10. Imágenes con dimensiones (Sprint 5.4) 🟡

`width` y `height` en HTML evitan CLS y permiten al navegador reservar espacio antes de cargar. Aplica también a `loading="lazy"` (ver `frontend-performance`).

## Anti-patrones

- `<div onClick>` o `<img onClick>` para acciones.
- `placeholder` como única etiqueta.
- `outline: none` global sin estilo de focus alternativo.
- `tabIndex="-1"` en elementos interactivos visibles.
- Anunciar errores solo visualmente (sin `aria-live`).
- `alt="image"` o `alt="logo"` (redundante con el contexto).
- Usar emojis como iconos accionables sin texto alternativo.

## Smoke test rápido

Tras un cambio, comprueba:

1. Recorre la página solo con `Tab` desde la URL bar — ¿llegas a todo?
2. Activa cada control con `Enter` y `Space`.
3. Cierra un menú con `Esc`.
4. Revisa en DevTools → Accessibility tree que los botones tengan accessible name.
5. Lighthouse Accessibility ≥ 95 (objetivo del Sprint 5).

## Referencias cruzadas

- Convenciones generales React → `frontend-component-author`.
- Performance/imágenes → `frontend-performance` (Sprint 5.4–5.6).
- Tests de a11y → `frontend-testing` (queries `getByRole`, `getByLabelText`).
