# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**AgroCaptura Pro** is a single-file, offline-first agricultural field management PWA for tomato/vegetable farms. The entire application lives in one file: `index.html` (~384 lines of dense, minified-style HTML/CSS/JS). There is no build system, no package manager, no dependencies, and no server. Open `index.html` directly in a browser.

## Running the App

```bash
# Any of these work:
open index.html                          # macOS
xdg-open index.html                      # Linux
python3 -m http.server 8080              # Serve locally (avoids some browser restrictions)
```

There are no tests, no linter config, and no CI pipeline.

## Architecture: Single-File App

Everything — HTML structure, CSS, and all JavaScript — is in `index.html`. The JS is written in ES5-style (no arrow functions, no modules, no `const`/`let`), for maximum mobile browser compatibility.

### Data Layer

All state is stored in `localStorage` under the key `"agro9"` (variable `K`). The in-memory cache is `_DB` (global). Two functions manage all data access:

- **`G()`** — Get: reads from `_DB` cache, falls back to `localStorage.getItem(K)`, falls back to `I()` (seed data)
- **`P(d)`** — Persist: writes to both `_DB` and `localStorage.setItem(K, JSON.stringify(d))`

The data object shape:
```
{ ce, plagas, riego, cosecha, insumos, consumos, personal, clima,
  maquinaria, finanzas, users, config }
```

Each key holds an array of records. Records use numeric `id` fields assigned via `Math.max(...ids) + 1` or `Date.now()`.

`config` is a special object (not an array) holding user-editable dropdown lists: `sectores`, `equipos`, `plagas`, `cultivos`, `fertilizantes`, `actividades`, `fenologias`. Populated with defaults by `initCfg(d)`.

### Module System

Modules are defined in the `M` array at the top of the script. Each has `{i: moduleId, c: emoji, n: displayName}`.

Navigation flow:
- `RG()` renders the home grid of module cards (filtered by user role)
- `OM(id)` opens a module: hides grid, shows `#mc` div, calls the module's render function
- `goHome2()` returns to the home grid

Module render functions follow the pattern `X<XX>()` — e.g., `XCE()`, `XPL()`, `XRI()`. Each function injects HTML into `document.getElementById("mc")` and attaches event listeners inline. List display functions are `L<XX>()`.

**To add a new module:**
1. Add entry to `M` array with a unique `id`
2. Add `id` to the `mods` arrays in `ROLES` for appropriate roles
3. Add `id: XNN` entry in the `fn` object inside `OM()`
4. Write `XNN()` (render form + list) and `LNN()` (render list HTML) functions
5. Add `d[id] = []` in `I()` (seed data) and initialize in `P()`

### Authentication & RBAC

- `CU` (Current User) is the global holding the logged-in user object `{id, user, pass, role, name, active}`
- `doLogin()` searches `d.users` array, stores session in `localStorage` under `K+"_session"`, sets `CU`
- `doLogout()` clears `CU` and removes the session key

Four roles are defined in the `ROLES` object:

| Role | `canDel` | Module access |
|------|----------|---------------|
| `admin` | ✓ | All modules |
| `supervisor` | ✓ | All except `finanzas`, `usuarios` |
| `tecnico` | ✗ | `ce`, `plagas`, `riego`, `cosecha`, `clima`, `reportes` |
| `capturista` | ✗ | `ce`, `plagas`, `riego`, `cosecha`, `clima` |

Guard access with:
- `hasMod(id)` — checks `ROLES[CU.role].mods.indexOf(id) >= 0`
- `ROLES[CU.role].canDel` — controls whether delete buttons appear

Default users (stored in `d.users`, passwords in plaintext):
- `admin` / `admin123`
- `supervisor` / `super123`
- `tecnico` / `tec123`
- `campo` / `campo123`

### Google Sheets Sync (`XSY`)

The sync module posts all data as JSON to a user-deployed Google Apps Script URL. The Apps Script code (`GS_CODE` string in the JS) is shown in the UI so the user can deploy it themselves. The sync URL is stored in `localStorage` under `K+'_gsurl'`. Photo base64 data is uploaded to Google Drive in a folder called `AgroCaptura_Fotos`.

## Key Conventions

### Shorthand functions used everywhere

| Function | Purpose |
|----------|---------|
| `V(id)` | `document.getElementById(id).value` |
| `H()` | Today's date as `YYYY-MM-DD` |
| `D(str)` | Format date `YYYY-MM-DD` → `DD/MM/YYYY` |
| `E(str)` | HTML-escape a string (XSS prevention — always use for user data in innerHTML) |
| `SO()` | Returns `<option>` tags for the `sectores` config list |
| `OPTS(key, addEmpty)` | Returns `<option>` tags from `d.config[key]` array |
| `CL(idsArray)` | Clears `.value` on a list of element IDs |
| `TT(msg, type)` | Shows toast notification (`"ok"` or `"error"`) |
| `CF(title, msg, fn)` | Shows confirmation modal, calls `fn` on OK |
| `FF(arr, fn)` | Find-first: equivalent to `arr.find(fn)` |

### CRUD pattern

- **Create**: `d[mod].unshift(record)`, then `P(d)`, then re-render list
- **Read**: `d[mod]` array, rendered via `innerHTML` with `.map()` + `E()` for escaping
- **Update**: Not supported — users delete and re-enter records
- **Delete**: `d[mod] = d[mod].filter(r => r.id !== id)`, then `P(d)`, then re-render

### CSS class conventions

Classes are single short names (`.mc2`, `.mg`, `.bs`, `.fl`, etc.) — all defined in the `<style>` block. Status badge colors: `.co` (green/OK), `.cba` (blue/low), `.cal` (yellow/alert), `.ccr` (red/critical). Severity spans use `.s0`–`.s4`. Role badges: `.r-admin`, `.r-supervisor`, `.r-tecnico`, `.r-capturista`.

### Photo handling

Photos are captured via `<input type="file" accept="image/*" capture="environment">`. Images are compressed to JPEG via canvas (max 800×800, quality 0.6) and stored as base64 data URIs in the record's `foto` field. During Google Sheets sync, photos are uploaded to Google Drive and replaced with shareable URLs.

### ID generation

New record IDs: `(d[mod].length ? Math.max(...d[mod].map(r=>r.id)) + 1 : 1)`. Some modules use `Date.now()`.

### Sectors

The farm sectors are `["Las Rosas","Las Pitas","Las Nuevas","Patron B","El Pozo","El Esquino"]` — configurable via the `config` module but defaulted in `initCfg()`.

## CE (Soil Electrical Conductivity) Module Specifics

This is the most complex module. It records CE readings for 3 irrigation points (`i`) and 3 drain points (`d`), each with sub-points `a`, `b`, `c`. The `CC()` function analyzes the readings against optimal ranges per phenological stage and returns color-coded alert levels. The analysis uses hardcoded optimal CE ranges per `fn` (fenología) stage.
