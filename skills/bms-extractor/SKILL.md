---
name: bms-extractor
description: Extracts levels and zones from a BMS (Building Management System) web interface using built-in browser tools (Navigate, Click, Take screenshot, Read page, Execute JavaScript). Produces a TSV file for Peak platform import and a JSON site model with screenshots for verification. Triggers for "extract levels from BMS", "get zones from BMS", "map BMS levels and zones", "BMS site structure extraction", or "extract BMS hierarchy for Peak import".
---

# BMS Extractor

Extracts **levels and zones** from a BMS web interface using built-in browser tools, producing a TSV ready for Peak platform import and a JSON site model with screenshot evidence.

---

## Workflow Overview

```
1. SETUP      → Collect BMS URL + site name, create output directory
2. AUTH       → Navigate to BMS, check if already logged in, handle login if needed
3. DISCOVER   → Identify nav tree pattern, screenshot the full nav tree
4. EXTRACT    → Walk levels and zones in the nav hierarchy, screenshot each
5. OUTPUT     → Write site_model.json, levels_and_zones.tsv, manifest.json
```

---

## Step 1: SETUP

Collect inputs and prepare the workspace.

```bash
SITE_NAME="example-site"  # kebab-case site name from user
OUTPUT_DIR="bms-extract/${SITE_NAME}"
mkdir -p "${OUTPUT_DIR}/screenshots"
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

## Step 3: DISCOVER — Identify Navigation Structure

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

## Step 4: EXTRACT — Walk Levels and Zones

For each level found:
1. **Navigate** to the level page URL
2. **Take a screenshot** and save it to the output directory
3. **Read the page** or **Execute JavaScript** to extract zone names
4. **Click** sub-navigation to explore zones if needed

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

## Step 5: OUTPUT — Write Files

### 5a. `site_model.json`

Write the complete JSON structure built during extraction.

### 5b. `levels_and_zones.tsv`

Generate the TSV from the JSON — two columns: `level` and `zones` (comma-separated zone list).

```bash
jq -r '
  ["level", "zones"],
  (.levels[] | [.name, ([.zones[].name] | join(", "))])
  | @tsv
' "${OUTPUT_DIR}/site_model.json" > "${OUTPUT_DIR}/levels_and_zones.tsv"
```

### 5c. `manifest.json`

Write extraction metadata (source URL, site name, platform, timestamps, counts).

### Final summary

Report to the user:
> **Extraction complete for [Site Name]**
> - Levels: N
> - Total zones: M
> - Output: `bms-extract/<site-name>/`
> - Files: `site_model.json`, `levels_and_zones.tsv`, `manifest.json`
> - Screenshots: `screenshots/` (K files)

---

## Troubleshooting

### Browser tab not visible
The browser doesn't always pop up automatically. Tell the user to look for the Chrome browser tab.

### Network blocked
If the BMS IP/domain is blocked, add it under settings → "Additional allowed domains".

### Tool errors (Click, Execute JavaScript, Take screenshot)
Always Navigate to a page first and wait for it to load before using other browser tools.

### BMS content in iframes
If Read page or Execute JavaScript returns limited content, check for iframes and navigate directly to the iframe URL.

### Can't identify levels vs zones
Ask the user to clarify. Take a screenshot and ask:
> "I can see these nodes in the navigation. Which ones are levels and which are zones?"
