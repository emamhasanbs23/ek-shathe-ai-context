# Auth Register / Login / Refresh / Logout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement register, login, refresh, and logout on `ek-shathe-backend` with response envelopes, Swagger UI at `/api-docs`, and a Postman collection.

**Architecture:** Modular Express CommonJS app: `routes → Joi validate → controller → service → Sequelize models`. Refresh JWTs are hashed (SHA-256) and stored in `RefreshToken` rows for rotation, logout, and reuse detection. OpenAPI YAML drives Swagger UI.

**Tech Stack:** Node.js, JavaScript (CommonJS), Express 5, Sequelize 6, PostgreSQL, Joi, bcrypt, jsonwebtoken, swagger-ui-express, yaml

**Spec:** `docs/superpowers/specs/2026-11-23-auth-signup-login-design.md` (in `ek-shathe-ai-context`)

## Global Constraints

- Work in sibling repo: `/Users/bs1100/Documents/Ek-Shathe All/ek-shathe-backend` (ask once per session before first write)
- Language: JavaScript CommonJS (`require` / `module.exports`) — not TypeScript
- Validation: Joi (not Zod for this slice)
- Base path: `/api/v1/auth`
- Success envelope: `{ success: true, data }` · Error envelope: `{ success: false, message, errors }`
- Never return `passwordHash`; never log passwords or raw tokens
- Login failures always use message `Invalid email or password`
- Verification for this learning project: `curl` + Postman (no Jest suite in this plan)
- Do not push remotes or commit sibling-repo git without explicit user permission each time
- Read sibling `CLAUDE.md` / `.cursor/rules` before writing if present

## File map

| Path | Responsibility |
|------|----------------|
| `package.json` | Add deps + keep scripts |
| `.env.example` | Document JWT + DB vars (no real secrets) |
| `src/app.js` | Express app: JSON, routes, swagger, error handler |
| `src/server.js` | DB authenticate, sync({ alter: true }), listen |
| `src/config/database.js` | Existing Sequelize instance (unchanged unless needed) |
| `src/config/swagger.js` | Load OpenAPI YAML and export swagger setup |
| `src/common/AppError.js` | Operational error with statusCode |
| `src/common/response.js` | `sendSuccess` / helpers |
| `src/common/validate.js` | Joi middleware factory |
| `src/common/errorHandler.js` | Central error middleware |
| `src/utils/password.util.js` | hash / compare |
| `src/utils/jwt.util.js` | access + refresh sign/verify |
| `src/utils/tokenHash.util.js` | SHA-256 hex of refresh token |
| `src/models/user.model.js` | Add passwordHash; email STRING(255) |
| `src/models/refreshToken.model.js` | RefreshToken model |
| `src/models/index.js` | Load models, associations, export |
| `src/modules/auth/auth.validation.js` | Joi schemas |
| `src/modules/auth/auth.service.js` | Business logic |
| `src/modules/auth/auth.controller.js` | HTTP ↔ service |
| `src/modules/auth/auth.routes.js` | Route wiring |
| `src/docs/openapi.yaml` | API contract |
| `postman/Ek-Shathe-Auth.postman_collection.json` | Manual tests |
| `postman/Ek-Shathe.local.postman_environment.json` | `baseUrl` |

---

### Task 1: Dependencies, env template, and Express app shell

**Files:**
- Modify: `package.json`
- Modify: `.env.example`
- Create: `src/common/AppError.js`
- Create: `src/common/response.js`
- Create: `src/common/errorHandler.js`
- Create: `src/app.js`
- Modify: `src/server.js`

**Interfaces:**
- Produces: `AppError`, `sendSuccess(res, data, statusCode)`, `errorHandler`, `createApp()` / `app` export, server listening on `PORT`

- [ ] **Step 1: Install packages**

```bash
cd "/Users/bs1100/Documents/Ek-Shathe All/ek-shathe-backend"
npm install joi bcrypt jsonwebtoken swagger-ui-express yaml
```

Expected: packages appear in `dependencies` in `package.json`.

- [ ] **Step 2: Update `.env.example`**

Replace contents with:

