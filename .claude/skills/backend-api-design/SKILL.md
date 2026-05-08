---
name: backend-api-design
description: Patrones de diseño de APIs REST/HTTP forward-looking para FarmLabs. Aplica cuando se planifique reemplazar el catálogo estático (public/data/products.json) por un backend real, o cuando el AuthContext mock evolucione a una API de autenticación. Cubre convenciones REST (verbos, status codes, paginación, errores), shape de respuesta, versionado, idempotencia, OpenAPI y framework de elección (Node + Express/Fastify). Forward-looking: este repo aún no tiene backend, así que se invoca cuando aparezca una tarea fuera del roadmap actual o se decida añadir un Sprint 10.
---

# Skill: backend-api-design

Aporta patrones de diseño de APIs HTTP. **Forward-looking**: FarmLabs aún no tiene backend. Esta skill se activa cuando:

- Se planifique reemplazar `public/data/products.json` por una API real.
- El `AuthContext` mock del Sprint 3 evolucione a un backend real.
- Se añada un Sprint 10 al roadmap con endpoints de checkout, orders, etc.
- El usuario pida explícitamente "diseña la API de productos", "qué endpoints expongo", etc.

## Stack recomendado para este repo

| Capa | Recomendación | Motivo |
|------|---------------|--------|
| Runtime | Node.js ≥ 20 | LTS; stack ya es JS |
| Framework | **Fastify** o Express | Fastify si se quiere validación con schema; Express si se prioriza familiaridad |
| Validación | `zod` o `@fastify/type-provider-typebox` | Type-safe en TS |
| Documentación | OpenAPI 3.1 (`@fastify/swagger` o `swagger-jsdoc`) | Autogenerable |
| Persistencia | Ver `backend-data` |
| Auth | Ver `backend-auth` |

## Convenciones REST

### Verbos HTTP

| Verbo | Uso | Idempotente |
|-------|-----|-------------|
| `GET` | Lectura, nunca cambia estado | Sí |
| `POST` | Crear recurso o ejecutar acción | No |
| `PUT` | Reemplazar recurso completo | Sí |
| `PATCH` | Actualizar parcial | No (típicamente) |
| `DELETE` | Borrar recurso | Sí |

### Naming de rutas

- Recursos en plural: `/products`, `/orders`, `/users`.
- Identificadores en path: `/products/:id`.
- Acciones no-CRUD como sub-recurso: `/orders/:id/cancel` (POST).
- Versionado en path: `/v1/products`. Bump mayor solo para breaking changes.

### Status codes

| Code | Cuándo |
|------|--------|
| 200 | OK con body |
| 201 | Created (POST exitoso, devuelve el recurso creado y `Location` header) |
| 204 | No Content (DELETE exitoso) |
| 400 | Bad Request (validación fallida — incluir detalle) |
| 401 | Unauthorized (sin token o expirado) |
| 403 | Forbidden (token válido pero sin permisos) |
| 404 | Not Found |
| 409 | Conflict (ej. email ya registrado) |
| 422 | Unprocessable Entity (validación semántica, alternativa a 400) |
| 429 | Too Many Requests (rate limit) |
| 500 | Internal Server Error (no exponer stack) |

### Shape de respuesta

```json
// Éxito
{
  "data": { "id": 1, "title": "Amnesia Haze", "price": 320 }
}

// Lista
{
  "data": [...],
  "meta": { "page": 1, "perPage": 20, "total": 142 }
}

// Error
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Email is required.",
    "details": [{ "field": "email", "message": "required" }]
  }
}
```

Mantén una sola envolvente. NO mezcles `{ data: ... }` con respuestas crudas.

### Paginación

- **Offset/limit:** `GET /products?page=1&perPage=20` para listas pequeñas/medianas.
- **Cursor:** `GET /products?cursor=abc&limit=20` para feeds o cuando los datos cambian frecuentemente.
- Devuelve `meta.total`, `meta.nextCursor` o `Link` header.

### Filtros y sort

- Filtros: `?category=Flowers&minPrice=100`.
- Sort: `?sort=price` o `?sort=-price` (- = desc), múltiples: `?sort=-price,title`.
- Search: `?q=haze`.

## Idempotencia

- Para POSTs idempotentes (ej. crear orden), aceptar header `Idempotency-Key`. Si llega dos veces el mismo key en N minutos, devuelve el mismo resultado.
- Para PUTs, garantiza que aplicar dos veces deja el mismo estado.

## Validación

Schema-first con `zod` (ejemplo):

```javascript
import { z } from 'zod';

const CreateOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.number().int().positive(),
    quantity: z.number().int().min(1).max(99),
  })).min(1),
  email: z.string().email(),
});

// En el handler:
const result = CreateOrderSchema.safeParse(req.body);
if (!result.success) {
  return reply.code(400).send({
    error: {
      code: 'VALIDATION_FAILED',
      message: 'Invalid payload',
      details: result.error.errors,
    },
  });
}
```

## Endpoints sugeridos para FarmLabs

Si se construye el backend que reemplace `products.json` y el mock auth:

```
GET    /v1/products                  → lista de productos
GET    /v1/products/:id              → detalle
GET    /v1/categories                → lista de categorías
POST   /v1/auth/register             → crear cuenta
POST   /v1/auth/login                → login (devuelve tokens)
POST   /v1/auth/refresh              → refresh access token
POST   /v1/auth/logout               → revocar refresh
GET    /v1/users/me                  → perfil (requiere auth)
PATCH  /v1/users/me                  → actualizar perfil
GET    /v1/orders                    → órdenes del usuario
POST   /v1/orders                    → crear orden (idempotente)
GET    /v1/orders/:id                → detalle de orden
POST   /v1/orders/:id/cancel         → cancelar
```

## OpenAPI y contrato

- Mantener un `openapi.yaml` en el repo del backend.
- Generar tipos TS para el frontend con `openapi-typescript` para que `useGetProducts` reciba `Product[]` tipado.
- En CI, validar que el código actual cumpla el spec (ej. `dredd` o test contractuales).

## CORS y headers

```javascript
// Solo orígenes propios:
fastify.register(import('@fastify/cors'), {
  origin: ['https://jonahsly.github.io'],
  credentials: true,
});

// Headers de seguridad básicos:
fastify.register(import('@fastify/helmet'));
```

## Rate limiting

Aplicar al menos en `/auth/login`, `/auth/register`, `/auth/refresh`:

```javascript
fastify.register(import('@fastify/rate-limit'), {
  max: 10,
  timeWindow: '1 minute',
});
```

## Logging y observabilidad

- Loggea `req.id`, método, ruta, status, latencia.
- NUNCA loguees `Authorization`, `Cookie`, `password` ni `token` (ver `audit-security`).
- Estructura JSON (Pino con Fastify es lo natural).

## Anti-patrones

- Verbos en URLs (`/getProducts`, `/createOrder`) — usa REST.
- Devolver 200 con `{ "success": false }` — usa el código HTTP correcto.
- Tener `error.message` legible para usuarios mezclado con stack traces.
- Aceptar cualquier campo en `PATCH` (whitelist explícita).
- Versionar con headers custom — usa path `/v1/`.
- Devolver IDs internos de DB (autoincrementales) sin necesidad — considera UUIDs públicos.

## Referencias cruzadas

- Auth (JWT, cookies, refresh) → `backend-auth`.
- Schema, migraciones, queries → `backend-data`.
- Cliente frontend que consumirá esta API → `frontend-component-author` (`useGetProducts`).
- Auditoría de seguridad del backend → `audit-security` (extender checklist con nuevos puntos para servidor).
