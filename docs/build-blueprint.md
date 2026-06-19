# Build Blueprint — Cheaply App

_**Execution checklist** for the Expo app — the ordered **how** (tasks, DoD, security gates, screens). The detailed **what** lives in `docs/build-plan-mobile-app.md` (reference spec); the **why** in `docs/prd-mobile-app.md`; the architecture in `AGENTS.md` §15. Build from this doc; open the spec for the detail behind each task. Verify every Expo API against https://docs.expo.dev/versions/v54.0.0/ before coding._

Status: Draft · Date: 2026-06-19

---

## How to use this doc

- **Two tracks, cross-linked.** Track A = ordered tasks (`T#`). Track B = screens, each pointing at the Track-A tasks that build it.
- **Work top-down inside a phase.** A task is startable only when its `Dep:` tasks are `[x]`.
- **A phase isn't "done" until its Security gate passes.**

### Legend
- `- [ ]` open · `- [x]` done · `- [~]` in progress (edit the box).
- `T#` = stable task id (never renumber; append only).
- **Layer:** `scaffold | models | lib | services | hooks | screens | backend`.
- **Dep:** task ids that must be done first.
- **DoD:** testable definition of done.
- **🔒 Sec:** security criteria + the concrete threat mitigated (only on tasks touching auth/data/I-O/storage/push).
- Screens reference tasks as `→ T44`.

### Open decisions (resolve before the tasks they block)
- [ ] **Brand color.** gluestack v3 + NativeWind locked; primary hue TBD. Mock default = teal (`#0F6E56`/`#9FE1CB`). Blocks: T3 theme tokens, all Track-B design steps. _Run `impeccable` to lock a palette before T44._
- [ ] **App API hosting** — same Vercel deploy vs split service (build-plan §6.4). Blocks: T5 base URL/env.
- [ ] **AnalyticsService sink** — lightweight `AppEvents` collection vs external analytics (build-plan §2.7). Blocks: T33.
- [ ] **Apple/Google client ids** — provisioning before T17/T37/T38.

### Global conventions
- Layered imports enforced by lint (T2): screens → hooks → services → lib; models flow up. UI (gluestack) only in `screens`/components.
- Every service throws on error; every hook catches via TanStack Query.
- All network through `lib/apiClient` (HTTPS only, auth header injected, refresh rotation). No `fetch` in services directly.
- Tokens only in `expo-secure-store`. Never AsyncStorage, never in JS-visible global state at rest.
- Security skills in play: `security-guidance` (while coding) · `owasp-security` (auth/data design) · `/security-review` (phase gates).

---

# PHASE 1 — Consumer MVP

Surfaces: Home + dedicated pages, Map, Discover, Business profile, Saved/Following, Profile, Auth soft-gate. Goal: prove daily-habit retention in Leiden + follow-driven online discovery.

## Milestone 0 — Foundation (scaffold)

