# Mobile Auth Integration & Branding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `ek_shathe_mobile` to Ek-Shathe backend auth (`/api/v1/auth/*`, Bearer, `{ success, data }`), add a Register screen, and apply navy/orange/white branding plus logos.

**Architecture:** Rewrite the auth/network layer in place (Retrofit DTOs, envelope helpers, interceptor, repository). Presentation keeps Riverpod + GoRouter patterns; Register is a new route. Branding updates `AppPalette` and `assets/branding/`.

**Tech Stack:** Flutter, Riverpod, GoRouter, Dio/Retrofit, Freezed/json_serializable, flutter_secure_storage

**Spec:** `docs/superpowers/specs/2026-11-24-mobile-auth-branding-design.md`

## Global Constraints

- Work in sibling repo: `/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile` (ask once per session before first write)
- Mobile must match backend: `{ success, data, message?, errors? }`, `Authorization: Bearer`, paths under `/api/v1/auth/`
- Register returns user only (no tokens); Login/Refresh return `user` + tokens
- No `/users/me` in this slice — use `data.user` from login/refresh
- Brand: navy `#0B2455`, orange `#FF7900`, white `#FFFFFF`
- Logos from `/Users/bs1100/Downloads/ek_shathe_logos`
- Run `dart run build_runner build --delete-conflicting-outputs` after DTO/Retrofit/Riverpod changes
- Git commits in sibling only with explicit user permission each time
- Dev baseUrl: `http://localhost:5050/` (iOS/desktop); document Android emulator `http://10.0.2.2:5050/`

## File map

| Path | Responsibility |
|------|----------------|
| `lib/core/network/api/api_endpoints.dart` | Auth paths |
| `lib/core/network/api/api_envelope.dart` (+ `.g.dart`/`.freezed.dart`) | `{ success, message, errors }` |
| `lib/core/network/api/api_response_utils.dart` | Parse/fail from new envelope |
| `lib/core/network/network_guard.dart` | `guardApi` uses `success == false` |
| `lib/core/network/auth_interceptor.dart` | Bearer + refresh |
| `lib/core/network/api/auth_api.dart` (+ `.g.dart`) | register/login/refresh/logout |
| `lib/features/auth/data/models/*` | Request/response DTOs |
| `lib/features/auth/domain/repositories/auth_repository.dart` | Add `register`; login uses email |
| `lib/features/auth/data/repositories/auth_repository_impl.dart` | Wire new API |
| `lib/features/auth/domain/usecases/register_use_case.dart` | New |
| `lib/features/auth/presentation/*` | Login links + Register page |
| `lib/core/router/*` | `/register` + redirect allowlist |
| `lib/core/theme/app_palette.dart` | Brand colors |
| `lib/core/theme/app_assets.dart` | Logo paths |
| `assets/branding/` | Copied logo files |
| `lib/features/splash/...` | Use full logo + orange progress |
| `lib/features/auth/auth.md` | Update paths/envelope notes |

---

### Task 1: Brand assets and palette

**Files:**
- Copy into: `assets/branding/ek_shathe_full_logo_transparent.png`, `ek_shathe_symbol_transparent_1024.png` (optional jpg if needed)
- Modify: `lib/core/theme/app_assets.dart`, `lib/core/theme/app_palette.dart`
- Modify splash/login widgets that hardcode purple or old logos

- [ ] **Step 1: Copy logos**

```bash
cp "/Users/bs1100/Downloads/ek_shathe_logos/ek_shathe_full_logo_transparent.png" \
  "/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile/assets/branding/"
cp "/Users/bs1100/Downloads/ek_shathe_logos/ek_shathe_symbol_transparent_1024.png" \
  "/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile/assets/branding/"
```

- [ ] **Step 2: Update `AppAssets`**

```dart
abstract final class AppAssets {
  static const branding = 'assets/branding/';
  static const appLogo = '${branding}ek_shathe_full_logo_transparent.png';
  static const appSymbol = '${branding}ek_shathe_symbol_transparent_1024.png';
  // Keep old keys as aliases pointing to new files if referenced widely, or update call sites.
}
```

- [ ] **Step 3: Update `AppPalette`**

