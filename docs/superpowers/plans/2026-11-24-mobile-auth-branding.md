# Mobile Auth Integration & Branding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `ek_shathe_mobile` to Ek-Shathe backend auth (`/api/v1/auth/*`, `{ success, data }`, Bearer), add a Register screen, and apply navy/orange/white branding with logos.

**Architecture:** Rewrite auth DTOs, envelope helpers, Retrofit `AuthApi`, repository, and interceptor in place. Persist user from login/refresh payloads (no `/me`). Brand via `AppPalette` + assets under `assets/branding/`.

**Tech Stack:** Flutter, Dio, Retrofit, Freezed, Riverpod, GoRouter, flutter_secure_storage

**Spec:** `docs/superpowers/specs/2026-11-24-mobile-auth-branding-design.md`

## Global Constraints

- Work in sibling repo: `/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile` (ask once per session before first write)
- Match backend: register returns user only; login/refresh return `user` + `accessToken` + `refreshToken`
- Header: `Authorization: Bearer <accessToken>` (not `x-auth-token`)
- Envelope: `{ success, data, message?, errors? }`
- Brand: navy `#0B2455`, orange `#FF7900`, white `#FFFFFF`
- Logos from `/Users/bs1100/Downloads/ek_shathe_logos`
- Run `dart run build_runner build --delete-conflicting-outputs` after Freezed/Retrofit/Riverpod changes
- Git in sibling: ask before every commit
- Prefer feature branch `feat/mobile-auth-branding` (ask if on main)

## File map

| Path | Change |
|------|--------|
| `assets/branding/*` | Copy Ek-Shathe logos; retire unused purple-era icons if unused |
| `lib/core/theme/app_palette.dart` | Brand colors |
| `lib/core/theme/app_assets.dart` | Logo paths |
| `lib/core/config/flavor.dart` | Confirm/fix dev baseUrl `:5050` |
| `lib/core/network/api/api_envelope.dart` (+ generated) | New envelope shape |
| `lib/core/network/api/api_response_utils.dart` | Parse `success: false` |
| `lib/core/network/network_guard.dart` | Use new envelope failure mapping |
| `lib/core/network/api/api_endpoints.dart` | v1 auth paths |
| `lib/core/network/api/auth_api.dart` (+ `.g.dart`) | register/login/refresh/logout |
| `lib/core/network/auth_interceptor.dart` | Bearer |
| `lib/features/auth/data/models/*` | New DTOs; drop or stop using me/`results` |
| `lib/features/auth/domain/*` | `register`; login uses `email` |
| `lib/features/auth/data/repositories/auth_repository_impl.dart` | Wire new API |
| `lib/features/auth/presentation/*` | Login links; new register page |
| `lib/core/router/*` | `/register` + redirect allowlist |
| `lib/features/splash/**` | Logo + progress tint |
| `lib/features/auth/auth.md` | Update paths |

---

### Task 1: Brand assets and palette

**Files:**
- Copy into: `assets/branding/ek_shathe_full_logo_transparent.png`, `ek_shathe_symbol_transparent_1024.png` (and jpg optional)
- Modify: `lib/core/theme/app_palette.dart`, `lib/core/theme/app_assets.dart`
- Modify: splash widget(s) that reference `AppAssets` / `progressTint`

- [ ] **Step 1: Copy logos**

```bash
cp "/Users/bs1100/Downloads/ek_shathe_logos/ek_shathe_full_logo_transparent.png" \
  "/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile/assets/branding/"
cp "/Users/bs1100/Downloads/ek_shathe_logos/ek_shathe_symbol_transparent_1024.png" \
  "/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile/assets/branding/"
```

- [ ] **Step 2: Update `AppPalette`**

Set `brand = Color(0xFF0B2455)`, add `accent = Color(0xFFFF7900)`, `progressTint = accent` (or navy), keep `white`. Remove purple brand constants or leave unused neutrals only.

- [ ] **Step 3: Update `AppAssets`**

```dart
static const appLogo = '${branding}ek_shathe_full_logo_transparent.png';
static const appSymbol = '${branding}ek_shathe_symbol_transparent_1024.png';
// Point splash/icon helpers at symbol or full logo as appropriate
```

- [ ] **Step 4: Point splash/login logo widgets at `AppAssets.appLogo`**

- [ ] **Step 5: Verify** — `flutter analyze` on touched theme files; hot-restart splash shows new logo colors.

- [ ] **Step 6: Commit** (if user approved)

---

### Task 2: Success envelope helpers

**Files:**
- Modify: `api_envelope.dart`, regenerate freezed/json
- Modify: `api_response_utils.dart`, `network_guard.dart`
- Grep and update any `error:` / `results` envelope assumptions in auth path

**Interfaces:**
- `ApiResponse` exposes `bool? get success` and `String? get message` (drop required `error`/`code` or map them unused)
- `guardApi` throws when `success == false`

- [ ] **Step 1: Redesign envelope**

