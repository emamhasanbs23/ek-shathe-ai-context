# Trip Management Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Phase 1 trip management on `ek-shathe-backend`: auth middleware, Trip/TripMember/JoinRequest models, create/discover/mine/detail, join requests accept/reject, members list, status transitions, OpenAPI, and Postman.

**Architecture:** Single `trips` module under modular Express CommonJS (`routes → auth → Joi → controller → service → Sequelize`). Create and accept run in transactions; accept locks the trip row for capacity safety. Bearer access JWT via shared middleware.

**Tech Stack:** Node.js, JavaScript (CommonJS), Express 5, Sequelize 6, PostgreSQL, Joi, jsonwebtoken, swagger-ui-express

**Spec:** `docs/superpowers/specs/2026-11-25-trip-management-design.md` (in `ek-shathe-ai-context`)

## Global Constraints

- Work in sibling repo: `/Users/bs1100/Documents/Ek-Shathe All/ek-shathe-backend` (ask once per session before first write; ask every time before git ops)
- Language: JavaScript CommonJS — not TypeScript
- Validation: Joi; envelopes match auth (`success` / `message` / `errors`)
- Base path: `/api/v1/trips`; all routes require Bearer access token
- Status enums: Trip `OPEN|STARTED|CANCELLED`; member role `ORGANIZER|MEMBER`; join `PENDING|ACCEPTED|REJECTED`
- Capacity ≥ 2; organizer counts as one seat; join only while `OPEN` with seats left
- Schema: keep `sequelize.sync({ alter: true })` for this slice
- Verification: curl + Postman (no Jest)
- Do not push remotes or commit sibling-repo git without explicit user permission each time

## File map

| Path | Responsibility |
|------|----------------|
| `src/middlewares/auth.middleware.js` | Verify access JWT → `req.user = { id }` |
| `src/common/validate.js` | Extend to validate `body` and/or `params` / `query` |
| `src/models/trip.model.js` | Extended trip fields + `OPEN` default |
| `src/models/tripMember.model.js` | TripMember model |
| `src/models/joinRequest.model.js` | JoinRequest model |
| `src/models/index.js` | Register models + associations |
| `src/modules/trips/trip.validation.js` | Joi schemas |
| `src/modules/trips/trip.service.js` | Business logic + transactions |
| `src/modules/trips/trip.controller.js` | HTTP ↔ service |
| `src/modules/trips/trip.routes.js` | Route wiring (`/mine`, `/discover` before `/:tripId`) |
| `src/app.js` | Mount `/api/v1/trips` |
| `src/docs/openapi.yaml` | Trip paths + schemas |
| `postman/Ek-Shathe-API.postman_collection.json` | Trip requests |

---

### Task 1: Auth middleware + validate helper

**Files:**
- Create: `src/middlewares/auth.middleware.js`
- Modify: `src/common/validate.js`

**Interfaces:**
- Consumes: `verifyAccessToken` from `src/utils/jwt.util.js`; `AppError`
- Produces: `authenticate` middleware setting `req.user = { id: Number(decoded.sub) }`; `validate({ body?, params?, query? })` factory

- [ ] **Step 1: Create auth middleware**

```js
// src/middlewares/auth.middleware.js
const AppError = require("../common/AppError");
const { verifyAccessToken } = require("../utils/jwt.util");

const authenticate = (req, res, next) => {
  const header = req.headers.authorization;
  if (!header || !header.startsWith("Bearer ")) {
    return next(new AppError("Authentication required", 401));
  }
  const token = header.slice(7).trim();
  if (!token) {
    return next(new AppError("Authentication required", 401));
  }
  try {
    const decoded = verifyAccessToken(token);
    const id = Number(decoded.sub);
    if (!Number.isInteger(id) || id < 1) {
      return next(new AppError("Invalid access token", 401));
    }
    req.user = { id };
    return next();
  } catch (err) {
    return next(new AppError("Invalid or expired access token", 401));
  }
};

module.exports = { authenticate };
```

- [ ] **Step 2: Extend validate for params/query**

Replace `src/common/validate.js` with:

