# Build Plan — Cheaply Mobile App (React Native)

_Engineering plan for the app in `docs/prd-mobile-app.md`. Backend + frontend work, sequenced, grounded in the current repo._

Status: Draft
Date: 2026-06-17
Companion to: `docs/prd-mobile-app.md`

---

## 0. Current state (what already exists in the repo)

- **Payload CMS 3 + Postgres.** Collections: `products/`, `companies/`, `company-locations/`, `Categories.ts`, `BlogPosts.ts`, `Media.ts`, `Users.ts`, `ApiTokens.ts`.
- **Typesense** product search with geo sort + facets.
- **Scraper** (`apify-actors/universal-ai-scraper/`) syncing products via CMS API.
- **Some company-session/auth already exists** — `src/app/api/auth/clear-company-session`, `construction-bypass`. Confirm how businesses authenticate today before building business accounts (may extend, not build fresh).
- **API routes** under `src/app/api/` (scraper batch, cron sync, seed, social-card, etc.). Payload also exposes REST/GraphQL at `/api/...` automatically.
- **Email** via Nodemailer. **Cron** sync endpoints exist (`cron/sync-cms-products-*`).
- **No consumer (end-user) accounts, no follow/saved, no push.** These are the main net-new backend.

**Principle:** the app is a new Expo client over a thin public API in front of the existing backend. Backend work = mostly *expose + add 3 relationships + push*, not rebuild.

---

## 1. Workstreams overview

| # | Workstream | Backend | Frontend | Phase |
|---|---|---|---|---|
| A | Public App API (read) | ✅ | — | 1 |
| B | Consumer identity (auth) | ✅ | ✅ | 1 |
| C | Follow + Saved | ✅ | ✅ | 1 |
| D | App shell + navigation | — | ✅ | 1 |
| E | Home + content pages (deals/products/businesses) | ✅ (rank) | ✅ | 1 |
| F | Map | ✅ (geo endpoint) | ✅ | 1 |
| G | Discover/Search | ✅ (expose Typesense) | ✅ | 1 |
| H | Business profile | ✅ | ✅ | 1 |
| I | Push notifications | ✅ | ✅ | 1 |
| J | Business self-serve | ✅ | ✅ | 2 |
| K | Events / market ads | ✅ | ✅ | 2 |
| L | Reels / video | ✅ | ✅ | 3 |
| M | Monetization (boost) | ✅ | ✅ | 3 |

---

## 2. Backend work

### 2.1 New Payload collections / fields

**New collections:**

1. **`Consumers`** (end-user accounts — distinct from admin `Users`)
   - `email` (unique), `name?`, `locale`, `homeLocation` (postcode + geo, reuse postcode lookup), `pushTokens[]` (Expo tokens + platform), `notificationPrefs` (frequency, quiet hours), `createdAt`.
   - Passwordless: store one-time login codes / magic-link tokens (or delegate to an auth provider — see 2.3).

2. **`Follows`** (edges)
   - `consumer` (rel), `targetType` (`business` | `category`), `business?` (rel), `category?` (rel), `createdAt`.
   - Indexed on `consumer` and on `business`/`category` for fan-out queries.

3. **`Saves`** (saved products/deals)
   - `consumer` (rel), `product` (rel), `createdAt`.

4. **`Events`** (Phase 2 — market/pop-up ads)
   - `company` (rel), `title`, `description`, `startsAt`, `endsAt`, `location` (geo, reuse company-location pattern), `image`, `published`.

5. **`Reels`** (Phase 3)
   - `company` (rel), `video` (storage ref), `thumbnail`, `product?` (rel), `caption`, `status` (moderation), `published`.

6. **`Collections`** (Phase 2 — curated discovery)
   - `slug`, `title`, `description?`, `companies` (rel[]), `curatedBy`, `published`. Editorial discovery lists; reuse admin UI.

**Field additions:**
- `companies`: ensure `followerCount` (denormalized, updated by hook), `claimedBy` (rel → Consumer/business account), `hours`, `phone`.
- `companies`: add `type` (`local` | `online` | `hybrid`), `shopUrl` (online/hybrid), `shipsTo` (optional). Online-only → no `company-locations` rows; exclude from geo/map endpoints. Default existing scraped companies to `local` (or infer `online` when no geocodable address).

