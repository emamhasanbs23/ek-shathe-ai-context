# Tech Stack

## Backend (`ek-shathe-backend` / planned `ek-shathe-api`)

| Area | Choice |
|------|--------|
| Runtime / language | Node.js + **JavaScript** |
| HTTP framework | Express.js |
| Database / ORM | PostgreSQL + Sequelize |
| Validation / API docs | Zod + OpenAPI / Swagger UI |
| Authentication | Password hashing + access/refresh token flow |
| Cache / rate limiting | Redis |
| Background processing | BullMQ + Redis; scheduled jobs |
| File storage | S3-compatible object storage |
| Containerization | Docker + Docker Compose |
| Manual API verification | Postman collections and environments |

## Mobile (`ek-shathe-mobile`)

| Area | Choice |
|------|--------|
| Client | Flutter — integrate after API contracts stabilize |

## Local tooling

- DB client: pgAdmin or DBeaver
- Database name: `ek_shathe`
- Money: integer minor units (poisha for BDT); never floating-point