```dart
static const progressTint = Color(0xFFFF7900);
static const brand = Color(0xFF0B2455);
static const brandDark = Color(0xFF0B2455);
static const accent = Color(0xFFFF7900);
static const white = Color(0xFFFFFFFF);
```

Remove purple `#723DA7` brand usage; grep and replace remaining references.

- [ ] **Step 4: Point splash (and login header if present) at `AppAssets.appLogo`; progress tint `AppPalette.progressTint`**

- [ ] **Step 5: Verify** — `flutter analyze` on touched theme files; hot restart shows logo/colors

- [ ] **Step 6: Commit** (if user approved)

```bash
git add assets/branding lib/core/theme
git commit -m "style: apply Ek-Shathe brand colors and logos"
```

---

### Task 2: Envelope + failure helpers

**Files:**
- Modify: `api_envelope.dart`, regenerate freezed/json
- Modify: `api_response_utils.dart`, `network_guard.dart`
- Modify: any `ApiResponse` implementers that still expose `error`/`code`

**Interfaces:**
- `ApiResponse` has `bool? get success`, `String? get message` (drop required `error`/`code` or keep optional unused)
- `failureFromSuccessFlag({ required bool? success, String? message })` → failure when `success == false`

- [ ] **Step 1: Redefine envelope**

```dart
@freezed
abstract class ApiEnvelope with _$ApiEnvelope implements ApiResponse {
  const factory ApiEnvelope({
    bool? success,
    String? message,
    @Default([]) List<dynamic> errors,
  }) = _ApiEnvelope;

  factory ApiEnvelope.fromJson(Map<String, dynamic> json) =>
      _$ApiEnvelopeFromJson(json);
}
```

Update `ApiResponse` interface accordingly.

- [ ] **Step 2: Update utils + `guardApi`**

```dart
Failure? failureFromSuccessFlag({
  required bool? success,
  String? message,
}) {
  if (success != false) return null;
  final trimmed = message?.trim();
  return ApiFailure(
    message: trimmed != null && trimmed.isNotEmpty ? trimmed : null,
  );
}

Future<R> guardApi<R extends ApiResponse>(...) async {
  final response = await guard(tag, call);
  final failure = failureFromSuccessFlag(
    success: response.success,
    message: response.message,
  );
  if (failure != null) throw failure;
  return response;
}
```

Update `parseApiEnvelope` to look for `success`/`message` keys.

- [ ] **Step 3: Grep for `.error`, `failureFromResponseFields`, `results` in network/auth and fix compile breaks**

- [ ] **Step 4: `dart run build_runner build --delete-conflicting-outputs`**

- [ ] **Step 5: Commit** (if user approved)

```bash
git commit -m "refactor: parse backend success/data API envelope"
```

---

### Task 3: Auth DTOs, endpoints, AuthApi

**Files:**
- Modify: `api_endpoints.dart`
- Rewrite/add models under `features/auth/data/models/`
- Modify: `auth_api.dart` + regenerate

**Endpoints:**

```dart
abstract final class AuthEndpoints {
  static const register = '/api/v1/auth/register';
  static const login = '/api/v1/auth/login';
  static const refresh = '/api/v1/auth/refresh';
  static const logout = '/api/v1/auth/logout';
}
```

**DTO shapes (Freezed + json_serializable):**

- `RegisterRequest`: `name`, `email`, `password`
- `LoginRequest`: `email`, `password` (rename from username)
- `RefreshTokenRequest` / `LogoutRequest`: `refreshToken`
- Shared `AuthUserDto`: `id`, `name`, `email`
- `AuthTokenData`: `user`, `accessToken`, `refreshToken`
- Responses wrap envelope:

```dart
@freezed
abstract class LoginResponse with _$LoginResponse implements ApiResponse {
  const factory LoginResponse({
    bool? success,
    String? message,
    AuthTokenData? data,
  }) = _LoginResponse;
  // fromJson...
}
```

- `RegisterResponse`: `success`, `message`, `data: { user }` (or `RegisterData` with only `user`)

Remove or stop using `UserMeResponse` / `getMe` in auth path.

**AuthApi:**