```js
const validate = (schemas) => (req, res, next) => {
  // Back-compat: validate(joiSchema) still validates body only
  const map =
    schemas && typeof schemas.validate === "function"
      ? { body: schemas }
      : schemas || {};

  for (const key of ["params", "query", "body"]) {
    if (!map[key]) continue;
    const { error, value } = map[key].validate(req[key], {
      abortEarly: false,
      stripUnknown: true,
    });
    if (error) {
      error.isJoi = true;
      return next(error);
    }
    req[key] = value;
  }
  return next();
};

module.exports = validate;
```

- [ ] **Step 3: Smoke-check require**

```bash
cd "/Users/bs1100/Documents/Ek-Shathe All/ek-shathe-backend"
node -e "require('./src/middlewares/auth.middleware'); require('./src/common/validate'); console.log('ok')"
```

Expected: `ok`

- [ ] **Step 4: Commit** (only if user asked)

---

### Task 2: Models + associations

**Files:**
- Modify: `src/models/trip.model.js`
- Create: `src/models/tripMember.model.js`
- Create: `src/models/joinRequest.model.js`
- Modify: `src/models/index.js`

**Interfaces:**
- Produces: `Trip`, `TripMember`, `JoinRequest` exports; unique `(tripId,userId)` on members

- [ ] **Step 1: Extend Trip model**

Fields to add/change on existing `Trip.init`:
- `title` STRING(150) allowNull true
- `description` TEXT allowNull true
- `origin` STRING(150) allowNull true
- `capacity` INTEGER allowNull false
- `status` default `"OPEN"` (replace `"open"`)

Keep existing `destination`, `organizerId` (`organizer_id`), `startDate`, `endDate`, timestamps.

- [ ] **Step 2: Create TripMember**

```js
// src/models/tripMember.model.js
const { Model, DataTypes } = require("sequelize");
const sequelize = require("../config/database");

class TripMember extends Model {}

TripMember.init(
  {
    id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true },
    tripId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      field: "trip_id",
    },
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      field: "user_id",
    },
    role: { type: DataTypes.STRING(30), allowNull: false },
    status: {
      type: DataTypes.STRING(30),
      allowNull: false,
      defaultValue: "ACTIVE",
    },
    joinedAt: {
      type: DataTypes.DATE,
      allowNull: false,
      field: "joined_at",
    },
    createdAt: { type: DataTypes.DATE, allowNull: false },
    updatedAt: { type: DataTypes.DATE, allowNull: false },
  },
  {
    sequelize,
    tableName: "trip_members",
    modelName: "TripMember",
    indexes: [{ unique: true, fields: ["trip_id", "user_id"] }],
  }
);

module.exports = TripMember;
```

- [ ] **Step 3: Create JoinRequest**

```js
// src/models/joinRequest.model.js
const { Model, DataTypes } = require("sequelize");
const sequelize = require("../config/database");

class JoinRequest extends Model {}

JoinRequest.init(
  {
    id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true },
    tripId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      field: "trip_id",
    },
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      field: "user_id",
    },
    message: { type: DataTypes.TEXT, allowNull: true },
    status: {
      type: DataTypes.STRING(30),
      allowNull: false,
      defaultValue: "PENDING",
    },
    decidedAt: {
      type: DataTypes.DATE,
      allowNull: true,
      field: "decided_at",
    },
    decidedBy: {
      type: DataTypes.INTEGER,
      allowNull: true,
      field: "decided_by",
    },
    createdAt: { type: DataTypes.DATE, allowNull: false },
    updatedAt: { type: DataTypes.DATE, allowNull: false },
  },
  {
    sequelize,
    tableName: "join_requests",
    modelName: "JoinRequest",
  }
);

module.exports = JoinRequest;
```

- [ ] **Step 4: Wire associations in `index.js`**

Import models; add:

