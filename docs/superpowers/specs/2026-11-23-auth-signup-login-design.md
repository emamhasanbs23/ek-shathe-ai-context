# Auth API Design — Register, Login, Refresh, Logout

**Date:** 2026-11-23  
**Repo:** `ek-shathe-backend`  
**Status:** Draft for review  

## Goal

Build the first auth slice for Ek-Shathe: **register**, **login**, **refresh**, **logout**, **Swagger UI**, and a **Postman collection**, following Week 4 learning notes and modular Express layering. Flutter client alignment comes later.

## Decisions (agreed)

| Topic | Choice |
|-------|--------|
| Scope | Register + login + refresh + logout + Swagger + Postman |
| Contract source | Week 4 notes under `/api/v1/auth/*` |
| Validation | Joi (Week 4; memory-bank lists Zod for later consolidation) |
| Auth header (future protected routes) | `Authorization: Bearer <accessToken>` |
| Response shape | Consistent envelope for success and errors |
| Structure | Modular: routes → controllers → services → models |
| Refresh lifecycle | Persist token **hashes**; rotate on refresh; reuse detection revokes all user refresh tokens |
| Out of scope | Auth middleware on other modules, rate limiting, Redis, Helmet/CORS hardening, Flutter changes, access-token denylist |

## Architecture

```
src/
  config/          # env helpers, database, swagger wiring
  common/          # response envelope, AppError, validate middleware, error handler
  modules/auth/    # routes, controller, service, Joi schemas
  models/          # User (+ passwordHash), RefreshToken
  utils/           # bcrypt helpers, JWT helpers, token hash helper
  docs/            # openapi.yaml
  app.js           # Express app (middleware + mount routes)
  server.js        # connect DB, sync/listen
postman/           # collection + local environment
```

**Request flow**

```
HTTP
  → auth.routes
  → Joi validate middleware
  → auth.controller
  → auth.service
  → User / RefreshToken models, bcrypt, JWT
  → success or error envelope
```

## Data model

### User

| Field | Type | Notes |
|-------|------|-------|
| `id` | INTEGER PK | Existing |
| `name` | STRING(100) | Required |
| `email` | STRING(255) | Required, unique; store normalized (trim + lowercase); widen from 150 to match Joi max |
| `passwordHash` | STRING | Required; never returned in API responses |
| `createdAt` / `updatedAt` | DATE | Existing |

### RefreshToken

| Field | Type | Notes |
|-------|------|-------|
| `id` | INTEGER PK | |
| `userId` | INTEGER FK → users | Required; cascade delete on user delete |
| `tokenHash` | STRING | Required, unique; SHA-256 hex of the raw refresh JWT |
| `expiresAt` | DATE | Required |
| `revokedAt` | DATE | Null until logout, rotation, or reuse response |
| `createdAt` / `updatedAt` | DATE | |

**Associations:** `User` hasMany `RefreshToken`; `RefreshToken` belongsTo `User`.

Use `sequelize.sync({ alter: true })` for this learning slice only (migrations later). If alter fails on a messy local DB, drop/recreate the `ek_shathe` database once and restart.

## API contracts

### Shared success envelope

```json
{
  "success": true,
  "data": { }
}
```

Optional `message` may be included when useful.

### Shared error envelope

```json
{
  "success": false,
  "message": "Human-readable reason",
  "errors": []
}
```

`errors` holds field-level Joi details when validation fails; otherwise an empty array.

### Auth payload helper

Register, login, and refresh success `data` share:

```json
{
  "user": {
    "id": 1,
    "name": "Ada Lovelace",
    "email": "ada@example.com"
  },
  "accessToken": "<jwt>",
  "refreshToken": "<jwt>"
}
```

Logout success `data` may be `null` or `{}` with an optional message such as `"Logged out"`.

---

### `POST /api/v1/auth/register`

**Body**

```json
{
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "password": "secret123"
}
```

**Validation (Joi)**

- `name`: string, trim, min 2, max 100, required  
- `email`: string, trim, email, max 255, required  
- `password`: string, min 8, max 72, required  

**Behavior**

1. Validate body  
2. Normalize email  
3. If email exists → `409`  
4. Hash password with bcrypt (cost factor 12)  
5. Create user  
6. Issue access + refresh JWTs  
7. Persist refresh token **hash** + `expiresAt`  
8. Return `201` with safe user + tokens  

---

### `POST /api/v1/auth/login`

**Body**

```json
{
  "email": "ada@example.com",
  "password": "secret123"
}
```

**Validation (Joi)**

- `email`: required email  
- `password`: required  

**Behavior**

1. Validate body  
2. Normalize email; find user  
3. If missing user or `bcrypt.compare` fails → same `401` message (`Invalid email or password`)  
4. Issue access + refresh JWTs  
5. Persist refresh token hash + `expiresAt`  
6. Return `200` with safe user + tokens  

---

### `POST /api/v1/auth/refresh`

**Body**

```json
{
  "refreshToken": "<jwt>"
}
```

**Validation (Joi)**

- `refreshToken`: string, required  

