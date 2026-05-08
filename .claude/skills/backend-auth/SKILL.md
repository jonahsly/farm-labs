---
name: backend-auth
description: Patrones de autenticación server-side para FarmLabs cuando se vaya más allá del mock del Sprint 3. Cubre access + refresh tokens, cookies HttpOnly + SameSite + Secure, hashing de contraseñas con bcrypt/argon2, CSRF, rate limiting en login, password reset y cierre de sesiones. Forward-looking: complementa al frontend-component-author del Sprint 3 cuando exista un backend real. Úsalo cuando el usuario pida "implementa auth real", "JWT", "OAuth", "session", o cuando el AuthContext mock se conecte a un backend.
---

# Skill: backend-auth

Aporta los patrones de autenticación server-side. **Forward-looking**: hoy FarmLabs solo tiene un `AuthContext` mock (Sprint 3). Esta skill se activa cuando:

- Se construye un backend real que valide credenciales.
- Se reemplaza el `sessionStorage` del Sprint 3 por cookies `HttpOnly`.
- Se añaden flujos de refresh token, password reset, OAuth, etc.

## Decisión central: cookies vs. localStorage

Para una SPA con backend propio:

| Opción | Pros | Cons | Recomendación |
|--------|------|------|---------------|
| **Cookies HttpOnly + SameSite=Lax + Secure** | Inmunes a XSS leyendo el token; el navegador las envía solas | Requieren CSRF protection en POSTs cross-site | ✅ Default |
| `localStorage` con JWT | Fácil; el front controla el flujo | XSS puede leer el token; no se envía solo | ❌ Evitar |
| `sessionStorage` | Solo vive en pestaña | Mismas vulnerabilidades XSS que `localStorage` | Mock del Sprint 3 |

**Decisión por defecto:** access token corto + refresh token, ambos en cookies HttpOnly.

## Esquema de tokens

```
┌─────────────┐  Login (email, password)            ┌──────────────┐
│   Cliente   │  ─────────────────────────────────▶ │   Servidor   │
│             │                                     │              │
│             │  Set-Cookie: access=<JWT 15min>     │              │
│             │  Set-Cookie: refresh=<opaque 30d>   │              │
│             │  ◀───────────────────────────────── │              │
│             │                                     │              │
│             │  GET /v1/users/me  (cookies)        │              │
│             │  ─────────────────────────────────▶ │              │
│             │  ◀───────────────────────────────── │              │
│             │                                     │              │
│             │  POST /v1/auth/refresh (cookies)    │              │
│             │  ─────────────────────────────────▶ │              │
│             │  rotación: nuevos access+refresh    │              │
│             │  ◀───────────────────────────────── │              │
└─────────────┘                                     └──────────────┘
```

- **Access token:** JWT firmado, `exp` 10–15 minutos. Contiene `sub` (userId), `role`, `iat`, `exp`. Sin PII.
- **Refresh token:** opaco (random 256 bits), almacenado en DB con expiración 7–30 días, vinculado a un `sessionId`. Permite revocación.
- **Rotación:** cada `/refresh` invalida el refresh anterior y emite uno nuevo. Detecta reuse → revoca toda la familia.

## Cookies — flags obligatorios

```javascript
reply.setCookie('access', accessJwt, {
  httpOnly: true,
  secure: true,             // HTTPS only en prod
  sameSite: 'lax',          // 'strict' si no necesitas navegación cross-site con cookies
  path: '/',
  maxAge: 60 * 15,          // 15 min
});

reply.setCookie('refresh', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  path: '/v1/auth',         // limita el scope
  maxAge: 60 * 60 * 24 * 30,
});
```

⚠️ En desarrollo local con `http://localhost`, `secure: true` rompe el set. Usa `secure: process.env.NODE_ENV === 'production'`.

## CSRF

Con `SameSite=Lax`, los POSTs cross-site no envían cookies por defecto, lo que mitiga CSRF. Para defensa en profundidad:

- **Double-submit cookie**: emite también un `csrf-token` no-HttpOnly que el cliente lee y envía en header `X-CSRF-Token`. El servidor verifica que ambos coincidan.
- O **token de sesión**: el endpoint `/auth/me` devuelve un token CSRF que el frontend mete en cada mutación.

## Hashing de contraseñas

Usa **argon2id** (preferido) o **bcrypt** (rounds ≥ 12).

