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

## Step 5: EXTRACT — Walk Levels and Zones

For each level found:
1. **Navigate** to the level page URL
2. **Take a screenshot** and save to `${OUTPUT_DIR}/screenshots/level-<slug>.png`
3. **Save page source** via Execute JavaScript and save to `${OUTPUT_DIR}/pages/level-<slug>.html`:
   ```javascript
   document.documentElement.outerHTML
   ```
4. **Read the page** or **Execute JavaScript** to extract zone names
5. **Click** sub-navigation to explore zones if needed

For each zone found:
1. **Navigate** to the zone page (if it has its own page/graphic)
2. **Take a screenshot** and save to `${OUTPUT_DIR}/screenshots/level-<slug>-zone-<slug>.png`
3. **Save page source** to `${OUTPUT_DIR}/pages/level-<slug>-zone-<slug>.html`

### Saving page source

For every graphics page visited during extraction, **always save the page source**:
- **Page source** (HTML) — preserves the full DOM for later analysis, point extraction, or re-processing. Extract via Execute JavaScript (`document.documentElement.outerHTML`) and write to file.

```bash
# Save page source captured via Execute JavaScript
cat > "${OUTPUT_DIR}/pages/level-1.html" << 'HTMLEOF'
... page source from Execute JavaScript ...
HTMLEOF
```

**Note on screenshots:** In CoWork, browser screenshots cannot be saved directly to the filesystem — the Chrome extension is sandboxed. Use Take screenshot to visually inspect pages during extraction, but rely on saved page source (HTML) as the persistent artifact. If screenshots are needed as files, tell the user to save them manually from the open Chrome tabs.

### Identifying zones

Zones are often **equipment names** visible on floor plan graphics (VAVs, FCUs, AHUs) rather than room/tenancy names. When walking each level:
- Screenshot the floor plan graphic — equipment labels on the plan are the zone names
- Check for sub-navigation (child nodes, tabs) listing zones
- Zone summary pages (if they exist) may show tenancy names but miss equipment zones — always check the floor plan too

### Build the structure as you go

Maintain a running JSON structure:

```json
{
  "site": "99 Elizabeth St",
  "bms_platform": "detected platform",
  "extracted_at": "2026-03-16T14:30:00+11:00",
  "levels": [
    {
      "name": "Level 1",
      "screenshot": "screenshots/level-1.png",
      "page_source": "pages/level-1.html",
      "zones": [
        { "name": "Zone A", "screenshot": "screenshots/level-1-zone-a.png" },
        { "name": "Zone B", "screenshot": "screenshots/level-1-zone-b.png" }
      ]
    }
  ]
}
```

### Progress reporting

After each level, report progress:
> "Extracted Level 3: found 4 zones (Lobby, North Wing, South Wing, Plant Room). Moving to Level 4..."

---

## Step 6: OUTPUT — Write Files

### 6a. `site_model.json`

Write the complete JSON structure built during extraction.

### 6b. `levels_and_zones.csv`

Generate the CSV from the JSON — two columns: `level` and `zones`.

```bash
jq -r '
  ["level", "zones"],
  (.levels[] | [.name, ([.zones[].name] | join(", "))])
  | @csv
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
