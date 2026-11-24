# Mobile Auth Integration & Branding Design

**Date:** 2026-11-24  
**Repos:** `ek_shathe_mobile` (API contract from `ek-shathe-backend`)  
**Status:** Draft for review  

## Goal

Point the Flutter client at the Ek-Shathe backend auth APIs, adopt the `{ success, data }` envelope and Bearer tokens, add a Register screen, and apply brand colors plus logos on splash/login/register.

## Decisions (agreed)

| Topic | Choice |
|-------|--------|
| Contract source | Mobile adapts to backend (not the reverse) |
| Scope | Auth wiring + brand colors + logos (not a heavy UI redesign) |
| Register UX | Separate Register screen; success → Login (no tokens on register) |
| Approach | Rewrite auth/envelope in place |

## Backend contract (source of truth)

| Action | Method | Path | Success |
|--------|--------|------|---------|
| Register | `POST` | `/api/v1/auth/register` | `201` — `{ success, message, data: { user } }` — **no tokens** |
| Login | `POST` | `/api/v1/auth/login` | `200` — `{ success, data: { user, accessToken, refreshToken } }` |
| Refresh | `POST` | `/api/v1/auth/refresh` | `200` — same token pair shape |
| Logout | `POST` | `/api/v1/auth/logout` | `200` — body `{ refreshToken }` |

**Auth header for protected calls:** `Authorization: Bearer <accessToken>` (replace legacy `x-auth-token`).

**Dev base URL:** keep flavor default aligned with local API (`http://localhost:5050/` for iOS sim / desktop; document `http://10.0.2.2:5050/` for Android emulator).

## Mobile changes

### Network / envelope

- Replace legacy `{ error, code, results }` parsing with `{ success, data, message?, errors? }`.
- Update `AuthEndpoints` to `/api/v1/auth/register|login|refresh|logout`.
- Update Retrofit `AuthApi` + Freezed/JSON DTOs for register/login/refresh/logout.
- `AuthInterceptor`: Bearer header; 401 → refresh with `{ refreshToken }`; clear session on failure.
- After login/refresh: persist tokens + user id/email from `data.user` (no `/users/me` until that API exists).
- Logout: call backend logout with stored refresh token, then clear secure storage (best-effort if network fails — still clear local).

### Domain / presentation

- Login: email + password (drop “username or email” framing where it conflicts; use email).
- Register: name + email + password; repository returns success without storing tokens; navigate to Login.
- New route `/register` + Register page/form/providers mirroring login patterns.
- Link Login ↔ Register (“Create account” / “Already have an account?”).

### Branding

| Token | Value |
|-------|--------|
| Navy (primary) | `#0B2455` |
| Orange (accent) | `#FF7900` |
| White | `#FFFFFF` |

- Update `AppPalette` (remove purple `#723DA7` brand usage).
- Copy logos from `/Users/bs1100/Downloads/ek_shathe_logos` into `assets/branding/`:
  - Full transparent → splash + auth headers
  - Symbol → compact / icon uses
- Update `AppAssets` paths; splash/login/register show the full logo.

## Out of scope

- Trip / other API modules  
- `GET /users/me` (use login/refresh user payload)  
- Full visual redesign of form layout beyond colors/logo  
- Changing backend contracts  
- Mandatory native splash/launcher regen beyond asset swap if tooling is heavy (do if straightforward)

## Success criteria

- Register against local backend creates user and returns no tokens; UI goes to Login  
- Login stores tokens, reaches home with user from response  
- Refresh on 401 works with Bearer + refresh body  
- Logout revokes refresh server-side when possible and clears local session  
- Splash/login/register use navy/orange/white and Ek-Shathe logos  
- No remaining dependency on Evolv paths (`login-or-register`, `x-auth-token`, `results` envelope) in the auth path  