```dart
@freezed
abstract class ApiEnvelope with _$ApiEnvelope {
  const factory ApiEnvelope({
    required bool success,
    String? message,
    @Default([]) List<dynamic> errors,
  }) = _ApiEnvelope;
  factory ApiEnvelope.fromJson(Map<String, dynamic> json) =>
      _$ApiEnvelopeFromJson(json);
}
```

- [ ] **Step 2: `failureFromSuccessFlag({required bool? success, String? message})`** — if `success == false` → `ApiFailure`

- [ ] **Step 3: Update `guardApi`** to check `success` instead of `error`

- [ ] **Step 4: `dart run build_runner build --delete-conflicting-outputs`**

- [ ] **Step 5: Commit** (if approved)

---

### Task 3: Auth DTOs + AuthApi + endpoints

**Files:**
- Modify: `api_endpoints.dart`
- Rewrite models: `login_request.dart` (`email`, `password`), `login_response.dart` (`success`, `data: AuthSessionData`), `register_*`, `refresh_*`, `logout_*`
- Remove or stop calling `getMe` / `UserMeResponse` from auth flow
- Modify: `auth_api.dart`

**AuthSessionData shape:**

```dart
@freezed
abstract class AuthSessionData with _$AuthSessionData {
  const factory AuthSessionData({
    required AuthUserDto user,
    required String accessToken,
    required String refreshToken,
  }) = _AuthSessionData;
}

@freezed
abstract class AuthUserDto with _$AuthUserDto {
  const factory AuthUserDto({
    required int id,
    required String name,
    required String email,
  }) = _AuthUserDto;
}
```

Register response: `{ success, message?, data: { user: AuthUserDto } }` only.

- [ ] **Step 1: Endpoints**

```dart
abstract final class AuthEndpoints {
  static const register = '/api/v1/auth/register';
  static const login = '/api/v1/auth/login';
  static const refresh = '/api/v1/auth/refresh';
  static const logout = '/api/v1/auth/logout';
}
```

- [ ] **Step 2: Write Freezed models + `AuthApi` methods** (register, login, refresh, logout)

- [ ] **Step 3: build_runner**

- [ ] **Step 4: Commit** (if approved)

---

### Task 4: Interceptor + repository

**Files:**
- Modify: `auth_interceptor.dart` — `Authorization: Bearer …`
- Modify: `auth_repository.dart` — `login({email, password})`, add `register({name, email, password})`
- Modify: `auth_repository_impl.dart`
- Modify: use cases / providers accordingly

**Login flow:**
1. `POST login` → save tokens + user fields  
2. Return `AuthUser` from `data.user` (no `getMe`)

**Register flow:**
1. `POST register` → return `Right(AuthUser)` or `Right(void)` without writing tokens  

**Logout flow:**
1. Read refresh token; if present, `POST logout` with skip-auth options  
2. Clear storage always  

**Refresh:**
1. Body `{ refreshToken }`  
2. Save new access (and rotated refresh)  

- [ ] **Step 1–4: Implement + analyze**

- [ ] **Step 5: Manual test against backend** (register → login → kill access mentally via wait or force refresh path)

- [ ] **Step 6: Commit** (if approved)

---

### Task 5: Register UI + router + login polish

**Files:**
- Create: `register_page.dart`, form widget, register state/notifier (mirror login)
- Modify: `login_page` / form — email label, link to register
- Modify: `app_routes.dart`, `app_router.dart` — `/register`; allow unauthenticated access to register in `authRedirect`
- Update `auth.md`
- l10n strings for register (en/es arb + gen-l10n)

**Redirect rules:**
- Not logged in may visit `/login` and `/register`
- Logged in visiting `/register` → home (same as login)

- [ ] **Step 1: Routes + redirect**

- [ ] **Step 2: Register page + provider**

- [ ] **Step 3: Login ↔ Register navigation**

- [ ] **Step 4: `flutter gen-l10n` / analyze / widget smoke**

- [ ] **Step 5: Commit** (if approved)

---

### Task 6: Flavor URL check + E2E manual checklist

- [ ] Confirm `Flavor.dev` `baseUrl` is `http://localhost:5050/` (trailing slash consistent with Dio)
- [ ] Document Android emulator: `--dart-define=BASE_URL=http://10.0.2.2:5050/`
- [ ] Checklist:
  - [ ] Register creates user, no tokens stored, navigates to login  
  - [ ] Login stores tokens, home shows  
  - [ ] Restart app stays logged in (token present)  
  - [ ] Logout clears session  
  - [ ] Splash/login show Ek-Shathe logo + navy/orange  

---

## Spec coverage

| Spec item | Task |
|-----------|------|
| Brand colors/logos | 1 |
| Envelope | 2 |
| Endpoints + DTOs + AuthApi | 3 |
| Bearer, repo, refresh, logout | 4 |
| Register screen + router | 5 |
| Dev URL / verify | 6 |
