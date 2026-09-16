# Pharma Work Allocation App — Multi-Tenant Architecture & Schema

This covers the two-database split you described: **your control-plane database** (stores companies + their DB credentials) and **each company's own tenant database** (stores their employees, tasks, etc.). Plus subscription handling, and a placeholder for the future machine-integration plan.

---

## 1. How the System Fits Together

```
                        ┌─────────────────────────────┐
                        │   CONTROL-PLANE DATABASE     │
                        │   (yours — one, shared)      │
                        │   Companies, Subscriptions,  │
                        │   tenant DB credentials      │
                        └──────────────┬───────────────┘
                                       │ resolves companyId → tenant dbUrl
                                       ▼
                        ┌─────────────────────────────┐
                        │        NestJS API            │
                        │  (hosted free-tier — Render  │
                        │   / Fly.io — see note below)  │
                        └──────────────┬───────────────┘
                     ┌─────────────────┼─────────────────┐
                     ▼                 ▼                 ▼
            ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
            │ Tenant DB       │ │ Tenant DB       │ │ Tenant DB       │
            │ Company A       │ │ Company B       │ │ Company C       │
            │ Users, Tasks,   │ │ Users, Tasks,   │ │ Users, Tasks,   │
            │ TaskEvents      │ │ TaskEvents      │ │ TaskEvents      │
            └────────────────┘ └────────────────┘ └────────────────┘
                     ▲                 ▲                 ▲
              ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
              │ Desktop .exe │   │ Mobile .apk  │   │ Desktop .exe │
              │ Company A    │   │ Company B    │   │ Company C    │
              └─────────────┘   └─────────────┘   └─────────────┘
```

Every company's app (desktop or mobile) talks to **the same NestJS API**. The API looks up which tenant database to use based on the `companyId` the user enters at login, then all queries for that session go to that company's own database only. This is exactly the pattern from the earlier Neon guide's Phase 4 — this file just fills in the actual columns.

**Hosting note (see correction above):** Neon (free tier) can host your control-plane DB and every tenant DB as separate databases inside one Neon project. The NestJS API itself needs a free host like Render or Fly.io. This is the only piece that isn't "zero hosting" — it is zero *cost*, at least until you have paying companies and need better uptime.

---

## 2. Control-Plane Database (Yours)

This is the **one** database you own and manage. It never holds any company's actual work data — only who the companies are, how to reach their database, and whether they're paid up.

### `Company`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `companyId` | string, unique | Human-typed code at login, e.g. `ACME123` |
| `name` | string | |
| `contactEmail` | string | Primary contact for billing/support |
| `contactPhone` | string | |
| `dbName` | string | Name of their database inside your Neon project |
| `dbUrl` | string, **encrypted** | Full connection string to their tenant DB — see security note below |
| `status` | enum | `TRIAL`, `ACTIVE`, `SUSPENDED`, `EXPIRED`, `CANCELLED` |
| `createdAt` | datetime | |
| `updatedAt` | datetime | |

> **Security note:** `dbUrl` contains a password. Don't store it as plain text even in your own DB — encrypt it at the application layer (e.g. with a key from an environment variable) before saving, and decrypt only in memory when the API needs to connect. This one field is the single most sensitive thing in your whole system.

### `Plan`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `name` | string | e.g. `Basic`, `Pro` |
| `maxUsers` | int | Optional cap |
| `priceMonthly` | decimal | For your own reference |
| `features` | json | Freeform, e.g. `{ "exportEnabled": true }` |

### `Subscription`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `companyId` | uuid, FK → `Company.id` | |
| `planId` | uuid, FK → `Plan.id` | |
| `billingCycle` | enum | `MONTHLY`, `YEARLY`, `LIFETIME` |
| `startDate` | datetime | |
| `endDate` | datetime | When access should stop if unpaid |
| `status` | enum | `TRIAL`, `ACTIVE`, `EXPIRED`, `CANCELLED` |
| `lastPaymentDate` | datetime, nullable | |
| `amountPaid` | decimal, nullable | |
| `paymentMethod` | enum | `MANUAL` (bank/UPI, you update this by hand), `RAZORPAY` (placeholder for later automation) |
| `notes` | string, nullable | e.g. "paid via UPI, screenshot on file" |

**How this gets checked:** on every `resolve-tenant` call (or on login), the API checks `Subscription.status` and `endDate` for that company before returning the tenant `dbUrl`. Expired → the API refuses to connect them and returns a "subscription expired, contact admin" response. Everything else (users, tasks) never even gets touched.

**Zero-expense subscription handling for now:** since you don't want payment gateway costs/complexity yet, keep `paymentMethod = MANUAL` — a company pays you by bank transfer/UPI, you log into a simple internal tool (or just update the row directly) and set `status = ACTIVE` with a new `endDate`. The schema already has a `RAZORPAY` slot ready for later if you ever want to automate this — you won't need to redesign the table, just add the integration.

### `SuperAdmin`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `name` | string | |
| `email` | string, unique | |
| `password` | string, hashed | |
| `role` | enum | `OWNER`, `SUPPORT` |

