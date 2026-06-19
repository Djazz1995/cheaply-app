# AGENTS.md

Instructions for any agent (Claude Code, etc.) working in this repo.

App: **Cheaply** — local-business social-commerce mobile app (Expo React Native). Discover, follow, walk into nearby businesses via their live products, deals, reels. Backend reused: Payload CMS + Postgres + Typesense, fronted by a thin `/api/app/v1` layer. See `docs/prd-mobile-app.md` + `docs/build-plan-mobile-app.md`.

## Expo HAS CHANGED

Read the exact versioned docs at https://docs.expo.dev/versions/v54.0.0/ before writing any code. Do not rely on memory of older Expo APIs.

## Working rules

- **Respect the layered architecture (Section 15).** UI calls hooks only. Hooks call services. Services own all I/O and wrap `lib`. No screen imports `services` or `lib` directly.
- **Services throw on error; hooks catch.** Services return models directly (no result wrappers). Hooks hold loading/error state (via TanStack Query).
- **Keep `lib` thin.** It only wraps vendor SDKs (`apiClient`, secure-store, expo-location, react-native-maps, expo-notifications, Apple/Google auth) so providers stay swappable.
- **Models are pure data.** No logic, no I/O in `src/models`. Mirror `/api/app/v1` response shapes; reuse generated Payload types where possible (avoid drift).
- **Anonymous-first.** Browse (feed/map/search/profile/directions/call) needs no account. Gate only follow/save/push behind auth (soft-gate modal). See build plan §4a.
- **Local + online businesses.** `Business.type` = `local | online | hybrid`. Online has no geo → off the map, no directions; profile shows **Visit shop** CTA instead. Feed mixes both (filter scope local/online/all); local ranks by distance, online by relevance/discount/recency/follow. Don't assume every business has a `BusinessLocation`.
- **Business is a first-class discoverable unit.** Not only deals. A business is discoverable + followable with **no active deal**. Don't deal-gate business visibility. Curated `Collection`s (Phase 2) layer on top.
- **Home = categorized overview, not one mixed feed.** `HomeService.getHome` returns preview rows per type (deals/products/reels/events/businesses); each "See all" → a **dedicated page** backed by its own service (`DealService`, `ProductService`, `BusinessService.discover`, `EventService` P2, `ReelService` P3) with its own filters/sort + pagination. Discover (`SearchService`) = query-driven, separate from browse.
- **Tokens in secure-store only** (Keychain/Keystore via `expo-secure-store`) — never AsyncStorage.
- **Instrument outbound actions** (directions/call/visit) from day one — core success metric (PRD §11).
- **UI = gluestack-ui v3 + NativeWind.** Current stable (v2 legacy, v5 alpha — skip). Copy-paste components (owned in-repo), Tailwind-style classes. UI lives only in `screens`/components — never import UI into `services`/`hooks`/`lib`. Check versions against Expo SDK 54 before adding.
- **TypeScript everywhere.** Strict types; no `any` without reason.
- **Reference the PRD by section** (e.g. PRD 6.1) — the architecture tables map models/services to PRD sections.

## Docs

- PRD → `docs/prd-mobile-app.md` (app; positioning + web/discovery context folded into §1a)
- Execution checklist → `docs/build-blueprint.md` (ordered how: tasks, DoD, security gates, screens) — build from this
- Build workflow → `docs/build-workflow.md` (the per-screen/feature procedure to follow) + `docs/od-design-workflow.md` (Open Design wiring + recipe)
- Reference spec → `docs/build-plan-mobile-app.md` (detailed what: collections, endpoints, auth §4a, ranking)
- Architecture → this file, Section 15.

---

## 15. Frontend Architecture (service-based)

Layered architecture. UI calls hooks only; services own all I/O and return models. No SDK/network access from screens.

### 15.1 Layers

```
src/
  models/      pure data shapes (TS interfaces/types) — no logic, no I/O
  services/    business logic + all I/O; call /api/app/v1, return models, throw on error
  hooks/       thin React wrappers (useHome, useDeals, useBusiness…); TanStack Query, hold state, catch
  lib/         thin SDK clients (apiClient, secure-store, location, maps, notifications, auth)
  screens/     UI (PRD §6 surfaces: Home + dedicated pages, Map, Discover, profile…) — call hooks only
```