```
PORT=5000

DB_HOST=localhost
DB_PORT=5432
DB_NAME=ek_shathe
DB_USER=postgres
DB_PASSWORD=your_password

JWT_ACCESS_SECRET=change-me-access-secret-min-32-chars
JWT_REFRESH_SECRET=change-me-refresh-secret-min-32-chars
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

Ensure the developer's real `.env` has the same JWT keys set (do not commit `.env`).

- [ ] **Step 3: Create `src/common/AppError.js`**

```js
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

module.exports = AppError;
```

- [ ] **Step 4: Create `src/common/response.js`**

```js
const sendSuccess = (res, data, statusCode = 200, message) => {
  const body = { success: true, data };
  if (message) body.message = message;
  return res.status(statusCode).json(body);
};

module.exports = { sendSuccess };
```

- [ ] **Step 5: Create `src/common/errorHandler.js`**

```js
const { ValidationError } = require("joi");
const { UniqueConstraintError } = require("sequelize");
const AppError = require("./AppError");

const errorHandler = (err, req, res, next) => {
  if (res.headersSent) return next(err);

  if (err.isJoi || err instanceof ValidationError) {
    return res.status(400).json({
      success: false,
      message: "Validation failed",
      errors: (err.details || []).map((d) => ({
        path: d.path.join("."),
        message: d.message,
      })),
    });
  }

  if (err instanceof UniqueConstraintError) {
    return res.status(409).json({
      success: false,
      message: "Email already registered",
      errors: [],
    });
  }

  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      message: err.message,
      errors: [],
    });
  }

  console.error(err);
  return res.status(500).json({
    success: false,
    message: "Internal server error",
    errors: [],
  });
};

module.exports = errorHandler;
```

- [ ] **Step 6: Create `src/app.js`**

```js
const express = require("express");
const errorHandler = require("./common/errorHandler");

const createApp = () => {
  const app = express();
  app.use(express.json({ limit: "100kb" }));

  app.get("/health", (req, res) => {
    res.status(200).json({ success: true, data: { status: "ok" } });
  });

  // auth routes mounted in a later task
  app.use(errorHandler);
  return app;
};

module.exports = { createApp };
```

- [ ] **Step 7: Rewrite `src/server.js`**

```js
require("dotenv").config();

const sequelize = require("./config/database");
const { createApp } = require("./app");

const PORT = Number(process.env.PORT) || 5000;

const startServer = async () => {
  try {
    await sequelize.authenticate();
    console.log("Database connection established successfully.");

    // Models registered in later task via require("./models")
    await sequelize.sync({ alter: true });
    console.log("Database tables synchronized.");

    const app = createApp();
    app.listen(PORT, () => {
      console.log(`Server listening on port ${PORT}`);
    });
  } catch (error) {
    console.error("Unable to start server:", error);
    process.exit(1);
  }
};

startServer();
```

- [ ] **Step 8: Verify health**

```bash
npm run start:dev
# in another terminal:
curl -s http://localhost:5000/health
```

Expected: `{"success":true,"data":{"status":"ok"}}`

- [ ] **Step 9: Commit in `ek-shathe-backend` (only if user approved git)**

```bash
git add package.json package-lock.json .env.example src/app.js src/server.js src/common
git commit -m "feat: add Express app shell with envelope errors and health check"
```

---

### Task 2: Password, JWT, and token-hash utilities

**Files:**
- Create: `src/utils/password.util.js`
- Create: `src/utils/jwt.util.js`
- Create: `src/utils/tokenHash.util.js`

**Interfaces:**
- Produces:
  - `hashPassword(password: string): Promise<string>`
  - `comparePassword(password: string, hashedPassword: string): Promise<boolean>`
  - `generateAccessToken(user: { id: number }): string`
  - `generateRefreshToken(user: { id: number }): string`
  - `verifyAccessToken(token: string): object`
  - `verifyRefreshToken(token: string): object`
  - `hashToken(rawToken: string): string`

- [ ] **Step 1: Create `src/utils/password.util.js`**

```js
const bcrypt = require("bcrypt");

const SALT_ROUNDS = 12;

const hashPassword = async (password) => bcrypt.hash(password, SALT_ROUNDS);

