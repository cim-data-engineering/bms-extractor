---
name: bms-extractor
description: Extracts HVAC levels and zones from a BMS (Building Management System) web interface using built-in browser tools (Navigate, Click, Take screenshot, Read page, Execute JavaScript). Produces a CSV file for Peak platform import and a JSON site model with page source for verification. Triggers for "extract levels from BMS", "get zones from BMS", "map BMS levels and zones", "BMS site structure extraction", or "extract BMS hierarchy for Peak import".
---

# BMS Extractor

Extracts **HVAC levels and zones** from a BMS web interface using built-in browser tools, producing a CSV ready for Peak platform import and a JSON site model with page source evidence.

**Focus on HVAC only** — skip lighting, security, fire, vertical transport, and other non-HVAC systems.

---

## Workflow Overview

```
1. SETUP      → Collect BMS URL + site name, create output directory
2. AUTH       → Navigate to BMS, check if already logged in, handle login if needed
3. SCOPE      → Show user what's available, ask what to extract
4. DISCOVER   → Identify nav tree pattern for selected scope
5. EXTRACT    → Walk levels and zones in the nav hierarchy, screenshot each
6. OUTPUT     → Write site_model.json, levels_and_zones.csv, manifest.json
```

---

## Step 1: SETUP

Collect inputs and prepare the workspace.

```bash
SITE_NAME="example-site"  # kebab-case site name from user
OUTPUT_DIR="bms-extract/${SITE_NAME}"
mkdir -p "${OUTPUT_DIR}/pages"
```

Ask the user for:
1. **BMS URL** — the web interface URL (e.g. `https://bms.example.com`)
2. **Site name** — human-readable name (e.g. "99 Elizabeth St")

---

## Step 2: AUTH — Authentication

**CRITICAL: Never ask the user to type credentials into the chat.**

### Check authentication status first

**Do not assume the user needs to log in.** The browser may already have an active session. Always check first:

1. **Navigate** to the BMS URL
2. **Take a screenshot** of the page
3. **Read the screenshot** — if it shows the BMS dashboard/home page, the user is already authenticated → skip to Step 3
4. Only if it shows a login page, proceed with the login flow below

### Login flow

If login is needed, tell the user:

> "The BMS login page is showing. Please log in using the Chrome browser — you can find it in your browser tabs. Let me know once you're on the dashboard."

Wait for the user to confirm, then take another screenshot to verify.

**Note:** The browser tab doesn't always pop up automatically — the user may need to find it themselves.

**CRITICAL: Never write credentials to output files.**

---

## Browser Tab Setup

**CRITICAL:** Before attempting screenshots or JavaScript execution on a BMS page, always ensure you have a working tab.

### Correct sequence:
1. Call `tabs_context_mcp(createIfEmpty: true)` to initialize the tab group
2. Call `tabs_create_mcp` to create a **new, clean tab** — use this tab ID for all screenshot and interaction work
3. Navigate to the BMS URL on the new tab
4. Wait 5 seconds for the page to fully render
5. Take screenshot — this should now work reliably

### If screenshots fail on a tab:
- Create another new tab with `tabs_create_mcp` and retry
- The old tab can still be used for `read_page` and `navigate` (DOM-based tools), just not for screenshots or JavaScript execution

### Use the working screenshot tab for:
- Taking screenshots of each level/zone page
- Capturing floor plan graphics for verification
- Any page where content is rendered as canvas/graphics (not in the DOM)

---

## Step 3: SCOPE — Confirm Extraction Scope

After authentication, **do not start extracting immediately**. First discover the navigation structure, then present what's available and ask the user what they want:

1. Take a screenshot of the dashboard/nav tree
2. Identify the levels available (e.g. "I can see Roof, Levels 15–1, Ground, and B1 — 18 levels total")
3. Identify HVAC-related navigation (Mechanical, HVAC, Floor Layouts, etc.) — ignore Lighting, Security, Fire, Vertical Transport
4. Ask the user:

> "What would you like to extract?"
> - **All levels and zones** — full building hierarchy
> - **Specific levels only** — e.g. "just Levels 1–5" or "only Ground and B1"
> - **A single level** — e.g. "Level 3 only, with all its zones"

Wait for the user to confirm the scope before proceeding to Step 4.

---

## Step 4: DISCOVER — Identify Navigation Structure (for selected scope)

Use Navigate, Take screenshot, Read page, Click, and Execute JavaScript to explore the BMS interface:

1. **Take a screenshot** of the dashboard to identify the nav pattern
2. **Read the page** to get link text, button labels, and navigation structure
3. **Execute JavaScript** to extract DOM elements, especially from iframes:
   ```javascript
   document.querySelectorAll('a, input[type="button"], [onclick]')
   ```
4. **Click** navigation elements to explore the hierarchy

**Iframe handling:** Many BMS platforms load content in iframes. If Read page/Execute JavaScript returns limited content, try navigating directly to iframe URLs.

### What to identify

- **Nav pattern** — sidebar tree, top tabs, button grid, or breadcrumb-based
- **Level/zone structure** — how levels and zones appear in the hierarchy
- **Direct URLs** — whether each level/zone has a navigable URL

---

## Step 5: EXTRACT — Walk Levels, Equipment, and Zones

Extraction happens in two passes: first extract levels and equipment, then derive zones.

### Pass 1: Levels and Equipment