- [ ] **T1 — Init Expo (managed) + TypeScript + Expo Router.** Layer: scaffold. DoD: app boots on iOS + Android sim; a blank route renders; SDK = 54 confirmed.
- [ ] **T2 — Folder skeleton + path aliases + import-boundary lint.** Layer: scaffold. Dep: T1. DoD: `src/{models,services,hooks,lib,screens}` exist; `@/…` aliases resolve; lint **fails** when a screen imports `services`/`lib`.
- [ ] **T3 — gluestack-ui v3 + NativeWind install + theme tokens + root provider.** Layer: scaffold. Dep: T1, Open:brand. DoD: a themed gluestack `Button` + `Box` render with brand tokens; verified against SDK 54 (gluestack v3 shipped on SDK 53 — confirm 54).
- [ ] **T4 — TanStack Query + Zustand stores (auth, location).** Layer: lib. Dep: T2. DoD: `QueryClientProvider` mounts; `authStore`/`locationStore` typed + readable.
- [ ] **T5 — `lib/apiClient`.** Layer: lib. Dep: T2. DoD: typed `get/post/patch/del` over `/api/app/v1`; base URL from env; throws typed error on non-2xx. 🔒 Sec: HTTPS-only enforced; no secrets in bundle; base URL via `expo-constants`/env, not hardcoded. Threat: secret exposure in client (OWASP A02).
- [ ] **T6 — `lib/secureStore`.** Layer: lib. Dep: T2. DoD: `setToken/getToken/clear` round-trip via expo-secure-store. 🔒 Sec: Keychain/Keystore only. Threat: token theft from insecure storage.
- [ ] **T7 — i18n (i18next + expo-localization, en/nl).** Layer: lib. Dep: T2. DoD: `t('key')` resolves both locales; nl/en parity lint or test.
- [ ] **T8 — Author all Phase-1 models.** Layer: models. Dep: T2. DoD: `User, Business(+BusinessType), BusinessLocation, Product, Deal, Category, HomeSection, CardItem, MapPin, SearchFilters, Follow, Save, AuthSession, NotificationPrefs, NotificationPayload` typed; mirror `/api/app/v1` shapes; `tsc` clean.

## Milestone 1 — API + auth skeleton (backend-led)

> Backend tasks land in the existing Payload/Next repo (`/api/app/v1`), not the Expo app. Sequenced here because FE auth depends on them.

- [ ] **T9 — BE: `Consumers` collection + migration.** Layer: backend. DoD: fields email(unique)/name?/locale/homeLocation/pushTokens[]/notificationPrefs/createdAt; admin-only read. 🔒 Sec: PII minimized; field-level access control; no public list endpoint. Threat: PII over-exposure (A01).
- [ ] **T10 — BE: `Follows` + `Saves` collections + indexes.** Layer: backend. Dep: T9. DoD: edges with consumer rel; indexed on consumer + business/category; owner-scoped access.
- [ ] **T11 — BE: `companies` field additions + migration.** Layer: backend. DoD: `type(local|online|hybrid)`, `shopUrl`, `shipsTo`, `followerCount`, `claimedBy`, `hours`, `phone`; existing rows default `local` (infer `online` when no geocodable address); online rows have no `company-locations`.
- [ ] **T12 — BE: login-code store.** Layer: backend. Dep: T9. DoD: 6-digit code, hashed at rest, ~10 min TTL, single-use, max-attempts→lockout. 🔒 Sec. Threat: code brute-force → max attempts + lockout + hashing.
- [ ] **T13 — BE: `POST /auth/request-code`.** Layer: backend. Dep: T12. DoD: sends code via Nodemailer; returns generic 200 regardless of email existence; rate-limited per email + per IP. 🔒 Sec. Threat: email enumeration + email-bomb → generic response + rate limit.
- [ ] **T14 — BE: `POST /auth/verify`.** Layer: backend. Dep: T12. DoD: validates code; create-or-login `Consumer`; returns JWT (~15 min) + rotating refresh token. 🔒 Sec. Threat: token theft / replay → short JWT, refresh rotation, server-side revoke on sign-out.
- [ ] **T15 — BE: JWT auth middleware.** Layer: backend. Dep: T14. DoD: guards `/me`, `/follow`, `/save`, `/following`, `/saved`, push register; rejects missing/expired/invalid. 🔒 Sec. Threat: broken access control (A01) → authZ verified on every protected route, not just first.
- [ ] **T16 — BE: `GET/PATCH /me`.** Layer: backend. Dep: T15. DoD: returns profile+prefs; PATCH updates location/prefs/push token; owner-only. 🔒 Sec: validate push-token format + locale enum; reject foreign-id writes.
- [ ] **T17 — BE: Apple + Google token verification endpoints.** Layer: backend. Dep: T14. DoD: verifies provider id-token server-side; maps to create-or-login. 🔒 Sec. Threat: forged id tokens → verify signature/`aud`/`iss`/expiry server-side, never trust client claims.
- [ ] **T18 — FE: auth libs.** Layer: lib. Dep: T3. DoD: `lib/notifications` (expo-notifications), `lib/location` (expo-location), `lib/maps` (react-native-maps), `lib/appleAuth` (expo-apple-authentication), `lib/googleAuth` — each a thin typed wrapper.
- [ ] **T19 — FE: `AuthService`.** Layer: services. Dep: T5, T6, T13, T14, T17. DoD: `requestCode/verify/refresh/signOut/appleSignIn/googleSignIn`; persists tokens via secureStore; silent refresh. 🔒 Sec: clear tokens + server revoke on signOut. Threat: stale-session reuse.
- [ ] **T20 — FE: `UserService`.** Layer: services. Dep: T16, T19. DoD: `getMe/updateMe/registerPushToken`; tested vs apiClient mock.
- [ ] **T21 — FE: `useAuth` + `useUser`.** Layer: hooks. Dep: T19, T20. DoD: expose `{user, loading, error, requestCode, verify, signOut}`; gate state for soft-gate.