const comparePassword = async (password, hashedPassword) =>
  bcrypt.compare(password, hashedPassword);

module.exports = { hashPassword, comparePassword };
```

- [ ] **Step 2: Create `src/utils/jwt.util.js`**

```js
const jwt = require("jsonwebtoken");

const generateAccessToken = (user) =>
  jwt.sign({ sub: user.id }, process.env.JWT_ACCESS_SECRET, {
    expiresIn: process.env.JWT_ACCESS_EXPIRES_IN || "15m",
    algorithm: "HS256",
  });

const generateRefreshToken = (user) =>
  jwt.sign({ sub: user.id, type: "refresh" }, process.env.JWT_REFRESH_SECRET, {
    expiresIn: process.env.JWT_REFRESH_EXPIRES_IN || "7d",
    algorithm: "HS256",
  });

const verifyAccessToken = (token) =>
  jwt.verify(token, process.env.JWT_ACCESS_SECRET, { algorithms: ["HS256"] });

const verifyRefreshToken = (token) => {
  const decoded = jwt.verify(token, process.env.JWT_REFRESH_SECRET, {
    algorithms: ["HS256"],
  });
  if (decoded.type !== "refresh") {
    const err = new Error("Invalid token type");
    err.name = "JsonWebTokenError";
    throw err;
  }
  return decoded;
};

module.exports = {
  generateAccessToken,
  generateRefreshToken,
  verifyAccessToken,
  verifyRefreshToken,
};
```

- [ ] **Step 3: Create `src/utils/tokenHash.util.js`**

```js
const crypto = require("crypto");

const hashToken = (rawToken) =>
  crypto.createHash("sha256").update(rawToken).digest("hex");

module.exports = { hashToken };
```

- [ ] **Step 4: Smoke-check utils in Node REPL**

```bash
node -e "
require('dotenv').config();
const { hashPassword, comparePassword } = require('./src/utils/password.util');
(async () => {
  const h = await hashPassword('secret123');
  console.log(await comparePassword('secret123', h));
})();
"
```

Expected: prints `true`

- [ ] **Step 5: Commit (if user approved)**

```bash
git add src/utils
git commit -m "feat: add password, JWT, and refresh token hash utilities"
```

---

### Task 3: User + RefreshToken models and associations

**Files:**
- Modify: `src/models/user.model.js`
- Create: `src/models/refreshToken.model.js`
- Create: `src/models/index.js`
- Modify: `src/server.js` (require models before sync)
- Keep: `src/models/trip.model.js` (still load via index if desired, or leave required from index)

**Interfaces:**
- Produces: `module.exports = { sequelize, User, RefreshToken, Trip }` from `src/models/index.js`
- `User.hasMany(RefreshToken)`; `RefreshToken.belongsTo(User)`

- [ ] **Step 1: Update `src/models/user.model.js`**

```js
const { Model, DataTypes } = require("sequelize");
const sequelize = require("../config/database");

class User extends Model {}

User.init(
  {
    id: {
      type: DataTypes.INTEGER,
      autoIncrement: true,
      primaryKey: true,
    },
    name: {
      type: DataTypes.STRING(100),
      allowNull: false,
    },
    email: {
      type: DataTypes.STRING(255),
      allowNull: false,
      unique: true,
    },
    passwordHash: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    createdAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
    updatedAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
  },
  {
    sequelize,
    tableName: "users",
    modelName: "User",
  }
);

module.exports = User;
```

- [ ] **Step 2: Create `src/models/refreshToken.model.js`**

```js
const { Model, DataTypes } = require("sequelize");
const sequelize = require("../config/database");

class RefreshToken extends Model {}

RefreshToken.init(
  {
    id: {
      type: DataTypes.INTEGER,
      autoIncrement: true,
      primaryKey: true,
    },
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
    },
    tokenHash: {
      type: DataTypes.STRING,
      allowNull: false,
      unique: true,
    },
    expiresAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
    revokedAt: {
      type: DataTypes.DATE,
      allowNull: true,
    },
    createdAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
    updatedAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
  },
  {
    sequelize,
    tableName: "refresh_tokens",
    modelName: "RefreshToken",
  }
);