This is **you** (and anyone helping you), separate from any company's own users — logs into a separate internal admin view to manage companies/subscriptions. Not exposed to client companies at all.

### `CompanyAuditLog`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `companyId` | uuid, FK | |
| `action` | string | e.g. `CREATED`, `SUSPENDED`, `PLAN_CHANGED`, `REACTIVATED` |
| `performedBy` | uuid, FK → `SuperAdmin.id` | |
| `timestamp` | datetime | |

Keeps a record of what you did to any company's account — useful if a client ever disputes being suspended or asks "when did our plan change."

---

## 3. Tenant Database (One Per Company)

Every company gets its own database with this same schema. None of these tables ever reference another company's data — that separation is what makes it "multi-tenant."

### `User`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `name` | string | |
| `email` | string, unique (within this DB) | |
| `password` | string, hashed | |
| `role` | enum | `ADMIN`, `MANAGER`, `EMPLOYEE` |
| `department` | string, nullable | |
| `phone` | string, nullable | |
| `isActive` | boolean, default true | For disabling a user without deleting them |
| `createdAt` | datetime | |

**Role suggestion:**
- `ADMIN` — company-level owner account. Manages users, sees everything, changes settings. Usually 1–2 people.
- `MANAGER` — creates/assigns tasks, oversees their team's work.
- `EMPLOYEE` — receives tasks, accepts/completes them.

(This is a small addition to the earlier guide, which only had `MANAGER`/`EMPLOYEE` — worth having a distinct `ADMIN` since pharma companies will want one account that isn't tied to day-to-day task assignment.)

### `Task`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `title` | string | |
| `description` | string, nullable | |
| `status` | enum | `PENDING`, `IN_PROGRESS`, `COMPLETED`, `ON_HOLD` |
| `priority` | enum, nullable | `LOW`, `MEDIUM`, `HIGH` |
| `assignedToId` | uuid, FK → `User.id`, nullable | Employee doing the task |
| `assignedById` | uuid, FK → `User.id` | Manager/Admin who created it |
| `dueDate` | datetime, nullable | |
| `createdAt` | datetime | |
| `updatedAt` | datetime | |

### `TaskEvent` (audit log — append-only, never update/delete rows)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `taskId` | uuid, FK → `Task.id` | |
| `action` | string | `CREATED`, `ASSIGNED`, `ACCEPTED`, `STARTED`, `COMPLETED`, `REJECTED`, `COMMENTED` |
| `actorId` | uuid, FK → `User.id` | |
| `notes` | string, nullable | |
| `timestamp` | datetime | |

This is the compliance backbone — for pharma work, being able to show "who did what, when" without any possibility of edits is often more important than the task feature itself.

### `Department` (optional — add only if companies actually want team grouping)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `name` | string | |
| `managerId` | uuid, FK → `User.id`, nullable | |

---

## 4. Future Plan — Machine Integration (placeholder only)

You mentioned wanting to eventually connect this to pharma company machines. Don't build this now — but reserving the shape avoids a painful migration later.

### `MachineLog` (stub table, tenant DB)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid, PK | |
| `machineName` | string | e.g. "Tablet Press 2" |
| `taskId` | uuid, FK → `Task.id`, nullable | Link a reading to a task/batch if relevant |
| `parameter` | string | e.g. "temperature", "batch_no", "pressure" |
| `value` | string | Keep as string for now — flexible until you know real data formats |
| `recordedAt` | datetime | |
| `recordedBy` | uuid, FK → `User.id`, nullable | Null if the machine pushes data automatically, not a person |

When this becomes real, you'll likely need a separate lightweight ingestion endpoint (so a flaky machine connection doesn't affect the main app), and possibly a message queue if machines push data faster than the DB should be hit directly — but that's a bridge to cross once you know what protocol the actual machines speak (Modbus, OPC-UA, plain HTTP, etc.).

---

## 5. Other Suggestions

- **Indexes:** put indexes on `Company.companyId`, `User.email`, `Task.assignedToId`, and `Task.status` early — these are the columns you'll filter/search by constantly.
- **Backups:** Neon has point-in-time restore on paid tiers; on the free tier, periodically export each tenant DB (the per-company export endpoint from your original plan doubles as a backup mechanism).
- **Company deletion:** never hard-delete a `Company` row — set `status = CANCELLED` and keep the row (and ideally their tenant DB, or an export of it) for a retention period. Pharma clients may have compliance reasons to need old records even after cancelling.
- **App versioning:** since you're distributing `.exe`/`.apk` directly rather than through a store, keep the `GET /app/latest-version` check from your original Phase 7 plan — it's your only way to nudge people onto new builds without a store doing it for you.

---

## Decisions Confirmed

- **Hosting:** one central NestJS API (yours), hosted free-tier initially (Render/Fly.io). No per-company self-hosting for now — the control-plane DB only works with a central API enforcing it.
- **Subscriptions:** manual for now — companies pay via bank transfer/UPI, you update `Subscription.status` and `endDate` by hand. The `paymentMethod` field already has a `RAZORPAY` slot ready if you automate this later.