### 2.2 Public App API layer

Build a versioned, app-facing API. Two options — **recommend custom Next.js route handlers under `src/app/api/app/v1/...`** (thin, shaped for the app, cacheable) over exposing raw Payload REST (too chatty, leaks admin shape).

Endpoints (Phase 1):
- `POST /api/app/v1/auth/request-code` — passwordless start.
- `POST /api/app/v1/auth/verify` — returns session/JWT.
- `GET  /api/app/v1/me` — profile + prefs.
- `PATCH /api/app/v1/me` — location, prefs, push token register.
- `GET  /api/app/v1/home?lat&lng&scope` — categorized overview: preview rows per type (deals/products/reels/events/businesses).
- `GET  /api/app/v1/deals?filters&cursor` — dedicated deals page (paginated).
- `GET  /api/app/v1/products?filters&cursor` — dedicated products page.
- `GET  /api/app/v1/events?filters&cursor` — events page (Phase 2).
- `GET  /api/app/v1/reels?filters&cursor` — reels page (Phase 3).
- `GET  /api/app/v1/map?bbox|lat,lng,radius&filters` — businesses + pins (geo).
- `GET  /api/app/v1/search?q&filters&sort` — Typesense passthrough (already have the query layer).
- `GET  /api/app/v1/business/:slug` — profile + products + deals + locations.
- `GET  /api/app/v1/businesses?sort=new|nearby|trending&geo&scope&category` — **business discovery** (deal-agnostic shop list; followers/recency/geo). Feeds business cards in feed + Discover.
- `GET  /api/app/v1/collections` / `:slug` — curated discovery lists (Phase 2).
- `POST /api/app/v1/follow` / `DELETE /api/app/v1/follow` — toggle.
- `POST /api/app/v1/save` / `DELETE /api/app/v1/save`.
- `GET  /api/app/v1/following` / `GET /api/app/v1/saved`.

Reuse: the existing product service + Typesense query helpers (`src/app/(frontend-v2)/services/products/products.service.ts`, `src/typesense/`). Wrap, don't duplicate.

### 2.3 Auth

- Passwordless email (one-time code) — reuses Nodemailer. Issue JWT (short) + refresh token.
- Add Apple + Google sign-in (App Store norm; Apple **requires** Sign in with Apple if other social logins exist).
- **First: audit existing company-session auth** (`api/auth/clear-company-session`) — business accounts in Phase 2 may extend this rather than start fresh.

### 2.4 Ranking (powers Home rows + dedicated pages)
- **Local items:** Typesense geo sort (exists) + discount + recency + validity-ending-soon + follow-boost.
- **Online items:** no distance → rank by relevance + discount + recency + validity + follow-boost. Merge with local results.
- **Scope param** (`local` | `online` | `all`, default `all`): filter `companies.type`. Map/geo endpoints always exclude `online`.
- Follow-boost: fetch consumer's followed business/category ids, boost matching results (Typesense `query_by_weights` / pinning, or post-merge).
- **Home overview** (`/home`): run a capped query per type → preview rows; hide empty sections. Dedicated pages reuse the same ranking with full pagination + per-page filters.
- **Business discovery:** `/businesses` (new/nearby/trending, deal-agnostic) backs the Businesses row + page.
- Cache anonymous location-bucketed home (local); online slices location-independent → cache separately, higher reuse.

### 2.5 Push pipeline
- Store Expo push tokens on `Consumers`.
- **Phase 1 triggers = followed-only** (PRD §7): hook into existing product-save → Typesense-sync hook. On new qualifying product/deal for a followed business/category → enqueue push (+ optional email digest). Deal-ending-soon via cron over followed entities.
- **Deferred — proximity triggers** (new business/reel near you, event this weekend nearby): noisier, harder opt-in. Not Phase 1.
- Use existing `jobs/` + cron infra for batching/digest. Respect prefs + quiet hours.
- Send via Expo Push API.

### 2.6 Tiered ingestion (supply — extend scraper)
- **Platform detection:** Shopify (`/products.json`), Woo, feeds → structured API ingest, skip LLM. (Cheapest, most reliable.)
- Keep current Crawlee+OpenAI scraper as **custom-site fallback**.
- Feed both into the same CMS product sync API already in use.