module.exports = RefreshToken;
```

- [ ] **Step 3: Create `src/models/index.js`**

```js
const sequelize = require("../config/database");
const User = require("./user.model");
const RefreshToken = require("./refreshToken.model");
const Trip = require("./trip.model");

User.hasMany(RefreshToken, {
  foreignKey: "userId",
  as: "refreshTokens",
  onDelete: "CASCADE",
});
RefreshToken.belongsTo(User, {
  foreignKey: "userId",
  as: "user",
});

module.exports = { sequelize, User, RefreshToken, Trip };
```

- [ ] **Step 4: Update `src/server.js` to load models before sync**

Ensure this line exists before `sequelize.sync`:

```js
require("./models");
```

Remove direct `require("./models/user.model")` / trip requires if they are only for side-effect registration.

- [ ] **Step 5: Restart server and confirm sync**

```bash
npm run start:dev
```

Expected logs include sync without crash. If alter fails, drop/recreate DB `ek_shathe` once and retry.

- [ ] **Step 6: Commit (if user approved)**

```bash
git add src/models src/server.js
git commit -m "feat: add passwordHash and RefreshToken model with associations"
```

---

### Task 4: Auth validation middleware + register

**Files:**
- Create: `src/common/validate.js`
- Create: `src/modules/auth/auth.validation.js`
- Create: `src/modules/auth/auth.service.js` (register + shared token helpers)
- Create: `src/modules/auth/auth.controller.js`
- Create: `src/modules/auth/auth.routes.js`
- Modify: `src/app.js` (mount routes)

**Interfaces:**
- Consumes: User, RefreshToken, password/jwt/hash utils, AppError, sendSuccess
- Produces: `POST /api/v1/auth/register` → 201 envelope with user + tokens; persists refresh hash

- [ ] **Step 1: Create `src/common/validate.js`**

```js
const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, {
    abortEarly: false,
    stripUnknown: true,
  });
  if (error) {
    error.isJoi = true;
    return next(error);
  }
  req.body = value;
  return next();
};

module.exports = validate;
```

- [ ] **Step 2: Create `src/modules/auth/auth.validation.js`**

```js
const Joi = require("joi");

const registerSchema = Joi.object({
  name: Joi.string().trim().min(2).max(100).required(),
  email: Joi.string().trim().email().max(255).required(),
  password: Joi.string().min(8).max(72).required(),
});

const loginSchema = Joi.object({
  email: Joi.string().trim().email().required(),
  password: Joi.string().required(),
});

const refreshSchema = Joi.object({
  refreshToken: Joi.string().required(),
});

const logoutSchema = Joi.object({
  refreshToken: Joi.string().required(),
});

module.exports = {
  registerSchema,
  loginSchema,
  refreshSchema,
  logoutSchema,
};
```

- [ ] **Step 3: Create `src/modules/auth/auth.service.js` with register + helpers**

```js
const jwt = require("jsonwebtoken");
const { User, RefreshToken } = require("../../models");
const AppError = require("../../common/AppError");
const { hashPassword, comparePassword } = require("../../utils/password.util");
const {
  generateAccessToken,
  generateRefreshToken,
  verifyRefreshToken,
} = require("../../utils/jwt.util");
const { hashToken } = require("../../utils/tokenHash.util");

const normalizeEmail = (email) => email.trim().toLowerCase();

const toPublicUser = (user) => ({
  id: user.id,
  name: user.name,
  email: user.email,
});

const persistRefreshToken = async (user, refreshToken) => {
  const decoded = jwt.decode(refreshToken);
  await RefreshToken.create({
    userId: user.id,
    tokenHash: hashToken(refreshToken),
    expiresAt: new Date(decoded.exp * 1000),
  });
};

const issueTokenPair = async (user) => {
  const accessToken = generateAccessToken(user);
  const refreshToken = generateRefreshToken(user);
  await persistRefreshToken(user, refreshToken);
  return {
    user: toPublicUser(user),
    accessToken,
    refreshToken,
  };
};