## Milestone 2 — Read surfaces (backend + FE)

- [ ] **T22 — BE: read endpoints.** Layer: backend. Dep: T11. DoD: `/home`, `/deals`, `/products`, `/businesses`, `/map`, `/search`, `/business/:slug` live; wrap existing product service + Typesense; `scope` param filters `type`; map/geo excludes online; pagination cursors. 🔒 Sec: validate + clamp all query params (lat/lng/bbox/limits); cap page size; never leak Payload admin shape. Threat: injection / resource exhaustion (A03) → param validation + caps.
- [ ] **T23 — BE: ranking compose.** Layer: backend. Dep: T22. DoD: local = geo+discount+recency+ending-soon+follow-boost; online = relevance+discount+recency+follow-boost; home = capped per-type rows, hide empty.
- [ ] **T24 — FE: `HomeService`.** Layer: services. Dep: T5, T22. DoD: `getHome(geo,scope)→HomeSection[]`; tested.
- [ ] **T25 — FE: `DealService` + `ProductService`.** Layer: services. Dep: T5, T22. DoD: `list(filters,cursor)`/`get(id)` each; tested.
- [ ] **T26 — FE: `BusinessService`.** Layer: services. Dep: T5, T22. DoD: `get(slug)→Business`; `discover(filters,cursor)→{items,cursor}`; tested.
- [ ] **T27 — FE: `MapService` + `SearchService`.** Layer: services. Dep: T5, T22. DoD: `getPins(...)` (online excluded); `search(filters)` typed results; tested.
- [ ] **T28 — FE: read hooks.** Layer: hooks. Dep: T24–T27. DoD: `useHome, useDeals, useProducts, useBusinesses, useBusiness, useMap, useSearch, useLocation` — Query-backed, paginated where relevant.

## Milestone 3 — Engagement (follow / save)

- [ ] **T29 — BE: follow/save endpoints + `followerCount` hook.** Layer: backend. Dep: T15, T10. DoD: `POST/DELETE /follow`, `POST/DELETE /save`, `GET /following`, `GET /saved`; `followerCount` denormalized via hook; owner-scoped. 🔒 Sec: authZ owner-only; idempotent toggles. Threat: forced-browsing other users' edges (A01).
- [ ] **T30 — FE: `FollowService` + `SaveService`.** Layer: services. Dep: T5, T29. DoD: follow/unfollow/listFollowing, save/unsave/listSaved; tested.
- [ ] **T31 — FE: engagement hooks.** Layer: hooks. Dep: T30. DoD: `useFollow, useFollowing, useSave, useSaved`; mutations invalidate related queries.
- [ ] **T32 — FE: `AnalyticsService.track`.** Layer: services. Dep: T5, Open:analytics. DoD: `track(directions|call|visit, businessId)` → events sink. 🔒 Sec: no PII in event payload.