### 2.7 Business self-serve API (Phase 2)
- Auth'd business account endpoints: CRUD products/deals/events/reels, Shopify connect (OAuth), basic stats (views, clicks, directions taps).
- Stats = event log table + aggregation (new lightweight `AppEvents` collection or analytics sink).

### 2.8 Media/video (Phase 3)
- Video upload → object storage + transcode (e.g. Mux or Cloudflare Stream) + CDN. Don't roll your own transcode.
- Moderation gate on `Reels.status` before `published`.

---

## 3. Frontend work (Expo React Native)

Layered service-based architecture per `AGENTS.md` §15. Build bottom-up: `models → lib → services → hooks → screens`. UI calls hooks only; services own all I/O (`/api/app/v1`) and throw on error; hooks catch via TanStack Query.

### 3.1 Foundation (do once)
- Init Expo (managed) + TypeScript. Read https://docs.expo.dev/versions/v54.0.0/ first.
- Share types from `src/payload-types.ts` (shared client types package or copy generated types) — avoid drift.
- Navigation: Expo Router (file-based, matches Next mental model).
- **UI: gluestack-ui v3** (current stable; copy-paste components owned in-repo) + **NativeWind** (Tailwind for RN) as styling base. Components live only in `screens`/components — orthogonal to `services`/`hooks`/`lib`. v2 = legacy; v5 = alpha (skip). Verify gluestack v3 + NativeWind versions against Expo SDK 54 before install (v3 announced on SDK 53 — confirm 54 compat).
- Data: TanStack Query + typed API client over `/api/app/v1`. State: small auth/location store (Zustand).
- Folder skeleton: `src/{models,services,hooks,lib,screens}`. Path aliases (`@/models`…). Lint convention: screens import `models` + `hooks` only.
- i18n: reuse `messages/en.json` + `messages/nl.json` (next-intl → i18next/expo-localization). nl/en parity.
- Maps: `react-native-maps`. Location: `expo-location`. Push: `expo-notifications`. Media: `expo-image`, `expo-av` (reels).
- OTA: EAS Update. Builds: EAS Build. Stores: EAS Submit.

### 3.2 Models (`src/models`)
Pure TS, no logic/I/O (§15.2). Mirror `/api/app/v1` shapes; reuse generated Payload types.
- `User`, `Business` (+ `BusinessType` local/online/hybrid; `locations[]` optional), `BusinessLocation`, `Product`, `Deal`, `Category`
- `HomeSection`, `CardItem` (`type` incl `business`), `MapPin`, `SearchFilters`
- `Follow`, `Save`, `AuthSession`, `NotificationPrefs`, `NotificationPayload`
- `Collection` (P2), `Event` (P2), `Reel` (P3); `index.ts` barrel.

### 3.3 Lib (`src/lib`)
Thin vendor + transport wrappers only (§15.4). No business logic.
- `apiClient.ts` (typed fetch over `/api/app/v1`; auth header + silent refresh), `secureStore.ts` (tokens), `queryClient.ts`, `location.ts`, `maps.ts`, `notifications.ts`, `appleAuth.ts`, `googleAuth.ts`.

### 3.4 Services (`src/services`)
Plain TS, framework-agnostic, throw on error (§15.3). Compose `/api/app/v1` via `apiClient`. Build order:
1. `AuthService`, 2. `UserService`, 3. `HomeService` (getHome → preview rows), 4. `DealService` (list/get), 5. `ProductService`, 6. `MapService`, 7. `SearchService` (typed), 8. `BusinessService` (get + discover), 9. `FollowService`, 10. `SaveService`, 11. `NotificationService`, 12. `AnalyticsService`.
- P2: `EventService`, `CollectionService`. P3: `ReelService`. Unit-test each against `apiClient` mocks.

### 3.5 Hooks (`src/hooks`)
Thin React wrappers over services + TanStack Query. Hold loading/error; catch; mutations invalidate queries.
- `useAuth`, `useUser`, `useHome`, `useDeals`, `useProducts`, `useBusinesses` (discover), `useMap`, `useSearch`, `useBusiness(slug)`, `useFollow`, `useFollowing`, `useSave`, `useSaved`, `useLocation`, `usePush`.
- P2: `useEvents`, `useCollections`. P3: `useReels`.