**Flow:** `screens → hooks → services → lib`. Models flow back up.

**Rules:**

- Screens import models for typing + call hooks. Never import `services` or `lib` directly.
- Services are plain TS (framework-agnostic, testable without React). Return the model directly; **throw on error**. They compose `/api/app/v1` calls via `lib/apiClient`.
- Hooks wrap services with TanStack Query (cache, loading/error). Mutations (follow/save) invalidate relevant queries.
- `lib` wraps vendor SDKs + the API client so providers/transport stay swappable.

### 15.2 Models (`src/models`)

| Model                 | Core fields                                                                       | PRD     |
| --------------------- | --------------------------------------------------------------------------------- | ------- |
| `User`                | id, email, name?, locale, homeLocation, notificationPrefs, tier?                  | 4a, 8   |
| `Business`            | id, slug, name, type (local/online/hybrid), categories[], locations[] (empty for online), hours?, phone?, website, shopUrl?, shipsTo?, followerCount, claimed | 6.4 |
| `BusinessType`        | enum: local / online / hybrid                                                     | 4       |
| `BusinessLocation`    | id, businessId, geo (lat,lng), address, openNow? (local/hybrid only)              | 6.2     |
| `Product`             | id, businessId, name, image, price, link                                          | 5       |
| `Deal`                | id, businessId, title, discount, validFrom, validTo                               | 5       |
| `Category`            | enum: food/drinks/fashion/home/furniture/gifts                                    | 6.3     |
| `Event`               | id, businessId, title, startsAt, endsAt, location, image (Phase 2)                | 5       |
| `Reel`                | id, businessId, video, thumbnail, productId?, caption, status (Phase 3)           | 6.5     |
| `HomeSection`         | type (deals/products/reels/events/businesses), title, items[], seeAllRoute        | 6.1     |
| `CardItem`            | type (deal/product/reel/event/business), ref, business, distance? (null online), score | 6.1a |
| `Collection`          | id, slug, title, description?, businessIds[], curatedBy, published (Phase 2)       | 6.3     |
| `MapPin`              | businessId, geo, hasDeal, category, openNow (local/hybrid only)                   | 6.2     |
| `Follow`              | id, targetType (business/category), businessId?/categoryId?                       | 8       |
| `Save`                | id, productId, createdAt                                                          | 6.6     |
| `SearchFilters`       | q?, category?, scope? (local/online/all), openNow?, hasDeal?, sort, geo           | 6.3     |
| `AuthSession`         | accessToken, refreshToken, expiresAt                                              | 4a      |
| `NotificationPrefs`   | frequency, quietHours                                                             | 7       |
| `NotificationPayload` | type, businessId?, dealId?, deepLink                                              | 7       |

### 15.3 Services (`src/services`)