const register = async ({ name, email, password }) => {
  const normalizedEmail = normalizeEmail(email);
  const existing = await User.findOne({ where: { email: normalizedEmail } });
  if (existing) {
    throw new AppError("Email already registered", 409);
  }

  const passwordHash = await hashPassword(password);
  const user = await User.create({
    name: name.trim(),
    email: normalizedEmail,
    passwordHash,
  });

  return issueTokenPair(user);
};

// login / refresh / logout added in later tasks — export stubs only if needed
module.exports = {
  register,
  normalizeEmail,
  toPublicUser,
  issueTokenPair,
  comparePassword,
  verifyRefreshToken,
  hashToken,
  User,
  RefreshToken,
  AppError,
};
```

Note: Prefer keeping only domain exports in the final file; temporary re-exports above are for stepwise growth — collapse to clean exports once login/refresh/logout are added:

Final exports should be: `register`, `login`, `refresh`, `logout`.

- [ ] **Step 4: Create controller + routes for register only**

`src/modules/auth/auth.controller.js`:

```js
const authService = require("./auth.service");
const { sendSuccess } = require("../../common/response");

const register = async (req, res, next) => {
  try {
    const data = await authService.register(req.body);
    return sendSuccess(res, data, 201);
  } catch (err) {
    return next(err);
  }
};

module.exports = { register };
```

`src/modules/auth/auth.routes.js`:

```js
const express = require("express");
const validate = require("../../common/validate");
const { registerSchema } = require("./auth.validation");
const authController = require("./auth.controller");

const router = express.Router();

router.post("/register", validate(registerSchema), authController.register);

module.exports = router;
```

- [ ] **Step 5: Mount in `src/app.js`**

Before `app.use(errorHandler)`:

```js
const authRoutes = require("./modules/auth/auth.routes");
app.use("/api/v1/auth", authRoutes);
```

- [ ] **Step 6: Test register with curl**

```bash
curl -s -X POST http://localhost:5000/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada Lovelace","email":"ada@example.com","password":"secret123"}'
```

Expected: HTTP 201, `success: true`, `data.user` without `passwordHash`, both tokens present.

Duplicate:

```bash
curl -s -o /tmp/out.json -w "%{http_code}" -X POST http://localhost:5000/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com","password":"secret123"}'
```

Expected: `409` and `Email already registered`.

- [ ] **Step 7: Commit (if user approved)**

```bash
git add src/common/validate.js src/modules/auth src/app.js
git commit -m "feat: add auth register endpoint with Joi validation"
```

---

### Task 5: Login

**Files:**
- Modify: `src/modules/auth/auth.service.js`
- Modify: `src/modules/auth/auth.controller.js`
- Modify: `src/modules/auth/auth.routes.js`

**Interfaces:**
- Produces: `login({ email, password })` → token pair; `POST /api/v1/auth/login`

- [ ] **Step 1: Add `login` to `auth.service.js`**

```js
const login = async ({ email, password }) => {
  const normalizedEmail = normalizeEmail(email);
  const user = await User.findOne({ where: { email: normalizedEmail } });
  if (!user) {
    throw new AppError("Invalid email or password", 401);
  }

  const ok = await comparePassword(password, user.passwordHash);
  if (!ok) {
    throw new AppError("Invalid email or password", 401);
  }

  return issueTokenPair(user);
};
```

Export `login`.

- [ ] **Step 2: Wire controller + route**

Controller:

```js
const login = async (req, res, next) => {
  try {
    const data = await authService.login(req.body);
    return sendSuccess(res, data, 200);
  } catch (err) {
    return next(err);
  }
};
```

Route:

```js
const { loginSchema } = require("./auth.validation");
router.post("/login", validate(loginSchema), authController.login);
```

- [ ] **Step 3: curl verify**

```bash
curl -s -X POST http://localhost:5000/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"secret123"}'

curl -s -o /tmp/out.json -w "%{http_code}" -X POST http://localhost:5000/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"wrong"}'
```

Expected: 200 with tokens; then `401` with `Invalid email or password`.

- [ ] **Step 4: Commit (if user approved)**

```bash
git add src/modules/auth
git commit -m "feat: add auth login endpoint"
```

---

### Task 6: Refresh (rotate + reuse detection)

**Files:**
- Modify: `src/modules/auth/auth.service.js`
- Modify: `src/modules/auth/auth.controller.js`
- Modify: `src/modules/auth/auth.routes.js`

**Interfaces:**
- Produces: `refresh({ refreshToken })` → new token pair; reuse of revoked token revokes all user refresh rows

- [ ] **Step 1: Add `refresh` to service**

```js
const { Op } = require("sequelize");

