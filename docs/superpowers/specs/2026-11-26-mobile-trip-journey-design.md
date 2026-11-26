# Mobile Trip Journey Design

**Date:** 2026-11-26  
**Repos:** `ek_shathe_mobile` (API contract from `ek-shathe-backend`)  
**Status:** Draft for review  
**Depends on:** `docs/superpowers/specs/2026-11-25-trip-management-design.md` (Phase 1 trip APIs)

## Goal

Wire the Flutter client to all Phase 1 trip APIs and deliver an end-to-end journey: discover open trips, create a trip, request to join, organizer accept/reject, view members, and start/cancel — using the existing auth session and branding.

## Decisions (agreed)

| Topic | Choice |
|-------|--------|
| Shell | Bottom tabs: **Discover** \| **My Trips** |
| Create entry | FAB on **My Trips** |
| Detail UX | One role-aware trip detail screen |
| Architecture | Full `features/trips/` module mirroring auth (Approach 1) |
| Contract source | Mobile adapts to backend trip APIs |
| UI fidelity | Functional screens with existing navy/orange theme — not a visual redesign |
| Out of scope | Expenses/settlements, invite codes, discover filters, auto-reject on full, profile tab |

## Backend contract (source of truth)

Base: `/api/v1/trips` · Auth: `Authorization: Bearer <accessToken>`  
Envelope: `{ success, data, message?, errors? }`

| Action | Method | Path |
|--------|--------|------|
| Create | `POST` | `/` |
| My trips | `GET` | `/mine` |
| Discover | `GET` | `/discover` |
| Detail | `GET` | `/:tripId` |
| Join request | `POST` | `/:tripId/join-requests` |
| List requests | `GET` | `/:tripId/join-requests` (organizer; optional `?status=`) |
| Accept | `POST` | `/:tripId/join-requests/:requestId/accept` |
| Reject | `POST` | `/:tripId/join-requests/:requestId/reject` |
| Members | `GET` | `/:tripId/members` |
| Status | `PATCH` | `/:tripId/status` body `{ status: "STARTED" \| "CANCELLED" }` |

Create body: required `destination`, `startDate`, `endDate`, `capacity` (≥ 2); optional `title`, `description`, `origin`.

## Navigation & shell

After login, replace placeholder home with a bottom-nav shell:

| Tab | Route (example) | Content |
|-----|-----------------|---------|
| Discover | `/discover` (shell) | Open trips with seats left |
| My Trips | `/my-trips` (shell) | Memberships + FAB create + logout |

Pushed routes (above shell):

| Route | Screen |
|-------|--------|
| `/trips/create` | Create trip form |
| `/trips/:tripId` | Role-aware trip detail |

Auth redirect unchanged: splash → login/register or authenticated shell. Trip routes require auth.

Keep GoRouter + Riverpod refresh pattern from `app_router.dart`. Prefer `StatefulShellRoute` (or equivalent) so tabs preserve state; exact GoRouter wiring follows existing project conventions during implementation.

## Screens

### Discover

- Load `GET /discover`
- List tiles: title/destination, dates, `seatsLeft` / `capacity`, status
- Pull-to-refresh; empty and error states
- Tap → trip detail

### My Trips

- Load `GET /mine`
- Same list tile pattern; pull-to-refresh; empty/error
- FAB → create trip
- App bar: logout (existing logout controller → login)

### Create trip

- Form fields matching API; client validation (capacity ≥ 2, end ≥ start, required fields)
- Submit `POST /trips` → on success navigate to `/trips/:id`
- Show API validation/conflict messages

### Trip detail (role-aware)

Loads: `GET /:tripId`, `GET /:tripId/members`.  
If organizer: also `GET /:tripId/join-requests?status=PENDING`.

**Role detection:** compare stored/current user id to `trip.organizerId` and member `userId`s (from auth session / JWT `sub`). No `/users/me`.

| Role | Behavior |
|------|----------|
| Non-member, trip `OPEN`, seats left | “Request to join” (+ optional message) → `POST .../join-requests` |
| Non-member, full or not `OPEN` | Hide/disable join; short reason text |
| Member (not organizer) | Show members; no join CTA |
| Organizer | Members; pending requests with Accept / Reject; if `OPEN`, Start and Cancel actions |

After join / accept / reject / status change: refresh detail data. Surface backend `message` on 403/409.

**Pending when full:** leftover `PENDING` requests still list for organizer (backend does not auto-reject). Accept may return 409; Reject still works. UI shows the error message; do not invent auto-reject in this slice.

## Data layer

```
TripApi (Retrofit) → DTOs (Freezed/json) → TripRepository → Riverpod → UI
```

| Piece | Responsibility |
|-------|----------------|
| `TripEndpoints` / `TripApi` | HTTP contract on shared Dio (Bearer + refresh) |
| Models | Trip, JoinRequest, TripMember, create/status request bodies, envelope wrappers as needed |
| Repository | Map Dio/envelope errors to `Failure`; return `Either` like auth |
| Providers | Lists (discover/mine), detail, create, mutations (join/accept/reject/status) |
| Presentation | Discover, My Trips, Create, Detail (+ small list tiles/widgets) |

Reuse: `network_guard`, `dio_failure_mapper`, `api_response_utils`, theme (`AppPalette` navy/orange), `AppSize`, l10n where practical (add English strings for new UI; Spanish stubs OK if project requires both).

## File placement (indicative)

```
lib/core/network/api/trip_api.dart (+ endpoints)
lib/features/trips/
  data/models/…
  data/repositories/trip_repository_impl.dart
  domain/… (entities/repo interface/usecases as thin as auth)
  presentation/
    shell/… (or home shell hosting tabs)
    discover_page.dart
    my_trips_page.dart
    create_trip_page.dart
    trip_detail_page.dart
    riverpod/…
    widgets/…
lib/core/router/app_router.dart / app_routes.dart  # shell + trip routes
```

Exact folder naming should match auth/splash conventions in-repo during implementation.

## Success criteria

- Logged-in user sees Discover + My Trips tabs
- Organizer creates trip → appears in My Trips; seats reflect organizer counting as 1
- Joiner sees trip on Discover → requests join → organizer accepts on detail → joiner appears in members / My Trips
- Organizer can reject a pending request
- Organizer can Start or Cancel an `OPEN` trip; join after Start shows clear error
- Logout from My Trips clears session and returns to login
- No trip calls without Bearer token; 401 refresh path unchanged

## Out of scope

- Expense / balance / settlement mobile UI
- Invite codes, public unauthenticated browse, discover filters/pagination UI
- Auto-reject pending requests when capacity fills
- Leave trip / remove member
- Heavy visual redesign or new design system
- Committing/pushing without explicit user permission

## Implementation notes

- Sibling write: ask once per session before first write to `ek_shathe_mobile`
- Run `build_runner` after Retrofit/Freezed/Riverpod annotations
- Prefer hot reload/restart via Dart tooling when a device/emulator is connected
- Verify against local backend (`BASE_URL` / flavor already documented for simulator vs Android emulator)