## Milestone 4 — Push + polish

- [ ] **T33 — BE: push pipeline (followed-only).** Layer: backend. Dep: T9, T29. DoD: store Expo tokens; trigger on new qualifying product/deal for followed business/category (hook into product-save→Typesense-sync); deal-ending-soon cron; send via Expo Push; digest batching; respect prefs + quiet hours. Proximity triggers deferred. 🔒 Sec: validate token ownership; rate/volume caps. Threat: push spam / token abuse.
- [ ] **T34 — BE: GDPR endpoints.** Layer: backend. Dep: T9. DoD: data export + account delete (cascades follows/saves/tokens). 🔒 Sec. Threat: GDPR non-compliance → export + erase + consent record.
- [ ] **T35 — FE: `NotificationService` + register wiring.** Layer: services+hooks. Dep: T18, T20, T33. DoD: `register()` post-auth → `registerPushToken`; `handleTap(payload)→deepLink`; `usePush`.
- [ ] **T36 — FE: deep links.** Layer: scaffold. Dep: T35. DoD: notification tap → Expo Router route (deal/business); cold + warm start handled.
- [ ] **T37 — FE: states + i18n QA pass.** Layer: screens. Dep: all M2/M3 screens. DoD: every screen has loading/empty/error/offline + anon-vs-authed; nl/en parity verified.

## Milestone 5 — Ship

- [ ] **T38 — EAS Build + Submit + store listings.** Layer: scaffold. Dep: T37, Phase-1 gate. DoD: iOS TestFlight + Android internal track live in Leiden; privacy disclosures (GDPR) filed; store assets in nl/en.

---

## Track B — Phase 1 screens

> Each: purpose → hooks/services → components (gluestack v3) → states → design step → security → build tasks. Design step for every screen: **(1) `impeccable` pass** (hierarchy, spacing, motion, copy), **(2) Claude Design mockup**, **(3) record reused tokens/components**.

### S1 — Onboarding / location · PRD §6.1, build-plan §3.6
- Hooks: `useLocation`, `useUser`. Services: device location, postcode→geo (reuse backend lookup).
- Components: `Box`, `Input` (postcode), `Button`, permission sheet.
- States: permission-denied (manual postcode), geocode-fail, anonymous (default — no sign-in here).
- 🔒 Sec/privacy: location consent prompt; store home geo on device + `/me` only after auth; nothing requires account.
- Build: → T8, T28 (`useLocation`), T35 (perms). Design: impeccable + mockup.

### S2 — Home (categorized overview) · PRD §6.1
- Hooks: `useHome`. Service: `HomeService`.
- Components: location header, search entry, scope chips (All/Local/Online), section rows (`HomeSection`) of `CardItem` (Deal/Product/Business cards), bottom tab bar.
- States: loading (skeleton rows), empty (hide section), error (retry), offline (cached home), anon (location+online) vs authed (follow-personalized).
- 🔒 Sec: anonymous-safe; no auth-only data; follow CTA → soft-gate (S9).
- Build: → T24, T28, T44-equiv (this screen). Design: impeccable + mockup (✅ first pass done).

### S3 — Dedicated pages: Deals / Products / Businesses · PRD §6.1a
- Hooks: `useDeals` / `useProducts` / `useBusinesses`. Services: Deal/Product/Business.
- Components: filter/sort bar, infinite list/grid, card, scope filter.
- States: loading (skeleton), empty ("no results, widen filters"), error, end-of-list, offline.
- 🔒 Sec: read-only, anonymous-safe; param validation server-side (T22).
- Build: → T25, T26, T28. Design: impeccable + mockup per page (shared card components).

### S4 — Map · PRD §6.2
- Hooks: `useMap`, `useLocation`. Service: `MapService`.
- Components: map (react-native-maps), clustered pins, filter bar, tap → preview card → profile.
- States: loading pins, empty viewport, location-denied (default region), error, offline.
- 🔒 Sec: local/hybrid only (online excluded server-side); no precise-location upload beyond viewport query.
- Build: → T27, T28. Design: impeccable + mockup.

