# Open Design — workflow & wiring

_How the Cheaply app screens get designed via Open Design (OD). Captures the setup so it's not re-derived._

## What OD is here

Local-first design engine at `~/Desktop/Personal/open-design`. It ingested the **gluestack v3 Figma kit** and extracted a faithful design system. Drives agent-built screens via skills + runs. Replaces Claude Design for this project (don't use `DesignSync`).

## Daemon

- Runs with the **OD desktop app** — keep the app open while designing. Durable, survives Claude Code restarts.
- Listens on an **ephemeral port** (e.g. `127.0.0.1:61664`), not a fixed one. Don't hardcode it.
- Manual start (if app closed): `cd ~/Desktop/Personal/open-design && node apps/daemon/bin/od.mjs --no-open`. CLI bin: `apps/daemon/bin/od.mjs`.

## MCP wiring (`.mcp.json`)

```json
{ "mcpServers": { "open-design": {
  "command": "node",
  "args": ["/Users/d.vanderhoogt/Desktop/Personal/open-design/apps/daemon/bin/od.mjs", "mcp"]
}}}
```

- **No `OD_DAEMON_URL` env.** Setting it overrides OD's auto-discovery and pins a (wrong, ephemeral) port. Leave it out → the MCP discovers the live app daemon every spawn.
- MCP server caches the daemon URL at spawn → **restart Claude Code after a daemon restart** to re-discover.
- Verify connection: `list_projects` (MCP tool) returns the projects.

## Key MCP tools

`list_projects` · `list_skills` · `list_agents` · `create_project` · `start_run` · `get_run` · `get_artifact` · `get_file` / `write_file` / `list_files` / `search_files`.

## Assets in OD

- **gluestack DS** = project `3801baef-74be-45a2-9dba-79c038c94a5f` ("Figma migration", status succeeded). Holds `tokens.css` (gluestack v3 scale, NativeWind v4-compatible; primary default indigo `#6366F1`), `tailwind.config.js`, `token-map/`, `components/` (forms, navigation, overlay, feedback, media, primitives…).
- **Design skill** = `frontend-design` (platform mobile, design-system-aware, covers application screens).
- **Agent** = `claude` (Claude Code). Codex + Cursor also available.

## Build recipe (per screen / batch)

Token wiring = **option A**: a dedicated screens project seeded with the gluestack tokens.

1. `create_project(name: "cheaply-screens", skill: "frontend-design")`.
2. Seed it: copy `tokens.css` + `tailwind.config.js` + `components/` from `3801baef` via `write_file` (or `get_file` → `write_file`).
3. `start_run(project: "cheaply-screens", skill: "frontend-design", agent: "claude", prompt: <screen spec>)` → `runId`.
   - Prompt = the blueprint Track-B block for the screen (states, security notes) + "use the project's gluestack tokens.css/components, stay on-system, colors tokenized (brand recolor later = single primary-token swap)".
4. Poll `get_run(runId)` until `succeeded` → `get_artifact` to pull HTML (or open `previewUrl`).
5. Build order: **Home + Business profile first** → extract shared components → other screens reuse.

## Conventions

- Brand color deliberately deferred → design on gluestack indigo default; recolor = one `--color-primary-*` swap. Keep everything tokenized.
- Every screen covers loading / empty / error / offline / anonymous-vs-authed (mirror blueprint Track B + 🔒 Sec).
- OD is design only. App architecture/layers stay per `AGENTS.md` §15; tasks per `docs/build-blueprint.md`.
