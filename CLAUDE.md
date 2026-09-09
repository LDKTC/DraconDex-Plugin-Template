# CLAUDE.md

This file guides Claude Code (and other agents) working in a
[DraconDex](https://github.com/ZYDRAXYL/DraconDex-APP) plugin repo. This repo
itself is the **starter template** — forked/used-as-template to create new
plugins. Keep this file generic and accurate for that purpose; when you fork
this repo into a real plugin, update the project-specific bits (title, table
names, feature list) but keep the architecture/constraints sections, since
they describe the platform, not this template's business logic.

Full human-facing docs (quick start, manifest field rules, install-link
shapes, legacy `extApi` support) are in [README.md](README.md) — read that
first for anything not covered below.

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
| `style.css` | Optional styling; mirrors the app's dark theme tokens. |
| `scripts/validate-manifest.mjs` | Local manifest check, same rules the app enforces on install. Not shipped (not in `files`). |

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
- Optional `panels` (dock into the Module Inspector slot, DraconDex 4.3.0+),
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
- If you add a docked `panels` entry: it is **reloaded whenever DraconDex
  re-renders its pane** (e.g. editing a tag is enough). Nothing may live only
  in a JS variable — persist state to your table at the moment it exists, and
  rebuild the panel's view from the table on every load.

## Commands

```bash
node scripts/validate-manifest.mjs   # same rules the app enforces on install
node --check app.js                  # add every shipped .js file here as you add them
```

CI (`.github/workflows/validate.yml`) runs both on every push/PR. No
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
4. Keep `scripts/validate-manifest.mjs` and the CI workflow, and extend the
   `node --check` list as you add scripts.
