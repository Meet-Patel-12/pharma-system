# Database Design & Setup — Step-by-Step Flow

Build order: this file comes **first**, before backend/mobile/desktop — every other piece depends on these two databases existing. Full column-level schema already lives in `multi-tenant-architecture-and-schema.md`; this file is the *build order* for it.

---

## Step 1: Create your Neon account and project

1. Sign up at [neon.tech](https://neon.tech)
2. Create one project, e.g. `pharma-system`
3. Inside this one project you will create **multiple databases**:
   - `control_plane` — yours, one only
   - `tenant_<companyId>` — one per company, created as companies sign up
4. Copy the base connection details (host, user, password) — you'll build per-database URLs from these.

---

## Step 2: Build the control-plane database first

This is the one database that exists before any company does.

```bash
mkdir pharma-task-app && cd pharma-task-app
mkdir control-plane && cd control-plane
npm init -y
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```

In `control-plane/.env`:
```
DATABASE_URL="postgresql://<user>:<password>@<endpoint>.neon.tech/control_plane?sslmode=require"
```

`control-plane/prisma/schema.prisma` — enter the `Company`, `Plan`, `Subscription`, `SuperAdmin`, `CompanyAuditLog` models from the architecture file, then:

```bash
npx prisma migrate dev --name init
```

**Done when:** `npx prisma studio` shows five empty tables in your `control_plane` database.

### Step 2b: Seed one SuperAdmin row for yourself
Write a small one-off script (`control-plane/seed.ts`) that inserts your own admin row with a hashed password, run it once:
```bash
npx ts-node seed.ts
```
You'll use this to log into your own internal admin view later.

---

## Step 3: Build the tenant database template

This is the schema every company's database will use. You won't run this against a real company yet — you're just getting the schema file ready so the backend's provisioning script can apply it automatically later.

```bash
cd ../
mkdir backend && cd backend
npm i -g @nestjs/cli
nest new . --skip-git
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```

`backend/prisma/schema.prisma` — enter `User`, `Task`, `TaskEvent`, `Department` (and the `MachineLog` stub if you want it in from day one) from the architecture file.

```
DATABASE_URL="postgresql://<user>:<password>@<endpoint>.neon.tech/tenant_template?sslmode=require"
```

```bash
npx prisma migrate dev --name init
```

Running it against a database called `tenant_template` gives you a tested, working migration set. When a real company signs up, the backend's provisioning script (built in the backend flow file) copies this exact migration against a freshly created `tenant_<companyId>` database — so you never hand-write SQL per company.

**Done when:** `tenant_template` has `User`, `Task`, `TaskEvent`, `Department` tables, and the migration files exist in `backend/prisma/migrations/` (these get reused for every future company).

---

## Step 4: Confirm the split is correct before moving on

Checklist before you touch any backend code:
- [ ] `control_plane` database exists, has 5 tables, and your own `SuperAdmin` row is in it
- [ ] `tenant_template` database exists with `User`/`Task`/`TaskEvent`/`Department`
- [ ] You have **two separate** `DATABASE_URL` values saved somewhere safe (control-plane, tenant template) — you'll need both when building the backend
- [ ] You understand: real company data will **never** live in `control_plane`, and company credentials will **never** live in a tenant database

Once this checklist is done, move to `backend-build-flow.md`.
