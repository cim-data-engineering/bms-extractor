# Extracting BMS Points from Equipment Graphics

## Navigating to Equipment Graphics Pages

Each equipment in `equipment_list` has a `source_url` — the page where it was discovered during Part B. Use this as the starting point for navigation.

### Decision tree

1. **Navigate to the equipment's `source_url`**
2. **Determine page type:**
   - **Direct graphics page** — shows a single equipment schematic with live values → proceed directly to the extraction workflow below
   - **Summary/list page** — shows a table or list of multiple equipment → find the specific equipment entry and click through to its detail/graphics page
   - **No graphics available** — page has no drill-down to individual equipment graphics → log and skip

### Summary page drill-down patterns

When `source_url` loads a summary/list page, use these platform-specific patterns to navigate to the equipment graphic:

- **Niagara/Tridium** — equipment names are clickable links in the nav tree or summary table. Look for the equipment name as a tree node or table row and click it.
- **Alerton** — summary pages list equipment as card tiles or table rows. Click the equipment name or its associated "View" / "Detail" link.
- **Siemens Desigo CC** — equipment appears in a hierarchical tree. Expand nodes and click the equipment to load its graphic in the main pane.
- **Schneider EcoStruxure** — summary dashboards show equipment as interactive widgets or list items. Click through to the individual graphic view.

### Fallback: URL pattern construction

If clicking doesn't work, try constructing the graphics URL by appending the equipment name to the base path:
- `{base_url}/graphic/{equipment_name}`
- `{base_url}/view/{equipment_name}`
- `{base_url}/#/equipment/{equipment_name}`

### When to skip

Skip an equipment if:
- The `source_url` returns a 404 or login page
- No clickable link to an individual graphic can be found
- The graphics page is canvas-only with no readable DOM elements (and screenshot extraction yields nothing useful)

Log skipped equipment and continue with the next item.

---

## Workflow

Copy this checklist and track your progress:
```
BMS Point Extraction:
- [ ] Step 1: Identify equipment name and page structure
- [ ] Step 2: Detect rendering technology and binding convention
- [ ] Step 3: Bulk-extract all bound elements from the DOM
- [ ] Step 4: Extract static labels and match spatially to bound values
- [ ] Step 5: Parse values and units
- [ ] Step 6: Deduplicate and assemble output table
```

### Step 1: Identify equipment name and page structure

Take a screenshot. Determine the equipment name from the page heading, `<title>`, or last path segment of the URL. This populates `equipment_name` for every row.

Check for iframes — BMS platforms commonly render graphics inside one. If an iframe exists, all subsequent DOM queries must target `iframe.contentDocument`.

### Step 2: Detect rendering technology and binding convention

Run a single JavaScript pass to detect the platform and binding style:
```javascript
const doc = /* iframe.contentDocument or document */;
const probes = {
  niagara_title:  doc.querySelectorAll('[title]').length,
  data_bind:      doc.querySelectorAll('[data-bind]').length,
  data_point:     doc.querySelectorAll('[data-point]').length,
  data_ord:       doc.querySelectorAll('[data-ord]').length,
  ord:            doc.querySelectorAll('[ord]').length,
  aria_label:     doc.querySelectorAll('[aria-label]').length,
  data_ref:       doc.querySelectorAll('[data-ref]').length,
  svg_elements:   doc.querySelectorAll('svg [id]').length,
  canvas:         doc.querySelectorAll('canvas').length
};
```

Use the dominant non-zero result to choose the extraction strategy:

- **`title` attributes with `Name = Value {status}` pattern** → Niagara/Tridium
- **`data-bind`, `data-point`, `data-ref`** → Distech / KnockoutJS-based
- **`ord` or `data-ord`** → Niagara ORD-based
- **`aria-label`** → Schneider EcoStruxure or accessibility-annotated
- **SVG `id` attributes** → SVG-based schematic (Siemens Desigo, etc.)
- **Canvas only, no DOM bindings** → Screenshot fallback required

### Step 3: Bulk-extract all bound elements