```javascript
import argon2 from 'argon2';

const hash = await argon2.hash(plainPassword, {
  type: argon2.argon2id,
  memoryCost: 19456,  // 19 MB
  timeCost: 2,
  parallelism: 1,
});

// Verificación:
const ok = await argon2.verify(hash, candidatePassword);
```

**NUNCA almacenes la contraseña en plano**, hasheada-pero-no-salada, ni con MD5/SHA1.

## Rate limiting (anti brute force)

```javascript
fastify.register(import('@fastify/rate-limit'), {
  max: 5,
  timeWindow: '15 minutes',
  keyGenerator: (req) => `${req.ip}:${req.body?.email || ''}`,
});
```

Aplicar a:
- `POST /v1/auth/login`
- `POST /v1/auth/register`
- `POST /v1/auth/password-reset`

Combina con account lockout temporal (5 intentos fallidos → bloqueo 15 min) **persistido en DB**, no in-memory.

## Password reset

1. `POST /v1/auth/password-reset/request` con `{ email }`. Responde **siempre 204** (no filtres si el email existe).
2. Si el email existe: emite token aleatorio (256 bits), guarda hash + `expiresAt` (15 min) + `userId`. Envía link `https://app/.../reset?token=...`.
3. `POST /v1/auth/password-reset/confirm` con `{ token, newPassword }`. Verifica hash + expiración, hashea la nueva password, **invalida todas las sesiones del usuario**.
4. Tokens de un solo uso: marca como `used` o bórralo tras consumirlo.

## Logout

- `POST /v1/auth/logout`: limpia cookies (`maxAge: 0`), revoca el refresh token actual.
- `POST /v1/auth/logout-all`: revoca todos los refresh tokens del usuario.

## OAuth / Social login (opcional)

Si añades Google/Apple/GitHub:
- Usa **Authorization Code Flow + PKCE**. Nunca Implicit.
- El frontend redirige a `/v1/auth/oauth/<provider>` que hace el redirect server-side.
- El callback `/v1/auth/oauth/<provider>/callback` valida el code, obtiene el ID token, vincula o crea cuenta y emite los tokens del propio sistema (no uses el de Google directamente).

## JWT — qué meter y qué NO

✅ Sí:
- `sub` (userId).
- `role` o `permissions` (si caben en pocos bytes).
- `sid` (sessionId opaco para invalidación rápida).
- `iat`, `exp`.

❌ No:
- Email.
- Nombre.
- Datos de pago.
- Cualquier PII.

Firma con **RS256** (par de claves) o **EdDSA**, no `HS256` con secret compartido en repo. Las claves se cargan de variables de entorno o secret manager.

## Logging

NUNCA loguees:
- `Authorization` headers.
- `Cookie` headers.
- `password`, `passwordConfirm`, `currentPassword`, `newPassword`.
- `refresh_token`, `access_token`.
- Tokens de password reset.

Sí loguea: `userId`, `email` hasheado o solo dominio (`@gmail.com`) si necesitas debug, IP, user agent, resultado (`success`/`fail`/`locked`).

## Sesiones en DB

Tabla mínima:

```
sessions
  id            uuid (sid del JWT)
  user_id       fk users
  refresh_hash  hash del refresh actual (no el plano)
  user_agent    text
  ip            inet
  created_at    timestamp
  last_used_at  timestamp
  revoked_at    timestamp NULL
  expires_at    timestamp
```

Permite ver "sesiones activas" y revocar específicas.

## Anti-patrones

- Tokens en `localStorage`.
- JWT con email, nombre o cualquier PII.
- Sin expiración o expiración de 30 días en access tokens.
- Refresh tokens sin rotación.
- Hashing con MD5/SHA1/SHA256 sin salt.
- Mensaje "user not found" en login (filtra existencia de cuentas).
- `app.use(cors({ origin: '*', credentials: true }))` — incompatible y peligroso.
- Reset tokens reutilizables.
- Sin rate limit en endpoints de auth.

## Referencias cruzadas

- AuthContext mock del frontend (Sprint 3) → `frontend-component-author`.
- Diseño general de la API → `backend-api-design`.
- Schema de la tabla `users` y `sessions` → `backend-data`.
- Auditoría de los nuevos endpoints → `audit-security` (añade puntos al checklist).
