# `auth.ts` — Line-by-Line, Theory-First
### The authentication router · read it until you can rewrite it by hand

> **What this file is.** It's the **auth router** — the set of HTTP endpoints that handle *who you are*:
> `login`, `refresh`, `logout`, `register`, and `me`. In `index.ts` it's mounted at `/api/auth`, so
> `router.post('/login')` becomes the real URL **`POST /api/auth/login`**.
>
> **The one job of every route here:** receive an HTTP **request**, do some work (check a password, issue a
> token, write to the database), and send back an HTTP **response** (a status code + JSON).

---

## PART 1 — The theories you must hold in your head

You cannot rewrite this file by memorizing characters. You rewrite it by **understanding six ideas** and then the code writes itself. Learn these first.

### Theory 1 — HTTP request/response & REST
- A **request** arrives with: a **method** (`POST`/`GET`), a **path** (`/login`), **headers**, and a **body** (JSON data).
- Your handler sends a **response**: a **status code** (200 ok, 400 bad request, 401 unauthorized, 409 conflict, 201 created) + a **body** (JSON).
- **REST convention:** `POST` = "create/do an action" (login, register), `GET` = "read" (me). Status codes communicate the outcome so the frontend knows what happened.

### Theory 2 — Express Router, handlers & middleware
- `Router()` is a **mini-app** that groups related routes. You attach handlers with `router.post(path, handler)`.
- A **handler** is `async (req, res) => { ... }`: `req` holds the incoming request, `res` is how you reply.
- **Middleware** is a function that runs *before* your handler (e.g. `requireAuth`). You place it between the path and the handler: `router.post('/logout', requireAuth, handler)`. It can reject the request early or attach data (like `req.auth`).

### Theory 3 — `async/await` & Promises
- Talking to a database or hashing a password is **slow** (I/O). Those functions return a **Promise** — a placeholder for a future value.
- `await` pauses the function until the Promise resolves, then gives you the value — so async code reads top-to-bottom like normal code.
- Every handler here is `async` because every one of them awaits the database or bcrypt.

### Theory 4 — JWT (JSON Web Tokens)
- A **JWT** is a signed string: `header.payload.signature`. The **payload** holds claims (e.g. `userId`, `role`). The **signature** is created with a **secret** only the server knows.
- **`jwt.sign(payload, secret, options)`** creates one. **`jwt.verify(token, secret)`** checks the signature and returns the payload — or throws if it's forged/expired.
- **Two-token pattern:** a short-lived **access token** (15 min, sent on every request to prove identity) and a long-lived **refresh token** (7 days, used only to get a new access token). Two different secrets so a leaked access token can't forge a refresh token.

### Theory 5 — bcrypt (password hashing)
- You **never store or compare plaintext passwords.** `bcrypt.hash(password, 12)` turns a password into an irreversible hash (12 = "cost", how slow/strong).
- `bcrypt.compare(plaintext, hash)` returns `true`/`false` — it re-hashes and compares safely (including the salt baked into the hash).

### Theory 6 — Cookies & the schema-per-tenant twist
- The **refresh token** is stored in an **HttpOnly cookie** — a cookie JavaScript *cannot* read, so a XSS attack can't steal it. The browser sends it automatically on requests to `/api/auth`.
- **Schema-per-tenant (BayWork-specific):** each garage has its own Postgres schema. A login only has an email — it doesn't know *which* schema. So we look the email up in the shared **`public.user_index`** table to find the `schemaName`, then run the rest inside that garage's schema via **`withTenantDb(schemaName, ...)`**.

Also lean on: **destructuring** (`const { a, b } = obj`), and the **Drizzle query builder** (`db.select().from(table).where(eq(col, val)).limit(1)` builds a SQL `SELECT ... WHERE col = val LIMIT 1`).

---

## PART 2 — The skeleton (the shape you reproduce from memory)