For each level in scope:
1. **Navigate** to the level's HVAC floor plan / graphics page
2. **Take a screenshot** to visually inspect the layout
3. **Save page source** via Execute JavaScript and write to `${OUTPUT_DIR}/pages/level-<slug>.html`:
   ```javascript
   document.documentElement.outerHTML
   ```
4. **Extract equipment names and types** from the page — look for:
   - VAVs (Variable Air Volume boxes)
   - FCUs (Fan Coil Units)
   - AHUs (Air Handling Units)
   - Other HVAC equipment labels on the graphic
5. Record each equipment item with its name, type, and level

### Pass 2: Derive Zones

After extracting equipment, determine zone names using this priority:

1. **Explicit zone names on graphics** (best case) — some floor plans label zones directly (e.g. room names, area names next to equipment). If the graphic shows VAVs with associated room/zone labels, use those.

2. **Equipment naming convention** (common case) — when zone names aren't explicitly shown, the equipment names themselves often encode zone info. Decode the naming pattern, e.g.:
   - `VAV-L03-INT1` → Level 3, Internal Zone 1
   - `VAV-L03-NE` → Level 3, North East perimeter
   - `VAV-L03-SW` → Level 3, South West perimeter
   - `FCU-L02-BOARDROOM` → Level 2, Boardroom

   Ask the user to confirm the naming convention if it's ambiguous.

3. **Zone summary pages** (fallback) — some BMS platforms have zone summary tables, but these often show tenancy names rather than HVAC zones. Use as a supplement, not primary source.

### Saving page source

For every graphics page visited, **always save the page source**:
- **Page source** (HTML) — preserves the full DOM for later analysis, point extraction, or re-processing

```bash
cat > "${OUTPUT_DIR}/pages/level-1.html" << 'HTMLEOF'
... page source from Execute JavaScript ...
HTMLEOF
```

**Note on screenshots:** In CoWork, browser screenshots cannot be saved directly to the filesystem — the Chrome extension is sandboxed. Use Take screenshot to visually inspect pages during extraction, but rely on saved page source (HTML) as the persistent artifact. If screenshots are needed as files, tell the user to save them manually from the open Chrome tabs.

### Build the structure as you go

Maintain a running JSON structure:

```json
{
  "site": "99 Elizabeth St",
  "bms_platform": "detected platform",
  "extracted_at": "2026-03-16T14:30:00+11:00",
  "levels": [
    {
      "name": "Level 3",
      "page_source": "pages/level-3.html",
      "equipment": [
        { "name": "VAV-L03-INT1", "type": "VAV" },
        { "name": "VAV-L03-NE", "type": "VAV" },
        { "name": "VAV-L03-SW", "type": "VAV" },
        { "name": "FCU-L03-SERVER", "type": "FCU" }
      ],
      "zones": [
        { "name": "Internal Zone 1", "equipment": ["VAV-L03-INT1"] },
        { "name": "North East Perimeter", "equipment": ["VAV-L03-NE"] },
        { "name": "South West Perimeter", "equipment": ["VAV-L03-SW"] },
        { "name": "Server Room", "equipment": ["FCU-L03-SERVER"] }
      ]
    }
  ]
}
```

### Progress reporting

After each level, report progress:
> "Extracted Level 3: found 4 VAVs, 1 FCU → derived 4 zones (Internal Zone 1, NE Perimeter, SW Perimeter, Server Room). Moving to Level 4..."

---

## Step 6: OUTPUT — Write Files

### 6a. `site_model.json`

Write the complete JSON structure built during extraction.

### 6b. `levels_and_zones.csv`

Generate the CSV from the JSON — columns: `level`, `zone`, `equipment_type`, `equipment_names`.

```bash
jq -r '
  ["level", "zone", "equipment_type", "equipment_names"],
  (.levels[] as $l | $l.zones[] |
    [$l.name, .name,
     ([.equipment[] as $e | ($l.equipment[] | select(.name == $e) | .type)] | unique | join("/")),
     (.equipment | join(", "))]
  ) | @csv
' "${OUTPUT_DIR}/site_model.json" > "${OUTPUT_DIR}/levels_and_zones.csv"
```

### 6c. `manifest.json`

Write extraction metadata (source URL, site name, platform, timestamps, counts).

### Final summary

Report to the user:
> **Extraction complete for [Site Name]**
> - Levels: N
> - Total zones: M
> - Output: `bms-extract/<site-name>/`
> - Files: `site_model.json`, `levels_and_zones.csv`, `manifest.json`
> - Page sources: `pages/` (M files)

---

## Troubleshooting

### Browser tab not visible
The browser doesn't always pop up automatically. Tell the user to look for the Chrome browser tab.

### Screenshot/JS errors ("Cannot access chrome-extension:// URL")
Create a fresh tab with `tabs_create_mcp` and use that tab ID. Do not reuse the default tab for screenshots or JavaScript execution.

### Network blocked
If the BMS IP/domain is blocked, add it under settings → "Additional allowed domains".

### Tool errors (Click, Execute JavaScript, Take screenshot)
Always Navigate to a page first and wait for it to load before using other browser tools. If errors persist, create a new tab and retry.

### BMS content in iframes
If Read page or Execute JavaScript returns limited content, check for iframes and navigate directly to the iframe URL.

### Can't identify levels vs zones
Ask the user to clarify. Take a screenshot and ask:
> "I can see these nodes in the navigation. Which ones are levels and which are zones?"