### S5 — Discover / search · PRD §6.3
- Hooks: `useSearch`. Service: `SearchService`.
- Components: search input, typed result rows (business/deal/product), category hubs → dedicated page (filtered), scope filter.
- States: idle (categories), loading, no-results, error, offline.
- 🔒 Sec: query sanitized + length-capped server-side; anonymous-safe.
- Build: → T27, T28. Design: impeccable + mockup.

### S6 — Business profile (local + online variants) · PRD §6.4
- Hooks: `useBusiness(slug)`, `useFollow`. Services: `BusinessService`, `FollowService`, `AnalyticsService`.
- Components: header (logo, follow btn, follower count), tabs (deals/products + locations/about for local/hybrid), action bar — **local/hybrid:** Directions/Call/Visit/Follow; **online:** Visit shop/Follow.
- States: loading, not-found, no-offers (still followable), error, offline; follow btn anon → soft-gate.
- 🔒 Sec/privacy: outbound actions (`track`) carry no PII; follow gated; Directions/Call open native intents (validate URLs/tel).
- Build: → T26, T30, T31, T32. Design: impeccable + **two mockups** (local, online).

### S7 — Saved & Following · PRD §6.6
- Hooks: `useSaved`, `useFollowing`. Services: Save/Follow.
- Components: tabbed lists, card, empty CTA → Discover.
- States: loading, empty (anon → soft-gate prompt + value pitch), error, offline.
- 🔒 Sec: auth-required; never render another user's data; 401 → re-auth.
- Build: → T30, T31. Design: impeccable + mockup.

### S8 — Profile / Settings · PRD §6.7
- Hooks: `useUser`, `useAuth`, `usePush`. Service: `UserService`.
- Components: location editor, notification prefs (frequency, quiet hours), sign-in/out, GDPR (export/delete), locale toggle.
- States: anon (sign-in CTA) vs authed (full), saving, error.
- 🔒 Sec: GDPR export/delete (T34); sign-out clears secureStore + revokes; prefs validated.
- Build: → T20, T21, T34, T35. Design: impeccable + mockup.

### S9 — Auth soft-gate modal · build-plan §4a
- Hooks: `useAuth`. Service: `AuthService`.
- Components: modal, email input, 6-digit code input, Apple/Google buttons, resend (rate-aware).
- States: email-entry, code-sent, verifying, error (invalid/expired/locked-out), success → resume gated action.
- 🔒 Sec: no enumeration (generic copy), resend rate-limited UI, code masked, tokens → secureStore, never log code/email. Threat: enumeration + brute-force surfaced in UX.
- Build: → T19, T21. Design: impeccable + mockup.

---

## 🔒 Phase 1 — Security gate (must pass before "Phase 1 done")

`security-guidance` runs automatically during coding (edit warnings + stop/commit review) — no unresolved findings allowed into the gate. At the gate, also run `owasp-security` review + `/security-review` on the branch. All boxes green:

- [ ] **`security-guidance` clean** — no unresolved plugin findings on the phase's diff/commits.