Every file of this type has the **same five parts, in this order**. Memorize the skeleton; the details hang off it.

```
1. IMPORTS        — framework, libraries, your own modules
2. SETUP          — const router = Router()
3. HELPERS        — small reusable functions (sign tokens, set cookie)
4. ROUTES         — router.post/get(...) for each endpoint
5. EXPORT         — export default router
```

If you can write those five headings on a blank page, you already have the backbone.

---

## PART 3 — Line by line

### Imports (lines 1–9)

```ts
import { Router, Request, Response } from 'express';   // 1
import bcrypt    from 'bcryptjs';                       // 2
import jwt       from 'jsonwebtoken';                   // 3
import { eq }    from 'drizzle-orm';                    // 4
import { publicDb, withTenantDb } from '../db/client';        // 6
import { userIndex, pendingRegistrations } from '../db/publicSchema'; // 7
import * as T    from '../db/tenantSchema';             // 8
import { requireAuth } from '../middleware/auth';       // 9
```

- **Line 1** — *named* imports from Express. `Router` (value) builds the route group; `Request`/`Response` are **TypeScript types** used to type `req`/`res`. **Rule:** import the framework pieces at the very top.
- **Line 2** — *default* import of the bcrypt library (password hashing). Default import = no `{ }`.
- **Line 3** — default import of the JWT library.
- **Line 4** — named import of `eq` ("equals") — Drizzle's helper that builds a `WHERE column = value` condition.
- **Lines 6–9** — your own modules (relative paths `../`). `publicDb` = a DB handle for the shared `public` schema; `withTenantDb` = run a callback inside one garage's schema; `userIndex`/`pendingRegistrations` = table definitions in the public schema; `* as T` = import **all** tenant tables under the namespace `T` (so `T.users`, `T.refreshTokens`); `requireAuth` = the JWT-checking middleware.
- **Rule:** third-party imports first, your own imports second. **Theory:** ES-module imports; a *namespace import* (`* as T`) bundles a whole module under one name.

### Setup (line 11)

```ts
const router = Router();
```
- Creates the router instance you'll attach every route to. **Rule:** one router per domain/resource; `index.ts` mounts it with `app.use('/api/auth', router)`. **Theory:** Express Router = a modular, mountable group of routes.

### Helper 1 — `signAccess` (lines 15–21)

```ts
function signAccess(userId, garageId, schemaName, role, email) {
  return jwt.sign(
    { userId, garageId, schemaName, role, email },   // payload (claims)
    process.env.JWT_SECRET!,                          // secret
    { expiresIn: '15m' },                             // options
  );
}
```
- Builds a **short-lived access token**. The payload is exactly the identity the rest of the app needs on every request (which user, which garage, which schema, what role). **Rule:** the access-token payload = the minimum identity your middleware/routes need — no passwords, ever.
- `process.env.JWT_SECRET!` — reads the secret from the environment. The **`!`** is TypeScript's *non-null assertion*: "trust me, this exists." **Rule:** secrets come from env vars, never hardcoded.
- `expiresIn: '15m'` — **short** on purpose: if the token leaks, it's useless in 15 minutes. **Theory:** JWT = `header.payload.signature`; signing = hashing the payload with the secret so nobody can tamper with it.

### Helper 2 — `signRefresh` (lines 23–30)

```ts
function signRefresh(userId, schemaName) {
  return jwt.sign(
    { userId, schemaName },              // minimal payload
    process.env.JWT_REFRESH_SECRET!,     // DIFFERENT secret
    { expiresIn: '7d' },                 // long-lived
  );
}
```
- The **refresh token**. Two deliberate differences from the access token: (1) **minimal payload** — just enough (`userId` + `schemaName`) to route the refresh; (2) a **separate secret** (`JWT_REFRESH_SECRET`). **Rule:** access and refresh tokens use *different* secrets, so a stolen access token can't be used to mint refresh tokens. `7d` = long-lived, because its only job is to renew access tokens.

