# Backend (NestJS API) — Step-by-Step Flow

Build order: **second**, right after the two databases exist (`database-design-flow.md`). Everything mobile and desktop do goes through this API — build and test it fully with Postman before touching any frontend code.

---

## Step 1: Confirm project structure

You already created `backend/` in the database step. Final shape by the end of this file:

```
backend/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── jwt-auth.guard.ts
│   │   ├── roles.guard.ts
│   │   └── roles.decorator.ts
│   ├── tasks/
│   │   ├── tasks.module.ts
│   │   ├── tasks.controller.ts
│   │   └── tasks.service.ts
│   ├── tenants/
│   │   ├── tenants.module.ts
│   │   ├── tenants.controller.ts
│   │   ├── tenants.service.ts
│   │   ├── tenant-client.ts
│   │   └── subscription.guard.ts
│   ├── control-plane/
│   │   ├── control-plane.module.ts
│   │   ├── company.service.ts
│   │   └── provision-tenant.ts
│   └── admin/
│       ├── admin.module.ts
│       ├── admin.controller.ts        # your own SuperAdmin endpoints
│       └── admin.service.ts
├── prisma/
│   ├── schema.prisma                  # tenant schema (reused per-company)
│   └── migrations/
├── control-plane-prisma/
│   └── schema.prisma                  # control-plane schema (separate client)
├── .env
└── package.json
```

Two separate Prisma schemas, two separate generated clients — one for control-plane, one for whichever tenant DB is active in the current request.

---

## Step 2: Wire up two Prisma clients

Install:
```bash
npm install @prisma/client
```

Generate a client for each schema (control-plane and tenant), pointing each at its own `.env` variable — `CONTROL_PLANE_DATABASE_URL` and `DATABASE_URL`. `tenant-client.ts` (from your earlier plan) manages a cache of tenant clients, keyed by `dbUrl`, created on demand.

**Done when:** you can write a throwaway script that connects to `control_plane` and reads your seeded `SuperAdmin` row, and a second script that connects to `tenant_template` and creates a test `User`.

---

## Step 3: Build tenant resolution + subscription check

`POST /auth/resolve-tenant`
1. Input: `companyId`
2. Look up `Company` in control-plane DB
3. Look up its `Subscription` — if `status` is `EXPIRED`/`CANCELLED` or `endDate` has passed → reject with a clear "subscription expired" message, stop here
4. If active → return which tenant `dbUrl` to use internally (never expose the raw connection string to the client app)

Wrap this subscription check as a reusable guard (`subscription.guard.ts`) so it runs before every tenant-scoped request, not just login — a company's access should cut off mid-session too if their subscription lapses.

**Done when:** hitting `resolve-tenant` with a valid `companyId` returns success; with an expired one, it's blocked.

---

## Step 4: Build authentication

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
```

`POST /auth/login`:
1. Input: `companyId`, `email`, `password`
2. Resolve tenant (Step 3)
3. Look up `User` by email in that tenant DB
4. `bcrypt.compare()` the password
5. Sign a JWT containing `{ userId, role, tenantId }`

```bash
nest generate guard auth/jwt-auth
nest generate guard auth/roles
```
`@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles('ADMIN', 'MANAGER')` on protected routes.

**Done when:** login returns a JWT, and a request with a `MANAGER`-only or `ADMIN`-only route rejects `EMPLOYEE` tokens.

---

## Step 5: Build the tasks module

```bash
nest generate module tasks
nest generate controller tasks
nest generate service tasks
```
- `POST /tasks` (Manager/Admin only) — creates task, writes a `TaskEvent` with action `CREATED`
- `GET /tasks` — list, filtered by role (employee sees their own, manager/admin see all)
- `PATCH /tasks/:id` — status changes (`ACCEPTED`, `STARTED`, `COMPLETED`) — each one writes a `TaskEvent`

Every write to `Task` should be paired with a `TaskEvent` row in the same transaction — never let the two get out of sync.

**Done when:** you can create/assign/update tasks purely through Postman, and `TaskEvent` rows appear for each action.

---

## Step 6: Build provisioning (new company signup)

`control-plane/provision-tenant.ts`:
1. Create a new database on Neon: `tenant_<companyId>`
2. Run the saved migrations from `backend/prisma/migrations/` against it
3. Insert a `Company` row (with encrypted `dbUrl`) and a `Subscription` row (`status: TRIAL`) into `control_plane`

This can start as a script **you** run manually per new client (fine at low volume), and only needs to become a fully automatic signup flow once you have enough companies that manual provisioning is a bottleneck.

**Done when:** running the script for a test company creates a working, isolated tenant database, reachable through `resolve-tenant` + `login`.

---

## Step 7: Build your own admin endpoints

`admin/admin.controller.ts` — protected by a separate `SuperAdmin` login (not the same JWT/guard as company users):
- `POST /admin/companies` — trigger provisioning
- `PATCH /admin/companies/:id/subscription` — manually update `status`/`endDate` when a company pays
- `GET /admin/companies` — list all companies + their subscription status

This is the internal tool you'll actually use day-to-day to manage clients.

---

## Step 8: Compliance endpoints

- `GET /export` (Admin only, tenant-scoped) — dumps that company's `Task`/`TaskEvent` data as JSON/CSV
- Confirm `TaskEvent` rows are genuinely never updated or deleted anywhere in your code — grep your own service files for this before moving on

---

## Step 9: Test everything end-to-end with Postman

Before writing a single line of mobile/desktop code, confirm this full chain works:
`resolve-tenant` → `login` → `create task` → `list tasks` → `update task status` → `export` — for **two different test companies**, confirming their data never crosses over.

---

## Step 10: Host it (free tier)

Deploy to Render.com or Fly.io free tier:
1. Push `backend/` to GitHub
2. Connect the repo on Render/Fly, set `CONTROL_PLANE_DATABASE_URL` and other secrets as environment variables in their dashboard (never commit `.env`)
3. Note the public URL you get (e.g. `https://pharma-api.onrender.com`) — this is what mobile and desktop apps will point at

**Done when:** you can hit your live URL from Postman, not just `localhost`.

Once this file's checklist is fully green, move to `mobile-build-flow.md` and `desktop-build-flow.md` — both just consume this API, so they can be built in either order or in parallel.
