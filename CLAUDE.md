# CLAUDE.md

This file guides Claude Code (and other agents) working in a
[DraconDex](https://github.com/ZYDRAXYL/DraconDex-APP) plugin repo. This repo
itself is the **minimal starter template** for DraconDex 5 — one window, one
table — forked/used-as-template to create new plugins. Keep this file generic
and accurate for that purpose; when you fork this repo into a real plugin,
update the project-specific bits (title, table names, feature list) but keep
the architecture/constraints sections, since they describe the platform, not
this template's business logic.

Full human-facing docs (quick start, manifest field rules, install-link
shapes, adding a side panel, legacy `extApi` support) are in
[README.md](README.md) — read that first for anything not covered below.

## This repo is in the DraconDex chain

`DraconDex-PGI-Template` sits at the tail of all three chains
(`… > PGI, EXT`, `chain/chain.json`, key `PGI`). Two edges arrive, none leave:

| edge | carries | lands as |
|---|---|---|
| APP → PGI | `claude-tooling` | `.claude/`, `chain/`, `tools/chain-*.mjs`, `tools/validate-manifest.mjs`, `tools/plugin-contract.mjs` — **edit them in DraconDex-APP**, never here |
| EXE → PGI | `plugin-contract` | `tools/plugin-manifest.cjs` (byte-identical copy of EXE's `electron/src/db/plugin-manifest.js`) + `plugin-contract.lock.json` |

What this repo owns: `dracondex-plugin.json`, `index.html`, `app.js`,
`style.css`, `.dracondex`, the CI workflow, README and this file. Its sibling
`DraconDex-EXT-Template` is the full template (window + side panel); keep the
two consistent on platform facts.

A plugin *made from* this template is not in the chain: the
`extension-scaffold` skill removes `chain/` and the chain skills and keeps the
contract (`tools/plugin-manifest.cjs`, `tools/validate-manifest.mjs`,
`tools/plugin-contract.mjs`, the lock).

## What a DraconDex plugin is

A plugin is **not** code running inside the main app. It opens in its own
window with **no access** to the main app's data or `window.api`, and no
access to any other plugin's data — only to the SQLite table(s) *this*
plugin declares, through `window.pluginApi`. Ownership is resolved from the
calling window, not from anything the page sends, so there is no way for a
plugin to reach data that isn't its own. See DraconDex-APP's
`docs/PLUGINS.md` for the full architecture and its honest limits.

## Structure

| File | Purpose |
| --- | --- |
| `.dracondex` | Marker file tagging this repo as a plugin, for DraconDex's discovery list. Not downloaded on install. |
| `dracondex-plugin.json` | Manifest: id, name, version, entry point, `files`, table schema, optional `panels`/`permissions`/`dependencies`. |
| `index.html` | Entry point — must be listed in `files` and match `entry`. |
| `app.js` | Plugin logic. Talks to its own table(s) via `window.pluginApi.table.*`. |
| `style.css` | Optional styling; DraconDex 5's `daylight`/`midnight` palettes, following the OS. |
| `tools/validate-manifest.mjs` | Local manifest check running the app's own `validateManifest()`. Not shipped (not in `files`). |
| `tools/plugin-manifest.cjs` + `plugin-contract.lock.json` | The app's rules, vendored from DraconDex-EXE at a pinned release. Never hand-edit — `npm run contract` fails on it. |

Only paths listed in the manifest's `files` are ever downloaded by an
installing user — README, scripts, CI, and tests cost them nothing. Add new
source files to `files` when you add them, or they silently won't ship.

## Manifest rules (enforced by DraconDex on install)

- `id` — `^[a-z0-9_]{1,20}$`. **Becomes part of real DB table names
  (`plg_<id>_<table>`) — pick it once and never change it after anyone has
  installed.** Two plugins with the same `id` can't install side by side.
- `name` — string, max 80 chars. `version` — optional string, max 40 chars.
- `entry` — an HTML file, and it must also appear in `files`.
- `files` — 1 to 30 relative paths (no `..`, no leading `/`, no `\`), each
  fetched individually, capped at 2 MB. Subdirectories are fine.
- `tables` — up to 10, each with 1 to 25 columns.
  - table `name`: `^[a-z0-9_]{1,20}$`; `id`+`name` combined must stay within
    41 characters.
  - column `name`: `^[a-z][a-z0-9_]{0,29}$`; can't be `id`, `rowid`, `oid`,
    or `_rowid_`.
  - column `type`: `TEXT`, `INTEGER`, or `REAL` only — no `DEFAULT`, `CHECK`,
    or `FOREIGN KEY`.
  - every table gets an implicit `id INTEGER PRIMARY KEY AUTOINCREMENT` you
    don't declare and can't override.
- Optional `panels` (a page in the DraconDex 5 side panel; 4.3–4.x docked it
  in place of the Module Inspector),
  `permissions.net`/`permissions.context` (declare allowed origins / context
  a panel can receive, 4.3.0+), and `dependencies` (auto-install other
  plugin repos, 4.8.0+) — see README for the full shape if you add these.

## The `window.pluginApi` surface

Inside a plugin window there is no `window.api`, no Node, no filesystem —
only:

```js
await window.pluginApi.table.getSchema(localName)       // { columns: [{ name, type }, …] }
await window.pluginApi.table.query(localName, filter)   // rows, newest id first
await window.pluginApi.table.insert(localName, row)     // { id }
await window.pluginApi.table.update(localName, id, row) // { changes }
await window.pluginApi.table.delete(localName, id)      // { changes }
```

- `localName` is the `name` declared under `tables` (e.g. `"notes"`), not the
  internal `plg_*` name.
- `filter` is exact-match column equalities ANDed together; `{}` returns
  everything — no operator syntax, no raw SQL, no pagination. Filter/sort in
  JS.
- Keys in `filter`/`row` must be declared columns, or the call rejects with
  `unknown column: …`. Calls reject with `not an owned table` for any
  `localName` that isn't yours.
- If the manifest declares `permissions.net`, `window.pluginApi.net.fetch`
  can reach exactly those origins and nothing else — the install preview
  shows them, and DraconDex enforces them at runtime.

## Writing the window

Plugin windows are created frameless (`frame: false`, 900×650, min 480×360,
dark `#050506` background). Practical consequences:

- **Draw your own title bar** — `-webkit-app-region: drag` on the bar,
  `-webkit-app-region: no-drag` on every button inside it.
- **Give the user a way out** — there is no OS close button; call
  `window.close()` from your own UI.
- **The app's stylesheets are not injected** — ship whatever CSS you need in
  your own `files`.
- `window.prompt()` is unsupported in Electron renderers — build your own UI
  for it.
- **Nothing sanitizes your rendering** — treat stored rows as data; use
  `textContent`/DOM APIs, never `innerHTML`, for anything derived from a
  table or the network.
- If you add a `panels` entry (DraconDex 5 side panel): the host draws its
  header and ×, so the panel page has **no** title bar; it stays open across
  page changes and is **destroyed when closed** (on a 4.x host it was also
  reloaded on every pane re-render). Nothing may live only in a JS variable —
  persist state to your table at the moment it exists and rebuild from the
  table on load. With `permissions.context: ["module"]` the host pushes a new
  `context` message on every page change: keep `panel.onMessage` installed.
  Design for the side panel's 220px minimum width.
- Every HTML entry keeps the CSP `<meta>` from `index.html`: no remote
  resources, no inline `style=""` (`style-src 'self'` drops it silently).

## Commands

```bash
npm run validate                     # the app's own validateManifest (vendored), first error first
npm run contract                     # the vendored copy is byte-identical to the pin
npm run contract:upstream            # has DraconDex-EXE moved the contract past the pin?
node --check app.js                  # add every shipped .js file here as you add them
```

CI (`.github/workflows/validate.yml`) runs the contract check, the validator
and `node --check` on every push/PR, plus an advisory upstream check. No
dependencies to install, no build step — DraconDex downloads these files
as-is, so don't introduce one without also updating how the plugin installs.

## Developing / testing locally

In DraconDex: **Settings → Plugin → Plugins**, paste this repo's link, confirm
the preview (accepts `https://github.com/...`, `git@github.com:...`,
`owner/repo` shorthand, and GitLab equivalents; tries `main` then `master` if
no branch is given). Reinstalling after a change means uninstalling first
(same `id` can't install twice), and **uninstalling permanently deletes that
plugin's files and tables** — don't develop against data you care about.

## When forking this template

1. Keep `.dracondex` at the repo root.
2. Pick a real `id` in `dracondex-plugin.json` before anyone installs it —
   see the immutability warning above.
3. Update `files`, `index.html`/`app.js`/`style.css` (or replace them),
   `README.md`, and this `CLAUDE.md`'s title/structure table for your plugin.
4. Keep `tools/validate-manifest.mjs`, `tools/plugin-manifest.cjs`,
   `tools/plugin-contract.mjs`, the lock and the CI workflow; delete `chain/`
   and `tools/chain-*.mjs`. Extend the
   `node --check` list as you add scripts.