### Helper 3 — `setRefreshCookie` (lines 32–40)

```ts
res.cookie('refresh_token', token, {
  httpOnly: true,                                  // JS can't read it → XSS-safe
  secure:   process.env.NODE_ENV === 'production', // HTTPS-only in prod
  sameSite: 'strict',                              // not sent cross-site → CSRF defense
  maxAge:   7 * 24 * 60 * 60 * 1000,               // 7 days in milliseconds
  path:     '/api/auth',                           // only sent to auth routes
});
```
- Stores the refresh token in a cookie with four security flags. **Rules to memorize:**
  - `httpOnly: true` → the browser hides it from JavaScript, so an XSS script can't steal it.
  - `secure` → only sent over HTTPS (on in production, off on localhost so dev works).
  - `sameSite: 'strict'` → the browser won't attach it to requests coming from other sites → **CSRF** defense.
  - `maxAge` → lifetime in **milliseconds** (`7*24*60*60*1000`). Match it to the token's `7d` expiry.
  - `path: '/api/auth'` → the cookie is only sent to auth endpoints, not every request.
- **Theory:** access token → lives in the client's memory and is sent in the `Authorization` header; refresh token → lives in an HttpOnly cookie the browser manages. This split is the standard secure pattern.

---

### Route: `POST /login` (lines 49–112)