### 3.6 Screens (`src/screens`, Expo Router)
Call hooks only; import models for typing. Wire order:
1. **Onboarding/location** — postcode → geo, optional sign-in (browse anonymous first).
2. **Home** — categorized overview: preview rows per type (Deals/Products/Businesses; Events P2, Reels P3), each "See all" → dedicated page. Scope filter (local/online/all), pull-to-refresh.
3. **Dedicated pages** — Deals, Products, Businesses (discovery) in P1; Events (P2), Reels (P3). Full list/grid, own filters/sort, infinite scroll.
4. **Map** — pins (local/hybrid only; online excluded), cluster, filters (category/open-now/has-deal/restaurants), tap → profile.
5. **Discover/search** — query-driven Typesense across types + category hubs → funnel into dedicated pages. Curated collections (P2).
6. **Business profile** — header (logo, follow, follower count), tabs (deals/products + locations/about for local/hybrid). Actions by type — local/hybrid: Directions/Call/Visit/Follow; online: Visit shop/Follow (→ `AnalyticsService.track`).
7. **Saved & Following** — lists.
8. **Profile/Settings** — location, notification prefs, sign-in/out.
9. **Auth soft-gate modal** on Follow/Save (email→code, Apple/Google).

### 3.7 Auth flow (frontend)
- Email → 6-digit code → verified. Apple/Google buttons. Anonymous browsing allowed; gate only follow/save/push. Tokens in `expo-secure-store`; silent refresh rotation. See §4a.

### 3.8 Push (frontend)
- Register Expo token post-auth → `UserService.registerPushToken` (`PATCH /me`). Notification tap → `NotificationService.handleTap` → Expo Router deep link (deal/business).

### 3.9 Business mode (Phase 2)
- Section in the same app (role-gated) or companion flow: manage profile, add/edit products/deals/events, connect Shopify, upload reels (Phase 3), view stats.

---

## 4. Sequenced timeline (Phase 1 = consumer MVP)

**Milestone 1 — API + auth skeleton (backend-led)**
- BE: `Consumers`, `Follows`, `Saves` collections + migrations.
- BE: `/api/app/v1` auth (passwordless) + `me`.
- FE: Expo init, navigation, API client, auth flow, onboarding/location.

**Milestone 2 — Read surfaces**
- BE: `home`, `deals`, `products`, `businesses`, `map`, `search`, `business/:slug` endpoints (wrap existing product service + Typesense).
- FE: home + dedicated pages (deals/products/businesses), map, discover/search, business profile screens.

**Milestone 3 — Engagement**
- BE: follow/save endpoints + `followerCount` hook.
- FE: follow/save UI, Saved & Following screens.

**Milestone 4 — Push + polish**
- BE: push token storage + trigger hook + Expo send + digest batching.
- FE: push registration, deep links, notification settings.
- QA: nl/en parity, geo edge cases, empty/loading states, store assets.

**Milestone 5 — Ship**
- EAS Build/Submit, store listings (App Store + Play), privacy disclosures (GDPR), TestFlight/internal track in Leiden.

**Phase 2** — business self-serve + events. **Phase 3** — reels + monetization (boost).

---

## 4a. Authentication & registration

Decisions locked (2026-06-17): consumer = passwordless email + Apple + Google; verification via **6-digit code** (not magic link — keeps the user in-app, no app-switching); business auth audited below.

### Audit of existing auth (done)

- **No real business/consumer auth exists today.** `src/app/api/auth/clear-company-session/` is an **empty directory** (no route file).
- `src/app/(frontend-v2)/lib/coming-soon-business.server.ts` is the only auth-adjacent code and it is **stubbed/disabled**: `getComingSoonSessionContext` hardcodes `user: null`, `isCompanyUser: false`; guest-auth path checks return `false`; the company dashboard always redirects to a "business coming soon" page.
- The **function/route naming is a blueprint** for an intended business-account system (company users, self-serve business dashboard, business registration, read-only merchants) that was scaffolded then shut off for pre-launch.
- Only live mechanism = `cheaply-construction-bypass` cookie = **staff preview bypass, not user auth.**
- **Conclusion:** consumer auth = build fresh (nothing exists). Business auth (Phase 2) = build fresh, but reuse the existing naming/route scaffolding + intended design as a guide.

