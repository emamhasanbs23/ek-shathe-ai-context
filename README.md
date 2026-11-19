# Ek-Shathe — AI Development Workspace

Shared AI workspace for **Ek-Shathe** (Bengali: doing something together) — a trip discovery and shared-expense platform. Open this repo as the editor root. It holds project standards, roadmap context, and (later) feature planning artifacts shared by sibling code repos.

> Context first. Backend and Flutter come next in sibling folders.

---

## What is this repo?

| Folder | Purpose |
|--------|---------|
| `memory-bank/` | Project brain — standards, intents, bolt artifacts |
| `docs/superpowers/` | Design specs and implementation plans |
| `.cursor/rules/` | Always-on agent policies |

**Rule:** Application code goes in sibling repos (`../ek-shathe-backend/`, later `../ek-shathe-api/`, `../ek-shathe-mobile/`). Planning and standards live here.

---

## Platform repos

| Repo | Stack | Local path | Status |
|------|-------|------------|--------|
| ek-shathe-backend | Node.js, JavaScript, Express, PostgreSQL, Sequelize | `../ek-shathe-backend/` | Active (learning / Day 5+) |
| ek-shathe-api | Node.js, JavaScript, Express, PostgreSQL, Sequelize | `../ek-shathe-api/` | Planned (formal API repo) |
| ek-shathe-mobile | Flutter | `../ek-shathe-mobile/` | After API contracts stabilize |

---

## First-time setup

1. Open this AI context folder in Cursor (or open `ek-shathe-siblings.code-workspace`).
2. Read `memory-bank/standards/` — start with `product-scope.md`, `domain-model.md`, `tech-stack.md`, `learning-roadmap.md`.
3. Backend work lives in `../ek-shathe-backend/`. Create intents under `memory-bank/intents/` as features start.

Expected layout:

```
Ek-Shathe All/                         ← parent folder (may still be named SharedExpense Tracker All on disk)
├── SharedExpenseTracker-ai-context/   ← this repo (Ek-Shathe AI context)
├── ek-shathe-backend/                 ← active backend
├── ek-shathe-api/                     ← later
└── ek-shathe-mobile/                  ← later
```

---

## Daily workflow

```bash
git pull
# edit memory-bank/ standards or intents
git add memory-bank/
git commit -m "docs: describe change"
```

Push `memory-bank/` when an intent or bolt stage is ready for reuse.

---

## Learning roadmap pointer

See `memory-bank/standards/learning-roadmap.md` for the 8-week backend plan (10–12 hours/week).
