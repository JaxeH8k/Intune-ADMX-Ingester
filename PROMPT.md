# Build: ADMX → Intune Custom Ingestion (single-file web app)

Build a **single self-contained `index.html`** (inline CSS + vanilla JS, no build step,
no framework, no CDN, no network calls) that a user opens by double-clicking from
`file://`. It lets an admin import a Group Policy **ADMX** file plus its matching
language file (**ADML XML** *or* legacy **ADM** text), presents a modern
Group-Policy-Management-Editor–style UI, and generates the **Custom OMA-URI settings**
needed to create an Intune "ADMX Ingestion" custom device configuration profile.

Reference behavior (do not copy code, just match capability):
IntuneManagement's ADMXImport (github.com/Micke-K/IntuneManagement/blob/master/ADMXImport.md).

## Hard constraints
- One file, no server, no external requests. All assets inline. Must run offline from disk.
- Modern, clean UI (NOT a Windows MMC clone) — but keep the GPME mental model:
  Computer vs User configuration, a category tree, a settings list, a per-setting editor,
  and an explain-text pane.
- Pure client-side. Files are read via `<input type=file>` / drag-drop; nothing uploaded.

## File loading & encoding (critical)
- Read files as `ArrayBuffer`, then detect encoding before parsing:
  - BOM `FF FE` → UTF-16LE, `FE FF` → UTF-16BE, `EF BB BF` → UTF-8; otherwise sniff for
    interleaved NUL bytes (UTF-16) and fall back to UTF-8. Decode with `TextDecoder`.
  - (The common real-world ADMX/ADML files are UTF-16LE with a BOM — handle this or parsing breaks.)
- Keep the **raw decoded ADMX text** verbatim — it feeds the ingestion/trimming step later.

## Parse the ADMX (XML via DOMParser)
Extract:
- `policyNamespaces`: the `target` namespace + **prefix** (used to build OMA-URIs), and all
  `using` namespaces (for cross-namespace `parentCategory` refs like `Google:Cat_Google`).
- `categories` → build the full parent/child tree. `parentCategory ref` may be `name` or
  `prefix:name` (external/`windows:` refs may be unresolved — show them as roots gracefully).
- `policies/policy`, each with: `name`, `class` (`Machine` | `User` | `Both`), `displayName`,
  `explainText`, `key`, `valueName`, `parentCategory`, `supportedOn`, and its child
  `enabledValue`/`disabledValue`, `enabledList`/`disabledList`, and `elements`.
- `elements` types to support: `text`, `decimal`/`longDecimal`, `boolean`, `enum` (with
  `item`/`value`), `list` (with `valuePrefix`, `explicitValue`, `additive`), `multiText`.
  Capture `id`, `valueName`, `required`, `minValue`/`maxValue`, `maxLength`.

## Resolve display strings & presentation
Support BOTH language file formats:
- **ADML (XML):** resolve `$(string.ID)` from `//stringTable/string[@id]` and
  `$(presentation.ID)` from `//presentationTable/presentation[@id]` (decimalTextBox,
  textBox, checkBox, dropdownList, listBox, text, etc. — use these for input labels/order).
- **Legacy ADM (text):** parse the token grammar (`CLASS`, `CATEGORY`, `POLICY`, `PART`,
  `VALUENAME`, `ITEMLIST`, `MIN`/`MAX`, `VALUEON`/`VALUEOFF`) and the trailing `[strings]`
  section to resolve `!!ID` → text. If only an ADM is provided, derive display names from it.
- Anything unresolved: fall back to the raw ID string rather than failing.

## UI (modern, functional)
- Two-pane layout: left = tree (top split into **Computer Configuration** (Machine/Both) and
  **User Configuration** (User/Both), then the category hierarchy); right = settings list for
  the selected category with a state badge (Not configured / Enabled / Disabled).
- Selecting a setting opens an editor with three states (Not configured / Enabled / Disabled)
  and, when Enabled, the element inputs rendered per presentation order with validation
  (numeric min/max, required, enum dropdowns, list key/value grids, textbox maxLength).
- Explain-text pane shows the resolved `explainText`.
- Global search/filter across policy display names; a "Configured settings (N)" view.
- Let the user set the **App/namespace label** used in OMA-URIs (default: the ADMX target
  prefix, e.g. `update`) and a profile name.

## Minimize the ingested ADMX (trim to referenced settings)

Instead of ingesting the entire ADMX, build a **reduced ADMX** containing only what the
configured settings actually reference. This keeps the ingestion OMA-URI value small.
Provide a toggle **"Minimal ADMX (recommended)" / "Full ADMX"** (default: Minimal) and show a
byte-count readout (full vs trimmed) so the size saving is visible.

Given `S` = the set of policies the user configured, construct a new `<policyDefinitions>`:

1. **Root:** copy the root element's attributes (`revision`, `schemaVersion`) and keep the
   `<resources minRequiredRevision="…"/>` element. Drop `<supersededAdm>`.