```js
Trip.belongsTo(User, { foreignKey: "organizerId", as: "organizer" });
User.hasMany(Trip, { foreignKey: "organizerId", as: "organizedTrips" });

Trip.hasMany(TripMember, { foreignKey: "tripId", as: "members" });
TripMember.belongsTo(Trip, { foreignKey: "tripId", as: "trip" });
User.hasMany(TripMember, { foreignKey: "userId", as: "tripMemberships" });
TripMember.belongsTo(User, { foreignKey: "userId", as: "user" });

Trip.hasMany(JoinRequest, { foreignKey: "tripId", as: "joinRequests" });
JoinRequest.belongsTo(Trip, { foreignKey: "tripId", as: "trip" });
User.hasMany(JoinRequest, { foreignKey: "userId", as: "joinRequests" });
JoinRequest.belongsTo(User, { foreignKey: "userId", as: "user" });

module.exports = {
  sequelize,
  User,
  RefreshToken,
  Trip,
  TripMember,
  JoinRequest,
};
```

- [ ] **Step 5: Commit** (only if user asked)

---

### Task 3: Trip validation + service

**Files:**
- Create: `src/modules/trips/trip.validation.js`
- Create: `src/modules/trips/trip.service.js`

**Interfaces:**
- Produces service functions: `createTrip`, `listMine`, `discover`, `getTripById`, `createJoinRequest`, `listJoinRequests`, `acceptJoinRequest`, `rejectJoinRequest`, `listMembers`, `updateTripStatus`
- All organizer checks use `trip.organizerId === authenticatedUserId`
- Seat helpers: `countActiveMembers`, `toTripDto(trip, memberCount)`

- [ ] **Step 1: Write Joi schemas** (`trip.validation.js`)

```js
const Joi = require("joi");

const tripIdParams = Joi.object({
  tripId: Joi.number().integer().positive().required(),
});

const requestIdParams = Joi.object({
  tripId: Joi.number().integer().positive().required(),
  requestId: Joi.number().integer().positive().required(),
});

const createTripSchema = Joi.object({
  destination: Joi.string().trim().min(1).max(150).required(),
  startDate: Joi.date().iso().required(),
  endDate: Joi.date().iso().min(Joi.ref("startDate")).required(),
  capacity: Joi.number().integer().min(2).required(),
  title: Joi.string().trim().max(150).allow("", null),
  description: Joi.string().trim().allow("", null),
  origin: Joi.string().trim().max(150).allow("", null),
});

const joinRequestBodySchema = Joi.object({
  message: Joi.string().trim().max(1000).allow("", null),
});

const listJoinRequestsQuery = Joi.object({
  status: Joi.string().valid("PENDING", "ACCEPTED", "REJECTED").optional(),
});

const updateStatusSchema = Joi.object({
  status: Joi.string().valid("STARTED", "CANCELLED").required(),
});

module.exports = {
  tripIdParams,
  requestIdParams,
  createTripSchema,
  joinRequestBodySchema,
  listJoinRequestsQuery,
  updateStatusSchema,
};
```

- [ ] **Step 2: Implement service** with rules from the design spec:

  - `createTrip({ userId, ...fields })` — transaction: Trip `OPEN` + TripMember `ORGANIZER`/`ACTIVE`/`joinedAt=now`
  - `listMine(userId)` — trips via memberships
  - `discover()` — `OPEN` trips where active member count `< capacity` (SQL subquery or load+filter; prefer Sequelize `literal`/`having` or two-step; document choice)
  - `getTripById(tripId)` — 404 if missing; include memberCount/seatsLeft
  - `createJoinRequest({ tripId, userId, message })` — 404 trip; 409 if not OPEN / full / already member / pending exists; else create PENDING
  - `listJoinRequests({ tripId, userId, status? })` — 403 if not organizer
  - `acceptJoinRequest({ tripId, requestId, userId })` — transaction + `Trip.findByPk(..., { lock: t.LOCK.UPDATE })`; 403/404/409 as mapped; return `{ request, member, trip }` shape from spec
  - `rejectJoinRequest` — similar without capacity/member create
  - `listMembers(tripId)` — 404 if trip missing
  - `updateTripStatus({ tripId, userId, status })` — only `OPEN` → `STARTED|CANCELLED`

Use `AppError` with correct status codes. Serialize dates via Sequelize; DTO should expose camelCase including `memberCount` and `seatsLeft`.