### Principle: no registration wall

Browse is **100% anonymous**. Login is never shown on launch. Identity is soft-prompted only when the user taps an action that needs it.

```
Anonymous (no account):  home, pages, map, search, profile, directions, call  ✅
Needs account:           follow, save, push notifications               🔒
```

There is **no separate "register" vs "login"** for consumers: first time an email is seen → auto-create `Consumer`; return visit → log in. Same flow, same screens.

### Consumer auth flow (Phase 1)

```
1. Tap Follow/Save → soft-gate modal
2. Enter email → POST /auth/request-code  (Nodemailer sends 6-digit code)
3. Enter code → POST /auth/verify → JWT (short) + refresh token
   (email unseen → create Consumer; seen → log in)
4. No password, ever.
+ Apple / Google buttons (Apple Sign-in REQUIRED by App Store when any social login is offered)
```

### Backend (Phase 1)

- `Consumers` collection: `email` (unique), `name?`, `locale`, `homeLocation`, `pushTokens[]`, `notificationPrefs`.
- Login-code store: 6-digit, hashed, short TTL (~10 min), single-use, max-attempts.
- `POST /api/app/v1/auth/request-code` — rate-limited; generic response regardless of whether email exists (no enumeration).
- `POST /api/app/v1/auth/verify` — returns JWT (~15 min) + rotating refresh token.
- `GET/PATCH /api/app/v1/me`.
- JWT middleware guarding follow/save/me/push endpoints.
- Apple/Google token verification endpoints.

### Frontend (Phase 1)

- Tokens in `expo-secure-store` (Keychain/Keystore) — **never** AsyncStorage.
- Soft-gate modal on Follow/Save.
- Email → code screens; `expo-apple-authentication` + Google sign-in buttons.
- Silent refresh-token rotation.

### Security requirements (do not skip)

- Rate-limit `request-code` (email-bomb + enumeration protection); generic responses.
- Codes: 6-digit, hashed at rest, ~10 min TTL, single-use, max attempts then lockout.
- JWT short-lived + rotating refresh; revoke on sign-out.
- Secure token storage (Keychain/Keystore via secure-store).
- GDPR: consent at first sign-up, data export + delete, privacy policy in store listing.

### Business auth (Phase 2)

- Build fresh, reusing the dormant scaffolding's design (company users, self-serve dashboard, business registration, read-only merchants).
- Claim flow: business proves ownership of an already-listed company (domain email or admin verification) → account.
- Likely passwordless or magic-link too; higher stakes (edits listings, later handles money) → plan for optional 2FA later.

---

## 5. Cross-cutting

- **Keep web alive.** Web storefront = SEO/GEO acquisition funnel; app = retention. Same backend. The `/api/app/v1` layer must not break existing web data paths — wrap shared services, don't fork them.
- **Types contract.** Generate API types once (from Payload + endpoint schemas), share to app. Avoid drift.
- **GDPR.** Consumer accounts + location + push = consent flow, data export/delete, privacy policy in store listing.
- **Analytics.** Instrument outbound business actions (directions/call/visit) from day one — that's the core success metric ("in the door").
- **Testing.** Reuse Vitest (int) for API endpoints; Playwright stays for web; add Maestro/Detox for app E2E later.

---

## 6. Open decisions (need your call)

1. ~~**Auth:** passwordless email + Apple/Google?~~ ✅ LOCKED: passwordless email (6-digit code) + Apple + Google. See §4a.
2. ~~**Business accounts:** extend existing company-session auth or build fresh?~~ ✅ AUDITED: no real auth exists (empty route + stubbed file); build fresh in Phase 2, reuse dormant scaffolding as design guide. See §4a.
3. ~~**Backend platform:** Payload vs Supabase?~~ ✅ LOCKED: keep Payload. Entire system (collections, scraper, Typesense sync, content automation, admin UI) already runs on it; migration = months of rewrite for one feature (auth) buildable in days. App eats JSON regardless of backend. Revisit only on a wall Payload genuinely can't scale past.
4. **Hosting for app API + push:** same Next deployment (Vercel) or split service? (Same to start.)
5. **Video provider (Phase 3):** Mux vs Cloudflare Stream. (Defer.)
6. **Expo managed vs bare:** managed recommended unless a native module forces bare.
```