const refresh = async ({ refreshToken }) => {
  let decoded;
  try {
    decoded = verifyRefreshToken(refreshToken);
  } catch {
    throw new AppError("Invalid or expired refresh token", 401);
  }

  const tokenHash = hashToken(refreshToken);
  const stored = await RefreshToken.findOne({ where: { tokenHash } });

  if (!stored) {
    throw new AppError("Invalid or expired refresh token", 401);
  }

  if (stored.revokedAt) {
    await RefreshToken.update(
      { revokedAt: new Date() },
      { where: { userId: stored.userId, revokedAt: null } }
    );
    throw new AppError("Invalid or expired refresh token", 401);
  }

  if (stored.expiresAt.getTime() <= Date.now()) {
    throw new AppError("Invalid or expired refresh token", 401);
  }

  const user = await User.findByPk(decoded.sub);
  if (!user || user.id !== stored.userId) {
    throw new AppError("Invalid or expired refresh token", 401);
  }

  stored.revokedAt = new Date();
  await stored.save();

  return issueTokenPair(user);
};
```

Export `refresh`. Remove unused `Op` import if not used.

- [ ] **Step 2: Wire controller + `POST /refresh`**

- [ ] **Step 3: curl verify rotation**

```bash
# LOGIN_JSON=$(curl -s ... login ...)
# OLD_REFRESH=...
curl -s -X POST http://localhost:5000/api/v1/auth/refresh \
  -H 'Content-Type: application/json' \
  -d "{\"refreshToken\":\"$OLD_REFRESH\"}"
# save NEW_REFRESH, then replay OLD_REFRESH → expect 401
```

Expected: first refresh 200 with new tokens; second use of old refresh → 401.

- [ ] **Step 4: Commit (if user approved)**

```bash
git add src/modules/auth
git commit -m "feat: add refresh token rotation and reuse detection"
```

---

### Task 7: Logout

**Files:**
- Modify: `src/modules/auth/auth.service.js`
- Modify: `src/modules/auth/auth.controller.js`
- Modify: `src/modules/auth/auth.routes.js`

**Interfaces:**
- Produces: `logout({ refreshToken })` → always succeeds with 200; revokes matching row when present

- [ ] **Step 1: Add `logout` to service**

```js
const logout = async ({ refreshToken }) => {
  const tokenHash = hashToken(refreshToken);
  const stored = await RefreshToken.findOne({ where: { tokenHash } });
  if (stored && !stored.revokedAt) {
    stored.revokedAt = new Date();
    await stored.save();
  }
  return null;
};
```

- [ ] **Step 2: Controller returns `sendSuccess(res, null, 200, "Logged out")`**

- [ ] **Step 3: Route `POST /logout` with `logoutSchema`**

- [ ] **Step 4: curl verify**

Login → logout with refresh → refresh with same token → 401.

- [ ] **Step 5: Commit (if user approved)**

```bash
git add src/modules/auth
git commit -m "feat: add auth logout that revokes refresh tokens"
```

---

### Task 8: Swagger / OpenAPI

**Files:**
- Create: `src/docs/openapi.yaml`
- Create: `src/config/swagger.js`
- Modify: `src/app.js`

**Interfaces:**
- Produces: `GET /api-docs` Swagger UI covering all four auth endpoints

- [ ] **Step 1: Create `src/docs/openapi.yaml`**

Include `openapi: 3.0.3`, `info.title: Ek-Shathe API`, servers `http://localhost:5000`, paths:

- `/api/v1/auth/register`
- `/api/v1/auth/login`
- `/api/v1/auth/refresh`
- `/api/v1/auth/logout`

Define schemas: `RegisterRequest`, `LoginRequest`, `RefreshRequest`, `AuthSuccess`, `ErrorEnvelope`. Match the design spec envelopes and status codes (201/200/400/401/409).

