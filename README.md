# Cheaply — mobile app

Local-business **and** online-webshop discovery/deals app. Expo (React Native), iOS + Android. Anonymous-first browse; follow/save/push behind a soft-gate. Reuses the existing Cheaply backend (Payload CMS + Postgres + Typesense) via a thin `/api/app/v1` layer.

> **Agents:** read `AGENTS.md` first — it's the project's operating contract (architecture §15, build workflow §16). `CLAUDE.md` imports it.

## Docs — start here

Four altitudes, each cites the next. Build from the blueprint; it pulls the others.

| Doc | Role | Read when |
| --- | --- | --- |
| [docs/prd-mobile-app.md](docs/prd-mobile-app.md) | **Why** — vision, positioning, surfaces, phasing | understanding the product |
| [docs/build-plan-mobile-app.md](docs/build-plan-mobile-app.md) | **What** — reference spec (collections, endpoints, auth §4a, ranking) | you need detail behind a task |
| [docs/build-blueprint.md](docs/build-blueprint.md) | **How / order** — tasks (T1–T47), screens (S1–S9), security gates | deciding what to build next |
| [docs/build-workflow.md](docs/build-workflow.md) | **Procedure** — the per-screen/feature loop to follow | actually building |
| [docs/od-design-workflow.md](docs/od-design-workflow.md) | **Design** — Open Design wiring + `start_run` recipe | designing a screen |

Flow: `PRD → build-plan → blueprint → build-workflow`, all bound by `AGENTS.md` §15 (layers) + §16 (workflow), with security skills (`security-guidance` auto, `owasp-security`, `/security-review`) and Open Design.

## Stack

Expo SDK 54 + Expo Router · gluestack-ui v3 + NativeWind · TanStack Query + Zustand · layered `screens → hooks → services → lib` (see `AGENTS.md` §15).

## Status

Planning + design setup complete. App not yet scaffolded — first build task is blueprint **T1** (Expo init). Verify Expo APIs against https://docs.expo.dev/versions/v54.0.0/ before coding.

## Get started (once scaffolded)

```bash
npm install
npx expo start
```

Open on an [iOS simulator](https://docs.expo.dev/workflow/ios-simulator/), [Android emulator](https://docs.expo.dev/workflow/android-studio-emulator/), or [Expo Go](https://expo.dev/go).
