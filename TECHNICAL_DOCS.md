# ServiceNow Power Search — Technical Documentation

**Version:** 2.1.0 · **Manifest:** V3 · **Author:** Raunak Kapoor

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture & Workflow](#2-architecture--workflow)
3. [Codebase Explanation](#3-codebase-explanation)
4. [UI/UX Functionality](#4-uiux-functionality)
5. [Technical Breakdown](#5-technical-breakdown)
6. [Development Notes](#6-development-notes)
7. [Function-Level Documentation](#7-function-level-documentation)
8. [Quick Summary](#8-quick-summary)

---

## 1. Project Overview

### 1.1 Purpose

ServiceNow Power Search is a Chrome extension that provides a keyboard-first query builder for the ServiceNow platform. It introspects the live instance's data dictionary at runtime, constructs encoded filter queries, and navigates the ServiceNow list view — all without the user ever needing to write a URL or encode query strings by hand.

### 1.2 Problem It Solves

ServiceNow's native filter builder requires multiple clicks, hides inherited fields from parent tables, and does not support dot-walking reference fields from the filter UI. Power users and developers frequently need to craft complex encoded query strings (e.g. `state=1^priority<=2^assigned_to.nameCONTAINSjohn`) directly in the address bar, which is error-prone and slow.

This extension solves that by:

- Fetching **real field metadata** (including field type and reference table) from `sys_dictionary` via the live session
- Walking the full **inheritance chain** of any table so inherited fields (e.g. `short_description` on `incident` via `task`) are available
- Providing **dot-walk navigation** for reference-type fields across multiple hops
- Resolving reference field values by **display name → sys_id** lookup in real time
- Allowing **multi-condition queries** (AND/OR) with chip-based visual feedback
- **Saving and restoring** searches as named templates or recent history

### 1.3 Key Features

| Feature | Description |
|---|---|
| Live field autocomplete | Fields fetched from `sys_dictionary` with 1-hour local cache |
| Inheritance chain walking | Follows `super_class` up to 12 levels; child fields always win dedup |
| Dot-walk reference drill | Click `→` on any REF field to explore its table's fields recursively |
| Reference value resolver | Type a display name → live typeahead → auto-fills correct `sys_id` |
| Multi-condition builder | Chip-based AND/OR conditions, individually removable |
| Import filters from page | Reads the active SN list view's applied query and imports it into the builder |
| Named templates | Save, name, export, and import searches as JSON |
| Recent history | Last 10 searches auto-saved and restored |
| Tab awareness | Real-time instance detection; badge updates as tabs switch |
| Zero re-authentication | Reuses the browser's existing `JSESSIONID` session cookie |
| No external servers | All network traffic is between the extension and the user's SN instance |

---

## 2. Architecture & Workflow

### 2.1 Extension Layer Map

```
┌─────────────────────────────────────────────────────────────────┐
│                        Chrome Browser                           │
│                                                                 │
│  ┌──────────────────────────┐    ┌──────────────────────────┐  │
│  │      Popup Context        │    │    ServiceNow Tab         │  │
│  │  (popup.html + popup.js  │    │   (*.service-now.com)    │  │
│  │   + search-core.js)      │    │                          │  │
│  │                          │    │  Injected functions:      │  │
│  │  - UI rendering          │◄──►│  - fetchFieldsFromSN()   │  │
│  │  - State management      │    │  - lookupRefRecords()     │  │
│  │  - Query building        │    │  - readCurrentPageQuery() │  │
│  │  - Template management   │    │                          │  │
│  └────────────┬─────────────┘    └──────────────────────────┘  │
│               │ chrome.runtime.sendMessage()                    │
│               ▼                                                 │
│  ┌──────────────────────────┐                                   │
│  │   Background Service     │                                   │
│  │   Worker (background.js) │                                   │
│  │                          │                                   │
│  │  - Message router        │                                   │
│  │  - executeScript() calls │                                   │
│  │  - Tab open/navigate     │                                   │
│  └──────────────────────────┘                                   │
│                                                                 │
│  ┌──────────────────────────┐                                   │
│  │   chrome.storage.local   │                                   │
│  │                          │                                   │
│  │  - Field cache (1hr TTL) │                                   │
│  │  - Recent searches       │                                   │
│  │  - Saved templates       │                                   │
│  │  - Last UI state         │                                   │
│  │  - Last known instance   │                                   │
│  └──────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 End-to-End Execution Flow

The following trace covers the primary user flow from popup open to search execution.

```
[User clicks toolbar icon]
        │
        ▼
popup.html loads
        │
        ├── search-core.js parsed (IIFE, exports window.SearchCore)
        └── popup.js parsed (imports from window.SearchCore)
                │
                ▼
           init() called
                │
                ├─ loadTemplates(state)
                │      └── chrome.storage.local.get(['recentSearches','savedTemplates'])
                │
                ├─ renderTemplates()
                │
                ├─ restoreState(state, els)
                │      └── chrome.storage.local.get([lastTable, lastField, lastOp, ...])
                │              └── Repopulates UI inputs + conditions chips
                │
                └─ detectInstance(state, els, callbacks)
                        │
                        └── chrome.tabs.query({ active:true, currentWindow:true })
                                │
                        ┌───── Is current tab *.service-now.com? ─────┐
                        │ YES                                         │ NO
                        ▼                                             ▼
                commitTab(tab)                               useCached() → showNoSn()
                  - state.snTab = tab
                  - state.instanceUrl = "https://xxx.service-now.com"
                  - showMain()
                  - auto-detect table from tab URL
                        │
                        ▼
              loadFieldsForTable(tableName)
                        │
                        ├─ chrome.storage.local.get(cacheKey)
                        │        │
                        │  ┌─────┴──── Cache hit and not expired? ────┐
                        │  │ YES                                       │ NO
                        │  ▼                                           ▼
                        │ state.fields = cached             sendMessage(FETCH_FIELDS)
                        │ renderDropdown()                          │
                        │                                           ▼
                        │                                 background.js receives message
                        │                                           │
                        │                                 chrome.scripting.executeScript(
                        │                                   { func: fetchFieldsFromSN,
                        │                                     args: [tableName] }
                        │                                 )
                        │                                           │
                        │                     ┌─────────────────────┘
                        │                     │ Runs inside SN tab
                        │                     ▼
                        │           fetchFieldsFromSN(tableName)
                        │                     │
                        │            Walk super_class chain
                        │            (up to 12 levels, 2 fetches/level)
                        │                     │
                        │            Bulk fetch sys_dictionary
                        │            (sysparm_limit=3000)
                        │                     │
                        │            Deduplicate, sort by label
                        │                     │
                        │            Return { fields, tables, count }
                        │                     │
                        │             ◄────────┘
                        │ chrome.storage.local.set(cacheKey, {fields, ts})
                        │ state.fields = fields
                        │ renderDropdown()
                        │
                        ▼
              startTabWatch()
              - Attach chrome.tabs.onActivated listener
              - Attach chrome.tabs.onUpdated listener
                        │
                        ▼
              setTimeout(() => els.fieldInput.focus(), 120)
              [Popup is fully initialized]
```

### 2.3 Message Passing Flow

The popup context and the background service worker communicate exclusively via `chrome.runtime.sendMessage`. The background worker then uses `chrome.scripting.executeScript` to inject functions into the ServiceNow tab. This indirection is required because the popup cannot directly call `chrome.scripting.executeScript` against another tab.

```
Popup                   Background SW           SN Tab (injected)
  │                          │                       │
  │──sendMessage(FETCH_FIELDS)──►│                   │
  │                          │                       │
  │                          │──executeScript()──►   │
  │                          │    fetchFieldsFromSN  │
  │                          │                       │
  │                          │           fetch(/sys_dictionary_list.do?JSONv2)
  │                          │                       │
  │                          │◄──result──────────────│
  │                          │                       │
  │◄──sendResponse(fields)───│                       │
  │                          │                       │
```

### 2.4 State Machine — Instance Detection

```
                    ┌─────────────┐
           Popup    │  DETECTING  │
           opens    └──────┬──────┘
                           │
              chrome.tabs.query({ active, currentWindow })
                           │
              ┌────────────┴────────────┐
              │                         │
        SN tab found               No SN tab found
              │                         │
              ▼                         ▼
      ┌──────────────┐         ┌──────────────────┐
      │    ONLINE    │         │   CHECK STORAGE   │
      │  (green dot) │         └────────┬─────────┘
      └──────┬───────┘                  │
             │                 lastInstance exists?
             │                 ┌────────┴────────┐
             │                 │ YES             │ NO
             │                 ▼                 ▼
             │          ┌────────────┐    ┌──────────┐
             │          │   CACHED   │    │ OFFLINE  │
             │          │(grey badge)│    │(no badge)│
             │          └────────────┘    └──────────┘
             │
      Tab switch events
      (onActivated / onUpdated)
             │
    ┌────────┴────────┐
    │                 │
  New SN        Non-SN tab
  tab/instance        │
    │                 ▼
    │          ┌──────────┐
    │          │  OFFLINE │
    ▼          └──────────┘
  ┌──────────────┐
  │   RELOAD     │ (clear fields, re-fetch for new instance)
  └──────────────┘
```

---

## 3. Codebase Explanation

### 3.1 File Overview

```
ServiceNow Power Search/
├── manifest.json       Extension manifest (MV3)
├── background.js       Service worker: message router + SN API injection
├── search-core.js      Shared module: state, query engine, caching, templates
├── popup.js            Popup controller: all UI event wiring
├── popup.html          Popup markup: all panels, inputs, dropdowns
├── styles.css          CSS: dark theme using CSS custom properties
└── icons/
    ├── icon16.png
    ├── icon48.png
    └── icon128.png
```

### 3.2 `manifest.json`

The extension manifest conforms to **Manifest V3** (MV3), Chrome's current extension platform.

```json
{
  "manifest_version": 3,
  "name": "ServiceNow Power Search",
  "version": "2.1.0",
  "permissions": ["activeTab", "scripting", "storage", "tabs"],
  "host_permissions": ["https://*.service-now.com/*"],
  "action": { "default_popup": "popup.html" },
  "background": { "service_worker": "background.js" }
}
```

**Key design decisions:**

- `"background": { "service_worker": ... }` — MV3 replaces persistent background pages with event-driven service workers that are terminated when idle. This means the background script cannot maintain in-memory state between messages.
- `host_permissions` is declared separately from `permissions` in MV3. The `https://*.service-now.com/*` entry is required for `chrome.scripting.executeScript` to inject scripts into SN tabs.
- No `content_scripts` are declared — the extension does not inject persistent scripts into pages. All injection happens on-demand via `executeScript`.

### 3.3 `background.js`

**Role:** MV3 service worker. Receives messages from the popup and either handles them natively (tab operations) or injects functions into the SN tab via `chrome.scripting.executeScript`.

**Why injection instead of content scripts?**

Content scripts run in an isolated JavaScript context and cannot access the page's own JavaScript globals (e.g. ServiceNow's `g_list` object). Scripts injected via `executeScript` with `world: 'MAIN'` run in the page's own execution context, giving access to both the DOM and SN's global variables. For the field-fetching functions, the default isolated world is used (they only need `fetch` access to the same origin); `readCurrentPageQuery` explicitly uses `world: 'MAIN'` to read `g_list`.

**Why JSONv2 instead of REST Table API?**

The REST Table API (`/api/now/table/`) triggers Chrome's Basic Auth dialog when the request includes session cookies but no Bearer token. The JSONv2 endpoint (`.do?JSONv2`) accepts the `JSESSIONID` cookie alone and returns a JSON 401 on auth failure instead of an HTML redirect, which the extension handles gracefully without any browser UI interruption.

**Injected functions (defined in `background.js`, executed inside the SN tab):**

| Function | Injected via | World | Purpose |
|---|---|---|---|
| `fetchFieldsFromSN` | `FETCH_FIELDS` | ISOLATED | Walk inheritance chain, bulk-fetch dictionary fields |
| `lookupRefRecords` | `FETCH_REF_LOOKUP` | ISOLATED | Search reference table by display name |
| `readCurrentPageQuery` | `FETCH_CURRENT_QUERY` | MAIN | Read current list view's applied query from URL and `g_list` |

**Message types handled:**

```
GET_ACTIVE_TAB      → chrome.tabs.query({ active, currentWindow }) → sendResponse({ tab })
FETCH_FIELDS        → executeScript(fetchFieldsFromSN, [tableName]) → sendResponse({ fields, tables, count })
FETCH_REF_LOOKUP    → executeScript(lookupRefRecords, [refTable, query]) → sendResponse({ records })
FETCH_CURRENT_QUERY → executeScript(readCurrentPageQuery, world:'MAIN') → sendResponse({ table, query })
OPEN_URL            → chrome.tabs.create() or chrome.tabs.update() → sendResponse({ ok: true })
```

### 3.4 `search-core.js`

**Role:** Shared logic module. Wrapped in an IIFE to prevent variable collisions with `popup.js`. Exports a single `window.SearchCore` object consumed by `popup.js`.

**Why an IIFE?**

`popup.html` loads both `search-core.js` and `popup.js` as inline scripts in the same scope. Without the IIFE wrapper, `const` declarations at the top level of `search-core.js` would collide with identically-named destructured imports in `popup.js`.

**Responsibilities:**

| Category | What it owns |
|---|---|
| State | `createState()` factory — single source of truth for popup state |
| Query building | `buildQuery()`, `buildUrl()`, `appendClause()` |
| Query parsing | `parseQueryString()`, `parseClause()` — SN encoded query → conditions array |
| Field fetching | `fetchFields()`, `clearFieldCache()` — cache + message dispatch |
| Instance detection | `detectInstance()` — tab query + cached fallback |
| Template storage | `loadTemplates()`, `persistRecent()`, `persistSaved()`, `recordRecentSearch()` |
| State persistence | `saveState()`, `restoreState()` — chrome.storage.local serialization |
| Multi-condition | `addCondition()`, `removeCondition()` |
| Utilities | `esc()`, `debounce()`, `highlight()`, `chipLabel()`, `conditionLabel()` |

**Constants:**

```javascript
MAX_RECENT            = 10        // max recent search history entries
MAX_DROPDOWN          = 50        // max items shown in field dropdown at once
MAX_INHERITANCE_DEPTH = 12        // max super_class levels to walk
CACHE_TTL_MS          = 3600000   // 1 hour field cache TTL
```

### 3.5 `popup.js`

**Role:** Popup controller. Imports from `window.SearchCore`, wires all DOM event handlers, and orchestrates every user-facing interaction. All UI state changes flow through this file.

**Key responsibilities:**

- Initializing and orchestrating the `init()` sequence
- Rendering all dynamic UI components (field dropdown, conditions chips, templates)
- Wiring keyboard navigation across all inputs
- Managing the dot-walk breadcrumb stack
- Debouncing reference value lookups
- Implementing custom `<select>` replacement (`initCustomSelect`)
- Tab watching (`startTabWatch`, `onTabActivated`, `onTabUpdated`)
- Import filters from active page (`importFiltersFromPage`)

**Notable patterns:**

**Debouncing:** Three separate debounce instances are created:
- `debouncedSave` (350ms) — persists UI state on every value/field/operator change
- `debouncedRefLookup` (400ms) — throttles reference table lookups to avoid per-keystroke API calls
- Custom table input handler (500ms inline debounce) — delays field fetch until the user pauses typing

**Request ID race guard:**
```javascript
const requestId = ++state.fetchRequestId;
// ...later in callback:
if (requestId !== state.fetchRequestId) return; // discard stale response
```
This prevents a slow field fetch from an earlier table selection from overwriting the results of a faster fetch for the currently selected table.

**Custom select replacement (`initCustomSelect`):**
The native `<select>` element is visually replaced with a custom panel-based dropdown to support per-option hint text, letter-key navigation, and styling that matches the dark theme. The native `<select>` remains in the DOM as the source of truth; the custom panel reads from and writes to it.

### 3.6 `popup.html`

**Role:** Static markup for the extension popup. All interactive elements are referenced by ID in `popup.js`.

**Panel structure:**

```
popup.html
├── #instanceBadge          Instance name + connection dot
├── #noSnScreen             Full-screen "no SN tab" error state
└── #mainBody               Main query builder (hidden when offline)
    ├── #importFiltersBtn   Import current page filters
    ├── Table selector      Native <select> + custom panel overlay
    ├── #conditionsContainer Multi-condition chips display
    ├── Field autocomplete  #fieldInput + #fieldDropdown + #fieldRefreshBtn
    ├── Operator selector   Native <select> + custom panel overlay
    ├── Value input         #valueInput + #valueDropdown (ref lookup)
    ├── #joinToggle         AND / OR toggle for next condition
    ├── #addConditionBtn    Commit current condition
    ├── #searchBtn          Execute search
    ├── #clearAllBtn        Reset all fields
    └── History panels
        ├── #recentChips    Recent search history
        └── #savedChips     Named templates
            ├── Save prompt (#savePrompt)
            └── Import modal (#importModal)
```

### 3.7 `styles.css`

**Role:** Full visual theme for the popup. Uses CSS custom properties (variables) for the color palette, making the theme easily adjustable.

**Design system highlights:**

- Dark background with layered surface colors for depth (`--bg`, `--surface`, `--surface2`)
- Accent color `--accent: #4a7cff` used for interactive states, the search button gradient, and focus rings
- `.dropdown-item`, `.hist-item`, `.cond-chip` all share hover/focus state conventions
- Spinner animation for loading indicators
- `.keyboard-focus` and `.focused` classes drive keyboard navigation highlights

---

## 4. UI/UX Functionality

### 4.1 Popup Lifecycle

The popup is created fresh every time the user opens it (Chrome destroys and recreates the popup DOM on each open). `init()` is called unconditionally and restores all persisted state from `chrome.storage.local` so the popup feels continuous across opens.

### 4.2 Field Dropdown

The field dropdown is entirely custom-rendered HTML (not a native `<select>`). It renders up to `MAX_DROPDOWN = 50` items at a time, filtered client-side from the in-memory `state.fields` array.

**States the dropdown can be in:**

| State | What renders |
|---|---|
| Loading | Spinner + "Loading fields…" |
| Dot-walk active | Breadcrumb header + back button above field list |
| Filtered results | Up to 50 matching fields with highlight on matched substring |
| No results | "No fields matching" or "No fields — open a SN tab to load" |
| Reference field item | "REF" badge + `→` drill-down button on the right |

**Keyboard navigation:**
`ArrowDown` / `ArrowUp` move `state.dropdownKeyIndex`. `Enter` or `Tab` selects the focused item. `Escape` closes without selecting.

### 4.3 Reference Field Drill-Down (Dot-Walk)

The dot-walk feature uses a stack (`state.dotWalkStack`) to track the navigation path.

```
state.dotWalkStack = [
  { fieldName: 'assigned_to', fieldLabel: 'Assigned to', refTable: 'sys_user' },
  { fieldName: 'department',  fieldLabel: 'Department',  refTable: 'cmn_department' }
]
```

When the user drills into a field:
1. The current field is pushed onto `dotWalkStack`
2. `loadFieldsForDotWalk(refTable)` fetches fields for the target table (with 10-min cache)
3. The breadcrumb header updates to show the current path
4. The `← Back` button pops the stack and reloads the parent level's fields

When a field is selected at any depth:
- `getDotWalkPrefix()` joins the stack `fieldName`s with `.` → e.g. `assigned_to.department.`
- The selected field's name is appended → `assigned_to.department.name`
- This full dot-walk path is stored as `state.selectedField.name`

### 4.4 Reference Value Lookup

Triggered only when:
- The selected field's `type === 'reference'`
- The field is at the root level (not a dot-walked sub-field, since those are treated as text)
- The user has typed at least 2 characters in the value input
- 400ms have elapsed since the last keystroke

The lookup makes 2 to 4 requests:
1. `nameSTARTSWITH{query}` (index-backed, fast) — with `active=true^`
2. `nameCONTAINS{query}` (full scan, catches mid-string matches) — with `active=true^`
3. If both return 0 results: retry steps 1 and 2 without `active=true^` (for custom tables)

Results are merged, deduplicated by `sys_id`, and capped at 8 records. The value dropdown shows each record's `name` (display value) and `sys_id`.

### 4.5 Multi-Condition Builder

Each committed condition is stored in `state.conditions` as:

```javascript
{
  id:            Date.now(),     // unique ID for removal
  field_name:    'state',
  field_label:   'State',
  op:            '=',
  value:         '1',
  display_value: null,           // set when sys_id picked from ref lookup
  is_ref:        false,
  join:          'AND'           // 'AND' | 'OR'
}
```

`renderConditions()` generates the chip HTML from this array and attaches a two-click-to-remove pattern (first click arms the button with a "?" state; second click within 2 seconds confirms deletion).

The AND/OR toggle (`#joinToggle`) sets `state.currentJoin` which is applied to the **next** condition added, not retroactively to existing ones.

### 4.6 Template History

**Recent searches** are stored in `state.recentSearches` (max 10). A new entry is added every time `doSearch()` is called. Duplicate searches (same signature) are deduplicated and moved to the front.

**Saved templates** are stored in `state.savedTemplates` (no limit). Each template is the full `currentSearchState` snapshot plus a user-provided name.

Both collections are serialized to `chrome.storage.local` after every mutation.

**Export format:**
```json
{
  "savedTemplates": [
    {
      "name": "Open P1 Incidents",
      "table": "incident",
      "field_name": "priority",
      "op": "=",
      "value": "1",
      "conditions": [...]
    }
  ]
}
```

### 4.7 Import Filters from Active Page

`importFiltersFromPage()` fires `FETCH_CURRENT_QUERY` which injects `readCurrentPageQuery()` into the SN tab with `world: 'MAIN'`. That function:

1. Decodes the URL (SN nav router double-encodes the list URL inside the path)
2. Extracts the table name via regex (`/([a-zA-Z0-9_]+)_list\.do/`)
3. Extracts `sysparm_query` via `URL.searchParams` (preferred) or regex fallback
4. Falls back to `g_list.getFilter()` from SN's own client-side API if URL parsing yields nothing

The returned query string is then parsed by `parseQueryString()` into condition objects and merged into `state.conditions`.

---

## 5. Technical Breakdown

### 5.1 Manifest Permissions — Detailed Rationale

| Permission | API Used | Why Required |
|---|---|---|
| `activeTab` | `chrome.tabs.query({ active, currentWindow })` | Inspect current tab URL to detect SN instance without broad tab access |
| `scripting` | `chrome.scripting.executeScript()` | Inject `fetchFieldsFromSN`, `lookupRefRecords`, `readCurrentPageQuery` into the SN tab |
| `storage` | `chrome.storage.local.*` | Persist field cache, templates, recent history, and UI state across popup sessions |
| `tabs` | `chrome.tabs.create()`, `chrome.tabs.update()`, `onActivated`, `onUpdated` | Navigate to search results; listen for tab switches to update instance badge |
| `https://*.service-now.com/*` (host) | Required by `scripting` | `executeScript` cannot inject into a tab without a matching host permission |

**Not requested and not used:**
- `history` — no browsing history access
- `cookies` — session cookie is used passively via `credentials: 'same-origin'`, never read
- `identity` — no OAuth or Google account access

### 5.2 Chrome Storage Schema

All data is stored in `chrome.storage.local`. Keys used:

| Key Pattern | Type | TTL | Description |
|---|---|---|---|
| `fields_v2_{instanceUrl}_{tableName}` | `{ fields: Field[], ts: number }` | 1 hour | Field metadata cache per table per instance |
| `recentSearches` | `SearchState[]` | Permanent | Last 10 search states |
| `savedTemplates` | `Template[]` | Permanent | Named saved searches |
| `lastInstance` | `string` | Permanent | Last detected SN instance URL |
| `lastTable` | `string` | Session | Last selected table name |
| `lastFieldName` | `string` | Session | Last selected field system name |
| `lastFieldLabel` | `string` | Session | Last selected field display label |
| `lastOp` | `string` | Session | Last selected operator |
| `lastValue` | `string` | Session | Last entered value |
| `lastConditions` | `JSON string` | Session | Serialized conditions array |
| `histRecentOpen` | `boolean` | Permanent | Accordion open state for Recent section |
| `histSavedOpen` | `boolean` | Permanent | Accordion open state for Saved section |

### 5.3 ServiceNow API Endpoints Used

All requests use `JSONv2` format, `credentials: 'same-origin'`, and `X-Requested-With: XMLHttpRequest`.

#### Inheritance chain walk — Step A (per level)
```
GET /sys_db_object_list.do?JSONv2
    &sysparm_action=getRecords
    &sysparm_query=name={tableName}
    &sysparm_fields=name,super_class
    &sysparm_limit=1
```

#### Inheritance chain walk — Step B (per level)
```
GET /sys_db_object_list.do?JSONv2
    &sysparm_action=getRecords
    &sysparm_query=sys_id={superClassSysId}
    &sysparm_fields=name
    &sysparm_limit=1
```

#### Bulk field fetch
```
GET /sys_dictionary_list.do?JSONv2
    &sysparm_action=getRecords
    &sysparm_query=name=table1^ORname=table2^...^active=true^elementISNOTEMPTY
    &sysparm_fields=element,column_label,name,internal_type,reference
    &sysparm_limit=3000
```

#### Reference value lookup (STARTSWITH — fast, index-backed)
```
GET /{refTable}_list.do?JSONv2
    &sysparm_action=getRecords
    &sysparm_query=active=true^nameSTARTSWITH{query}^ORnumberSTARTSWITH{query}
    &sysparm_fields=sys_id,name,number
    &sysparm_limit=5
```

#### Reference value lookup (CONTAINS — slower, catches mid-string)
```
GET /{refTable}_list.do?JSONv2
    &sysparm_action=getRecords
    &sysparm_query=active=true^nameCONTAINS{query}^ORnumberCONTAINS{query}
    &sysparm_fields=sys_id,name,number
    &sysparm_limit=5
```

### 5.4 Query Building Logic

`buildQuery(state, els)` assembles the final encoded ServiceNow query string.

**Clause assembly rules:**

| Condition | Generated clause |
|---|---|
| Normal field, normal op, value | `fieldOPERATORvalue` e.g. `state=1` |
| No-value operator (ISEMPTY, ISNOTEMPTY) | `fieldOPERATOR` e.g. `stateISEMPTY` |
| Reference field + plain text value + `=` | `field.nameCONTAINSvalue` (dot-walk to display name) |
| Reference field + plain text value + `!=` | `field.nameDOESNOTCONTAINvalue` |
| Reference field + `sys_id` (32-char hex) | `field=sys_id` (direct match) |
| Reference field picked from typeahead | `field=sys_id` (exact, using stored sys_id) |

**AND/OR join encoding:**

```
condition1^condition2        → AND join
condition1^ORcondition2     → OR join
```

**Generated URL format:**
```
https://{instance}.service-now.com/now/nav/ui/classic/params/target/{table}_list.do?sysparm_query={encoded_query}
```

### 5.5 Query Parsing Logic

`parseQueryString(raw)` converts an existing SN encoded query string back into a `conditions` array. This is used by the "Import filters from page" feature.

**Parsing algorithm:**
1. Scan left-to-right for `^` (AND) and `^OR` (OR) separators
2. Split into `{ clause, join }` segments
3. For each clause, try regex match against `PARSE_OPS` in priority order (longest operators first to avoid partial matches)
4. Build condition objects with `field_name`, `op`, `value`, `join`
5. Silently drop `ORDERBY`, `ORDERBYDESC`, `GROUPBY` clauses

**Operator parse priority (longest first to avoid greedy mismatches):**
```
'DOES NOT CONTAIN', 'ISNOTEMPTY', 'ISEMPTY',
'STARTSWITH', 'ENDSWITH', 'CONTAINS', 'NOTCONTAINS',
'NOT IN', 'IN', 'LIKE', 'NOTLIKE',
'!=', '>=', '<=', '>', '<', '='
```

### 5.6 Performance Considerations

#### What runs when

| Trigger | Network calls | When |
|---|---|---|
| Popup opens (cache hit) | 0 | Every time after first load within 1hr |
| Popup opens (cache miss) | 2–24 serial + 1 bulk | First open per table, or after cache expires |
| Reference value typing | 2–4 parallel | Per debounce cycle (400ms) |
| Dot-walk drill | 2–24 serial + 1 bulk | Per drill, with 10-min cache |
| Import filters from page | 0 (DOM read only) | On button click |
| Run search | 1 (list view navigation) | On search button |

#### Background activity

**Zero.** No content scripts, no timers, no polling. The service worker is terminated by Chrome when idle. Tab listeners (`onActivated`, `onUpdated`) are registered only in the popup context and are cleaned up when the popup closes.

#### Most expensive operation

The inheritance chain walk: up to `MAX_INHERITANCE_DEPTH = 12` levels × 2 requests = 24 sequential HTTP requests before the bulk dictionary query. These are lightweight metadata calls (limit=1, few fields) but they are serial by design because each level's result is needed to form the next request. After the first fetch, the 1-hour cache eliminates all of this cost.

#### Most frequent operation

The reference value lookup (400ms debounced). Two `CONTAINS` calls per cycle can cause full table scans on large tables (`sys_user`, `cmdb_ci`). The `sysparm_limit=5` keeps server response sizes minimal, but query execution time server-side depends on table size and index availability.

### 5.7 Error Handling

| Scenario | Handling |
|---|---|
| Active tab is not SN | `showNoSn()` — hides main UI, shows error screen with cached instance name |
| Field fetch HTTP error | `showStatus('Field fetch failed (http_NNN)', 'error')` |
| Session expired (HTML response) | `showStatus('Session expired — re-login on the SN tab.', 'error')` |
| Network error | Error swallowed in `safeJson()`, returns `{ error: 'network_...' }` |
| Empty field list | Status message; `state.fields = []`; dropdown shows "No fields" |
| `executeScript` failure | `chrome.runtime.lastError` checked; `sendResponse({ error: ... })` |
| Corrupt stored state | `restoreState` is wrapped in try/catch; malformed data is silently ignored |
| Duplicate search signature | Deduplicated before inserting into recent history |
| Ref lookup on table without `active` field | Retries without `active=true^` prefix if first pass returns 0 results |
| `readCurrentPageQuery` global JS error | Returns `{ error: String(e), table: null, query: null }` |

### 5.8 Security Considerations

**XSS prevention:** All user-provided and API-returned strings rendered into innerHTML pass through `esc(s)`, which escapes `&`, `<`, and `>`. The only raw HTML inserted is from trusted internal template literals (e.g. `<span class="match-highlight">...</span>`), which wrap already-escaped content.

```javascript
function esc(s) {
  return String(s)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;');
}
```

**Session isolation:** The extension never reads, stores, or transmits the `JSESSIONID` cookie. It is used implicitly by the browser's fetch mechanism via `credentials: 'same-origin'`. The extension has no access to cookie values.

**No external data transmission:** All `fetch()` calls target the user's own ServiceNow instance (`same-origin` within the injected SN tab context). No data leaves the user's browser except to their own SN server.

**URL construction:** The final search URL is built using `encodeURIComponent` on both the table name and query string before insertion into the navigation URL, preventing any injection through those values.

**Manifest V3 CSP:** MV3 enforces a strict Content Security Policy on extension pages. The popup HTML does not use inline `<script>` blocks, `eval()`, or dynamic code execution. All scripts are referenced as external files.

**Host permission scope:** `https://*.service-now.com/*` is the minimum scope required. The extension does not request `<all_urls>` or broad host permissions.

---

## 6. Development Notes

### 6.1 Local Installation

1. Clone or download the repository
2. Open Chrome and navigate to `chrome://extensions`
3. Enable **Developer mode** via the toggle in the top-right corner
4. Click **Load unpacked** and select the project root directory (the folder containing `manifest.json`)
5. The extension icon appears in the toolbar. Pin it via the puzzle-piece menu for easy access.

**Reloading after edits:**

- Changes to `background.js`: Click the ↺ refresh icon on the extension card in `chrome://extensions`
- Changes to `popup.html`, `popup.js`, `search-core.js`, `styles.css`: Close and reopen the popup (no reload needed since the popup is recreated fresh on each open)
- Changes to `manifest.json`: Reload the extension from `chrome://extensions`

### 6.2 Debugging

**Popup JavaScript:**
Right-click the extension icon → **Inspect popup**. This opens DevTools attached to the popup context. The popup must remain open while inspecting (clicking elsewhere closes it; use `chrome.action.openPopup()` in the service worker console or keep DevTools docked to prevent accidental closure).

**Service worker (background.js):**
Navigate to `chrome://extensions` → find the extension → click **Service Worker** link. This opens a dedicated DevTools instance for the background service worker context. Messages sent and received can be logged here.

**Injected scripts (fetchFieldsFromSN etc.):**
Open DevTools on the ServiceNow tab itself. All `console.log` calls inside injected functions appear in the SN tab's console. The injected functions can also be called manually from the console for testing:

```javascript
// Test field fetch directly in the SN tab console:
fetchFieldsFromSN('incident').then(r => console.log(r));
```

Note: `fetchFieldsFromSN` is not available in the SN tab's console by default since it runs in an isolated world. To test it, temporarily add `console.log` statements inside the function in `background.js` and trigger a field fetch from the popup.

**Storage inspection:**
In the popup DevTools console:
```javascript
chrome.storage.local.get(null, d => console.log(d));  // dump all storage
chrome.storage.local.clear();                           // wipe all storage
```

**Simulating cache miss:**
```javascript
// Clear the field cache for a specific table/instance
chrome.storage.local.remove('fields_v2_https://yourinstance.service-now.com_incident');
```

### 6.3 Known Limitations

| Limitation | Detail |
|---|---|
| JSONv2 max limit | `sysparm_limit=3000` may miss fields on tables with very large dictionaries (>3000 active dictionary entries across the hierarchy) |
| CONTAINS performance | The reference value CONTAINS fallback does a full table scan on SN. On tables with millions of rows, this can be slow regardless of the `limit=5` on results |
| Serial inheritance walk | Up to 24 sequential requests for deeply inherited tables before the first cache entry exists |
| No offline field editing | Without an active SN tab, the field dropdown is non-functional (no cached-mode field browsing) |
| Single instance at a time | The popup connects to the active tab's instance only; no multi-instance support |
| `g_list` availability | The `g_list.getFilter()` fallback in `readCurrentPageQuery` only works in the ServiceNow classic UI; Next Experience (Polaris) may not expose this global |
| Popup close clears tab listeners | `onActivated` / `onUpdated` are registered in the popup context and removed on popup unload, so tab events are not tracked between popup sessions |
| `sysparm_limit=3000` in storage | Each cached field set can consume up to ~500KB of `chrome.storage.local` space. Storage quota is 10MB by default in `chrome.storage.local` |

### 6.4 Recommended Future Enhancements

| Enhancement | Priority | Notes |
|---|---|---|
| Paginate dictionary fetch | High | Handle tables with >3000 dictionary entries by fetching in pages or increasing the limit dynamically |
| Parallel inheritance walk | Medium | Batch multiple `sys_db_object` lookups; reduce from up to 24 serial calls |
| CONTAINS-only toggle | Medium | Let users opt out of the CONTAINS fallback in ref lookups for performance-sensitive environments |
| Dot-walk on imported conditions | Medium | Currently imported conditions show raw field names; dot-walked imported fields need special parsing |
| Next Experience (Polaris) URL support | Medium | `readCurrentPageQuery` targets classic UI URLs; Polaris uses a different route structure |
| Sync storage for templates | Low | `chrome.storage.sync` would allow templates to follow the user across devices |
| Field type filtering | Low | Let users filter the field dropdown by type (string, boolean, reference, etc.) |
| Operator presets per field type | Low | Auto-select a sensible default operator based on the field's `internal_type` |
| Keyboard shortcut to open popup | Low | Register a `chrome.commands` shortcut (e.g. `Alt+S`) to open the popup without clicking the icon |
| Result count preview | Low | Before navigating, fetch `sysparm_count=true` to show how many records the query would return |

---

## 7. Function-Level Documentation

### 7.1 `background.js`

---

#### `fetchFieldsFromSN(tableName)`

**Context:** Injected into the ServiceNow tab via `chrome.scripting.executeScript`.

**Inputs:**
- `tableName` {string} — the ServiceNow table name (e.g. `'incident'`)

**Output:** `{ fields: Field[], tables: string[], count: number }` or `{ fields: [], error: string }`

**Execution:**

1. **`safeJson(url)`** (inner async helper) — wraps `fetch()` with `credentials: 'same-origin'` and `X-Requested-With: XMLHttpRequest`. Returns `{ data }` on success, `{ error }` on HTTP error, network failure, or HTML content-type (session expired).

2. **Inheritance chain walk** — iterates while `depth < MAX_DEPTH (12)`:
   - Fetches `sys_db_object_list.do?JSONv2` to get the current table's `super_class` field (returns raw sys_id in JSONv2)
   - Fetches `sys_db_object_list.do?JSONv2` again with that sys_id to resolve the parent table name
   - Pushes each table name into the `tables` array
   - Breaks on any error, empty result, or missing `super_class`

3. **Bulk dictionary fetch** — constructs `name=t1^ORname=t2^...^active=true^elementISNOTEMPTY` and fetches `sys_dictionary_list.do?JSONv2` with `sysparm_limit=3000`. Returns `element`, `column_label`, `name`, `internal_type`, `reference`.

4. **Deduplication** — assigns a priority score per table (child tables score highest). Iterates `rows`, keeping only the highest-priority entry per `element` name. Child overrides of inherited fields always win.

5. **Return** — maps the deduplicated map to an array, sorts alphabetically by label.

```javascript
// Field object shape:
{
  name:      'assigned_to',    // sys_dictionary.element
  label:     'Assigned to',   // sys_dictionary.column_label
  type:      'reference',     // sys_dictionary.internal_type
  reference: 'sys_user',      // sys_dictionary.reference (table name)
  table:     'task'           // which table in the hierarchy owns this field
}
```

---

#### `lookupRefRecords(refTable, query)`

**Context:** Injected into the SN tab via `executeScript`.

**Inputs:**
- `refTable` {string} — reference table name (e.g. `'sys_user'`)
- `query` {string} — partial display name or number typed by user

**Output:** `{ records: [{ sys_id, name }] }` — up to 8 deduplicated records

**Execution:**

1. **`safeGet(url)`** (inner helper) — same fetch wrapper as above, returns parsed records array.

2. **`runLookup(prefix)`** — runs two parallel requests:
   - `{prefix}nameSTARTSWITH{query}^ORnumberSTARTSWITH{query}` (index-backed)
   - `{prefix}nameCONTAINS{query}^ORnumberCONTAINS{query}` (full scan)

3. First run uses `prefix = 'active=true^'`. If 0 records returned, retries with `prefix = ''` (for custom tables without an `active` field).

4. Merges results from both branches, deduplicates by `sys_id`, caps at 8 records.

---

#### `readCurrentPageQuery()`

**Context:** Injected into the SN tab via `executeScript` with `world: 'MAIN'` (page context).

**Inputs:** None (reads `window.location.href` and optionally `g_list`)

**Output:** `{ table: string|null, query: string|null }` or `{ error, table: null, query: null }`

**Execution:**

1. Reads `window.location.href`
2. `decodeURIComponent()` once to unwrap SN's nav router encoding (the router places the list URL inside the path, URL-encoded)
3. Extracts table name via two regex patterns:
   - `/\/([a-zA-Z0-9_]+)_list\.do/`
   - `/target\/([a-zA-Z0-9_]+)_list/`
4. Extracts `sysparm_query` from `new URL(rawHref).searchParams` (primary — avoids the encoded path copy)
5. Falls back to regex match on decoded path
6. Falls back to `g_list.getFilter()` (SN classic UI global)

---

### 7.2 `search-core.js`

---

#### `createState()`

**Returns:** A fresh state object with all properties initialized to their defaults.

```javascript
{
  snTab:              null,     // Chrome Tab object for the active SN tab
  instanceUrl:        null,     // 'https://xxx.service-now.com'
  fields:             [],       // Field[] — loaded field list for current table
  selectedField:      null,     // Field object for the currently selected field
  dropdownKeyIndex:   -1,       // Keyboard nav cursor in field dropdown
  fieldsLoading:      false,    // True while field fetch is in flight
  recentSearches:     [],       // SearchState[]
  savedTemplates:     [],       // Template[]
  lastKnownTable:     null,
  tabWatchInterval:   null,     // Unused in current version
  fetchRequestId:     0,        // Monotonic counter for race-condition guard
  conditions:         [],       // Committed multi-condition chips
  currentJoin:        'AND',    // Join for the next condition
  dotWalkStack:       [],       // Navigation stack for reference drill-down
  dotWalkCurrentFields: null,   // Fields for the current dot-walk level
}
```

---

#### `buildQuery(state, els)`

**Inputs:**
- `state` — current state object
- `els` — DOM element references

**Output:** {string} — SN-encoded query string (e.g. `state=1^priority<=2^ORassigned_to.nameCONTAINSjohn`)

**Logic:** Iterates `state.conditions` and the current partial condition (field/op/value in the inputs). Each clause is appended with `^` (AND) or `^OR` (OR) prefix. Reference fields with plain-text values are automatically converted to dot-walk form (see §5.4).

---

#### `parseQueryString(raw)`

**Inputs:**
- `raw` {string} — raw SN encoded query string

**Output:** `Condition[]` — array of condition objects

**Algorithm:** See §5.5 for detail. Handles `^OR` and `^` separators correctly, drops ordering clauses, and uses length-priority operator matching to avoid greedy partial matches.

---

#### `fetchFields(tableName, state, els, callbacks)`

**Inputs:**
- `tableName` {string}
- `state` — current state
- `els` — DOM refs
- `callbacks` — `{ onLoading(bool), onFields(fields, count, tables), onError(msg, type) }`

**Execution:**

1. Checks `chrome.storage.local` for a valid cache entry (`fields_v2_{instanceUrl}_{tableName}`, not older than `CACHE_TTL_MS`)
2. Cache hit → calls `onFields` immediately, returns
3. Cache miss → increments `fetchRequestId`, calls `onLoading(true)`, sends `FETCH_FIELDS` message to background
4. On response: checks `requestId` matches (stale response guard), calls `onLoading(false)`
5. On success: stores result in cache, calls `onFields`
6. On error: calls `onError` with user-friendly message

---

#### `detectInstance(state, els, callbacks)`

**Inputs:**
- `state`, `els`, `callbacks` — `{ onOnline(name, url, tab), onCached(name), onOffline() }`

**Output:** `Promise<Tab|null>`

**Execution:**

Queries `chrome.tabs.query({ active: true, currentWindow: true })`. If the active tab matches `/^(https:\/\/[a-zA-Z0-9_-]+\.service-now\.com)/`:
- Calls `commitTab(tab)` → sets `state.snTab`, `state.instanceUrl`, fires `onOnline`

Otherwise falls back to `chrome.storage.local.get(['lastInstance'])`:
- If found: fires `onCached` with the cached instance name, then `onOffline`
- If not found: fires `onOffline`

---

#### `debounce(fn, ms)`

**Inputs:**
- `fn` {Function} — function to debounce
- `ms` {number} — delay in milliseconds

**Output:** Debounced wrapper function

Standard trailing-edge debounce. Each call resets the timer. `fn` fires only after `ms` ms of inactivity.

---

#### `esc(s)`

**Inputs:** `s` {any} — value to escape

**Output:** {string} — HTML-safe string

Escapes `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`. Must be applied to any user-controlled or API-returned string before inserting into `innerHTML`.

---

#### `addCondition(state, els)`

**Inputs:** `state`, `els`

**Output:** {boolean} — `true` if condition was added, `false` if inputs were incomplete

Reads current field/op/value from state and inputs. Returns `false` if `field_name` is empty or if `value` is empty for value-requiring operators. Pushes a new condition object to `state.conditions`, resets `state.currentJoin` to `'AND'`.

---

#### `saveState(state, els, prefix)`

Serializes the current UI state to `chrome.storage.local`. Stores: table, field name, field label, operator, value, and the conditions array (JSON-stringified).

---

#### `restoreState(state, els, prefix, onRestored)`

**Output:** `Promise<void>`

Reads keys from `chrome.storage.local` and repopulates the UI. Handles custom table (sets the `customTableGroup` visible and fills `customTableInput`). Parses the conditions JSON string back to an array. Calls `onRestored()` callback on completion.

---

### 7.3 `popup.js`

---

#### `init()`

**Entry point called once on popup load.**

Sequential initialization:
1. `loadTemplates(state)` — load recent + saved from storage
2. `renderTemplates()` — paint the history sections
3. `updateOpHint()` — show initial operator hint
4. `initHistoryToggles()` — wire accordion expand/collapse + restore persisted open/closed state
5. `restoreState(state, els, '', callback)` — repopulate all inputs; callback renders conditions chips
6. `detectInstance(state, els, callbacks)` — detect SN tab; on success: auto-detect table from URL, show main UI
7. `loadFieldsForTable(getTable(els))` — trigger field fetch if SN tab available
8. `setTimeout(() => els.fieldInput.focus(), 120)` — auto-focus field input after render
9. `startTabWatch()` — attach tab event listeners

---

#### `renderConditions()`

Regenerates the `#conditionsChips` innerHTML from `state.conditions`. Each condition renders as a `.sn-filter-row` with field name, operator, value, and a remove button. The remove button implements double-click-to-confirm: first click adds the `armed` class and shows `?`; second click within 2 seconds calls `removeCondition()`.

Also calls `syncJoinToggle()` (shows/hides the AND/OR toggle based on whether conditions exist) and `saveState()` to persist immediately.

---

#### `renderDropdown(fields, query)`

Generates the field dropdown HTML from a `Field[]` array. Handles four states: loading (spinner), empty with query (no match message), empty without query (no SN tab message), and populated (list of items).

Each item includes:
- The field label (with substring match highlighted via `highlight()`)
- The field system name in grey
- A `REF` badge if `f.type === 'reference'`
- A `→` drill-down button (`dw-drill-btn`) if `f.type === 'reference' && f.reference`

Attaches mouse event listeners immediately after rendering via `attachDwListeners()`.

---

#### `drillDownField(f)`

**Inputs:** `f` — `{ name, label, reference }` — the field being drilled into

Pushes an entry onto `state.dotWalkStack`, clears `state.dotWalkCurrentFields`, clears `els.fieldInput.value`, and calls `loadFieldsForDotWalk(f.reference)`.

---

#### `loadFieldsForDotWalk(tableName)`

Checks `chrome.storage.local` for a cached entry (10-minute TTL, shorter than the main 1-hour TTL). On cache miss, sends `FETCH_FIELDS` to background. On completion, sets `state.dotWalkCurrentFields` and re-renders the dropdown.

---

#### `selectField(name, label)`

**Inputs:** `name` {string}, `label` {string} — selected field's system name and display label

Prepends the dot-walk prefix from `getDotWalkPrefix()` to form the full field path. Updates `state.selectedField`, the field input value, and shows the clear button. Clears the value input and closes the dropdown. Focuses the value input.

---

#### `debouncedRefLookup` (400ms debounced function)

Fires `FETCH_REF_LOOKUP` to background with `state.selectedField.reference` as the target table. Only fires if:
- `state.selectedField.type === 'reference'`
- Field is at root level (no `.` in name)
- Query is at least 2 characters
- A SN tab is available

Renders `renderValueDropdown()` on success.

---

#### `importFiltersFromPage()`

Sends `FETCH_CURRENT_QUERY` to background. On success:
- Applies the detected table to the dropdown
- Parses the query string via `parseQueryString()`
- Merges imported conditions with existing ones (deduplicates by `field+op+value`)
- Calls `resolveConditionLabels()` once fields are loaded (replaces `field_name` placeholders with proper `column_label` values)
- Shows status message with count of imported conditions

---

#### `initCustomSelect({ nativeSelect, trigger, panel, valueEl, hints })`

**Inputs:**
- `nativeSelect` — the hidden native `<select>` element (source of truth)
- `trigger` — the visible click target
- `panel` — the dropdown panel container
- `valueEl` — the element showing the selected option's text
- `hints` — optional object mapping option values to hint text strings

**Output:** `{ syncTrigger, close }` — control handles for programmatic sync

Builds panel HTML from the native select's `<option>` and `<optgroup>` children. Wires:
- Click on trigger → open/close panel
- Click on panel option → update native select value, dispatch `change` event, sync trigger label, close
- Keyboard on trigger: `Enter`/`Space` → open or confirm, `Escape` → close, letter keys → jump to matching option and cycle through matches
- Global `mousedown` → close when clicking outside

Used for both the table selector and operator selector.

---

#### `startTabWatch()`

Attaches `chrome.tabs.onActivated` → `onTabActivated()` and `chrome.tabs.onUpdated` → `onTabUpdated()` listeners. Also attaches a `window.unload` handler to remove them when the popup closes, preventing listener leaks.

---

#### `applyTabState(tab)`

Called by both tab event handlers. Checks if the new tab's URL matches `*.service-now.com`.

- **Match:** Updates `state.snTab`, `state.instanceUrl`, badge label. If the instance changed, clears `state.fields` and reloads fields for the current table. Calls `showMain()`.
- **No match:** Sets `state.snTab = null`, calls `showNoSn()`.

---

#### `doSearch()`

**The primary action.** Calls `buildQuery(state, els)`, `recordRecentSearch()`, `renderTemplates()`, then sends `OPEN_URL` to background with the full navigation URL. Shows a brief "Opening…" status.

---

## 8. Quick Summary

### For Developers

ServiceNow Power Search is a **Manifest V3 Chrome extension** with three JavaScript execution contexts:

1. **Popup context** (`popup.html` + `popup.js` + `search-core.js`) — the user interface. All UI state is managed here. Communicates with the background worker via `chrome.runtime.sendMessage`.

2. **Background service worker** (`background.js`) — a stateless message router. Routes popup messages to either native Chrome APIs (tab navigation) or injected functions running inside the SN tab.

3. **Injected context** (functions defined in `background.js`, executed inside the SN tab) — performs all HTTP requests to the ServiceNow instance using the existing browser session cookie. Three functions: `fetchFieldsFromSN`, `lookupRefRecords`, `readCurrentPageQuery`.

**Data flows in one direction:** Popup → Background → SN Tab → Background → Popup. The SN tab context never initiates communication.

**State lives in two places:** In-memory in the popup (`state` object from `createState()`) and persistently in `chrome.storage.local` (field cache, templates, recent history, last UI state).

**All ServiceNow API calls use JSONv2** (`.do?JSONv2`) to avoid triggering Chrome's Basic Auth dialog, which the REST Table API would trigger when session cookies are present without a Bearer token.

### For Stakeholders

When a user opens the extension popup on a ServiceNow tab, the extension:

1. Detects which ServiceNow instance they are logged into from the active browser tab
2. Fetches the complete list of fields for the selected table directly from that instance's data dictionary — using the user's existing login session, no password required
3. Presents a guided form where the user picks a field, an operator, and a value using autocomplete
4. For reference fields (links to other tables), offers two options: drill down to explore related table fields (dot-walk), or search by display name with live record lookup
5. Allows multiple conditions to be chained together with AND/OR logic
6. Builds the correct ServiceNow filter query string and opens the filtered list view in the same tab

No data leaves the user's browser to any external server. The extension communicates only with the user's own ServiceNow instance. All field metadata is cached locally for one hour to minimize API calls on repeated use.

---

*ServiceNow Power Search — Technical Documentation v2.1.0*
*Not affiliated with ServiceNow, Inc.*