- [ ] **Input validation** — every `/api/app/v1` query/body param validated, typed, length/range-capped; page sizes clamped. (A03)
- [ ] **AuthZ on every protected endpoint** — `/me`, follow, save, following, saved, push verified server-side; forced-browsing another consumer's data returns 403/404. (A01)
- [ ] **Auth tokens** — JWT ≤15 min, refresh rotates, server-side revoke on sign-out; no token in logs. (A07)
- [ ] **Rate limiting** — `request-code` + `verify` limited per email + IP; lockout after N attempts. (A07)
- [ ] **Secure storage** — tokens only in Keychain/Keystore; grep confirms zero AsyncStorage token writes. (A02)
- [ ] **No client secrets** — bundle scanned; API base/keys via env; no embedded service credentials. (A02)
- [ ] **Provider tokens** — Apple/Google id-tokens verified server-side (sig/aud/iss/exp). (A07)
- [ ] **Transport** — HTTPS enforced everywhere; no cleartext. (A02)
- [ ] **GDPR** — consent at first sign-up; data export + account delete working; privacy policy in store listing.
- [ ] **Dependency audit** — `npm audit` / Snyk clean of high+; gitleaks/trufflehog scan clean.
- [ ] **Push** — token ownership validated; volume caps; quiet hours respected.
- [ ] **PII hygiene** — analytics + push payloads carry no PII; error responses generic.

---

# PHASE 2 — Business self-serve (lighter)

Build on Phase 1 patterns. New: business accounts, claim flow, manual CRUD, events, curated collections.

- [ ] **T39 — BE: business auth (fresh, reuse dormant scaffolding design).** 🔒 Sec: higher stakes → plan optional 2FA; claim verification (domain email / admin). Threat: listing takeover → ownership proof.
- [ ] **T40 — BE: `Events` + `Collections` collections + `/events`, `/collections` endpoints.**
- [ ] **T41 — BE: business self-serve CRUD (products/deals/events), Shopify connect (OAuth), stats (`AppEvents`).** 🔒 Sec: OAuth state/PKCE; scope-limited tokens; tenant isolation on every write. (A01)
- [ ] **T42 — FE: `EventService`, `CollectionService` + `useEvents`, `useCollections`.**
- [ ] **T43 — FE: Events page + Collections in Discover (Track B).** Design: impeccable + mockups.
- [ ] **T44 — FE: business-mode screens** (manage profile, CRUD, Shopify connect, stats). 🔒 Sec: role-gated UI; never expose other tenants' data.
- [ ] **🔒 Phase 2 security gate** — multi-tenant authZ (no cross-tenant read/write), OAuth hardening, claim-flow abuse, business-account rate limits, `/security-review`.

# PHASE 3 — Reels + monetization (lighter)

- [ ] **T45 — BE: video upload → storage + transcode (Mux/Cloudflare Stream) + CDN; `Reels.status` moderation gate before publish.** 🔒 Sec: signed upload URLs; content moderation; MIME/size validation.
- [ ] **T46 — FE: `ReelService` + `useReels` + Reels page (vertical player, `expo-av`).** Design: impeccable + mockup.
- [ ] **T47 — Monetization: boosted/featured placement (paid), priority ranking.** 🔒 Sec: payment provider PCI offload (no card data in app); server-side entitlement checks.
- [ ] **🔒 Phase 3 security gate** — upload abuse, moderation bypass, media access control, billing/entitlement integrity, `/security-review`.

---

## Definition of Done — whole app

- All Phase 1 tasks `[x]`; Phase 1 security gate fully green; `/security-review` clean on `main`.
- Every screen: loading/empty/error/offline + anon-vs-authed states; nl/en parity.
- Layered architecture lint passes; UI confined to `screens`; no service bypasses `apiClient`.
- Anonymous browse works with zero account; follow/save/push gated.
- Shipped to TestFlight + Play internal in Leiden with GDPR disclosures.
- Each screen has an `impeccable`-reviewed Claude Design mockup; tokens consistent across screens.

## Start here — first 5 tasks
1. **T1** — Expo init + Router.
2. **T2** — Folder skeleton + import-boundary lint (locks the architecture early).
3. **T3** — gluestack v3 + NativeWind + brand tokens (resolve Open:brand via `impeccable` first).
4. **T5 + T6** — `apiClient` + `secureStore` (the two lib spines everything else rides on).
5. **T8** — Author all Phase-1 models (unblocks every service).

_Parallel backend track can start T9–T14 (Consumers + auth) immediately — FE auth (T19) waits on them._
