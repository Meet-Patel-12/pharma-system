# Mobile App (React Native / Expo) — Step-by-Step Flow

Build order: **third**, after the backend is live and tested with Postman (`backend-build-flow.md`). Point this app at your hosted API URL, not `localhost`, from the start — saves a rewrite later.

---

## Step 1: Create the app

```bash
cd pharma-task-app
npx create-expo-app mobile
cd mobile
```

## Step 2: Install what you actually need

```bash
npx expo install @react-navigation/native @react-navigation/native-stack
npx expo install react-native-screens react-native-safe-area-context
npm install axios
npm install @react-native-async-storage/async-storage
```

That's the full list — no state-management library (React's built-in `useState`/`Context` is enough at this size), no UI kit required unless you want one for speed.

## Step 3: Folder structure

```
mobile/
├── App.tsx
├── app.json
├── package.json
└── src/
    ├── api/
    │   └── client.ts              # axios instance + JWT interceptor
    ├── context/
    │   └── AuthContext.tsx        # stores logged-in user, role, tenant, JWT
    ├── screens/
    │   ├── LoginScreen.tsx        # companyId + email + password
    │   ├── TaskListScreen.tsx
    │   ├── TaskDetailScreen.tsx
    │   └── CreateTaskScreen.tsx   # Manager/Admin only
    ├── navigation/
    │   └── AppNavigator.tsx
    └── types/
        └── index.ts               # Task, User, Role types shared across screens
```

## Step 4: API client

`src/api/client.ts`:
```ts
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';

const api = axios.create({ baseURL: 'https://<your-hosted-api-url>' });

api.interceptors.request.use(async (config) => {
  const token = await AsyncStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default api;
```

## Step 5: Build screens, in this exact order

1. **`LoginScreen.tsx`** — three fields: Company ID, email, password. Calls `POST /auth/login`, stores the returned JWT + role in `AsyncStorage` and `AuthContext`.
2. **`TaskListScreen.tsx`** — calls `GET /tasks`. Render differently based on role from context: employees see only their assigned tasks, managers/admins see everyone's.
3. **`TaskDetailScreen.tsx`** — shows one task's full detail + its `TaskEvent` history. Employees get Accept/Start/Complete buttons here.
4. **`CreateTaskScreen.tsx`** — Manager/Admin only, hidden entirely from the employee navigation stack. Form → `POST /tasks`.

Build and test each screen against your live API before starting the next one — don't build all four UI shells first and wire them up at the end.

## Step 6: Role-based navigation

In `AppNavigator.tsx`, read `role` from `AuthContext` and conditionally include `CreateTaskScreen` in the stack/tab only for `MANAGER`/`ADMIN`. Simpler than hiding a button — the screen genuinely doesn't exist in an employee's navigation.

## Step 7: Run it

```bash
npx expo start
```
Scan the QR code with Expo Go on a real Android phone (better for pharma-floor testing than an emulator — reflects actual signal/network conditions).

**Done when:** a manager logs in, creates and assigns a task; an employee logs in (same company), sees it, accepts it, completes it — all against your live hosted API, not localhost.

## Step 8: Build the distributable `.apk`

```bash
npm install -g eas-cli
eas build --platform android --profile production
```
Download the `.apk` from the build result and host it on your own site/portal for companies to download and sideload directly (no Play Store).

## Step 9: In-app update check (optional, add once you have real users)

On app launch, call `GET /app/latest-version` on your API and compare against the installed build; if outdated, show a prompt linking to the new `.apk`. This is your substitute for a Play Store's auto-update.

**Done when:** you have a working `.apk` file that installs and runs on a phone with "install unknown apps" enabled, with no dependency on `localhost` or your dev machine.
