# Pharma Work Allocation App — Full Step-by-Step Build Guide

This is the detailed, hands-on version of the project plan. Each phase has exact steps, commands, and what "done" looks like. Items marked **[CORE]** are required; items marked **[OPTIONAL]** can be skipped or added later without blocking progress.

---

## Before You Start

**Install these tools:**
- [ ] Node.js (LTS version) — [nodejs.org](https://nodejs.org)
- [ ] PostgreSQL — either install locally, or use Docker (recommended, easier to reset/clean)
- [ ] VS Code (or any editor)
- [ ] Postman or Insomnia (for testing APIs without a frontend)
- [ ] Git + a GitHub account

**Repo structure (simple version, no monorepo tooling needed):**
```
pharma-task-app/
├── backend/
├── mobile/
├── desktop/        (added in Phase 5)
└── README.md
```

---

## Phase 0 — Setup (Est. 2–3 days)

1. [ ] Create the GitHub repo with the folder structure above
2. [ ] Add a root `README.md` explaining what the project is and how to run it locally
3. [ ] Agree on Git workflow with your team:
   - `main` branch = stable
   - `feature/xyz` branches for new work
   - PRs reviewed before merging
4. [ ] Everyone runs `node -v` and `psql --version` to confirm tools are installed

**Done when:** everyone on the team can clone the repo and has Node + Postgres running.

---

## Phase 1 — Backend Basics (Est. 1 week) **[CORE]**

Goal: a working API + database, single company, no auth yet.

### Step 1: Create the NestJS project
```bash
npm i -g @nestjs/cli
cd pharma-task-app
nest new backend
cd backend
```

### Step 2: Set up PostgreSQL
Easiest way — run Postgres in Docker:
```bash
docker run --name pharma-db -e POSTGRES_PASSWORD=devpassword -e POSTGRES_DB=pharma_dev -p 5432:5432 -d postgres
```

### Step 3: Install and configure Prisma **[CORE — see note below]**
```bash
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```
This creates a `prisma/schema.prisma` file and a `.env` file. Set your `DATABASE_URL` in `.env`:
```
DATABASE_URL="postgresql://postgres:devpassword@localhost:5432/pharma_dev"
```

> Note: if your team prefers to skip Prisma and use raw SQL via the `pg` package instead, that's fine too — just replace Steps 3–5 with manual SQL table creation and hand-written queries. Prisma is recommended for speed while learning, not required.

### Step 4: Define your first schema
In `prisma/schema.prisma`:
```prisma
model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique
  password  String
  role      Role     @default(EMPLOYEE)
  tasks     Task[]   @relation("AssignedTasks")
  createdAt DateTime @default(now())
}

enum Role {
  MANAGER
  EMPLOYEE
}

model Task {
  id          String     @id @default(uuid())
  title       String
  description String?
  status      TaskStatus @default(PENDING)
  assignedTo  User?      @relation("AssignedTasks", fields: [assignedToId], references: [id])
  assignedToId String?
  createdAt   DateTime   @default(now())
}

enum TaskStatus {
  PENDING
  IN_PROGRESS
  COMPLETED
}
```

### Step 5: Run the migration
```bash
npx prisma migrate dev --name init
```
This creates the actual tables in your database.

### Step 6: Generate a NestJS module for tasks
```bash
nest generate module tasks
nest generate controller tasks
nest generate service tasks
```

### Step 7: Build the endpoints
In `tasks.service.ts`, use `PrismaClient` to implement:
- `createTask(data)`
- `getAllTasks()`
- `updateTaskStatus(id, status)`

In `tasks.controller.ts`, wire these up to:
- `POST /tasks`
- `GET /tasks`
- `PATCH /tasks/:id`

### Step 8: Test it
```bash
npm run start:dev
```
Use Postman to hit each endpoint and confirm tasks are created/listed/updated correctly.

**Done when:** you can create, list, and update tasks purely through API calls — no frontend, no login yet.

---

## Phase 2 — Authentication & Roles (Est. 3–4 days) **[CORE]**

### Step 1: Install auth packages
```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
```

### Step 2: Hash passwords on signup
Use `bcrypt.hash(password, 10)` before saving a new user.

### Step 3: Build login endpoint
`POST /auth/login`:
- Look up user by email
- Compare password with `bcrypt.compare()`
- If valid, sign a JWT containing `{ userId, role }`

### Step 4: Protect routes with Guards
```bash
nest generate guard auth/jwt-auth
```
Use `@UseGuards(JwtAuthGuard)` on routes that need login, and a custom `RolesGuard` to restrict manager-only routes (e.g. task creation).

### Step 5: Add refresh tokens **[OPTIONAL for MVP]**
Can be added later — for now, a simple JWT with a few hours' expiry is fine to get moving.

**Done when:** login returns a JWT, and task creation is blocked for non-managers.

---

## Phase 3 — React Native App, Android (Est. 1.5–2 weeks) **[CORE]**

### Step 1: Create the app
```bash
cd pharma-task-app
npx create-expo-app mobile
cd mobile
```

### Step 2: Install navigation and networking libs
```bash
npx expo install @react-navigation/native @react-navigation/native-stack
npx expo install react-native-screens react-native-safe-area-context
npm install axios
npm install @react-native-async-storage/async-storage
```

### Step 3: Build the screens (one at a time)
1. **Login screen** — email/password form → calls `POST /auth/login` → stores JWT in AsyncStorage
2. **Task List screen** — calls `GET /tasks` with JWT in header → shows list, styled differently for manager vs employee
3. **Task Detail screen** — shows one task, with an "Accept" or "Complete" button for employees
4. **Create Task screen** (manager only) — form → calls `POST /tasks`

### Step 4: Set up the API client
Create `mobile/src/api/client.ts`:
```ts
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';

const api = axios.create({ baseURL: 'http://YOUR_LOCAL_IP:3000' });

api.interceptors.request.use(async (config) => {
  const token = await AsyncStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default api;
```
(Use your machine's local network IP, not `localhost`, so a physical Android device can reach it.)

### Step 5: Run it
```bash
npx expo start
```
Scan the QR code with Expo Go on an Android phone, or run on an emulator.

**Done when:** a manager can log in, create a task, assign it; an employee can log in, see it, accept it, mark it complete — all on a real device.

---

## Phase 4 — Multi-Tenancy (Est. 1–1.5 weeks) **[CORE]**

This is the trickiest phase — take it slow.

### Step 1: Create a second, separate Prisma project for the control-plane DB
```bash
mkdir control-plane && cd control-plane
npx prisma init
```
Schema:
```prisma
model Company {
  id        String   @id @default(uuid())
  name      String
  companyId String   @unique   // this is what users type in on login
  dbUrl     String              // connection string to their tenant DB
  plan      String   @default("basic")
  createdAt DateTime @default(now())
}
```

### Step 2: Write a provisioning script
When a new company signs up:
1. Generate a new Postgres database (e.g. `tenant_<companyId>`)
2. Run your Phase 1 Prisma migrations against that new database
3. Save the company's row (with its `dbUrl`) in the control-plane DB

```bash
# Example: creating a new tenant DB from a script
psql -U postgres -c "CREATE DATABASE tenant_acme123;"
DATABASE_URL="postgresql://postgres:devpassword@localhost:5432/tenant_acme123" npx prisma migrate deploy
```
Wrap this in a Node script so it runs automatically on signup rather than by hand.

### Step 3: Build tenant resolution
`POST /auth/resolve-tenant`:
- Input: `companyId`
- Look up company in control-plane DB
- Return which tenant DB to use (internally, not exposed to the client)

### Step 4: Build a tenant-aware Prisma connection manager
```ts
const clientCache = new Map<string, PrismaClient>();

function getTenantClient(dbUrl: string): PrismaClient {
  if (!clientCache.has(dbUrl)) {
    clientCache.set(dbUrl, new PrismaClient({ datasources: { db: { url: dbUrl } } }));
  }
  return clientCache.get(dbUrl)!;
}
```
Use this inside your services instead of a single global Prisma client.

### Step 5: Update login flow
1. User submits `companyId` + email + password
2. Backend resolves tenant → looks up user in that tenant DB → verifies password
3. JWT now includes `tenantId` alongside `userId` and `role`
4. Every subsequent request extracts `tenantId` from the JWT and uses the tenant-aware client

### Step 6: Update the mobile app
Add a "Company ID" field to the login screen, sent along with email/password.

**Done when:** two test companies can both use the app at the same time, and their tasks never appear in each other's lists.

---

## Phase 5 — Desktop App (Est. 1 week)

### Step 1: Add react-native-web to your mobile project
```bash
cd mobile
npx expo install react-native-web react-dom
```

### Step 2: Create the desktop wrapper
```bash
cd pharma-task-app
mkdir desktop && cd desktop
npm init -y
npm install electron --save-dev
```

### Step 3: Point Electron at your web build
Build the Expo web output:
```bash
cd ../mobile
npx expo export:web
```
Then in `desktop/main.js`, load the exported `web-build/index.html` into an Electron `BrowserWindow`.

### Step 4: Test it
```bash
cd desktop
npx electron .
```

### Step 5: Package it **[OPTIONAL until you're ready to distribute]**
```bash
npm install electron-builder --save-dev
npx electron-builder --win --linux
```

**Done when:** the same login/task screens run inside a native-feeling desktop window.

---

## Phase 6 — Pharma-Specific Hardening (Est. 1 week)

### Step 1: Add audit log table **[CORE for pharma compliance]**
```prisma
model TaskEvent {
  id        String   @id @default(uuid())
  taskId    String
  action    String   // "created", "assigned", "accepted", "completed"
  actorId   String
  timestamp DateTime @default(now())
}
```
Write to this table on every task state change — never update or delete rows in it.

### Step 2: Add Socket.io for real-time updates **[OPTIONAL]**
```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
```
Skip this entirely if you want — a manual refresh button is a perfectly fine v1.

### Step 3: Add per-company data export **[CORE for compliance]**
A simple endpoint that dumps a company's tasks/events as JSON or CSV, for audits or offboarding.

### Step 4: Enable TLS **[CORE before going live]**
Use a reverse proxy (e.g. Caddy or Nginx) in front of your NestJS app to handle HTTPS certificates.

### Step 5: Compliance review **[CORE if targeting GxP/21 CFR Part 11 clients]**
Sit down with the client's QA/compliance team before finalizing e-signature and record-retention behavior — this is a regulatory decision, not just a technical one.

---

## Phase 7 — Distribution (Est. 3–4 days)

### Step 1: Android — sign and host your own APK
```bash
cd mobile
eas build --platform android --profile production
```
(Or use a manual React Native build if not using Expo's build service.) Host the resulting `.apk` on your own website/portal.

### Step 2: Add in-app update checking
On app launch, call your own API (`GET /app/latest-version`) and compare against the installed version; prompt the user to download the new APK if outdated.

### Step 3: Desktop — auto-update
```bash
npm install electron-updater
```
Configure it to check your own release server (not an app store) for new versions.

### Step 4: Write client install instructions
Document that Android users need to enable "install unknown apps" once — standard for any app distributed outside the Play Store.

---

## Quick Reference: Core vs Optional

| Item | Status |
|---|---|
| Node.js, NestJS/Express | **CORE** |
| PostgreSQL | **CORE** |
| React Native | **CORE** |
| JWT auth | **CORE** |
| Prisma | Recommended, not compulsory |
| Multi-tenant DB-per-company | **CORE** (this is the product's key feature) |
| Audit log (`TaskEvent`) | **CORE** for pharma |
| Socket.io / real-time | Optional, add later |
| Turborepo/Nx | Optional, add later if code duplication becomes painful |
| TLS/HTTPS | **CORE** before real client data touches it |

---

*Start here: Phase 1, Step 1. Get the NestJS project running and the first migration applied — everything else builds on that.*
