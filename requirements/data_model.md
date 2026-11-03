# Data Model
**ResortOS AI — Restaurant Booking App**

PostgreSQL 17 via Prisma 7. This is the source of truth for `prisma/schema.prisma`. All timestamps are `timestamptz` (stored in UTC).

## 1. Entity Overview
```
User 1 ──── * Reservation 1 ──── * ReservationTable * ──── 1 DiningTable
                                         (live table locks)
User 1 ──── * Session     (signed-in browsers; deleted on sign-out)
User 1 ──── 0..1 ApiKey   (for now only the AGENT account has one)

MenuItem   (standalone, seeded)
AuditLog   (standalone, append-only)
```

## 2. Enums
| Enum | Values |
|---|---|
| `Role` | `USER`, `ADMIN`, `AGENT` (the AI booking assistant's service account) |
| `BookingSource` | `APP` (guest in the app), `AGENT` (assistant on the guest's behalf) |
| `Branch` | `BRANCH_1`, `BRANCH_2` |
| `Sector` | `MAIN_HALL`, `VIP_WING` |
| `ReservationStatus` | `CONFIRMED`, `SEATED`, `COMPLETED`, `CANCELLED`, `NO_SHOW` (v1 writes only `CONFIRMED` / `CANCELLED`) |
| `MenuType` | `SEASONAL_FINE_DINING`, `A_LA_CARTE` |

## 3. Tables

### `User`
| Column | Type | Rules |
|---|---|---|
| `id` | text (cuid) | PK |
| `email` | text | unique, stored lowercase |
| `passwordHash` | text? | bcrypt, cost 12. **Null** for a guest account created by the assistant (sign-up with that email sets it, claiming the account) and for the assistant's own `AGENT` account |
| `name` | text | 1–100 chars |
| `phone` | text? | optional |
| `allergies` | text[] | lowercase tags, max 20 (e.g. `gluten`, `shellfish`) |
| `role` | `Role` | default `USER`. `ADMIN` and `AGENT` only via seed |
| `createdAt` | timestamptz | default now |

### `Session` (one row per signed-in browser)
| Column | Type | Rules |
|---|---|---|
| `id` | text (cuid) | PK |
| `tokenHash` | text | **unique**. SHA-256 (hex) of the random token in the `session` cookie; the token itself is never stored |
| `userId` | text | FK → `User`, cascade delete |
| `createdAt` | timestamptz | default now (sign-in time) |
| `expiresAt` | timestamptz | `createdAt + SESSION_TTL` (1 hour), never extended |

Index: `userId`.
* Sign-in / register: insert a row; delete that user's rows whose `expiresAt` has passed.
* Every request: valid only if the row exists and `expiresAt > now`.
* Sign-out: delete the row.

### `ApiKey` (keys for non-browser callers)
| Column | Type | Rules |
|---|---|---|
| `id` | text (cuid) | PK |
| `name` | text | who it's for, e.g. `AI booking assistant` |
| `keyHash` | text | **unique**. SHA-256 (hex) of the key; the key itself is never stored |
| `userId` | text | FK → `User`, cascade delete. **unique** for now: one key per account |
| `createdAt` | timestamptz | default now |

* For now there is **one** key, owned by the `AGENT` account, and the auth middleware accepts a key only if its owner's role is `AGENT`.
* Later (to be decided by the owner): keys for other callers, e.g. special users or other vendors. That would drop the one-key-per-account rule and may add scopes; not built now.

### `DiningTable` (seeded, read-only to the app)
| Column | Type | Rules |
|---|---|---|
| `id` | text | PK, readable code: `B1-MH-01`…`B1-MH-50`, `B1-VIP-1`…`B1-VIP-4` (same for `B2-`) |
| `branch` | `Branch` | |
| `sector` | `Sector` | |
| `seats` | int | 2 for Main Hall, 13 for VIP rooms |

Index: `(branch, sector)`. Seed total: 108 rows (2 branches × (50 + 4)).

### `Reservation`
| Column | Type | Rules |
|---|---|---|
| `id` | text (cuid) | PK (internal) |
| `code` | text | **unique**, `RSV-XXXXXX` (alphabet without 0/O/1/I), shown to guests and used by staff |
| `userId` | text | FK → `User` |
| `branch` | `Branch` | |
| `sector` | `Sector` | |
| `slotStart` | timestamptz | must equal one of the fixed slot starts |
| `slotEnd` | timestamptz | `slotStart + 150 min` |
| `partySize` | int | 1–50 |
| `specialRequests` | text? | max 500 chars |
| `status` | `ReservationStatus` | default `CONFIRMED` |
| `source` | `BookingSource` | default `APP` |
| `createdById` | text? | FK → `User`: the guest (app) or the assistant's account (`AGENT`) |
| `idempotencyKey` | text? | **unique**; a retried assistant confirmation returns the same booking |
| `tableIds` | text[] | **snapshot** of assigned tables. Kept after cancel so admins still see what was assigned |
| `createdAt` / `updatedAt` | timestamptz | |
| `cancelledAt` | timestamptz? | set on soft cancel |

Indexes: `userId`, `(branch, slotStart)`, `status`, `source`.

### `ReservationTable` (live locks)
| Column | Type | Rules |
|---|---|---|
| `reservationId` | text | FK → `Reservation`, cascade delete |
| `tableId` | text | FK → `DiningTable` |
| `slotStart` | timestamptz | copied from the reservation |

PK `(reservationId, tableId)`. **`UNIQUE (tableId, slotStart)`** is the double-booking guard.
* Create: insert rows inside the booking transaction. If a concurrent booking already took a table, the insert fails with `P2002`, mapped to `400 Table overlap conflict`.
* Cancel: delete the rows (table freed), keep the `Reservation`.
* Reschedule: delete old rows + insert new rows in **one** transaction. If capacity fails, the rollback keeps the original booking.

### `MenuItem` (seeded)
| Column | Type | Rules |
|---|---|---|
| `id` | text (cuid) | PK |
| `name` | text | unique |
| `description` | text | |
| `type` | `MenuType` | |
| `priceCents` | int | money stored as integer cents (IDR × 100 or chosen currency) |
| `allergens` | text[] | lowercase tags matching `User.allergies` |
| `chefPick` | boolean | sorted first in recommendations |

### `AuditLog` (append-only)
| Column | Type | Rules |
|---|---|---|
| `id` | bigint | PK, autoincrement |
| `timestamp` | timestamptz | default now |
| `actorId` | text? | no FK, so logs outlive users |
| `action` | text | see `audit_trail.md` |
| `resourceId` | text? | |
| `ipAddress` / `userAgent` | text? | |
| `metadata` | jsonb? | never contains passwords/tokens |

Index: `(action, timestamp)` (feeds the failed-login alert query).

**Immutability (hand-written SQL in the init migration):**
```sql
CREATE FUNCTION audit_log_immutable() RETURNS trigger AS $$
BEGIN RAISE EXCEPTION 'AuditLog is append-only'; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_log_no_update_delete
BEFORE UPDATE OR DELETE ON "AuditLog"
FOR EACH ROW EXECUTE FUNCTION audit_log_immutable();
```
`TRUNCATE` is blocked by grants (production DB role has `INSERT, SELECT` only).

## 4. Seed Data
* 108 `DiningTable` rows.
* ~10 `MenuItem` rows: Chicken Steak, Beef Steak, Barramundi, Fish & Chips, Classic Burger, plus seasonal fine-dining dishes, with allergen tags.
* 1 `ADMIN` user from `ADMIN_EMAIL` / `ADMIN_PASSWORD`.
* 1 `AGENT` user (the AI booking assistant, `assistant@resortos.internal`): no password, and its one `ApiKey` row (`keyHash` = SHA-256 of `AGENT_API_KEY`). Re-seeding with a new key replaces that row's hash.

## 5. Migrations
* `prisma migrate dev` generates migrations from the schema. The trigger SQL is appended to the init migration by hand.
* Production: `prisma migrate deploy`.
* The migration that adds `ApiKey` copies any existing `User.apiKeyHash` into it, then drops that column.
