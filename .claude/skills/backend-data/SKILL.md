---
name: backend-data
description: Patrones de capa de datos para FarmLabs cuando se construya el backend (esquemas, migraciones reversibles, queries, índices, ORM y seed data). Incluye decisiones sobre Postgres vs SQLite, ORMs (Prisma/Drizzle), prevención de N+1, transacciones y backups. Forward-looking: complementa a backend-api-design y backend-auth cuando exista una base de datos. Úsalo cuando el usuario pida "diseña la tabla X", "haz una migración", "modela las órdenes", "qué índices pongo" o cuando se mueva el catálogo de products.json a una DB.
---

# Skill: backend-data

Aporta los patrones de capa de datos del backend FarmLabs. **Forward-looking**: hoy el "catálogo" es un JSON estático. Esta skill se activa cuando:

- Se modela una base de datos para reemplazar `products.json`.
- Se necesita persistir órdenes, usuarios, sesiones, carritos.
- Se planifican migraciones/seed data o se añade un ORM.

## Decisión central: motor de DB

| Motor | Cuándo |
|-------|--------|
| **Postgres** | Default. Producción, transacciones, JSON nativo, full-text search. |
| **SQLite** | Dev/single-user, prototipo local, tests. |
| **MongoDB** | Solo si los documentos son realmente sin estructura uniforme. Para FarmLabs **no** aplica. |

**Decisión por defecto:** Postgres en producción + SQLite en dev/CI vía un ORM agnóstico.

## ORM recomendado

| ORM | Pros | Cons |
|-----|------|------|
| **Drizzle** | Schema en TS, queries SQL-like, ligero, type-safe | Joven, migraciones manuales |
| **Prisma** | Madurez, devx excelente, migrate command | Generated client pesa, runtime overhead |
| **Knex + tipos manuales** | Control total | Sin types automáticos |

Para este repo (futuro TS con Sprint 9), **Drizzle** o **Prisma** son las opciones razonables.

## Esquema mínimo para FarmLabs

```sql
-- users
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         CITEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  name          TEXT,
  role          TEXT NOT NULL DEFAULT 'customer',  -- customer | admin
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ
);
CREATE INDEX users_email_idx ON users (email) WHERE deleted_at IS NULL;

-- sessions (ver backend-auth)
CREATE TABLE sessions (
  id            UUID PRIMARY KEY,
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  refresh_hash  TEXT NOT NULL,
  user_agent    TEXT,
  ip            INET,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_used_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  revoked_at    TIMESTAMPTZ,
  expires_at    TIMESTAMPTZ NOT NULL
);
CREATE INDEX sessions_user_idx ON sessions (user_id) WHERE revoked_at IS NULL;

-- categories
CREATE TABLE categories (
  id    SERIAL PRIMARY KEY,
  slug  TEXT UNIQUE NOT NULL,
  name  TEXT NOT NULL
);

-- products
CREATE TABLE products (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sku          TEXT UNIQUE,
  title        TEXT NOT NULL,
  description  TEXT NOT NULL DEFAULT '',
  price_cents  INTEGER NOT NULL CHECK (price_cents >= 0),
  category_id  INTEGER REFERENCES categories(id),
  images       TEXT[] NOT NULL DEFAULT '{}',
  active       BOOLEAN NOT NULL DEFAULT true,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX products_category_idx ON products (category_id) WHERE active = true;
CREATE INDEX products_title_trgm_idx ON products USING gin (title gin_trgm_ops);  -- search

-- orders
CREATE TABLE orders (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  status          TEXT NOT NULL DEFAULT 'pending',  -- pending | paid | shipped | cancelled
  total_cents     INTEGER NOT NULL CHECK (total_cents >= 0),
  idempotency_key TEXT UNIQUE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX orders_user_idx ON orders (user_id, created_at DESC);

CREATE TABLE order_items (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id    UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id  UUID NOT NULL REFERENCES products(id),
  quantity    INTEGER NOT NULL CHECK (quantity > 0),
  unit_cents  INTEGER NOT NULL CHECK (unit_cents >= 0),
  title       TEXT NOT NULL,        -- denormalizado: histórico
  UNIQUE (order_id, product_id)
);
```

### Decisiones del esquema

- **UUIDs públicos** (no IDs autoincrementales) para evitar enumeración.
- **`price_cents` integer** no `numeric/float` — evita errores de redondeo.
- **`CITEXT` para emails** — comparación case-insensitive sin `LOWER()`.
- **Timestamps con TZ** — siempre UTC en DB, conversión en cliente.
- **Soft delete** (`deleted_at`) en `users`; hard delete en `sessions`.
- **Denormalización** de `title`/`unit_cents` en `order_items` para preservar historia ante cambios del producto.
- **Idempotency key** en `orders` para reintentos seguros desde el cliente (ver `backend-api-design`).

