# Trip Management Design — Create, Discover, Join, Accept/Reject

**Date:** 2026-11-25  
**Repo:** `ek-shathe-backend`  
**Status:** Draft for review  
**Phase:** 1 of 2 (Phase 2 = expenses / balances / settlements — separate design)

## Goal

Build the trip membership foundation for Ek-Shathe: organizers create trips, authenticated users discover trips with open seats, non-members submit join requests, organizers accept or reject, and accepted users become `TripMember`s. This unlocks Phase 2 financial features that depend on real trip membership.

## Decisions (agreed)

| Topic | Choice |
|-------|--------|
| Phasing | Trip management first; expenses / settlements after |
| Module shape | Single `trips` module (Approach 1) |
| Discovery | Authenticated-only list of `OPEN` trips with seats remaining |
| Discovery filters | Deferred |
| Join entry | By trip ID (from discovery or shared ID); no invite codes yet |
| Create fields | Required: destination, startDate, endDate, capacity. Optional: title, description, origin |
| Join window | Only while trip status is `OPEN` and seats remain |
| Capacity | Organizer counts as 1 seat (`capacity` ≥ 2) |
| Auth | Bearer access JWT on all trip endpoints; add shared auth middleware |
| Validation | Joi (match existing auth module) |
| Schema | Continue `sequelize.sync({ alter: true })` for this learning slice |
| Out of scope | Leave trip, remove member, invitations, announcements, public unauthenticated browse, expense/settlement APIs, formal migrations, automated test suite |

## Architecture

```
src/
  middlewares/
    auth.middleware.js     # Bearer access token → req.user
  models/
    trip.model.js          # extended
    tripMember.model.js    # new
    joinRequest.model.js   # new
    index.js               # associations
  modules/trips/
    trip.routes.js
    trip.validation.js
    trip.controller.js
    trip.service.js
  docs/openapi.yaml        # extended
  app.js                   # mount /api/v1/trips
postman/                   # trip collection or extended collection
```

**Request flow**

```
HTTP
  → trip.routes
  → auth middleware (req.user)
  → Joi validate (body / params / query as needed)
  → trip.controller
  → trip.service (transactions for create + accept)
  → Trip / TripMember / JoinRequest
  → success or error envelope
```

Layering matches auth: controllers stay HTTP-thin; services own business rules and transactions.

## Data model

### Trip (extend existing)

| Field | Type | Notes |
|-------|------|-------|
| `id` | INTEGER PK | Existing |
| `title` | STRING(150) | Optional |
| `description` | TEXT | Optional |
| `origin` | STRING(150) | Optional |
| `destination` | STRING(150) | Required (existing) |
| `startDate` | DATE | Required; DB `start_date` |
| `endDate` | DATE | Required; DB `end_date`; must be ≥ `startDate` |
| `capacity` | INTEGER | Required; ≥ 2; CHECK/app validation |
| `organizerId` | INTEGER | Required; FK intent → users; DB `organizer_id` |
| `status` | STRING(30) | `OPEN` \| `STARTED` \| `CANCELLED`; default `OPEN` (replace loose `"open"` default) |
| `createdAt` / `updatedAt` | DATE | Existing |

### TripMember

| Field | Type | Notes |
|-------|------|-------|
| `id` | INTEGER PK | |
| `tripId` | INTEGER | Required; DB `trip_id` |
| `userId` | INTEGER | Required; DB `user_id` |
| `role` | STRING(30) | `ORGANIZER` \| `MEMBER` |
| `status` | STRING(30) | `ACTIVE` for this slice |
| `joinedAt` | DATE | Set on create/accept; DB `joined_at` |
| `createdAt` / `updatedAt` | DATE | |

**Constraint:** unique `(tripId, userId)`.

### JoinRequest

