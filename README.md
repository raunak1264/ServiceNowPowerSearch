# ServiceNow Power Search

> A keyboard-first query builder for ServiceNow — dynamic field lookup, multi-condition search, dot-walk related fields, and reference field resolver, straight from your browser toolbar.

![Version](https://img.shields.io/badge/version-2.1.0-4a7cff?style=flat-square)
![Manifest](https://img.shields.io/badge/manifest-v3-0fd4a2?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-f5a428?style=flat-square)
![Chrome](https://img.shields.io/badge/Chrome-Extension-4285F4?style=flat-square&logo=googlechrome&logoColor=white)

**[🌐 Homepage](https://raunak1264.github.io/ServiceNowPowerSearch)** &nbsp;·&nbsp; **[🐛 Report a Bug](https://github.com/raunak1264/ServiceNowPowerSearch/issues)** &nbsp;·&nbsp; **[📧 Contact](mailto:rawnakkapoor@gmail.com)**

---

## Overview

ServiceNow Power Search is a Manifest V3 Chrome extension that replaces manual URL crafting and slow list-view filtering. It fetches real fields from your live `sys_dictionary`, builds encoded queries with a guided UI, and opens results in one keystroke — all without ever asking you to log in again.

```
incident → short_descriptionLIKEnetwork outage^state=1^priority<=2
```

Built for ServiceNow developers, admins, and power users who live in the platform.

---

## Installation

### Chrome Web Store *(recommended)*
1. Visit the **ServiceNow Power Search** listing on the [Chrome Web Store](https://chrome.google.com/webstore)
2. Click **Add to Chrome**
3. Confirm the permissions prompt — done

### Developer / Unpacked Load
1. Download or clone this repository
2. Open **Chrome** → `chrome://extensions`
3. Enable **Developer mode** (top-right toggle)
4. Click **Load unpacked** and select the project folder

---

## Usage

### Basic workflow

1. **Open a ServiceNow tab** and log in — the extension reuses your existing browser session (no extra login required)
2. Click the **Power Search** icon in the toolbar to open the popup
3. The active instance is auto-detected from the current tab — green dot means connected. Opening the popup on a non-ServiceNow tab shows an error screen; switch to an SN tab to use the extension
4. **Select a table** from the dropdown (or choose `── custom ──` and type any table name). You can press a letter key while the dropdown is open to jump to the first matching table name
5. Click the **Field** input and start typing — fields load live from `sys_dictionary`
6. Use `↑ ↓` to navigate, `Enter` or `Tab` to select a field
7. For **reference fields** (shown with a **REF** badge), click `→` to drill into related fields and build dot-walk queries like `assigned_to.department.name`
8. Pick an **operator** (default: `starts with`) and enter a **value**
9. Hit **Search** (or press `Enter`) — results open in the same tab

### Multi-condition queries

Instead of stopping at one condition, click **＋ Add** (or press `Ctrl+Enter` on the value input) to commit the current field/op/value as a chip. Repeat for as many conditions as needed. All chips are joined with `^` in the final query.

```
state=-1  ^  priority<=2  ^  assigned_to=John Smith
```

To remove a condition, click `✕` on its chip (click twice to confirm — no dialog boxes).

### Reference fields

When a `reference`-type field is selected (shown with a **REF** badge) two workflows are available:

**Dot-walk (related fields)** — click the `→` button on any REF field in the dropdown to drill into the referenced table's fields. A breadcrumb header shows the current path and a `← Back` button navigates up. You can go multiple levels deep (e.g. `assigned_to → department → cost_center`). Selecting a field at any level builds the full dot-walk query automatically.

**Value lookup** — type a display name (e.g. "John") and a live typeahead searches the reference table. Click a result to auto-fill the correct `sys_id`. If you type a plain text value without picking from the typeahead, the query dot-walks to the display name field automatically (e.g. `assigned_to.nameCONTAINSjohn`).

### Advanced query

The **Advanced** textarea appends a raw encoded query string joined with `^`. Use it for anything the UI doesn't cover directly:

```
active=true^state=1^assigned_to=javascript:gs.getUserID()
```

### Templates

| Action | How |
|--------|-----|
| Save a search | Click **＋ Save**, give it a name |
| Load a recent search | Click any chip in the **Recent** row |
| Load a saved template | Click any chip in the **Saved** row |
| Share templates | Click **Export** → copies JSON to clipboard |
| Import from a teammate | Click **Import** → paste their JSON |

The last 8 searches are saved automatically as **Recent** chips. Named templates persist across browser restarts.

---

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `↑` `↓` | Navigate field dropdown |
| `Enter` | Select focused field / run search |
| `Tab` | Select focused field and advance |
| `Escape` | Close field dropdown |
| `Ctrl+Enter` (value field) | Add current condition as a chip |
| `Ctrl+Enter` (advanced field) | Run search |

---

## Architecture

```
ServiceNowPowerSearch/
├── manifest.json        MV3 manifest — permissions, service worker, popup entry
├── background.js        Service worker — message router, executeScript injection
├── search-core.js       Shared module — state, query building, caching, templates
├── popup.js             Popup wiring — all UI event handlers and feature logic
├── popup.html           UI markup — all panels and elements
├── styles.css           Precision Dark theme — CSS variable system
└── icons/
    ├── icon16.png
    ├── icon48.png
    └── icon128.png
```

### How field fetching works

All ServiceNow API calls run **inside the active SN tab** via `chrome.scripting.executeScript`. This means:

- The browser's existing `JSESSIONID` cookie is reused — no re-authentication
- All requests use `X-Requested-With: XMLHttpRequest` so ServiceNow returns a JSON 401 instead of an HTML login redirect, which prevents Chrome's Basic Auth dialog from firing
- The REST Table API (`/api/now/table/`) is deliberately avoided — `JSONv2` (`.do?JSONv2`) is used for every request

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

### Instance detection and tab awareness

On popup open the extension inspects only the **active tab in the current window**. If it is a `*.service-now.com` tab the instance name is shown with a green dot and the UI becomes fully interactive. If the active tab is not ServiceNow a full-screen error is shown — the body, footer, and controls are hidden until the user switches to an SN tab.

While the popup is open, `chrome.tabs.onActivated` and `chrome.tabs.onUpdated` listeners keep the badge live: switching to a different SN instance clears the field cache and reloads fields; switching to any non-SN tab immediately shows the error screen.

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

| Operator | Notes |
|----------|-------|
| `STARTSWITH` | **Default** — prefix match, hits name index directly |
| `LIKE` | Fuzzy / approximate match |
| `CONTAINS` | Substring match |
| `=` | Exact match |
| `!=` | Not equal |
| `ENDSWITH` | Suffix match |
| `ISEMPTY` / `ISNOTEMPTY` | No value required |
| `IN` | Comma-separated list |
| `GT` / `LT` / `GTE` / `LTE` | Numeric or date comparison |

Changing the table resets the operator back to `STARTSWITH`.

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
| ITSM | `incident`, `incident_task`, `change_request`, `change_task`, `problem`, `problem_task`, `sc_request`, `sc_req_item`, `sc_task`, `task`, `sysapproval_approver` |
| CMDB | `cmdb_ci`, `cmdb_ci_computer`, `cmdb_ci_server`, `cmdb_ci_appl`, `cmdb_ci_service`, `sn_agent_cmdb_ci_agent` |
| Users & Groups | `sys_user`, `sys_user_group`, `sys_user_grmember`, `sys_user_has_role` |
| System / Admin | `sys_dictionary`, `sys_db_object`, `sys_properties`, `sys_script`, `sys_script_include`, `sys_update_set`, `sys_email` |
| Knowledge | `kb_knowledge` |
| Story | `rm_story` |
| Custom | Any table — choose `── custom ──` and type the table name |

---

## Privacy

SN Power Search collects **zero personal data** and communicates with **zero external servers**.

- **No credentials stored** — the extension uses your browser's existing session cookie passively via `credentials: 'same-origin'`. Your password and session token are never read or stored.
- **Local storage only** — saved templates, recent searches, field cache, and UI state are stored in Chrome's local extension storage. Nothing is synced to any cloud.
- **No external network calls** — every API call goes directly to your own ServiceNow instance. No third-party servers, no analytics, no telemetry.

---

## Changelog

### v2.1.0

#### New features
- **Dot-walk / related fields** — click `→` on any REF field in the field dropdown to drill into the referenced table's fields. A sticky breadcrumb shows the current path with a `← Back` button. Multi-level traversal supported (e.g. `assigned_to → department → cost_center`). Selecting a field at any depth produces the full dot-walk name automatically
- **Reference field text search** — typing a plain-text value on a reference field now generates a valid dot-walk query (`field.nameCONTAINS<value>`) instead of an invalid direct match. Picking a record from the typeahead still uses the `sys_id` path
- **Tab-aware instance badge** — the instance badge updates in real time as the user switches tabs. Switching to a different SN instance reloads fields; switching to any non-SN tab shows the error screen immediately
- **ServiceNow-only mode** — opening the popup on a non-SN tab shows a full-screen error with the last known instance name. The query builder is hidden until the user switches to an SN tab
- **Letter-key navigation in table dropdown** — press any letter while the dropdown is open to jump to the first table starting with that letter; press again to cycle through matches
- **`starts with` as default operator** — moved to the top of the operator list and selected by default
- **Table change resets state** — switching tables clears all conditions and resets the operator to `starts with`
- **`rm_story`** added to the built-in table list under a new Story group

#### Bug fixes
- **Import filter doubling** — `readCurrentPageQuery` now reads `sysparm_query` from the real URL query string via `URL.searchParams`, preventing the `nameSTARTSWITHa?sysparm_query=nameSTARTSWITHa` duplication caused by reading the encoded path fragment
- **REF lookup on custom tables** — removed the hard `active=true^` prefix that returned zero results for tables without an `active` field; now retries without the filter when the first pass returns nothing
- **Dot-walk "No fields matching" error** — field input is now cleared on drill-down and `← Back` so the new field list is always shown unfiltered
- **Instance detection scope** — `detectInstance` now queries only `{ active: true, currentWindow: true }` instead of scanning all SN tabs across every browser window, preventing a background SN tab from being picked up when the user is on an unrelated tab

#### Removed
- Query Preview section
- Operator hint text below the operator dropdown

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
- **Fix:** `clearAll` converted from hoisted function declaration to `const`
- **Fix:** Magic number `12` extracted to `MAX_INHERITANCE_DEPTH` constant
- **Style:** Refined dark palette, section-based layout, gradient search button, gradient logo icon
- **Style:** `optgroup` labels added to table dropdown for grouped navigation

### v1.4.1
- Minor bug fixes and version bump

### v1.4.0
- Initial release — keyboard-first query builder, field autocomplete, templates, encoded query preview

---

## Contributing

1. Fork and clone the repository
2. Load unpacked in Chrome (`chrome://extensions` → Developer mode → Load unpacked)
3. Edit source files — the extension reloads on popup close/open after saving
4. Open a pull request with a clear description of the change

---

## Support

| Channel | Link |
|---------|------|
| 🌐 Homepage | [raunak1264.github.io/ServiceNowPowerSearch](https://raunak1264.github.io/ServiceNowPowerSearch) |
| 📋 Support page | [raunak1264.github.io/ServiceNowPowerSearch/support.html](https://raunak1264.github.io/ServiceNowPowerSearch/support.html) |
| 🐛 Bug reports | [GitHub Issues](https://github.com/raunak1264/ServiceNowPowerSearch/issues) |
| 📧 Email | [rawnakkapoor@gmail.com](mailto:rawnakkapoor@gmail.com) |

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

*Not affiliated with ServiceNow, Inc. ServiceNow is a trademark of ServiceNow, Inc.*