## Migraciones

Reglas:
- **Una migración = un cambio reversible.** Cada `up` tiene su `down`.
- Nombrarlas con timestamp + descripción: `20260601120000_add_orders_table.sql`.
- NUNCA editar una migración ya aplicada. Si está en `main`, crea una nueva.
- En CI, correr migraciones contra una DB efímera y validar.

```sql
-- 20260601120000_add_idempotency_key.up.sql
ALTER TABLE orders ADD COLUMN idempotency_key TEXT UNIQUE;
CREATE INDEX orders_idem_idx ON orders (idempotency_key) WHERE idempotency_key IS NOT NULL;

-- 20260601120000_add_idempotency_key.down.sql
DROP INDEX IF EXISTS orders_idem_idx;
ALTER TABLE orders DROP COLUMN IF EXISTS idempotency_key;
```

### Cambios "online" en producción

Si la tabla es grande:
- `ADD COLUMN` con default → escribir como `NOT NULL` requiere backfill por lotes en Postgres < 11.
- `CREATE INDEX CONCURRENTLY` para no bloquear escrituras.
- Cambios de tipo → migración multi-paso (add new col → backfill → switch app → drop old col).

## Seed data para dev

```javascript
// scripts/seed.js
import { db } from '../src/db';
import productsJson from '../public/data/products.json' assert { type: 'json' };

await db.transaction(async (tx) => {
  for (const p of productsJson) {
    await tx.insert(products).values({
      sku: `seed-${p.id}`,
      title: p.title,
      description: p.description ?? '',
      priceCents: Math.round(p.price * 100),
      images: p.images ?? [],
    });
  }
});
```

Aprovecha el `products.json` actual como fuente para no perder el catálogo.

## Queries — patrones útiles

### Evitar N+1

```javascript
// ❌ N+1
const orders = await db.select().from(orders).where(eq(orders.userId, id));
for (const order of orders) {
  order.items = await db.select().from(orderItems).where(eq(orderItems.orderId, order.id));
}

// ✅ JOIN o IN
const items = await db.select().from(orderItems).where(inArray(orderItems.orderId, orders.map(o => o.id)));
const byOrder = groupBy(items, 'orderId');
orders.forEach(o => o.items = byOrder[o.id] ?? []);
```

### Transacciones

Toda operación que toque ≥ 2 tablas críticas (ej. crear orden + decrementar stock):

```javascript
await db.transaction(async (tx) => {
  const order = await tx.insert(orders).values({...}).returning();
  await tx.insert(orderItems).values(items.map(i => ({...i, orderId: order.id})));
});
```

### Paginación con cursor

```sql
SELECT * FROM products
WHERE active = true AND (created_at, id) < ($cursorCreatedAt, $cursorId)
ORDER BY created_at DESC, id DESC
LIMIT 21;  -- pide N+1 para saber si hay más
```

## Backups y recovery

- **Postgres managed** (Supabase, Neon, RDS): backup diario + PITR (point-in-time recovery) ≥ 7 días.
- **Self-hosted**: `pg_dump` diario + WAL archiving. Verifica restores cada mes.
- Almacena backups **fuera del mismo proveedor** que la DB principal.

## Connection pooling

- Local: pool del driver (`pg-pool`, 10 conexiones).
- Producción serverless: usar **PgBouncer** o **Supabase pooler** para no agotar conexiones.

## Anti-patrones

- IDs autoincrementales expuestos en URL (`/orders/142` revela volumen).
- Precios en `float`/`numeric` sin escala — usa cents.
- Soft delete sin filtro automático en queries (entrega datos borrados).
- Migraciones no reversibles (`DROP COLUMN` sin backup en `down`).
- ORMs sin migraciones versionadas (`db.sync()` en producción).
- Queries `SELECT *` en endpoints públicos (filtra columnas sensibles).
- Almacenar PII junto con datos analíticos sin separación.
- Índices "por si acaso" en cada columna — costo de escritura.
- Transacciones que mantienen locks largos (HTTP request abierto).

## Observabilidad

- Loggea queries lentas (`> 200ms`).
- Métricas: conexiones activas, queries/s, latencia p95.
- Alerta si la pool está al 80%.

## Referencias cruzadas

- API que consume estos datos → `backend-api-design`.
- Tablas `users` y `sessions` para login → `backend-auth`.
- Cliente que consume `/v1/products` → `frontend-component-author`.
- Auditoría: secrets de DB en `.env`, no en repo → `audit-security`.