| Field | Type | Notes |
|-------|------|-------|
| `id` | INTEGER PK | |
| `tripId` | INTEGER | Required; DB `trip_id` |
| `userId` | INTEGER | Required; DB `user_id` |
| `message` | TEXT | Optional |
| `status` | STRING(30) | `PENDING` \| `ACCEPTED` \| `REJECTED` |
| `decidedAt` | DATE | Null until accept/reject; DB `decided_at` |
| `decidedBy` | INTEGER | Organizer user id on decision; DB `decided_by` |
| `createdAt` / `updatedAt` | DATE | |

**Constraints / rules:**
- At most one `PENDING` request per `(tripId, userId)` (enforce in service; unique partial index later if/when migrations land).
- Already an active `TripMember` → cannot create a join request.
- Organizer is already a member and never submits a join request for their own trip.
- After `REJECTED`, the same user may submit a new join request as a **new** `PENDING` row (keep prior rows for history).
- After `ACCEPTED`, the user is a member; further join requests are rejected.

### Associations

- `Trip.hasMany(TripMember)` / `TripMember.belongsTo(Trip)`
- `Trip.hasMany(JoinRequest)` / `JoinRequest.belongsTo(Trip)`
- `User.hasMany(TripMember)` / `TripMember.belongsTo(User)`
- `User.hasMany(JoinRequest)` / `JoinRequest.belongsTo(User)`
- `Trip.belongsTo(User, { as: "organizer", foreignKey: "organizerId" })`

## Business rules

1. **Create trip:** Authenticated user becomes `organizerId`. In one transaction: insert `Trip` (`OPEN`) + `TripMember` (`ORGANIZER`, `ACTIVE`). One seat used immediately.
2. **Seats:** `seatsLeft = capacity - activeMemberCount`. Discoverable iff `status === OPEN` and `seatsLeft > 0`.
3. **Join request:** Caller must not be a member; trip must be `OPEN` with `seatsLeft > 0`; no existing `PENDING` request for same user/trip.
4. **Accept:** Organizer only. Request must be `PENDING`. Re-check capacity inside a transaction with a trip row lock (`LOCK.UPDATE` or equivalent). Create `TripMember` (`MEMBER`) + set request `ACCEPTED` with `decidedAt` / `decidedBy`. If capacity would be exceeded → `409`.
5. **Reject:** Organizer only. Request must be `PENDING` → `REJECTED` with decision metadata.
6. **Status change:** Organizer only. Allowed transitions: `OPEN` → `STARTED`, `OPEN` → `CANCELLED`. No reopen / no `STARTED` → `OPEN` in this slice. After `STARTED` or `CANCELLED`, new join requests are rejected.
7. **Identity:** Always use `req.user.id` from auth middleware — never trust a client-supplied user id as the actor.

## API contracts

Base path: `/api/v1/trips`  
Auth: `Authorization: Bearer <accessToken>` on every endpoint below.  
Success: `{ success: true, data }` (optional `message`).  
Error: `{ success: false, message, errors }` (Joi fills `errors`).

| Method | Path | AuthZ | Success | Notes |
|--------|------|-------|---------|-------|
| `POST` | `/` | any user | 201 | Create trip + organizer member |
| `GET` | `/mine` | any user | 200 | Trips where caller is active member |
| `GET` | `/discover` | any user | 200 | `OPEN` + seats left; no filters yet |
| `GET` | `/:tripId` | any user | 200 | Detail + `memberCount` + `seatsLeft` |
| `POST` | `/:tripId/join-requests` | non-member | 201 | Body may include optional `message` |
| `GET` | `/:tripId/join-requests` | organizer | 200 | Optional `?status=PENDING\|ACCEPTED\|REJECTED` |
| `POST` | `/:tripId/join-requests/:requestId/accept` | organizer | 200 | Capacity-safe accept |
| `POST` | `/:tripId/join-requests/:requestId/reject` | organizer | 200 | Reject pending request |
| `GET` | `/:tripId/members` | any user | 200 | Active members |
| `PATCH` | `/:tripId/status` | organizer | 200 | Body `{ status: "STARTED" \| "CANCELLED" }` |