| Service               | Responsibility               | Key methods → returns                                                                       |
| --------------------- | ---------------------------- | ------------------------------------------------------------------------------------------- |
| `AuthService`         | passwordless + social login  | `requestCode(email)`, `verify(email,code)→AuthSession`, `refresh()`, `signOut()`, `appleSignIn()`, `googleSignIn()` |
| `UserService`         | profile, prefs, push token   | `getMe()→User`, `updateMe(patch)→User`, `registerPushToken(token)`                          |
| `HomeService`         | categorized overview         | `getHome(geo,scope)→HomeSection[]` — preview rows per type, hides empty sections             |
| `DealService`         | deals page                   | `list(filters,cursor)→{items:Deal[],cursor?}`, `get(id)→Deal`                                |
| `ProductService`      | products page                | `list(filters,cursor)→{items:Product[],cursor?}`, `get(id)→Product`                          |
| `EventService`        | events page (Phase 2)        | `list(filters,cursor)→{items:Event[],cursor?}`                                               |
| `ReelService`         | reels page (Phase 3)         | `list(filters,cursor)→{items:Reel[],cursor?}`                                                |
| `MapService`          | geo pins (local/hybrid only) | `getPins(bbox\|radius,filters)→MapPin[]` — online-only businesses excluded                   |
| `SearchService`       | query-driven search          | `search(filters)→(Business\|Deal\|Product\|Event)[]` (typed)                                 |
| `BusinessService`     | profile + discovery page     | `get(slug)→Business` (products+deals+locations); `discover(filters,cursor)→{items:Business[],cursor?}` (new/nearby/trending, deal-agnostic) |
| `CollectionService`   | curated discovery (Phase 2)  | `list()→Collection[]`, `get(slug)→Collection`                                                |
| `FollowService`       | follow edges                 | `follow(target)`, `unfollow(target)`, `listFollowing()→Business[]`                          |
| `SaveService`         | saved products               | `save(productId)`, `unsave(productId)`, `listSaved()→Product[]`                             |
| `NotificationService` | push register + deep links   | `register()`, `handleTap(payload)→deepLink`                                                 |
| `AnalyticsService`    | outbound action tracking     | `track(action: directions\|call\|visit, businessId)`                                        |

All services call `/api/app/v1` via `lib/apiClient` (which injects auth + handles refresh). Auth gates: follow/save/me/push require a session.

### 15.4 Lib (`src/lib`) — swappable SDK + transport wrappers

| Module             | Wraps                              | Notes                                              |
| ------------------ | ---------------------------------- | -------------------------------------------------- |
| `apiClient.ts`     | typed fetch over `/api/app/v1`     | auth header inject + silent refresh-token rotation |
| `secureStore.ts`   | expo-secure-store                  | tokens (Keychain/Keystore) — never AsyncStorage    |
| `queryClient.ts`   | TanStack Query client              | cache config, defaults                             |
| `location.ts`      | expo-location                      | home + current geo                                 |
| `maps.ts`          | react-native-maps                  | pins, clustering                                   |
| `notifications.ts` | expo-notifications                 | push register + tap deep link                      |
| `appleAuth.ts`     | expo-apple-authentication          | Sign in with Apple (App Store requirement)         |
| `googleAuth.ts`    | Google sign-in                     | social login                                       |

Phase 2 adds `EventService`/business-mode services; Phase 3 adds `ReelService` + video lib (`expo-av` / player). Keep `lib` swappable per the same pattern.

---

## 16. Build workflow (blueprint · design · security)

**Follow `docs/build-workflow.md` for every screen and feature** — it's the canonical step-by-step procedure (foundation-first, build data spine before UI, OD design → user verifies → build RN UI → verify runs → security review → tick boxes → phase gate). Don't freelance around it.

`docs/build-blueprint.md` is the execution order. Work tasks in id order, respect `Dep:`, tick the boxes (`[ ]→[~]→[x]`) as you go, and don't advance a phase until its **Security gate** is green.

### Design — every screen
- Design via **Open Design** on the extracted gluestack DS, then **the user verifies** before building — see `docs/build-workflow.md` §1 (the procedure) + `docs/od-design-workflow.md` (the recipe).
- Record reused tokens/components so screens stay consistent (gluestack-ui v3 + NativeWind). `impeccable` is optional later for polish/critique (not required in the core loop).

### Security — built in, not bolted on
- **`security-guidance`** (installed plugin) runs automatically — pattern warnings on edits, diff review on stop, commit reviewer. **Do not ignore its findings**; fix or justify in the task before moving on.
- Lean on **`owasp-security`** knowledge when designing auth, data, and I/O tasks (OWASP Top 10:2025 / ASVS 5.0).
- Run **`/security-review`** on the branch at every phase Security gate; gate must pass to advance.
- Every task tagged `🔒 Sec` in the blueprint carries its acceptance criteria + the threat it mitigates — treat those as DoD, not suggestions.

### Definition of done (per task)
Code + states (loading/empty/error/offline, anon-vs-authed) + nl/en parity + the task's DoD and any `🔒 Sec` criteria. Layered-architecture lint (§15) stays green.
