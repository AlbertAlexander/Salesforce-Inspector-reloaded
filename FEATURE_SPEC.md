# Feature Specification: Data Import Enhancements

**Project:** Salesforce Inspector Reloaded
**Date:** 2026-03-06
**Scope:** Four new features for `addon/data-import.js`, each individually flag-gated via localStorage.

---

## Table of Contents

1. [Feature Flag Architecture](#1-feature-flag-architecture)
2. [Feature A: Auto-Format Field Values (`FLAG_AUTO_FORMAT`)](#2-feature-a-auto-format-field-values)
3. [Feature B: Enhanced Column Label Matching (`FLAG_SMART_COLUMN_MATCH`)](#3-feature-b-enhanced-column-label-matching)
4. [Feature C: ID Resolution via Live SOQL (`FLAG_ID_RESOLUTION`)](#4-feature-c-id-resolution-via-live-soql)
5. [Feature D: Persistent Column Mappings (`FLAG_SAVED_MAPPINGS`)](#5-feature-d-persistent-column-mappings)
6. [Loading the Fork as an Unpacked Chrome Extension](#6-loading-the-fork-as-an-unpacked-chrome-extension)

---

## 1. Feature Flag Architecture

### Overview

All four features are individually toggled via localStorage boolean flags. This mirrors the exact pattern already used throughout the extension (e.g., `greyOutSkippedColumns`, `displayQueryPerformance`, `openLinksInNewTab`). No new infrastructure is required.

### Flag Storage Keys

| Feature | localStorage Key | Default |
|---|---|---|
| Auto-Format Field Values | `FLAG_AUTO_FORMAT` | `false` |
| Enhanced Column Label Matching | `FLAG_SMART_COLUMN_MATCH` | `false` |
| ID Resolution via Live SOQL | `FLAG_ID_RESOLUTION` | `false` |
| Persistent Column Mappings | `FLAG_SAVED_MAPPINGS` | `false` |

### Reading a Flag at Runtime

Flags are read with a simple helper that can live in `addon/utils.js` alongside the existing `StorageHistory` class:

```js
export function isFeatureEnabled(flagKey) {
  return localStorage.getItem(flagKey) === "true";
}
```

Call sites in `data-import.js` then look like:

```js
import { isFeatureEnabled } from "./utils.js";

// inside guessColumn(), executeBatch(), etc.:
if (isFeatureEnabled("FLAG_SMART_COLUMN_MATCH")) {
  // enhanced logic
}
```

### Exposing Flags in the Options Page

The Options page (`addon/options.js`) already exposes an `Option` component with `type: "toggle"` that reads and writes localStorage directly via its `key` prop. Add a new tab entry — or append to the existing `"data-import"` tab — for each flag:

```js
// In the "data-import" tab content array inside OptionsTabSelector (options.js ~line 258):
{option: Option, props: {type: "toggle", title: "Auto-Format Field Values (experimental)", key: "FLAG_AUTO_FORMAT", default: false, tooltip: "Automatically convert dates, datetimes, and currency values to Salesforce-compatible formats before upload."}},
{option: Option, props: {type: "toggle", title: "Smart Column Label Matching (experimental)", key: "FLAG_SMART_COLUMN_MATCH", default: false, tooltip: "Match CSV headers to Salesforce field labels and common synonyms, not just API names."}},
{option: Option, props: {type: "toggle", title: "ID Resolution via SOQL (experimental)", key: "FLAG_ID_RESOLUTION", default: false, tooltip: "Resolve lookup/reference columns containing names or emails to Salesforce IDs before upload."}},
{option: Option, props: {type: "toggle", title: "Persistent Column Mappings (experimental)", key: "FLAG_SAVED_MAPPINGS", default: false, tooltip: "Save and auto-apply column-to-field mappings per SObject type across sessions."}},
```

This requires zero new UI components — the toggle already handles localStorage reads/writes.

---

## 2. Feature A: Auto-Format Field Values

**Flag:** `FLAG_AUTO_FORMAT`

### Problem Solved

Field values copied from spreadsheets or third-party exports arrive in arbitrary formats (e.g., `"3/15/2024"`, `"$1,250.00"`, `"John Smith"`). The Salesforce API rejects these silently or with cryptic errors. Currently `data-import.js` sends all cell values as raw strings with no preprocessing.

### Implementation Approach

**Where to hook in:** `executeBatch()` in `addon/data-import.js` (line 732) builds the array of `sObjects` that gets sent to the API. Cell values are read at line 753 from `importTable.data`. The transform step belongs immediately before the value is assigned to the sObject field — after column mapping is resolved but before the SOAP/REST payload is constructed.

**Field type source:** `DescribeInfo.describeSobject()` from `addon/data-load.js` returns a full `sobjectDescribe` whose `fields` array contains `type` and `soapType` for every field. The `Model` instance already holds `this.describeInfo` and calls `describeSobject` in `columnList()` and `idLookupList()` — the same pattern applies here.

**Transform table:**

| Salesforce `type` | Transform | Notes |
|---|---|---|
| `date` | Parse with `Date.parse()` fallback to common locale formats; output `YYYY-MM-DD` | Reject ambiguous dates (e.g., `04/05/06`) with a warning |
| `datetime` | Parse and output ISO 8601: `YYYY-MM-DDTHH:mm:ssZ` | Default to UTC if no timezone present |
| `currency`, `double`, `percent` | Strip leading currency symbols (`$`, `€`, `£`, etc.), strip thousands separators (`,`), strip currency codes (e.g., `USD`); parse as float | Warn if result is `NaN` |
| `string` fields named `Name` on `Contact` / `Lead` | Detect single-value "First Last" combined name; emit a warning and suggest splitting into `FirstName` + `LastName` columns. Do NOT auto-split silently. | Only trigger when both `FirstName` and `LastName` exist as writable fields on the SObject and neither is already mapped |

**Preview UI:** Before upload begins (when the user clicks the import button and `FLAG_AUTO_FORMAT` is on), render a collapsible summary section above the import table showing: field name, original value, transformed value, and transform type applied. The existing React render tree in the `App` component (bottom of `data-import.js`) already has a pattern for conditional status sections — follow that approach.

### Edge Cases and Warnings

- **Ambiguous dates:** `04/05/06` is DD/MM/YY in some locales and MM/DD/YY in others. Surface a per-row warning in the preview step; do not silently choose one interpretation.
- **Mixed currency symbols in a single column:** If column values include both `$1,000` and `€1,000`, emit a column-level warning that currency mixing was detected.
- **Name splitting:** Never auto-split silently. Show a yellow advisory banner: "Column 'Name' contains combined names. Add separate FirstName and LastName columns and re-map."
- **Field not yet resolved:** Auto-format only runs after column mapping is complete (i.e., after `refreshColumn()` has matched columns to known fields). If a column maps to an unknown or skipped field, skip formatting for that column entirely.
- **Empty cells:** Pass through as-is; do not convert empty strings to `null` (let Salesforce handle null semantics).

### Test Approach

Tests live in `tests/e2e/data-import.spec.js` using Playwright. The existing test infrastructure (`routeMock` in `test-mock.js`, `injectSessionData`, `pasteData`) is sufficient; no new fixtures are needed.

**Tests to add:**

1. **Date normalization:** Paste a CSV with a `date`-type field containing `"3/15/2024"`. Assert that after transformation (with flag on), the cell value shown in the preview is `"2024-03-15"`. Assert that with flag off, value is unchanged.
2. **Currency stripping:** Paste a CSV with a `currency`-type field containing `"$1,250.00"`. Assert preview shows `"1250"` (or `"1250.00"`). Assert `NaN` input (`"abc"`) produces a visible warning.
3. **Datetime ISO conversion:** Paste `"2024-03-15 09:30:00"` into a `datetime` column. Assert preview shows `"2024-03-15T09:30:00Z"`.
4. **Name advisory:** Paste a `Contact` CSV with a single `Name` column containing `"John Smith"`. Assert that a warning advisory is rendered when `FirstName` and `LastName` are available fields and the flag is on.
5. **Flag off = no change:** For each of the above, assert the preview step is absent and values are unchanged when `FLAG_AUTO_FORMAT` is `false` in localStorage.

Mock the describe response for `Inspector_Test__c` in `test-mock.js` to include fields with `type: "date"`, `type: "currency"`, and `type: "datetime"` — extend the existing `Inspector_Test__c` describe mock.

---

## 3. Feature B: Enhanced Column Label Matching

**Flag:** `FLAG_SMART_COLUMN_MATCH`

### Problem Solved

`guessColumn()` (line 659 of `data-import.js`) currently only handles `Object.Field` dot-notation. A CSV exported from a reporting tool typically uses human-readable column headers like `"Account Name"`, `"Owner"`, or `"Email Address"` — none of which match Salesforce API names. The user must manually remap every column.

### Implementation Approach

**Where to hook in:** Extend `guessColumn(col)` in `data-import.js`. The function is called from `refreshColumn()` (line 684) whenever the SObject or action changes, and from `makeColumn()` at parse time (line 189). The enhancement adds additional resolution passes after the existing dot-notation check fails.

**Resolution pass order (when flag is on):**

1. **Exact API name match** (existing behavior — always runs regardless of flag)
2. **Dot-notation match** (existing behavior)
3. **Exact label match:** Compare `col.trim().toLowerCase()` against `field.label.toLowerCase()` for all fields returned by `describeSobject()`. Return the API name of the first match.
4. **Normalized label match:** Strip punctuation and extra whitespace from both the column header and each field label before comparing (handles `"Account Name"` vs `"AccountName"`).
5. **Synonym table lookup:** A small static map in `data-import.js` (or a new `addon/column-synonyms.js`) covering common cases:

| CSV header (case-insensitive) | Maps to |
|---|---|
| `owner`, `owner name`, `owner full name` | `OwnerId` (with a note that ID resolution is needed) |
| `account`, `account name` | `AccountId` (with same note) |
| `email`, `email address` | `Email` |
| `phone`, `phone number`, `mobile`, `mobile phone` | `Phone` or `MobilePhone` depending on which exists on the SObject |
| `first name` | `FirstName` |
| `last name`, `surname` | `LastName` |
| `title`, `job title` | `Title` |
| `company` | `CompanyName` (Lead) or `AccountId` (Contact, with note) |

**Confidence indicator:** `makeColumn()` (line 689) builds the column view-model object returned to React. Extend it with a `columnMatchConfidence` property:
- `"exact"` — matched by API name (existing)
- `"label"` — matched by field label
- `"synonym"` — matched via synonym table
- `"none"` — no match

In the column header row of the import table UI, render a small visual badge next to inferred matches: a grey `~` prefix or a tooltip showing "Matched by label: 'Account Name' -> AccountId". The existing column render loop (around line 1304 of `data-import.js`) already inspects column properties for error states — add a similar check for `columnMatchConfidence`.

**No fuzzy string distance:** Do not implement Levenshtein/edit-distance matching. The synonym table plus label matching is sufficient and avoids false positives that would silently corrupt imports.

### Edge Cases and Warnings

- **Ambiguous synonym match:** `"Phone"` exists as both `Phone` and `MobilePhone` on Contact. When the synonym table could match multiple fields, do not auto-select — instead, leave the column unresolved and show a tooltip: "Ambiguous: could be Phone or MobilePhone. Please select manually."
- **OwnerId / AccountId from name columns:** These reference fields require a Salesforce ID, not a name. When `FLAG_SMART_COLUMN_MATCH` maps `"Owner"` to `OwnerId`, mark the column with a visual indicator (e.g., amber badge) noting "ID required — enable ID Resolution to auto-resolve names." This pairs with Feature C.
- **DescribeInfo not yet loaded:** `guessColumn()` is called during `makeColumn()` at parse time, before describe data may have returned. Guard with a null check on `sobjectDescribe`; the column will re-resolve when `refreshColumn()` is called after describe loads (existing behavior via the `didUpdate` callback chain).
- **Custom field labels with special characters:** Strip `__c` from API names before label comparison to avoid spurious matches.

### Test Approach

1. **Label match:** Paste a CSV with header `"Account Name"` for an Account import. Assert (with flag on) that `columnValue` resolves to `AccountId` or the correct relationship field, and that `columnMatchConfidence` is `"label"`. Assert (with flag off) that the column shows as unknown.
2. **Synonym match:** Paste a CSV with header `"Owner"` for a Contact import. Assert that with flag on, `columnValue` is `OwnerId` and `columnMatchConfidence` is `"synonym"`. Assert that an amber indicator or warning tooltip is present in the rendered UI.
3. **Ambiguous synonym:** Paste a CSV with header `"Phone"` for a Contact import (Contact has both `Phone` and `MobilePhone`). Assert that the column remains unresolved and a tooltip message about ambiguity is shown.
4. **Exact match still works:** Paste a CSV with header `"LastName"`. Assert it resolves exactly regardless of flag state (existing behavior must not regress).
5. **No false positives:** Paste a CSV with a completely unmapped header `"Synergy Score"`. Assert it remains unresolved even with the flag on.

---

## 4. Feature C: ID Resolution via Live SOQL

**Flag:** `FLAG_ID_RESOLUTION`

### Problem Solved

Update and Upsert operations require Salesforce record IDs. Users routinely have data with names, emails, or external identifiers instead of IDs. Currently the tool has no mechanism to resolve these — the user must pre-process the CSV manually or the import fails with lookup errors.

### Implementation Approach

**Where to hook in:** Add a new async method `resolveIds(model)` that runs after the user clicks the import button (currently `onImportAction` or equivalent) but before `executeBatch()` starts. The method gates on `FLAG_ID_RESOLUTION` and the presence of reference-type columns containing non-ID values.

**Resolution logic:**

1. **Identify candidate columns:** After column mapping, iterate `importTable.header`. For each column where `columnValue` corresponds to a field with `type === "reference"` (from `sobjectDescribe.fields`), inspect the column's cell values. Use the existing `isRecordId()` helper from `addon/utils.js` to determine if each value is already a valid Salesforce ID (15 or 18 chars, alphanumeric). Columns where all non-empty values pass `isRecordId()` are skipped — they are already resolved.

2. **Determine resolution strategy per column:** For each candidate column, determine the target SObject from `field.referenceTo[]`. Then:
   - If the target SObject has a field named `Email` that is `idLookup: true` (User, Contact), attempt email match first.
   - Otherwise attempt `Name` match.
   - Always attempt exact ID match as a no-op pass (already handled by step 1).

3. **Issue SOQL queries:** Use `sfConn.rest()` from `addon/inspector.js` (the same function used throughout the codebase). Batch distinct lookup values per column into a single SOQL `IN` clause:

   ```
   SELECT Id, Name, Email FROM User WHERE Email IN ('alice@example.com', 'bob@example.com')
   ```

   Limit to 200 values per query (matches the Salesforce SOQL `IN` clause practical limit). For large CSVs, issue multiple queries.

4. **Build a resolution map:** Key: original cell value (case-insensitive). Value: resolved Salesforce ID, or `null` if unresolved, or `["id1", "id2"]` if ambiguous.

5. **Pre-upload review step:** Before `executeBatch()` runs, render a review panel showing three tables:
   - **Resolved:** original value -> resolved ID (linked to Salesforce record using `sfLink` pattern already used in the extension)
   - **Unresolved:** rows where no match was found. User can enter an ID manually.
   - **Ambiguous:** rows where multiple matches exist. User must select one from a dropdown.

   The import button is disabled until the user has addressed all ambiguous rows. Unresolved rows may proceed (they will fail at the API level with a Salesforce error, which is the current behavior).

6. **Apply resolved IDs:** Before constructing the SOAP payload in `executeBatch()`, substitute resolved IDs into the data rows using the resolution map.

**Required fields resolution (Owner, Account):** When the import action is `create` and the SObject has a required `OwnerId` or `AccountId` field that is not present in the CSV, the resolution step should optionally pre-populate these fields. Specifically:
- If `OwnerId` is required and missing: query `SELECT Id FROM User WHERE Id = :currentUserId` (the session's user ID is available via `sfConn.getSession()`) and inject `OwnerId` for all rows.
- If `AccountId` is required and missing: do not auto-inject (Account is data-specific). Surface a warning: "AccountId is required but not mapped. Add an Account column or skip."

### Edge Cases and Warnings

- **Multiple `referenceTo` types:** A polymorphic lookup field (e.g., `WhatId` on Task) can reference multiple SObject types. Surface a warning: "Polymorphic lookup detected — ID Resolution is not supported for this column. Map manually." Skip resolution for that column.
- **SOQL governor limits:** Each resolution query consumes API calls. Display the count of queries that will be issued before running them, and allow the user to cancel. Store a count in `apiStatistics` (already used via `addon/api-statistics.js`) so the queries appear in the API Stats debug view.
- **Case sensitivity:** Salesforce SOQL `IN` clauses for `Email` are case-insensitive at the DB level but the comparison string should be normalized to lowercase before building the resolution map key.
- **Empty cells in reference columns:** Treat as "no resolution needed" — pass through as empty (Salesforce will interpret as null/no change for updates).
- **Name collisions across SObjects:** If `referenceTo` includes both `User` and `Contact` (e.g., a polymorphic field), skip resolution and warn.
- **Self-referential lookups:** A field like `ReportsToId` on User references User itself. Resolution works the same way but the SOQL query targets the same SObject.
- **Resolution result links in UI:** Each resolved ID in the review table links to `https://{sfHost}/{resolvedId}` so the user can visually verify the match before proceeding.

### Test Approach

The resolution step requires mocking SOQL query responses. The existing `routeMock` in `test-mock.js` already handles `/services/data/.../query/` routes.

1. **Email resolution - success:** Set up a CSV with an `OwnerId` column containing an email address. Mock the SOQL query to return a single User record. Assert that after resolution, the review panel shows the email in the "Resolved" table with the correct ID and a link. Assert that the import button becomes enabled.
2. **Name resolution - success:** Set up a CSV with an `AccountId` column containing `"Test Account 1"`. Mock the SOQL query to return the matching Account. Assert resolution.
3. **Ambiguous match:** Mock the SOQL query to return two records with the same Name. Assert that the "Ambiguous" table shows both options and the import button remains disabled until one is selected.
4. **Unresolved:** Mock the SOQL query to return zero records. Assert the row appears in the "Unresolved" table. Assert the import button is still enabled (unresolved rows are allowed through).
5. **Already-ID column skipped:** Set up a CSV where the reference column already contains valid 18-char IDs. Assert that no SOQL query is issued (verify by checking that the query route mock was not called).
6. **Flag off = no resolution step:** Assert that with `FLAG_ID_RESOLUTION = false`, the review panel never appears and `executeBatch()` runs immediately.
7. **OwnerId injection:** Set up a Create import with no `OwnerId` column. Assert (flag on) that an info banner appears offering to inject the current user's ID, and that after accepting, all rows get the session user ID in the payload.

---

## 5. Feature D: Persistent Column Mappings

**Flag:** `FLAG_SAVED_MAPPINGS`

### Problem Solved

Every time a user imports data for the same SObject (e.g., Contact updates from a monthly export), they must manually remap the same CSV columns to the same Salesforce fields. There is no memory between sessions.

### Implementation Approach

**Storage:** Use `StorageHistory` from `addon/utils.js` directly. The class already handles JSON serialization, deduplication, and a max-entry cap. Instantiate one `StorageHistory` per SObject type, keyed by `"columnMappings_" + sobjectName.toLowerCase()`:

```js
// Conceptual usage in data-import.js Model:
import { StorageHistory } from "./utils.js";

saveMappings(sobjectName, mappings) {
  // mappings = array of {originalHeader, columnValue} objects
  const store = new StorageHistory("columnMappings_" + sobjectName.toLowerCase(), 1, {
    matchAdd: (existing, incoming) => existing.sobject === incoming.sobject,
    addToFront: true
  });
  store.add({ sobject: sobjectName, mappings, savedAt: new Date().toISOString() });
}

loadMappings(sobjectName) {
  const store = new StorageHistory("columnMappings_" + sobjectName.toLowerCase(), 1);
  return store.list[0]?.mappings || null;
}
```

Only the most recent mapping per SObject is kept (max = 1). The `StorageHistory` deduplication via `matchAdd` ensures the entry is replaced on each save rather than accumulating.

**When to save:** After the user successfully completes an import (i.e., after `executeBatch()` finishes with at least one `Succeeded` row and zero failures), auto-save the current column mapping. Alternatively, add an explicit "Save Mapping" button in the column mapping area for user-initiated saves.

**When to apply:** In `refreshColumn()` (line 673 of `data-import.js`), after the existing `guessColumn()` resolution, check if a saved mapping exists for the current `this.importType`. For each column in `importTable.header`, if `savedMappings` contains an entry whose `originalHeader` matches `column.columnOriginalValue`, override `column.columnValue` with the saved `columnValue`. Only apply if the saved field still exists in `columnList()` — skip stale mappings silently.

**Saved mapping entry format:**

```json
{
  "sobject": "Contact",
  "savedAt": "2026-03-06T12:00:00Z",
  "mappings": [
    { "originalHeader": "Full Name", "columnValue": "Name" },
    { "originalHeader": "Work Email", "columnValue": "Email" },
    { "originalHeader": "Account", "columnValue": "AccountId" }
  ]
}
```

**UI indicators:** When a saved mapping is applied, show a dismissible info banner: "Saved column mapping for Contact applied. [Review] [Clear]". The "Clear" link removes the saved mapping from localStorage and re-runs `refreshColumn()` without it.

**Options page UI for managing saved mappings:** Add a new sub-section in the `"data-import"` Options tab to view and delete saved mappings. Since the Options page already uses the `Option` component and `StorageHistory`-like patterns, add a custom component `SavedMappingsManager` that:
- Lists all `localStorage` keys matching `"columnMappings_*"`
- Displays SObject name and `savedAt` date
- Provides a Delete button per entry

### Edge Cases and Warnings

- **Stale field names:** If a saved mapping references a field that no longer exists on the SObject (e.g., a custom field was deleted), silently skip that mapping entry and do not apply it. Do not error.
- **SObject rename:** The key is `sobjectName.toLowerCase()`, so a renamed SObject will not match the old key. Old mappings become orphaned and are visible in the Options page for manual deletion.
- **Multiple import actions:** A mapping saved during an Update import (which includes `Id`) may be applied to a Create import where `Id` is not a valid column. Before applying a saved mapping entry, verify each target `columnValue` exists in the current `columnList()` for the current action.
- **Conflict with Feature B (Smart Column Match):** If both `FLAG_SAVED_MAPPINGS` and `FLAG_SMART_COLUMN_MATCH` are on, apply saved mappings first. Only fall through to label/synonym matching for columns not covered by the saved mapping.
- **User declines to apply:** The info banner's dismiss action should suppress re-application for the remainder of the current page session (use a model-level boolean), but not delete the saved mapping.

### Test Approach

1. **Save on success:** Complete a mock import. Assert that `localStorage.getItem("columnMappings_inspector_test__c")` is populated with the correct mapping JSON after the batch finishes.
2. **Auto-apply on next load:** Pre-populate `localStorage` with a saved mapping for `Inspector_Test__c`. Load the import page, paste matching CSV data. Assert that column values in the UI reflect the saved mapping without manual intervention. Assert the info banner is visible.
3. **Stale field ignored:** Pre-populate a saved mapping that includes a field `"Deleted_Field__c"` which does not appear in the mock describe response. Assert that the stale entry is silently skipped and does not cause an error or unknown-field state.
4. **Flag off = no mapping applied:** With `FLAG_SAVED_MAPPINGS = false` in localStorage, assert that a pre-populated saved mapping is not applied even when CSV headers match.
5. **Clear mapping:** Click the "Clear" link in the info banner. Assert `localStorage.getItem("columnMappings_inspector_test__c")` is `null` and columns revert to default resolution.
6. **Options page list:** Navigate to the Options page Data Import tab. Assert that saved mappings are listed with the correct SObject name and date, and that clicking Delete removes the localStorage entry.

---

## 6. Loading the Fork as an Unpacked Chrome Extension

This project requires no build step. The `addon/` directory is loaded directly as the extension.

### Initial Load

1. Open Chrome and navigate to `chrome://extensions`.
2. Enable **Developer mode** using the toggle in the top-right corner.
3. Click **Load unpacked**.
4. Select the `addon/` directory from your local clone:
   ```
   /path/to/Salesforce-Inspector-reloaded/addon
   ```
5. The extension appears in the list with a generated extension ID (e.g., `abcdefghijklmnopqrstuvwxyz123456`). Note this ID — it is stable for a given unpacked directory on a given Chrome profile.

### Reloading After Changes

After editing any file in `addon/`:

1. Return to `chrome://extensions`.
2. Click the circular **Reload** icon on the extension card.
3. Refresh any open extension pages (e.g., `chrome-extension://<id>/data-import.html`).

There is no watch/hot-reload mechanism. For rapid iteration on a specific page, keep a terminal open and bind a key to reload the extension via the `chrome://extensions` page, or use the [Extensions Reloader](https://chromewebstore.google.com/detail/extensions-reloader/fimgfedafeadlieiabdeeaodndnlbhid) Chrome extension.

### Verifying the Extension ID for Tests

The Playwright test suite (`playwright.config.js`) loads the extension automatically via:

```js
args: [
  `--disable-extensions-except=${path.join(process.cwd(), "addon")}`,
  `--load-extension=${path.join(process.cwd(), "addon")}`,
]
```

The extension ID used in tests is obtained at runtime in `tests/e2e/fixtures.js`. When running tests manually, the same `addon/` directory is loaded so the ID is consistent within a test run.

To run the test suite:

```bash
npx playwright test
# or for a specific file:
npx playwright test tests/e2e/data-import.spec.js
# with UI mode for debugging:
npx playwright test --ui
```

### Debugging Extension Pages with Chrome DevTools

**Background script (`addon/background.js`):**
Go to `chrome://extensions`, find the extension card, click the **Service worker** link. This opens a dedicated DevTools window for the background script.

**Extension HTML pages (data-import, data-export, options, etc.):**
Navigate directly to the page URL, e.g.:
```
chrome-extension://<extension-id>/data-import.html?host=yourorg.salesforce.com
```
Then open DevTools with `F12` or right-click -> Inspect. The Console, Sources, Network, and Application tabs all work as normal.

**Inspecting localStorage:**
In DevTools -> Application -> Storage -> Local Storage, select the `chrome-extension://<id>` origin. All feature flags, saved mappings, and other settings are visible and editable here. This is the fastest way to toggle a feature flag without going through the Options page during development.

**Network tab for API calls:**
REST calls to the Salesforce org appear in the Network tab. SOAP calls go to `/services/Soap/u/<version>` and their XML request/response bodies are visible in the Payload and Response tabs.

**Content scripts (`addon/inject.js`, `addon/inspect-inline.js`):**
These run in the context of Salesforce pages. To debug them, open DevTools on the Salesforce page itself and look for the content script in Sources -> Content Scripts. Console logs from content scripts appear in the Salesforce page's console.