Register static paths (`/mine`, `/discover`) **before** `/:tripId` so they are not captured as ids.

### Create body (Joi)

```json
{
  "destination": "Cox's Bazar",
  "startDate": "2026-12-10",
  "endDate": "2026-12-14",
  "capacity": 4,
  "title": "Weekend by the sea",
  "description": "Optional notes",
  "origin": "Dhaka"
}
```

`title`, `description`, `origin` optional. `capacity` integer ≥ 2. Dates: `endDate` ≥ `startDate`.

### Representative responses

**Create / detail (shape sketch):**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "Weekend by the sea",
    "description": "Optional notes",
    "origin": "Dhaka",
    "destination": "Cox's Bazar",
    "startDate": "2026-12-10T00:00:00.000Z",
    "endDate": "2026-12-14T00:00:00.000Z",
    "capacity": 4,
    "organizerId": 7,
    "status": "OPEN",
    "memberCount": 1,
    "seatsLeft": 3
  }
}
```

**Accept join request `data` shape:**

```json
{
  "request": {
    "id": 10,
    "tripId": 1,
    "userId": 22,
    "status": "ACCEPTED",
    "decidedAt": "2026-11-25T12:00:00.000Z",
    "decidedBy": 7
  },
  "member": {
    "id": 3,
    "tripId": 1,
    "userId": 22,
    "role": "MEMBER",
    "status": "ACTIVE"
  },
  "trip": {
    "id": 1,
    "memberCount": 2,
    "seatsLeft": 2,
    "capacity": 4,
    "status": "OPEN"
  }
}
```

## Error mapping

| Case | Status |
|------|--------|
| Joi validation failure | 400 |
| Missing/invalid access token | 401 |
| Not organizer when required | 403 |
| Trip or join request not found | 404 |
| Already a member; duplicate pending request; trip full; trip not `OPEN`; invalid status transition; accept/reject non-pending | 409 |
| Unexpected failure | 500 (generic message; log server-side) |

Unique constraint races surface as 409 via existing `UniqueConstraintError` handling where applicable.

## Auth middleware

Add `src/middlewares/auth.middleware.js`:

- Read `Authorization: Bearer <token>`
- Verify access JWT with existing `jwt.util`
- Attach `req.user = { id, ...safe claims }`
- On failure → `AppError` 401

Trip routes use this middleware at router level. No refresh-token acceptance on protected routes.

## Concurrency

Accept path runs in a managed Sequelize transaction:

1. Lock trip row.
2. Load pending join request for that trip.
3. Count active members; if `count >= capacity` → 409.
4. Create `TripMember`; update join request.
5. Commit.

Create path: trip + organizer member in one transaction (rollback if either fails).

## Documentation & verification

- Extend `src/docs/openapi.yaml` with trip endpoints (Swagger UI already wired).
- Add/extend Postman collection for the happy path and conflict cases (full trip, duplicate pending, non-organizer accept, join when `STARTED`).
- Manual verification with Postman / curl — no Jest/Vitest in this slice (matches Week learning approach and auth plan).

## Phase 2 preview (not in this design)

After Phase 1 ships: `Expense`, `ExpenseParticipant`, balance derivation, debt simplification, `Settlement` / `SettlementPayment` (`PENDING` → `PAID` → `CONFIRMED`), per Week 5 summary. Those APIs will require active `TripMember` checks built here.

## Implementation notes

- Work in sibling repo `ek-shathe-backend`; ask once per session before first write; ask every time before git ops.
- Prefer extending existing patterns (CommonJS, Joi, `AppError`, `sendSuccess`, snake_case `field` mappings).
- Normalize trip `status` values to uppercase enums above; migrate the existing default `"open"` to `OPEN` as part of model update.
- Keep controllers free of SQL/ORM details; keep pure helpers minimal unless repeated seat-count logic needs a small util.