2. **Policies:** include only the policies in `S` (their child `<elements>`,
   `<enabledValue>`/`<disabledValue>`, enum items, etc. come along as children automatically).
3. **Categories — transitive closure (required):** for each policy in `S`, walk its
   `parentCategory ref` chain upward and include every **local** category referenced, deduped.
   Stop when a ref is external (prefixed, e.g. `Google:Cat_Google`, `windows:…`) — leave that
   ref intact but do NOT try to include the external category (it lives in another namespace).
   This closure is mandatory: Windows validates the `…~Policy~{CategoryChain}` area path in
   each Config OMA-URI against categories present in the ingested ADMX.
4. **supportedOn:** keep a `<supportedOn><definitions>` block containing **only** the
   `<definition>` elements whose `name` is referenced by a retained policy's
   `<supportedOn ref="…"/>`. This is usually the biggest size win, since the full block can hold
   hundreds of unused definitions. If a policy's `supportedOn ref` is external/unresolved,
   either keep the ref (its definition lives elsewhere) or omit the `<supportedOn>` child.
5. **Namespaces:** always keep `<target>`. Keep each `<using>` whose prefix is still referenced
   by a retained cross-namespace `parentCategory`/`supportedOn` ref; drop the rest.
   (Keeping all namespace declarations is also acceptable — they're tiny.)
6. Serialize with `XMLSerializer`. **Leave `$(string.*)` / `$(presentation.*)` references as-is**
   — enforcement doesn't require the ADML, and unresolved string refs don't block ingestion.
   Output must remain well-formed and schema-valid.

Edge handling:
- Dedupe categories/definitions shared across multiple configured policies.
- If `S` is empty, no ingestion row is emitted.
- If trimming would produce an invalid/empty tree, fall back to Full ADMX and warn inline.

## Output: Custom OMA-URI settings (the deliverable)
Produce a copyable list the admin pastes into an Intune **Custom** profile. Two parts:

1. **Ingestion row (once):**
   - OMA-URI: `./Device/Vendor/MSFT/Policy/ConfigOperations/ADMXInstall/{AppName}/Policy/{AppName}`
   - Data type: **String**
   - Value: the **reduced ADMX text** produced by the trimming step (or the raw full ADMX when
     "Full ADMX" is selected).

2. **One row per configured setting:**
   - OMA-URI: `./{Device|User}/Vendor/MSFT/Policy/Config/{AppName}~Policy~{CategoryPath}/{PolicyName}`
     - `{CategoryPath}` = the policy's parentCategory chain joined top-down with `~`
       (category `name`s, not display names).
     - Use `./User/...` for `class=User`, `./Device/...` for `Machine`; for `Both`, default to
       Device and let the user toggle.
   - Data type: **String**
   - Value XML, built from the editor state:
     - Disabled → `<disabled/>`
     - Enabled, no elements → `<enabled/>`
     - Enabled with elements → `<enabled/>` followed by, per element:
       - text/decimal/enum → `<data id="{elementId}" value="{value}"/>`
       - enum → emit the selected item's `value` (decimal or string) per ADMX
       - list → `<data id="{elementId}" value="{name1}&#xF000;{v1}&#xF000;{name2}&#xF000;{v2}..."/>`
         The payload is ALWAYS alternating name/value pairs separated by `&#xF000;` (U+F000).
         For `explicitValue` lists the admin supplies both name and value per row. For
         non-`explicitValue` lists the value name is `{valuePrefix}{index}` (index starting at 1)
         and the value is the admin's entry — e.g. `1&#xF000;dataA&#xF000;2&#xF000;dataB`.
       - multiText → each string separated by `&#xF000;` (U+F000), NOT a newline.
     - XML-escape all values; present the value ready to paste.

Provide per-row **Copy** buttons, a **Copy all** action, and an **Export** (JSON + CSV table
of `OMA-URI, DataType, Value`) so it can be scripted later.

## Validation target
Must correctly load the provided `assets/admx/GoogleUpdate.admx` + `assets/adml/GoogleUpdate.adm`
(UTF-16LE), render the Google Update category tree, let me enable e.g.
"Auto-update check period" with a numeric value, and emit a correct
`./Device/Vendor/MSFT/Policy/Config/update~Policy~Cat_GoogleUpdate~Cat_Preferences/Pol_AutoUpdateCheckPeriod`
row with `<enabled/><data id="Part_AutoUpdateCheckPeriod" value="..."/>`.
With only that one policy configured, the ingested (Minimal) ADMX must contain just that
`<policy>`, its ancestor category chain, and only the `supportedOn` definitions it references —
and must still be well-formed / ingest without schema errors.

## Quality
- Handle malformed/partial files with clear inline errors, never a blank page.
- Comment the ADMX/ADM/ADML parsers. Keep functions small and readable.