Full YAML should be written in-repo during implementation (too long to omit — do not leave placeholders; copy field names from the design spec exactly).

- [ ] **Step 2: Create `src/config/swagger.js`**

```js
const fs = require("fs");
const path = require("path");
const YAML = require("yaml");
const swaggerUi = require("swagger-ui-express");

const loadOpenApiSpec = () => {
  const file = fs.readFileSync(
    path.join(__dirname, "../docs/openapi.yaml"),
    "utf8"
  );
  return YAML.parse(file);
};

const setupSwagger = (app) => {
  const spec = loadOpenApiSpec();
  app.use("/api-docs", swaggerUi.serve, swaggerUi.setup(spec));
};

module.exports = { setupSwagger, loadOpenApiSpec };
```

- [ ] **Step 3: Call `setupSwagger(app)` in `createApp` before error handler**

- [ ] **Step 4: Verify**

Open `http://localhost:5000/api-docs` — all four operations visible; Try it out works for register/login.

- [ ] **Step 5: Commit (if user approved)**

```bash
git add src/docs/openapi.yaml src/config/swagger.js src/app.js
git commit -m "docs: add OpenAPI spec and Swagger UI for auth endpoints"
```

---

### Task 9: Postman collection + environment

**Files:**
- Create: `postman/Ek-Shathe.local.postman_environment.json`
- Create: `postman/Ek-Shathe-Auth.postman_collection.json`

**Interfaces:**
- Produces: importable collection with the 10 requests from the design spec; env `baseUrl=http://localhost:5000`

- [ ] **Step 1: Create environment JSON**

```json
{
  "id": "ek-shathe-local",
  "name": "Ek-Shathe Local",
  "values": [
    { "key": "baseUrl", "value": "http://localhost:5000", "enabled": true },
    { "key": "accessToken", "value": "", "enabled": true },
    { "key": "refreshToken", "value": "", "enabled": true }
  ]
}
```

- [ ] **Step 2: Create collection with requests**

1. Register happy → test script saves `accessToken` / `refreshToken`  
2. Register duplicate → expect 409  
3. Register invalid → expect 400  
4. Login happy → save tokens  
5. Login wrong password → 401  
6. Refresh happy → save new tokens; keep `previousRefreshToken`  
7. Refresh reuse previous → 401  
8. Refresh garbage → 401  
9. Logout happy  
10. Refresh after logout → 401  

Use `{{baseUrl}}/api/v1/auth/...` URLs and JSON bodies from the design spec.

- [ ] **Step 3: Import into Postman and run the folder once**

Expected: all assertions pass against local server.

- [ ] **Step 4: Commit (if user approved)**

```bash
git add postman
git commit -m "test: add Postman collection for auth happy and failure paths"
```

---

### Task 10: Final polish

**Files:**
- Modify: `src/modules/auth/auth.service.js` (clean exports only: register, login, refresh, logout)
- Modify: memory-bank in ai-context only if user asks (optional: update `backend-index.md`)

- [ ] **Step 1: Ensure auth.service exports only public API functions**

- [ ] **Step 2: Manual checklist from design success criteria**

- [ ] Register / login / refresh / logout work on port 5000  
- [ ] Rotation + reuse detection  
- [ ] Logout then refresh fails  
- [ ] Envelope consistency  
- [ ] Swagger shows four endpoints  
- [ ] Postman cases pass  
- [ ] No `passwordHash` in responses  

- [ ] **Step 3: Optional ai-context backend-index update + commit (user permission)**

---

## Spec coverage self-check

| Spec requirement | Task |
|------------------|------|
| Register | 4 |
| Login | 5 |
| Refresh + rotation + reuse | 6 |
| Logout | 7 |
| RefreshToken model + hash storage | 3, 4 |
| User passwordHash | 3 |
| Envelope + AppError | 1 |
| Joi validation | 4 |
| Swagger | 8 |
| Postman | 9 |
| JWT env vars | 1 |
| Teaching order | Tasks 1→9 |

## Placeholder scan

No TBD/TODO left in task steps; OpenAPI YAML body is specified by schemas/paths to write in Task 8 (implementer must write full YAML matching the design spec — do not ship empty paths).