In one JavaScript call, query all elements carrying binding metadata and extract:
```javascript
const selectors = '[title], [data-bind], [data-point], [data-ord], [ord], [aria-label], [data-ref]';
const elements = doc.querySelectorAll(selectors);
const results = Array.from(elements).filter(el => /* has binding pattern */).map(el => {
  const rect = el.getBoundingClientRect();
  return {
    binding: /* parse point name from the relevant attribute */,
    rawValue: el.textContent?.trim(),
    x: Math.round(rect.x),
    y: Math.round(rect.y),
    hasText: el.textContent?.trim().length > 0,
    tag: el.tagName,
    id: el.id
  };
});
```

Adapt the `binding` parse logic to the detected platform:

- **Niagara title**: split on `=`, take left side → point name
- **data-bind / data-point**: read attribute value directly
- **ord / data-ord**: extract last path segment as point name

**Do NOT hover** over individual points. Extract everything in this single bulk pass.

### Step 4: Extract static labels and match spatially

Query all non-bound visible text elements to find display labels:
```javascript
const labels = doc.querySelectorAll('div, span, td, text');
// Filter to: visible, has text, no binding attribute, text < 60 chars
// For each, capture: text, x, y
```

For each bound element from Step 3, find its display label by spatial proximity:

1. Same Y (±10px) and to the left
2. Directly above (within 30px) at similar X
3. Within the same parent container
4. If in a labelled panel/section (e.g. "S/A Pressure Control"), use the row label within that section

The matched label text becomes `graphic_point_name`. If no label matches confidently, use the underlying point name as a fallback.

### Step 5: Parse values and units

Apply this regex to each displayed text value:
```
/^([-+]?\d*\.?\d+)\s*(.*)$/
```

- Group 1 → `graphic_value` (numeric)
- Group 2 → `graphic_unit` (°C, Pa, %, %RH, Hz, s, etc.)

For non-numeric state values (On, Off, Stopped, Active, Clean, Disabled, AUTO, HAND, Open, Closed, etc.):
- Entire text → `graphic_value`
- `graphic_unit` → leave blank

### Step 6: Deduplicate and assemble output

- If the same `underlying_point_name` + `graphic_point_name` appears multiple times, include it once.
- For repeating grid rows (e.g. per-floor schedule), include one row and append the repetition pattern to `graphic_point_name` in parentheses, e.g. "Schedule (L6–L14)".
- Exclude site-wide/global points (current time, weather, logged-in user) unless positioned within the equipment graphic area.
- Sort alphabetically by `graphic_point_name`.

### Fallback: Screenshot extraction

If Step 2 finds canvas-only rendering or fewer than 5 DOM bindings, take a screenshot and visually read all point labels, values, and units. Additionally, check network requests (XHR/Fetch) for point subscription payloads as an alternative source of underlying point names.

## Output format

Output a single flat markdown table with these exact columns:
```
| equipment_name | graphic_point_name | graphic_value | graphic_unit | underlying_point_name |
```

Example rows:
```
| HR_PZN_AHU | S/A Duct Pressure       | 24    | Pa  | SA_StaticPressure        |
| HR_PZN_AHU | SAF Call                | On    |     | SAF_Call                 |
| HR_PZN_AHU | Supply Temp Setpoint    | 12.0  | °C  | SupplyTempSPT            |
| HR_PZN_AHU | Economy Dampers         | 0     | %   | Economy_Damper_ToScreen  |
| HR_PZN_AHU | Schedule (L6–L14)       | On    |     | Schedule                 |
| HR_PZN_AHU | Out Of Service          | No    |     | *(no DOM binding)*       |
```

Rules:
- `graphic_point_name` = exact human-readable label visible on screen next to the value
- `graphic_value` = the numeric value or state string displayed
- `graphic_unit` = unit of measure if shown; blank for state values
- `underlying_point_name` = the bound point/tag name from DOM attributes; use `*(no DOM binding)*` for controls (checkboxes, buttons) where no binding is found in the DOM

## Efficiency rules

- Never hover over individual points — extract everything via bulk JavaScript DOM queries.
- Combine extraction, label matching, and value parsing into as few JavaScript calls as possible (target 3–4 total).
- Use `javascript_tool` and `read_page` for data gathering. Reserve screenshots for visual verification and canvas fallback only.