```dart
@POST(AuthEndpoints.register)
Future<RegisterResponse> register(@Body() RegisterRequest request, ...);

@POST(AuthEndpoints.login)
Future<LoginResponse> login(@Body() LoginRequest request, ...);

@POST(AuthEndpoints.refresh)
Future<LoginResponse> refresh(@Body() RefreshTokenRequest request, {@DioOptions() Options? options, ...});

@POST(AuthEndpoints.logout)
Future<LogoutResponse> logout(@Body() LogoutRequest request, {@DioOptions() Options? options, ...});
```

- [ ] **Step 1: Write models + endpoints + AuthApi**
- [ ] **Step 2: build_runner**
- [ ] **Step 3: Commit** (if user approved)

```bash
git commit -m "feat: add v1 auth Retrofit DTOs and endpoints"
```

---

### Task 4: Interceptor + repository + use cases

**Files:**
- Modify: `auth_interceptor.dart` — header `Authorization: Bearer $token`
- Modify: `auth_repository.dart` / `auth_repository_impl.dart`
- Add: `register_use_case.dart`
- Modify: login use case param names if needed
- Modify: `auth_providers.dart`

**Repository:**

```dart
Future<Either<Failure, AuthUser>> login({
  required String email,
  required String password,
});

Future<Either<Failure, AuthUser>> register({
  required String name,
  required String email,
  required String password,
});
// register: call API, map data.user → AuthUser, do NOT write tokens
// login: write access+refresh, map data.user → AuthUser, cache id/email
// getCurrentUser: read cache / reconstruct AuthUser from storage (no me endpoint)
// refreshAccessToken: POST refresh, save new tokens, return accessToken
// logout: POST logout with refreshToken (ignore network failure), then clear storage
```

Map `AuthUser` from DTO (`id` as string if entity expects String).

- [ ] **Step 1: Interceptor Bearer**
- [ ] **Step 2: Repository + register use case + providers**
- [ ] **Step 3: build_runner for riverpod if needed**
- [ ] **Step 4: Manual test with running backend** — login via temporary debug or next task UI
- [ ] **Step 5: Commit** (if user approved)

```bash
git commit -m "feat: wire auth repository to backend login refresh logout register"
```

---

### Task 5: Register screen + router + login links

**Files:**
- Create: `register_page.dart`, `register_form_widget.dart`, register state/notifier (mirror login)
- Modify: `login_page.dart` / form — email label, link to register
- Modify: `app_routes.dart`, `app_router.dart` — path `/register`, name `register`
- Update `authRedirect` so unauthenticated users may open `/register` (same as login)

```dart
static const register = '/register';
// redirect: allow matchedLocation == register when logged out
```

- [ ] **Step 1: Register UI + notifier calling `register` then `context.go(login)` on success**
- [ ] **Step 2: Login “Create account” → register; Register “Sign in” → login**
- [ ] **Step 3: l10n strings in `app_en.arb` (and es if required by project) + gen-l10n**
- [ ] **Step 4: `flutter analyze`; fix issues**
- [ ] **Step 5: Commit** (if user approved)

```bash
git commit -m "feat: add register screen and auth navigation links"
```

---

### Task 6: Flavor URL note + auth.md + end-to-end verify

**Files:**
- Confirm `flavor.dart` dev `baseUrl` is `http://localhost:5050/` (trailing slash OK with Retrofit paths)
- Update `lib/features/auth/auth.md`
- Optional comment in flavor for Android emulator `10.0.2.2`

**Manual E2E (backend on 5050):**

1. Register new user → success message, land on Login, no tokens in storage  
2. Login → home  
3. Kill access token expiry or force 401 → refresh succeeds  
4. Logout → back to login; refresh with old token fails on server  

- [ ] **Step 1: Docs + flavor sanity**
- [ ] **Step 2: E2E checklist above**
- [ ] **Step 3: Commit** (if user approved)

```bash
git commit -m "docs: update auth feature notes for v1 backend contract"
```

---

## Spec coverage

| Spec item | Task |
|-----------|------|
| Envelope + Bearer | 2, 4 |
| Endpoints + DTOs | 3 |
| Register no tokens | 4, 5 |
| Login/refresh/logout | 4 |
| Register screen + route | 5 |
| Brand colors + logos | 1 |
| No `/users/me` | 4 |
| auth.md | 6 |

## Placeholder scan

No TBD left; implementers must run build_runner after Freezed/Retrofit edits and obtain sibling-repo write + git permission from the user.
