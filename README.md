# HordBox × Railway — Complete Setup Guide

## What You Have

| File | Role |
|---|---|
| `HordBoxLatest.jsx` | Main React app (TMDB frontend, 7 000+ lines) |
| `HordBoxAuth.jsx` | Auth UI component (Login/Register + ProfileScreen) |
| `hordbox-railway-server.js` | Express backend (JWT + bcrypt + PostgreSQL) |
| `hordbox-railway-package.json` | Backend npm manifest |

---

## PART 1 — Deploy the Backend on Railway

### Step 1 — Create a Railway Project

1. Go to **[railway.app](https://railway.app)** → sign up / log in
2. Click **"New Project"**
3. Choose **"Deploy from GitHub repo"** (recommended) **or** "Empty project" if you'll push manually

### Step 2 — Add a PostgreSQL Database

Inside your Railway project:

1. Click **"+ New"** → **"Database"** → **"Add PostgreSQL"**
2. Railway auto-provisions a Postgres instance and injects `DATABASE_URL` as an env var — you don't need to do anything else for the DB connection

### Step 3 — Add a Service for Your Backend

**Option A — GitHub (easiest)**

1. Push these two files to a GitHub repo:
   - `server.js` ← rename `hordbox-railway-server.js` to this
   - `package.json` ← rename `hordbox-railway-package.json` to this

2. In Railway → **"+ New"** → **"GitHub Repo"** → select your repo
3. Railway detects `package.json` and runs `npm start` automatically

**Option B — Railway CLI**

```bash
# Install CLI
npm install -g @railway/cli

# Log in
railway login

# In the folder with server.js + package.json
railway init
railway up
```

### Step 4 — Set Environment Variables

In Railway → your backend service → **"Variables"** tab, add:

| Variable | Value |
|---|---|
| `JWT_SECRET` | A long random string (e.g. `openssl rand -base64 48` in terminal) |
| `NODE_ENV` | `production` |
| `ALLOWED_ORIGIN` | Your frontend URL (e.g. `https://hordbox.vercel.app`) — use `*` while testing |

> `DATABASE_URL` is **already set automatically** by the PostgreSQL plugin. Do not touch it.

### Step 5 — Get Your Public API URL

1. Railway → backend service → **"Settings"** tab → **"Networking"**
2. Click **"Generate Domain"**
3. Copy the URL — it looks like: `https://hordbox-api-production.up.railway.app`

**Test it works:**
```
curl https://YOUR-URL.up.railway.app/
# Should return: {"status":"HordBox API running"}
```

The database table is created automatically on first boot — you'll see `✓ DB ready` in Railway logs.

---

## PART 2 — Wire the Auth into HordBoxLatest.jsx

The auth UI (`HordBoxAuth.jsx`) is already written. You need to:

1. Copy the relevant components into `HordBoxLatest.jsx`
2. Hook up the real API calls (currently commented-out stubs)
3. Gate the Profile page behind auth

### Step 6 — Add These Constants at the Top of HordBoxLatest.jsx

Right after the existing `const TMDB_KEY = ...` line, add:

```js
// ── Railway API ──────────────────────────────────────────
const API_URL = "https://YOUR-RAILWAY-URL.up.railway.app"; // ← paste your URL

const authApi = async (path, body, method = "POST") => {
  const token = localStorage.getItem("hb_token") || sessionStorage.getItem("hb_token");
  const res = await fetch(`${API_URL}${path}`, {
    method,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  const data = await res.json();
  if (!res.ok) throw new Error(data.error || "Request failed");
  return data;
};
```

### Step 7 — Copy Auth Components from HordBoxAuth.jsx

Copy these functions verbatim from `HordBoxAuth.jsx` into `HordBoxLatest.jsx`, 
**before** the `ProfilePage` function (around line 6284):

- `FloatInput` (line 444–476)
- `pwdScore` + the `STR` constant above it (line 478–489)
- `PosterMosaic` (line 491–531)
- `AuthScreen` (line 533–793)
- `ProfileScreen` (line 795–850) — optional, you can keep using HordBoxLatest's own ProfilePage

Also copy the **CSS block** from the top of `HordBoxAuth.jsx` and paste it inside 
HordBoxLatest's existing `<style>` tag (or inject it via `<style dangerouslySetInnerHTML>`).

### Step 8 — Replace the Demo Stub in AuthScreen with Real API Calls

Find this block inside `AuthScreen` → `handleSubmit`:

```js
// Demo simulation (remove when using real Railway backend)
await new Promise(r => setTimeout(r, 1300));
const user = { ... };
```

Replace the entire `try` block with:

```js
try {
  const endpoint = tab === "login" ? "/auth/login" : "/auth/register";
  const payload  = tab === "login"
    ? { email, password: pwd }
    : { username, email, password: pwd };

  const { token, user } = await authApi(endpoint, payload);

  // Persist token
  if (remember) localStorage.setItem("hb_token", token);
  else          sessionStorage.setItem("hb_token", token);

  setSuccess(user);
  setTimeout(() => onSuccess(user), 800);
} catch (e) {
  setErr(e.message || "Something went wrong. Please try again.");
} finally {
  setLoading(false);
}
```

### Step 9 — Add Auth State to the App Component

Inside `export default function App()`, add these new state variables and helpers 
right after the existing `useState` declarations (around line 6935):

```js
// ── Auth State ───────────────────────────────────────────
const [authUser, setAuthUser] = useState(() => {
  try { return JSON.parse(localStorage.getItem("hb_user")); } catch { return null; }
});

const handleLogin = (user) => {
  localStorage.setItem("hb_user", JSON.stringify(user));
  setAuthUser(user);
};

const handleLogout = () => {
  localStorage.removeItem("hb_user");
  localStorage.removeItem("hb_token");
  sessionStorage.removeItem("hb_token");
  setAuthUser(null);
};
```

### Step 10 — Gate the Profile View

Find the existing ProfilePage render (around line 7420):

```jsx
{view === "profile" && (
  <ProfilePage
    watchlistCount={watchlist.length}
    ...
  />
)}
```

Replace it with:

```jsx
{view === "profile" && (
  authUser
    ? <ProfilePage
        watchlistCount={watchlist.length}
        continueCount={Object.keys(continueWatching).length}
        likedCount={liked.length}
        likedItems={likedItems}
        theme={theme}
        toggleTheme={toggleTheme}
        onSelect={handleSelect}
        reminders={reminders}
        toggleReminder={toggleReminder}
        user={authUser}
        onLogout={handleLogout}
      />
    : <AuthScreen onSuccess={handleLogin} />
)}
```

---

## PART 3 — Sync User Data to the Backend (Optional but Recommended)

The backend has a `PUT /user/sync` endpoint that saves watchlist, likes, and ratings 
to the database so users keep their data across devices.

Add this sync call wherever you already persist to localStorage. For example, 
find your `toggleWatchlist` function and add after `store.set(...)`:

```js
// Sync to Railway backend if logged in
if (authUser) {
  authApi("/user/sync", {
    watchlist_ids: updatedWatchlist,
    liked_ids:     liked,
    ratings:       userRatings,
  }, "PUT").catch(() => {}); // silent fail — localStorage is the source of truth
}
```

---

## PART 4 — Frontend Deployment

### Deploying with Vite (Recommended)

```bash
# Create project if starting fresh
npm create vite@latest hordbox -- --template react
cd hordbox

# Replace src/App.jsx with HordBoxLatest.jsx content
# Add .env file:
echo "VITE_API_URL=https://YOUR-RAILWAY-URL.up.railway.app" > .env

# Then in HordBoxLatest.jsx change:
# const API_URL = "https://..."
# to:
# const API_URL = import.meta.env.VITE_API_URL;

npm install
npm run build    # → dist/ folder
```

Deploy the `dist/` folder to **Vercel**, **Netlify**, or **Cloudflare Pages** 
by dragging the folder or connecting your GitHub repo.

### Quick Local Test (No Build Step)

```bash
npm create vite@latest hordbox -- --template react
cd hordbox
cp /path/to/HordBoxLatest.jsx src/App.jsx
npm install
npm run dev
# → opens http://localhost:5173
```

---

## PART 5 — Checklist Before Going Live

- [ ] `JWT_SECRET` set to a long random string in Railway (not the default)
- [ ] `ALLOWED_ORIGIN` set to your actual frontend domain (not `*`)
- [ ] `API_URL` in HordBoxLatest.jsx points to your Railway service URL
- [ ] Railway logs show `✓ DB ready` on startup
- [ ] Test `/auth/register` and `/auth/login` via Postman or curl before touching the frontend
- [ ] HTTPS is enabled on Railway (it is by default on generated domains)

---

## Quick curl Tests for Your Backend

```bash
BASE="https://YOUR-RAILWAY-URL.up.railway.app"

# Register
curl -s -X POST "$BASE/auth/register" \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"secret123"}' | jq .

# Login (copy the token from above response)
curl -s -X POST "$BASE/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"secret123"}' | jq .

# Get profile (replace TOKEN)
curl -s "$BASE/auth/me" -H "Authorization: Bearer TOKEN" | jq .
```

---

## Architecture Summary

```
Browser (HordBoxLatest.jsx)
       │
       ├── TMDB API (movie/TV data)          — direct fetch, no backend needed
       │
       └── Railway Backend (server.js)
               │
               ├── POST /auth/register        → creates user, returns JWT
               ├── POST /auth/login           → verifies password, returns JWT
               ├── GET  /auth/me              → returns user profile (needs JWT)
               ├── PUT  /user/sync            → saves watchlist/likes (needs JWT)
               └── DELETE /auth/account       → deletes account (needs JWT)
                       │
                       └── PostgreSQL (Railway plugin)
                               └── users table (auto-created on startup)
```
