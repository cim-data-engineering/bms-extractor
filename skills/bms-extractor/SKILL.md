---
name: bms-extractor
description: Extracts levels and zones from a BMS (Building Management System) web interface using built-in browser tools (Navigate, Click, Take screenshot, Read page, Execute JavaScript). Produces a TSV file for Peak platform import and a JSON site model with screenshots for verification. Triggers for "extract levels from BMS", "get zones from BMS", "map BMS levels and zones", "BMS site structure extraction", or "extract BMS hierarchy for Peak import".
---

# BMS Extractor

Extracts **levels and zones** from a BMS web interface using built-in browser tools, producing a TSV ready for Peak platform import and a JSON site model with screenshot evidence.

This skill focuses on the spatial hierarchy only — the first step in onboarding a new site.

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
# Create output directory
SITE_NAME="example-site"  # kebab-case site name from user
OUTPUT_DIR="bms-extract/${SITE_NAME}"
mkdir -p "${OUTPUT_DIR}/screenshots"
```

Ask the user for:
1. **BMS URL** — the web interface URL (e.g. `https://bms.example.com`)
2. **Site name** — human-readable name (e.g. "99 Elizabeth St")

---

## Step 2: AUTH — Authentication

**CRITICAL: Never ask the user to type credentials into the chat.** Credentials typed in chat are visible in the conversation history.

### Check authentication status first

**Do not assume the user needs to log in.** The browser may already have an active session (e.g. from a previous conversation or persistent cookies). Always check first:

1. **Navigate** to the BMS URL
2. **Take a screenshot** of the page
3. **Read the screenshot** — if it shows the BMS dashboard/home page, the user is already authenticated → skip to Step 3
4. Only if it shows a login page, proceed with the login flow below

### Login flow

If login is needed, tell the user:

> "The BMS login page is showing. Please log in using the Chrome browser — you can find it in your browser tabs. Let me know once you're on the dashboard."

Then wait for the user to confirm they've logged in. Take another screenshot to verify.

**Note:** The browser tab doesn't always pop up automatically — the user may need to find it themselves.

**CRITICAL: Never write credentials to output files (site_model.json, manifest.json, TSV).**

---

## Step 3: DISCOVER — Identify Navigation Structure

Use the Navigate, Take screenshot, Read page, Click, and Execute JavaScript tools to explore the BMS interface:

1. **Take a screenshot** of the dashboard to identify the BMS platform and nav pattern
2. **Read the page** to get link text, button labels, and navigation structure
3. **Execute JavaScript** to extract DOM elements, especially from iframes:
   ```javascript
   // Get all clickable elements
   document.querySelectorAll('a, input[type="button"], [onclick]')
   ```
4. **Click** navigation elements to explore the hierarchy

**Iframe handling:** Many BMS platforms load content in iframes. If the main page shows navigation buttons but Read page/Execute JavaScript returns limited content, try navigating directly to iframe URLs. For Allerton/Optergy, replace `page?d=` with `page?dds=` in the URL.

### What to identify

- **BMS platform** (Niagara N4, Siemens Desigo, Schneider EBO, Metasys, Honeywell EBI, Allerton/Optergy, or unknown)
- **Nav pattern** — sidebar tree, top tabs, button grid, or breadcrumb-based
- **Iframe usage** — many BMS platforms render content in iframes; you may need to navigate to the iframe URL directly (e.g. `page?dds=...` for Allerton/Optergy)
- **Level/zone structure** — how levels and zones appear in the hierarchy

### Platform-Specific Navigation Patterns

#### Niagara N4 (Tridium / Fox)
- **Nav:** Left-side tree panel — hierarchical: station > network > controller > points
- **Levels:** Usually under a "Building" or site-name node, then floor/level nodes
- **Zones:** Sub-nodes under each level (e.g. "North Zone", "Zone A", or room names)
- **Tips:** Tree can be very deep — expand cautiously. `ord` routing in URLs tracks position.

