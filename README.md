# ServiceNow Power Search

> A keyboard-first query builder for ServiceNow — dynamic field lookup, multi-condition search, and reference field resolver, straight from your browser toolbar.

![Version](https://img.shields.io/badge/version-1.5.0-4a7cff?style=flat-square)
![Manifest](https://img.shields.io/badge/manifest-v3-0fd4a2?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-f5a428?style=flat-square)
![Edge](https://img.shields.io/badge/Microsoft%20Edge-Add--on-0078d4?style=flat-square)

---

## Overview

ServiceNow Power Search is a Manifest V3 browser extension that replaces manual URL crafting and slow list-view filtering. It fetches real fields from your live `sys_dictionary`, builds encoded queries with a guided UI, and opens results in one keystroke — all without ever asking you to log in again.

```
incident → short_descriptionLIKEnetwork outage^state=1^priority<=2
```

Built for ServiceNow developers, admins, and power users who live in the platform.

---

## Installation

### Microsoft Edge (Add-ons Store)
1. Visit the [Microsoft Edge Add-ons](https://microsoftedge.microsoft.com/addons/) store
2. Search for **"ServiceNow Power Search"**
3. Click **Get** → **Add extension**

### Developer / Unpacked Load
1. Download or clone this repository
2. Open **Edge** → `edge://extensions` (or **Chrome** → `chrome://extensions`)
3. Enable **Developer mode** (top-right toggle)
4. Click **Load unpacked** and select the project folder

---

## Usage

### Basic workflow

1. **Open a ServiceNow tab** and log in — the extension reuses your existing browser session (no extra login required)
2. Click the **SN** icon in the toolbar to open the popup
3. The active instance is auto-detected — green dot means connected
4. **Select a table** from the dropdown, or choose `── custom ──` and type any table name
5. Click the **Field** input and start typing — fields load live from `sys_dictionary`
6. Use `↑ ↓` to navigate, `Enter` or `Tab` to select a field
7. Pick an **operator** and enter a **value**
8. Hit **Search** (or press `Enter`) — results open in the same tab

### Multi-condition queries

Instead of stopping at one condition, click **＋ Add** (or press `Ctrl+Enter` on the value input) to commit the current field/op/value as a chip. Repeat for as many conditions as needed. All chips are joined with `^` in the final query.

```
state=-1  ^  priority<=2  ^  assigned_to=John Smith
```

To remove a condition, click `✕` on its chip (click twice to confirm — no dialog boxes).

### Reference field resolver

When a `reference`-type field is selected (shown with a **REF** badge), a resolver row appears below. Type a display name (e.g. "John") and the extension looks up matching records in the reference table. Click any result to auto-fill the value field with the correct `sys_id`.

### Advanced query

The **Advanced** textarea appends a raw encoded query string joined with `^`. Use it for anything the UI doesn't cover directly:

```
active=true^state=1^assigned_to=javascript:gs.getUserID()
```

### Templates

- **＋ Save** — name and save the current search as a reusable template
- **Recent** chips — the last 8 searches are saved automatically
- **Saved** chips — persistent named templates that survive browser restarts
- **Export** — copies all saved templates as JSON to your clipboard
- **Import** — paste JSON from a teammate to merge their templates into yours

---

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `↑` `↓` | Navigate field dropdown |
| `Enter` | Select focused field / run search |
| `Tab` | Select focused field and advance |
| `Escape` | Close field dropdown |
| `Ctrl+Enter` (in value field) | Add current condition as a chip |
| `Ctrl+Enter` (in advanced field) | Run search |

---

## Architecture

```
servicenow-power-search/
├── manifest.json        MV3 manifest — permissions, service worker, popup entry
├── background.js        Service worker — message router, executeScript injection
├── search-core.js       Shared module — state, query building, caching, templates
├── popup.js             Popup wiring — all UI event handlers and feature logic
├── popup.html           UI markup — all panels and elements
├── styles.css           Precision Dark theme — Outfit font, CSS variable system
└── icons/
    ├── icon16.png
    ├── icon48.png
    └── icon128.png
```

### How field fetching works

All ServiceNow API calls run **inside the active SN tab** via `chrome.scripting.executeScript`. This means:

- The browser's existing `JSESSIONID` cookie is reused — no re-authentication
- All requests use `X-Requested-With: XMLHttpRequest` so ServiceNow returns a JSON 401 instead of an HTML login redirect, which prevents Chrome's Basic Auth dialog from firing
- The REST Table API (`/api/now/table/`) is deliberately avoided for the same reason — `JSONv2` (`.do?JSONv2`) is used for every request

### Super-class chain walking

For tables like `change_request` that inherit fields from `task` → `sys_metadata`, the extension walks the `super_class` chain in `sys_db_object` up to 12 levels deep. It then fetches all `sys_dictionary` entries for the full hierarchy in a single query and dedupes them so child-table overrides always win.

### Message types

| Message | Direction | Purpose |
|---------|-----------|---------|
| `GET_ACTIVE_TAB` | popup → background | Detect current tab and instance URL |
| `FETCH_FIELDS` | popup → background | Inject field fetch into SN tab |
| `FETCH_REF_LOOKUP` | popup → background | Inject display-name lookup for reference fields |
| `OPEN_URL` | popup → background | Navigate the SN tab to the result list |

---

## Feature reference

### Instance detection

On popup open, the extension checks the active tab URL for a `*.service-now.com` hostname. If found, the instance name is shown with a green pulsing dot. If not, the last known instance is shown as cached and a warning banner appears.

If the active tab is a ServiceNow list view (`*_list.do`), the table is auto-detected and pre-selected in the dropdown.

### Field cache

Fetched fields are cached in `chrome.storage.local` with a **1-hour TTL** per `instance + table` combination. Click the **↻** button next to the Field label to force a refresh at any time.

Cache format: `fields_v2_{instanceUrl}_{tableName}` → `{ fields: [...], ts: timestamp }`

### State persistence

The last used table, field, operator, value, and advanced query are saved to `chrome.storage.local` and restored the next time the popup opens. Conditions (multi-condition chips) are session-only and are not persisted.

### Query URL format

```
https://<instance>.service-now.com/now/nav/ui/classic/params/target/<table>_list.do?sysparm_query=<encoded_query>
```

The **⧉** copy button in the query preview copies this full URL to your clipboard.

---

## Operators

| Symbol | Operator | Notes |
|--------|----------|-------|
| `~` | `LIKE` | Case-insensitive contains (default) |
| `=` | equals | Exact match |
| `≠` | not equals | |
| `^=` | `STARTSWITH` | |
| `$=` | `ENDSWITH` | |
| `⊂` | `CONTAINS` | |
| `∅` | `ISEMPTY` | No value required |
| `¬∅` | `ISNOTEMPTY` | No value required |
| `∈` | `IN` | Comma-separated list |
| `>` `<` `≥` `≤` | GT / LT / GTE / LTE | Numeric / date comparison |

---

## Permissions

| Permission | Why it's needed |
|------------|----------------|
| `activeTab` | Read the current tab URL to detect the ServiceNow instance |
| `scripting` | Inject fetch calls into the SN tab to load fields and resolve references |
| `storage` | Cache fields (1-hour TTL) and persist UI state between sessions |
| `tabs` | Navigate the active SN tab to the search results list |
| `https://*.service-now.com/*` | Required host permission for `executeScript` injection |

No data is sent to any external server. All network requests go directly to your ServiceNow instance using your existing browser session.

---

## Supported tables (built-in)

| Group | Tables |
|-------|--------|
| ITSM | `incident`, `change_request`, `problem`, `sc_request`, `sc_req_item`, `sc_task`, `task` |
| CMDB | `cmdb_ci`, `cmdb_ci_computer`, `cmdb_ci_server`, `cmdb_ci_appl`, `cmdb_ci_service`, `sn_agent_cmdb_ci_agent` |
| Users | `sys_user`, `sys_user_group` |
| Knowledge | `kb_knowledge` |
| Custom | Any table — choose `── custom ──` and type the table name |

---

## Changelog

### v1.5.0
- **New:** Multi-condition builder — commit conditions as chips with `＋ Add` or `Ctrl+Enter`
- **New:** Reference field resolver — type a display name, pick from live results, auto-fill `sys_id`
- **New:** Export / Import templates — share saved searches as JSON with teammates
- **New:** Cache refresh button (↻) — force field reload without clearing storage
- **New:** Auto-detect table from active SN tab URL on popup open
- **New:** `internal_type` and `reference` table fetched from `sys_dictionary` for REF detection
- **New:** `FETCH_REF_LOOKUP` message type in background.js for reference lookups
- **Fix:** Race condition — stale field fetch responses are now discarded via `fetchRequestId` stamp
- **Fix:** `window.confirm()` dialogs replaced with double-click-to-confirm pattern
- **Fix:** Field cache now uses TTL (`{ fields, ts }`) instead of never expiring; backward compatible
- **Fix:** `clearAll` uses `selectedIndex = 0` instead of fragile `value = ''`
- **Fix:** Error boundary added around `restoreState` — malformed stored state no longer crashes popup
- **Fix:** `saveState` on value/advanced inputs is now debounced (350ms) instead of per-keystroke
- **Fix:** `clearAll` converted from hoisted function declaration to `const` — consistent with rest of file
- **Fix:** Magic number `12` extracted to `MAX_INHERITANCE_DEPTH` constant
- **Style:** New `Outfit` UI font replacing Inter; JetBrains Mono retained for code elements
- **Style:** Gradient logo icon, refined dark palette, section-based layout, gradient search button
- **Style:** `optgroup` labels added to table dropdown for grouped navigation

### v1.4.1
- Minor bug fixes and version bump

### v1.4.0
- Initial release — keyboard-first query builder, field autocomplete, templates, encoded query preview

---

## Contributing

1. Fork and clone the repository
2. Load unpacked in Edge/Chrome (`edge://extensions` → Developer mode → Load unpacked)
3. Edit source files — the extension hot-reloads on popup close/open after saving
4. Open a pull request with a clear description of the change

---

## License

MIT — see [LICENSE](LICENSE) for details.