- [ ] **Step 3: Require service loads**

```bash
node -e "require('./src/modules/trips/trip.validation'); console.log('ok')"
```

---

### Task 4: Controller + routes + app mount

**Files:**
- Create: `src/modules/trips/trip.controller.js`
- Create: `src/modules/trips/trip.routes.js`
- Modify: `src/app.js`

**Interfaces:**
- Consumes: service + validation + `authenticate` + `sendSuccess`
- Mount: `app.use("/api/v1/trips", tripRoutes)`

- [ ] **Step 1: Controllers** — thin async handlers calling service with `req.user.id`, `req.params`, `req.body`, `req.query`; `sendSuccess(res, data, 201|200)`.

- [ ] **Step 2: Routes** — order matters:

```js
router.use(authenticate);
router.post("/", validate({ body: createTripSchema }), create);
router.get("/mine", listMine);
router.get("/discover", discover);
router.get("/:tripId", validate({ params: tripIdParams }), getById);
router.post(
  "/:tripId/join-requests",
  validate({ params: tripIdParams, body: joinRequestBodySchema }),
  createJoinRequest
);
router.get(
  "/:tripId/join-requests",
  validate({ params: tripIdParams, query: listJoinRequestsQuery }),
  listJoinRequests
);
router.post(
  "/:tripId/join-requests/:requestId/accept",
  validate({ params: requestIdParams }),
  acceptJoinRequest
);
router.post(
  "/:tripId/join-requests/:requestId/reject",
  validate({ params: requestIdParams }),
  rejectJoinRequest
);
router.get("/:tripId/members", validate({ params: tripIdParams }), listMembers);
router.patch(
  "/:tripId/status",
  validate({ params: tripIdParams, body: updateStatusSchema }),
  updateStatus
);
```

- [ ] **Step 3: Mount in `app.js`**

```js
const tripRoutes = require("./modules/trips/trip.routes");
// ...
app.use("/api/v1/trips", tripRoutes);
```

- [ ] **Step 4: Start server and hit health**

```bash
npm run start:dev
# other terminal:
curl -s http://localhost:5050/health
```

Expected: `{"success":true,"data":{"status":"ok"}}`

---

### Task 5: OpenAPI + Postman + manual verification

**Files:**
- Modify: `src/docs/openapi.yaml`
- Modify: `postman/Ek-Shathe-API.postman_collection.json`

- [ ] **Step 1: Add Trip paths/schemas** to OpenAPI (Bearer security scheme; all trip endpoints). Update `info.description` to mention trips.

- [ ] **Step 2: Add Postman folder** `Trips` with create, mine, discover, detail, join, list requests, accept, reject, members, status — plus conflict cases.

- [ ] **Step 3: Manual curl happy path**

1. Register/login two users (organizer + joiner); save access tokens  
2. Organizer `POST /api/v1/trips` → 201, `memberCount: 1`, `seatsLeft: capacity-1`  
3. Joiner `GET /discover` → includes trip  
4. Joiner `POST .../join-requests` → 201 PENDING  
5. Organizer accept → 200 ACCEPTED + MEMBER  
6. Duplicate pending / join as member → 409  
7. Non-organizer accept → 403  
8. `PATCH status` STARTED → join rejected with 409  

- [ ] **Step 4: Commit backend** (only if user asked)

---

## Spec coverage checklist

| Spec item | Task |
|-----------|------|
| Auth middleware | 1 |
| Trip / TripMember / JoinRequest + associations | 2 |
| Create + organizer seat | 3–4 |
| Mine / discover / detail | 3–4 |
| Join / list / accept / reject | 3–4 |
| Members / status | 3–4 |
| Capacity lock on accept | 3 |
| OpenAPI + Postman | 5 |
| Phase 2 expenses | Out of scope |

## Self-review notes

- No automated tests by design (matches auth / Week learning path); verify via curl/Postman.
- `validate` back-compat keeps auth routes working with `validate(schema)`.
- Discover implementation must exclude full trips; if using in-memory filter after fetch, still correct for MVP; prefer DB-side filter when practical.