**Behavior**

1. Validate body  
2. Verify refresh JWT with `JWT_REFRESH_SECRET` (algorithm HS256, expect `type: "refresh"`)  
3. Hash the presented token; look up `RefreshToken` by `tokenHash`  
4. If JWT invalid/expired, row missing, row expired, or user missing → `401` (`Invalid or expired refresh token`)  
5. If row already **revoked** (reuse of a rotated token) → revoke **all** refresh tokens for that `userId`, return `401`  
6. Otherwise **rotate**: set `revokedAt` on the old row; issue new access + refresh JWTs; persist new hash; return `200` with user + new tokens  

Refresh does **not** require an access token.

---

### `POST /api/v1/auth/logout`

**Body**

```json
{
  "refreshToken": "<jwt>"
}
```

**Validation (Joi)**

- `refreshToken`: string, required  

**Behavior**

1. Validate body  
2. Hash the presented token; find row by `tokenHash`  
3. If found and not revoked → set `revokedAt`  
4. Return `200` whether or not the row existed (idempotent; do not leak token validity beyond a generic success)  

**Note:** Access tokens remain usable until they expire (no denylist in this slice). Logout’s job is ending the **refresh** session so new access tokens cannot be minted from that refresh token.

---

### Status codes

| Situation | Status |
|-----------|--------|
| Register success | 201 |
| Login / refresh / logout success | 200 |
| Validation failure | 400 |
| Invalid credentials / invalid-or-reused refresh | 401 |
| Duplicate email | 409 |
| Unexpected server error | 500 |

## Tokens

| Token | Secret env | Default expiry | Claims |
|-------|------------|----------------|--------|
| Access | `JWT_ACCESS_SECRET` | `15m` | `sub` = user id |
| Refresh | `JWT_REFRESH_SECRET` | `7d` | `sub` = user id, `type: "refresh"` |

- Algorithm: HS256  
- Issued on register, login, and successful refresh  
- Raw refresh JWT is returned to the client once; server stores only `tokenHash`  
- Clients must not put tokens in logs; Flutter will use secure storage later  

`.env.example` additions:

```
JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

Never commit real secrets.

## Swagger

- Spec file: `src/docs/openapi.yaml`  
- UI: `GET /api-docs` via `swagger-ui-express`  
- Documents register, login, refresh, logout — bodies, envelopes, and status codes  
- Treat YAML as the human-facing contract; update it when handlers change  

## Postman

- `postman/Ek-Shathe-Auth.postman_collection.json`  
- `postman/Ek-Shathe.local.postman_environment.json` with `baseUrl=http://localhost:5000`  

**Requests to include**

1. Register — happy path (save tokens)  
2. Register — duplicate email → 409  
3. Register — invalid body → 400  
4. Login — happy path (save tokens)  
5. Login — wrong password → 401  
6. Refresh — happy path (expect new tokens; save them)  
7. Refresh — reuse old (already rotated) refresh → 401  
8. Refresh — invalid/garbage token → 401  
9. Logout — happy path  
10. Login again, then logout, then refresh with logged-out token → 401  

## Error handling

- `AppError(message, statusCode)` for expected failures  
- Central error middleware maps:
  - Joi validation → 400 + `errors`  
  - Sequelize `UniqueConstraintError` → 409  
  - `AppError` → its status  
  - Unknown → 500 with generic message  
- Do not expose `passwordHash`, raw tokens in logs, stack traces, or whether an email exists on failed login  

## Packages to add

- `joi`, `bcrypt`, `jsonwebtoken`  
- `swagger-ui-express`, `yaml` (or equivalent) to load OpenAPI  
- Hashing refresh tokens: Node `crypto.createHash('sha256')` (no extra package)  

## Teaching / build order

1. Envelope + `AppError` + error middleware + `app.js` listen  
2. User `passwordHash` + password utils + JWT utils + token-hash util  
3. `RefreshToken` model  
4. Register vertical slice → Postman happy path  
5. Login vertical slice → Postman success + 401  
6. Refresh (with rotation + reuse detection) → Postman cases  
7. Logout → Postman + “logout then refresh fails”  
8. OpenAPI + `/api-docs`  
9. Finish remaining Postman failure cases  

## Non-goals (explicit)

- `authenticate` middleware protecting trip/user modules  
- Rate limiting / Helmet / CORS hardening (later Week 4 days)  
- Access-token denylist / immediate access invalidation on logout  
- Changing Flutter to match these endpoints yet  
- Zod migration (document divergence from memory-bank; reconcile later)  

## Success criteria

- Register, login, refresh, and logout work against local Postgres on port 5000  
- Refresh rotates tokens; reusing a rotated refresh token fails and clears that user’s refresh sessions  
- Logout revokes the refresh session; a later refresh with that token fails  
- Responses always use the agreed envelope  
- Swagger UI shows and can exercise all four endpoints  
- Postman collection covers happy paths and the listed failure cases  
- Password hashes and raw refresh tokens are never stored or returned unsafely (refresh returned once to client; only hash in DB)
