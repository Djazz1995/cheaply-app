# Build Workflow — how to build screens & features

_Operating procedure for any agent building Cheaply. Follow this loop for every screen and every feature. It binds the blueprint (what/order), the layered architecture (`AGENTS.md` §15), Open Design (design), and the security skills (safety) into one repeatable process. Do not freelance around it._

---

## 0. Preconditions (once, before any screen)

Foundation must exist before building any screen or service. From `docs/build-blueprint.md`:

- **T1–T8 done:** Expo init + Router, folder skeleton + import-boundary lint, gluestack-ui v3 + NativeWind provider, TanStack Query + Zustand, `lib/apiClient`, `lib/secureStore`, i18n, all Phase-1 models.
- OD app running (daemon auto-discovered), `open-design` MCP connected (`list_projects` works). See `docs/od-design-workflow.md`.

If foundation is incomplete, build it first — a screen against no spine is invalid.

---

## 1. The build loop (per screen)

Repeat for each screen S1–S9 in blueprint build-order (**Home + Business profile first**, then the rest reuse shared components).

### Step 1 — Read the blueprint
- Open the screen's **Track B block** in `docs/build-blueprint.md`: purpose, hooks→services, components, **every state** (loading/empty/error/offline/anon-vs-authed), 🔒 Sec notes, cross-linked `T#` tasks.
- Open the **reference spec** (`docs/build-plan-mobile-app.md`) for the endpoints/data shapes behind it.
- Note the screen's Track-A dependency tasks (the `T#`s it cross-links).

### Step 2 — Build the screen's data spine (Track A) first
A screen is not just UI. Before the UI, build/confirm its cross-linked tasks in layer order `models → lib → services → hooks`:
- models exist (T8), service(s) implemented + unit-tested against `apiClient` mocks, hook(s) wrapping them with TanStack Query.
- Backend endpoints it needs are live (or mocked) per the spec.
- Mark each `T#` `[~]` while in progress, `[x]` when its DoD passes.

### Step 3 — Design with Open Design → you VERIFY
- Run OD per `docs/od-design-workflow.md` recipe:
  `start_run(project: "cheaply-screens", skill: "frontend-design", agent: "claude", prompt: <Track-B block + "use project gluestack tokens.css/components, stay on-system, all states, colors tokenized">)`.
- Poll `get_run` → `succeeded` → `get_artifact` (or open `previewUrl`).
- **STOP. Present the design to the user for verification.** Do not build UI until the user approves the design. Iterate via OD on feedback.

### Step 4 — Build the UI on the approved design
- Translate the OD artifact (HTML) into **React Native + gluestack-ui v3** components on the same tokens. This is a translation, not copy-paste (OD outputs web).
- UI lives only in `screens`/components. Screens import hooks + models — never `services`/`lib` (lint enforces, `AGENTS.md` §15).
- Implement **every state** from the Track-B block. Cover anon-vs-authed; follow/save/push → soft-gate (S9), never a hard wall.
- `security-guidance` runs automatically on edits — **fix its findings or justify them; never ignore.**

### Step 5 — VERIFY it runs
- Launch the screen on a simulator (`/run` or the `verify` skill). Design-correct ≠ working.
- Confirm states render (force loading/empty/error/offline), nl/en parity, navigation/deep-link in/out.

### Step 6 — Security review (right cadence)
- `security-guidance` already covered the edits (auto).
- Run **`/security-review`** now **only for auth/data screens** — S9 (auth), S8 (settings/GDPR), S6 (outbound actions), or any screen touching tokens/secure-store/PII.
- For pure-browse screens, **defer** the review to the phase Security gate (don't run it per screen — wasteful).
- Apply the screen's 🔒 Sec criteria from the blueprint as DoD.

### Step 7 — Close out
- Tick the screen's `T#` boxes `[x]` in the blueprint.
- Confirm DoD: code + all states + nl/en parity + 🔒 Sec criteria + layered-lint green.

---

## 2. Feature (non-screen) variant

For backend endpoints, services, libs, push, auth — same skeleton **minus Step 3/4 OD design**:
1. Read blueprint task + spec.  2. Build in layer order.  3. Unit-test (services vs `apiClient` mocks; endpoints vs Vitest).  4. `security-guidance` auto + `/security-review` if it touches auth/data/I-O/storage/push.  5. Tick `T#`.

---

## 3. Phase gate (end of each phase)

Do not advance a phase until its **Security gate** in `docs/build-blueprint.md` is fully green:
- Run `/security-review` on the branch + an `owasp-security` pass.
- All gate boxes checked (input validation, authZ, JWT/refresh, rate-limit, secure storage, no client secrets, provider-token verify, GDPR, dep audit, push, PII).
- `security-guidance` has no unresolved findings on the phase diff.

---

## 4. Rules that always hold

- **Layered architecture** (`AGENTS.md` §15): `screens → hooks → services → lib`; models flow up; UI never in services/hooks/lib; services throw, hooks catch.
- **Anonymous-first:** browse needs no account; gate only follow/save/push.
- **Tokens only in `expo-secure-store`.** Never AsyncStorage.
- **One component set:** shared cards (deal/product/business), headers, tab bar, action bar defined once, reused.
- **Brand color deferred:** design on gluestack defaults, everything tokenized → later recolor = single primary-token swap.
- **OD outputs web; the app is React Native.** Always translate, never paste.
- **Flag contradictions** between docs as you hit them; don't silently design around them.

---

## TL;DR loop

`foundation once → [ blueprint → data spine (Track A) → OD design → USER verifies → build RN UI (security-guidance auto) → verify runs → /security-review if auth/data → tick T# ] → phase security gate → advance`
