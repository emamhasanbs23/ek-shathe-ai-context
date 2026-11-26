# Mobile Trip Journey Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Discover / My Trips shell, create trip, and role-aware trip detail against Phase 1 backend trip APIs in `ek_shathe_mobile`.

**Architecture:** `features/trips/` module with Retrofit `TripApi`, Freezed DTOs, repository (`Either`/`guardApi`), Riverpod providers, GoRouter `StatefulShellRoute` (Discover | My Trips), FAB create, role-aware detail. Shared Dio Bearer auth unchanged.

**Tech Stack:** Flutter, Riverpod, GoRouter, Dio, Retrofit, Freezed, fpdart, existing theme/l10n

**Spec:** `docs/superpowers/specs/2026-11-26-mobile-trip-journey-design.md`

## Global Constraints

- Sibling repo: `/Users/bs1100/Documents/Ek-Shathe All/ek_shathe_mobile` (ask once per session before first write; ask every time before git ops)
- Contract: backend `/api/v1/trips/*` + `{ success, data }` envelope + Bearer access token
- Bottom tabs Discover | My Trips; FAB create on My Trips; one role-aware detail
- Reuse auth Dio interceptor; no new auth scheme
- Run `dart run build_runner build --delete-conflicting-outputs` after annotations
- Do not commit/push mobile without explicit user permission

## File map

| Path | Responsibility |
|------|----------------|
| `lib/core/network/api/api_endpoints.dart` | Add `TripEndpoints` |
| `lib/core/network/api/trip_api.dart` | Retrofit trip API |
| `lib/features/trips/data/models/*` | Freezed DTOs + response envelopes |
| `lib/features/trips/domain/repositories/trip_repository.dart` | Interface |
| `lib/features/trips/data/repositories/trip_repository_impl.dart` | Impl |
| `lib/features/trips/presentation/riverpod/trip_providers.dart` | Providers/controllers |
| `lib/features/trips/presentation/*_page.dart` | Discover, My Trips, Create, Detail |
| `lib/features/trips/presentation/trips_shell.dart` | Bottom nav shell |
| `lib/features/trips/presentation/widgets/trip_list_tile.dart` | Shared list tile |
| `lib/core/router/app_routes.dart` / `app_router.dart` | Shell + trip routes; default authed → discover |
| `lib/l10n/app_en.arb` (+ es) | Trip strings |
| Retire placeholder-only home as post-login landing (keep file or redirect) |

---

### Task 1: API + models + repository

- [ ] Add `TripEndpoints` constants
- [ ] Create Freezed models: `TripDto`, `JoinRequestDto`, `TripMemberDto`, `CreateTripRequest`, `UpdateTripStatusRequest`, `JoinRequestBody`, response wrappers (`TripResponse`, `TripListResponse`, `JoinRequestResponse`, `JoinRequestListResponse`, `MemberListResponse`, `AcceptJoinResponse`)
- [ ] Create `TripApi` Retrofit interface for all 10 endpoints
- [ ] `TripRepository` + `TripRepositoryImpl` with `guardApi`
- [ ] `build_runner`

### Task 2: Providers + UI screens

- [ ] `tripApi` / `tripRepository` providers
- [ ] Async providers: discover list, mine list, trip detail aggregate (trip + members + pending if organizer)
- [ ] Controllers: create, join, accept, reject, update status (invalidate lists/detail on success)
- [ ] Pages: Discover, My Trips (FAB + logout), Create form, Detail (role-aware)
- [ ] `TripsShell` with `NavigationBar` / `NavigationDestination`

### Task 3: Router + l10n + verify

- [ ] Routes: shell branches `/discover`, `/my-trips`; `/trips/create`; `/trips/:tripId`
- [ ] Auth redirect: logged-in default → `/discover` (not old `/` home)
- [ ] Add EN (and ES) arb keys; flutter gen-l10n
- [ ] Manual verify: two users create → discover → join → accept → start

## Spec coverage

| Spec item | Task |
|-----------|------|
| Tabs + FAB + create | 2–3 |
| All trip APIs | 1 |
| Role-aware detail | 2 |
| Logout on My Trips | 2 |
| Pending-when-full shows error on accept | 2 (show API message) |