```ts
router.post('/login', async (req, res) => {
  const { email, password } = req.body;                 // 50
  if (!email || !password) {                            // 51
    res.status(400).json({ error: 'Email and password required' });
    return;
  }
```
- **49** — declare the route + an `async` handler (async because we'll `await` the DB and bcrypt).
- **50** — pull `email` and `password` out of the request body. **Theory:** `req.body` is populated by the `express.json()` middleware (set up in `index.ts`) which parsed the incoming JSON.
- **51–54** — **guard clause:** if either field is missing, reply **400 Bad Request** and `return` to stop. **Rule:** validate inputs first and *return early* — never continue with bad data.

```ts
  const [index] = await publicDb                        // 57
    .select().from(userIndex)
    .where(eq(userIndex.email, email.toLowerCase().trim()))
    .limit(1);
  if (!index) {                                         // 63
    res.status(401).json({ error: 'Invalid credentials' });
    return;
  }
```
- **57–61 (Step 1)** — look the email up in `public.user_index` to discover **which garage schema** it belongs to. `email.toLowerCase().trim()` **normalizes** the email (users type inconsistent casing/spaces). `.limit(1)` = at most one row; `const [index] = ...` **destructures the first element** of the returned array. **Theory:** this lookup exists *only because of schema-per-tenant* — without it the server wouldn't know which schema to check.
- **63–66** — if no match, reply **401** with a **generic** "Invalid credentials". **Rule (security):** never say "email not found" vs "wrong password" — that lets attackers **enumerate** which emails exist. Same message for both.

```ts
  let user: typeof T.users.$inferSelect | undefined;    // 69
  await withTenantDb(index.schemaName, async (db) => {  // 71
    const [row] = await db.select().from(T.users)
      .where(eq(T.users.email, email.toLowerCase().trim())).limit(1);
    user = row;
  });
  if (!user || !user.active) { /* 401 */ }              // 80
```
- **69** — declare `user`; the type `typeof T.users.$inferSelect` is **Drizzle inferring the row shape** of the users table automatically. `| undefined` because it may not be found.
- **71–78 (Step 2)** — `withTenantDb` runs the callback **inside that garage's schema**. We find the user by email there and assign it to the outer `user`. **Theory:** the same query hits a *different physical table* depending on `schemaName` — that's the whole tenant-isolation mechanism.
- **80–83** — if there's no user, or the account is deactivated (`active` is false), reply 401 (again generic).

```ts
  const valid = await bcrypt.compare(password, user.passwordHash); // 85
  if (!valid) { /* 401 */ }
```
- **85–89** — **verify the password.** `bcrypt.compare` hashes the submitted password and checks it against the stored hash. **Rule:** never compare passwords directly — bcrypt handles the salt and does it in constant time. False → 401.

```ts
  const accessToken  = signAccess(user.id, index.garageId, index.schemaName, user.role, user.email); // 92
  const refreshToken = signRefresh(user.id, index.schemaName);
  const expiresAt    = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000);
  await withTenantDb(index.schemaName, async (db) => {  // 96
    await db.insert(T.refreshTokens).values({ userId: user!.id, token: refreshToken, expiresAt });
  });
  setRefreshCookie(res, refreshToken);                  // 100
  res.json({ accessToken, user: { ...fields } });       // 101
```
- **92–94 (Step 3)** — the password checked out, so **issue both tokens** and compute the refresh token's DB expiry (`Date.now()` is "now" in ms; add 7 days).
- **96–98** — **store the refresh token in the tenant's `refresh_tokens` table.** **Theory:** JWTs are stateless and normally can't be cancelled. By saving the refresh token in the DB, you can **revoke** it (delete the row) — e.g. on logout or "log out everywhere". `user!.id` — the **`!`** tells TypeScript `user` is definitely not undefined here.
- **100** — set the HttpOnly refresh cookie.
- **101–111** — reply **200** (default) with the **access token** (the frontend keeps it in memory/sessionStorage) and a safe subset of user fields. **Note:** the refresh token is *not* in the JSON — it's in the cookie only.

**Login mental model:** *find the schema (user_index) → find the user in that schema → check the password → hand back a short access token + a stored, cookie-carried refresh token.*

---

### Route: `POST /refresh` (lines 122–179)

Purpose: the access token expires every 15 min; this endpoint issues a fresh one **without** re-asking for the password, using the refresh cookie — and **rotates** the refresh token for extra safety.

```ts
  const token = req.cookies?.refresh_token;             // 123
  if (!token) { /* 401 'No refresh token' */ }
  let payload;
  try { payload = jwt.verify(token, process.env.JWT_REFRESH_SECRET!); } // 131
  catch { /* 401 'Invalid refresh token' */ }
  const { userId, schemaName } = payload;               // 137
```
- **123** — read the cookie. `req.cookies` is populated by the `cookie-parser` middleware; `?.` (optional chaining) avoids a crash if there are no cookies.
- **131–135** — **verify** the token with the refresh secret. If it's forged or expired, `jwt.verify` **throws**, and the `catch` replies 401. **Rule:** always wrap `jwt.verify` in try/catch — an invalid token is an exception, not a return value.
- **137** — pull `userId` + `schemaName` out of the verified payload — now we know which schema to enter.

```ts
  await withTenantDb(schemaName, async (db) => {        // 139
    const [stored] = await db.select().from(T.refreshTokens)
      .where(eq(T.refreshTokens.token, token)).limit(1);
    if (!stored || stored.expiresAt < new Date()) { /* 401 revoked/expired */ } // 146
    const [user] = await db.select().from(T.users).where(eq(T.users.id, userId)).limit(1); // 151
    if (!user || !user.active) { /* 401 */ }

    await db.delete(T.refreshTokens).where(eq(T.refreshTokens.token, token)); // 163 delete old
    const newRefresh = signRefresh(user.id, schemaName);                       // 164 make new
    const expiresAt  = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000);
    await db.insert(T.refreshTokens).values({ userId: user.id, token: newRefresh, expiresAt }); // 166 store new

    const [idx] = await publicDb.select({ garageId: userIndex.garageId }) // 169
      .from(userIndex).where(eq(userIndex.email, user.email)).limit(1);
    const accessToken = signAccess(user.id, idx.garageId, schemaName, user.role, user.email); // 175
    setRefreshCookie(res, newRefresh);                  // 176
    res.json({ accessToken });                          // 177
  });
```
- **139–149** — even though the JWT signature is valid, we **check the DB** that this exact token still exists and isn't past `expiresAt`. **Theory:** this is the **revocation** check — a logged-out or rotated token was deleted from the table, so it's rejected here even though its signature is fine. This is what makes stateless JWTs revocable.
- **151–160** — reload the user (still exists / still active).
- **162–166 — token rotation:** delete the old refresh token and issue a brand-new one. **Rule (security):** rotating on every refresh means a stolen refresh token only works once; if the real user and an attacker both use it, the reuse is detectable and you can revoke the family.
- **169–173** — the refresh payload didn't carry `garageId`, so fetch it from `user_index` (the access token needs it).
- **175–177** — mint a new access token, set the new refresh cookie, and return the access token.

---

### Route: `POST /logout` (lines 183–192)

```ts
router.post('/logout', requireAuth, async (req, res) => {   // 183
  const token = req.cookies?.refresh_token;
  if (token) {
    await withTenantDb(req.auth.schemaName, async (db) => {  // 186
      await db.delete(T.refreshTokens).where(eq(T.refreshTokens.token, token));
    });
  }
  res.clearCookie('refresh_token', { path: '/api/auth' });   // 190
  res.json({ ok: true });
});
```
- **183** — note the **`requireAuth` middleware** between the path and handler: you must be logged in to log out, and it attaches **`req.auth`** (the decoded access-token payload) to the request.
- **186–188** — delete the refresh token from the DB → it can never be used to refresh again (true server-side logout, thanks to storing tokens). `req.auth.schemaName` tells us which schema to delete from.
- **190** — clear the cookie in the browser (must use the same `path` it was set with).
- **Theory:** logout = *invalidate the refresh token server-side* + *clear the cookie client-side*. (The access token still works for up to 15 min — an accepted trade-off of JWTs.)

---

### Route: `POST /register` (lines 198–236)

Purpose: public sign-up. It does **not** create a live garage — it creates a **pending record** an admin later approves.

```ts
  const { garageName, ownerName, email, phone, address, city, plan, password } = req.body; // 199
  if (!garageName || !ownerName || !email || !phone || !password) { /* 400 */ }            // 201
  const existing = await publicDb.select({ id: pendingRegistrations.id })                   // 206
    .from(pendingRegistrations).where(eq(pendingRegistrations.email, email.toLowerCase().trim())).limit(1);
  if (existing.length > 0) { /* 409 'Email already registered' */ }                         // 212
  const passwordHash = await bcrypt.hash(password, 12);                                      // 217
  const refNumber    = 'BW-' + Date.now().toString(36).toUpperCase();                        // 218
  await publicDb.insert(pendingRegistrations).values({ ...fields, passwordHash, refNumber });// 220
  res.status(201).json({ message: 'Registration received — pending approval', refNumber });  // 232
```
- **199–204** — destructure all the sign-up fields; guard the required ones → 400 if missing.
- **206–215** — check for a **duplicate** pending registration by email. If found, reply **409 Conflict** (the correct status for "already exists"). **Rule:** 409 = conflict with current state.
- **217** — **hash the password now** (cost 12), so the plaintext is never stored even in the pending table.
- **218** — generate a human reference number. `Date.now().toString(36)` encodes the timestamp in base-36 (compact, alphanumeric), uppercased → e.g. `BW-LZ4F9K`. **Rule:** give users a tracking code.
- **220–230** — insert the pending row. `plan: plan || 'pro'` = default to `'pro'` if none was sent. **Theory:** the approval flow (admin later provisions the real schema) lives in `admin.ts` — register only *queues* the request.
- **232** — reply **201 Created** (correct status for "a new resource was created") + the ref number.

---

### Route: `GET /me` (lines 240–262)

Purpose: "who am I?" — the frontend calls this on load to get the current user.

```ts
router.get('/me', requireAuth, async (req, res) => {        // 240
  await withTenantDb(req.auth.schemaName, async (db) => {
    const [user] = await db.select({ id, name, email, role, phone, createdAt })
      .from(T.users).where(eq(T.users.id, req.auth.userId)).limit(1);
    if (!user) { /* 404 */ }
    res.json({ ...user, garageId: req.auth.garageId, schemaName: req.auth.schemaName });
  });
});
```
- **240** — a **`GET`** (reading data), protected by `requireAuth`. Identity comes from `req.auth` (the verified access token) — **no password involved**.
- **242–253** — select a **safe subset** of columns (note: *not* `passwordHash`) for the current user, found by `req.auth.userId` inside their schema. **Rule:** never select/return the password hash.
- **255–257** — 404 if the user vanished.
- **260** — merge the DB row with `garageId`/`schemaName` from the token and return it.

### Export (line 264)

```ts
export default router;
```
- **Default export** so `index.ts` can `import authRouter from './routes/auth'` and mount it. **Rule:** a route file exports its router as the default.

---

## PART 4 — Standard vs project-specific (what to keep vs change)

When you rebuild this in a *different* project, most of it is reusable boilerplate; only a few lines are BayWork-specific.

| Reusable in ANY app (memorize as the pattern) | BayWork-specific (would change per project) |
|---|---|
| Router setup, `signAccess`/`signRefresh`/`setRefreshCookie` helpers | The `user_index` lookup (only needed for schema-per-tenant) |
| Two-token JWT pattern + separate secrets | `withTenantDb(schemaName, …)` wrapping every DB call |
| bcrypt hash on register / compare on login | `pending_registrations` + admin-approval flow (vs. instant signup) |
| Storing refresh tokens in DB for revocation + rotation on refresh | The exact access-token claims (`garageId`, `schemaName`) |
| Guard clauses, generic 401s, correct status codes (400/401/409/201/404) | `refNumber` format |
| HttpOnly/secure/sameSite cookie flags | — |
| `requireAuth` on protected routes (`logout`, `me`) | — |

---

## PART 5 — Now write it yourself (the exam)

Close this file. On a blank page, reproduce `auth.ts` using only this checklist:

1. **Imports:** Express (`Router`, `Request`, `Response`), `bcrypt`, `jwt`, `eq`; your `publicDb`/`withTenantDb`, table defs, `requireAuth`.
2. `const router = Router()`.
3. **Three helpers:** `signAccess` (all claims, `JWT_SECRET`, 15m) · `signRefresh` (minimal, `JWT_REFRESH_SECRET`, 7d) · `setRefreshCookie` (httpOnly, secure, sameSite, maxAge, path).
4. **`POST /login`:** validate → find schema in `user_index` → find user in tenant schema → `bcrypt.compare` → issue tokens → store refresh in DB → set cookie → return `{ accessToken, user }`. Generic 401s throughout.
5. **`POST /refresh`:** read cookie → `jwt.verify` in try/catch → check token row in DB (revocation) → reload user → **rotate** (delete old, insert new) → get `garageId` → new access token + cookie.
6. **`POST /logout`** (`requireAuth`): delete refresh token from DB → clear cookie.
7. **`POST /register`:** validate → duplicate check (409) → `bcrypt.hash(…, 12)` → make `refNumber` → insert pending → 201.
8. **`GET /me`** (`requireAuth`): select safe user fields from tenant schema → return.
9. `export default router`.

**Self-test (answer out loud):**
- Why two different JWT secrets?
- Why store refresh tokens in the database if JWTs are stateless?
- Why is the login error message identical for "no such email" and "wrong password"?
- What do `httpOnly`, `secure`, and `sameSite: 'strict'` each defend against?
- Why does every DB call go through `withTenantDb`?
- What status code do you return for a duplicate registration, and why?
- Why is the access token 15 minutes but the refresh token 7 days?

When you can write all 9 parts cold and answer all 7 questions, you own this file. ✅