#### Siemens Desigo CC
- **Nav:** Left panel with Plant View / Logical View / Management View tabs
- **Levels:** Plant View mirrors physical location — Building > Floor > Room
- **Zones:** Appear as room or area nodes under each floor
- **Tips:** Use Plant View (not Logical View) for spatial hierarchy

#### Schneider EcoStruxure Building Operation (EBO)
- **Nav:** Left panel with server tree: Network > Controllers > Programs > Points
- **Levels:** Look for location-based grouping under the site node
- **Zones:** May be grouped by controller assignment rather than spatial layout
- **Tips:** List views (not graphics) are most structured

#### Johnson Controls Metasys
- **Nav:** Left panel: Network > Site > Equipment > Points. "Spaces" view organises by location.
- **Levels:** "Spaces" view has Floor nodes
- **Zones:** Zones/rooms appear under each floor in Spaces view
- **Tips:** Prefer "Spaces" view over "Equipment" view for spatial hierarchy

#### Honeywell EBI / SCADA Web
- **Nav:** Toolbar-based with system tree on left
- **Levels:** Under site node in system tree
- **Zones:** May use "Point Group Display" (PGD) for zone grouping
- **Tips:** Older EBI uses ActiveX — may not work. Flag to user if so.

#### Allerton / Optergy
- **Nav:** Button grid on main page; content loads in iframe (`page?d=...`)
- **Levels:** Look for "Floor Layouts" button — contains level buttons (Roof, Level 15..1, Ground, B1)
- **Zones:** Floor plan graphics per level show equipment zones (VAVs, FCUs). Also check "Zone Summary" pages if present — but these may only show tenancy names, not equipment zones.
- **Tips:** Navigate to iframe URLs directly using `page?dds=...` pattern. Content is positioned absolutely — use bounding-box coordinates to map text to table rows.

#### Generic / Unknown BMS
1. Identify the nav pattern — sidebar tree, top tabs, or breadcrumb-based
2. Look for spatial/location hierarchy nodes (building, floor, level, wing, zone, area, room)
3. Check URL scheme for hierarchy clues (`/building/level-1/zone-a`)
4. Screenshot each level of navigation as you explore

---

## Step 4: EXTRACT — Walk Levels and Zones

Systematically navigate the hierarchy, extracting level and zone names with a screenshot at each step.

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
  "bms_platform": "Allerton/Optergy",
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
# Generate TSV from site_model.json
jq -r '
  ["level", "zones"],
  (.levels[] | [.name, ([.zones[].name] | join(", "))])
  | @tsv
' "${OUTPUT_DIR}/site_model.json" > "${OUTPUT_DIR}/levels_and_zones.tsv"
```

Expected output format:
```
level	zones
Level 1	Zone A, Zone B
Level 2	North Wing
Level 3	Lobby, North Wing, South Wing, Plant Room
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

## BMS Platform Notes

See `references/bms-ui-patterns.md` for nav/extraction guidance on Niagara N4, Siemens Desigo CC,
Schneider EBO, JCI Metasys, and Honeywell EBI — including where each platform exposes its level/zone
hierarchy and quirks like iframes and lazy-loaded trees.

---

## Troubleshooting

### Browser tab not visible
The browser doesn't always pop up automatically. Tell the user to look for the Chrome browser tab in their interface.

### Network blocked
If the BMS IP/domain is blocked by a network allowlist, add it under settings → "Additional allowed domains".

### Tool errors (Click, Execute JavaScript, Take screenshot)
These browser tools may fail if the browser hasn't navigated to a page yet, or if the page hasn't loaded. Always Navigate first, wait for the page to load, then use other tools.

### Session expired
Navigate to the BMS URL — the browser may prompt for login again.

### BMS content in iframes
Many BMS platforms (Allerton/Optergy, some Niagara setups) load page content in iframes. If Read page or Execute JavaScript returns limited content:
1. Check for iframes in the page source
2. Navigate directly to the iframe URL (e.g. replace `page?d=` with `page?dds=` for Allerton/Optergy)

### Can't identify levels vs zones
Ask the user to clarify the hierarchy. Take a screenshot of the nav tree and ask:
> "I can see these nodes in the navigation. Which ones are levels and which are zones?"
