---
name: bms-extractor
description: Extracts levels and zones from a BMS (Building Management System) web interface using Playwright. Produces a TSV file for Peak platform import and a JSON site model with screenshots for verification. Handles authentication via Playwright browser state persistence. Works in CoWork without the --chrome flag. Triggers for "extract levels from BMS", "get zones from BMS", "map BMS levels and zones", "BMS site structure extraction", or "extract BMS hierarchy for Peak import".
---

# BMS Extractor

Extracts **levels and zones** from a BMS web interface using Playwright, producing a TSV ready for Peak platform import and a JSON site model with screenshot evidence.

Unlike `bms-web-extractor` (which requires `--chrome` and extracts full point data), this skill focuses on the spatial hierarchy only — the first step in onboarding a new site.

## Prerequisites

- Playwright installed: `npx playwright install chromium`
- BMS URL accessible (VPN connected if required)
- Site name and BMS login credentials ready

---

## Workflow Overview

```
1. SETUP      → Collect BMS URL + site name, launch Playwright, check for saved session
2. AUTH       → Navigate to BMS, handle login (form-fill or manual), save session state
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

Check for saved Playwright session state:

```bash
# Check for existing session
SESSION_FILE="${OUTPUT_DIR}/playwright-state.json"
if [ -f "$SESSION_FILE" ]; then
  echo "Found saved session — will attempt to reuse"
fi
```

---

## Step 2: AUTH — Authentication

**CRITICAL: Never ask the user to type credentials into the chat.** Credentials typed in chat are visible in the conversation history. Instead, open a headed browser for the user to log in directly.

### First run (no saved session)

Check whether a saved session exists:

```bash
SESSION_FILE="${OUTPUT_DIR}/playwright-state.json"
if [ -f "$SESSION_FILE" ]; then
  echo "Found saved session — skipping login"
fi
```

If no session exists, open a headed browser for the user to log in manually. Tell the user what's about to happen, then run the command:

> "I'm opening a browser window to the BMS. Please log in, then **close the browser window** when you're done — your session will be saved automatically."

```bash
npx playwright open --save-storage="${SESSION_FILE}" "${BMS_URL}"
```

This command blocks until the user closes the browser window, then returns.

### Verify the session

After the user confirms, take a headless screenshot using the saved session to verify it worked:

```bash
npx playwright screenshot \
  --load-storage="${SESSION_FILE}" \
  "${BMS_URL}" \
  "${OUTPUT_DIR}/screenshots/post-login.png"
```

Read the screenshot. If it shows the login page again, the session wasn't saved — ask the user to retry. If it shows the BMS dashboard, proceed to Step 3.

### Subsequent runs

If a session file already exists, go straight to the verification screenshot. If the session has expired (screenshot shows login page), delete the state file and repeat the manual login flow:

```bash
rm "${SESSION_FILE}"
```

**CRITICAL: Never write credentials to output files (site_model.json, manifest.json, TSV).**

---

## Step 3: DISCOVER — Identify Navigation Structure

The `npx playwright screenshot` CLI is useful for simple page captures, but BMS interfaces typically require **clicking nav elements, reading DOM text, and walking iframe-based hierarchies**. For this, you need programmatic Playwright scripts.

### Setting up programmatic Playwright

Write Node.js scripts to the **output directory** (not the repo root) to avoid polluting the project:

```bash
# Install playwright in the output directory
cd "${OUTPUT_DIR}"
npm init -y 2>/dev/null
npm install playwright
npx playwright install chromium
cd -  # return to original directory
```

**IMPORTANT:** All temporary scripts, `package.json`, and `node_modules` must go in the output directory (`bms-extract/<site>/`), not the repo root. Clean up scripts after extraction is complete.

### Discovery script pattern

Write a `.cjs` script to the output directory and run it from there so `require('playwright')` resolves:

```javascript
// ${OUTPUT_DIR}/discover.cjs
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({
    storageState: 'playwright-state.json',
    viewport: { width: 1920, height: 1080 }
  });
  const page = await context.newPage();

  // IMPORTANT: Use 'domcontentloaded' not 'networkidle' — BMS pages with
  // live data polling often never reach networkidle
  await page.goto('${BMS_URL}', { waitUntil: 'domcontentloaded', timeout: 30000 });
  await page.waitForTimeout(5000);  // allow async rendering

  await page.screenshot({ path: 'screenshots/dashboard-hires.png', fullPage: true });

  // BMS platforms often use iframes for content — check all frames
  for (const frame of page.frames()) {
    console.log('Frame:', frame.url());
    try {
      const elements = await frame.evaluate(() => {
        const els = document.querySelectorAll('a, input[type="button"], [onclick]');
        return Array.from(els).map(el => ({
          text: (el.textContent || el.value || el.title || '').trim(),
          href: el.href || el.getAttribute('onclick') || '',
          tag: el.tagName
        })).filter(e => e.text);
      });
      for (const el of elements) {
        console.log(`  [${el.tag}] "${el.text}" → ${el.href}`);
      }
    } catch(e) {}
  }

  await browser.close();
})().catch(e => console.error('FATAL:', e));
```

```bash
cd "${OUTPUT_DIR}" && node discover.cjs 2>&1 && cd -
```

### What to identify

Read the screenshot and script output to determine:
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

### Scripted extraction

For BMS interfaces that require clicking/DOM reading, write extraction scripts following this pattern:

```javascript
// ${OUTPUT_DIR}/extract.cjs
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({
    storageState: 'playwright-state.json',
    viewport: { width: 1920, height: 1080 }
  });
  const page = await context.newPage();

  const levels = [
    // Populate from discovery step
    { name: 'Level 1', url: 'http://bms.example.com/page?dds=...' },
    // ...
  ];

  for (const level of levels) {
    await page.goto(level.url, { waitUntil: 'domcontentloaded', timeout: 30000 });
    await page.waitForTimeout(3000);

    const slug = level.name.toLowerCase().replace(/\s+/g, '-');
    await page.screenshot({ path: `screenshots/level-${slug}.png`, fullPage: true });
    console.log(`Screenshot: ${level.name}`);

    // Extract zone names from the page DOM (platform-specific)
    // Check all frames if the BMS uses iframes
    for (const frame of page.frames()) {
      try {
        const zones = await frame.evaluate(() => {
          // Adapt selectors to the specific BMS platform
          const els = document.querySelectorAll('.zone-name, td.zone, [data-zone]');
          return Array.from(els).map(el => el.textContent.trim()).filter(Boolean);
        });
        if (zones.length) console.log(`  Zones: ${zones.join(', ')}`);
      } catch(e) {}
    }
  }

  await browser.close();
})().catch(e => console.error('FATAL:', e));
```

### For simple URL-navigable levels

If each level has a direct URL (no clicking needed), `npx playwright screenshot` CLI works:

```bash
LEVEL_SLUG=$(echo "${LEVEL_NAME}" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
npx playwright screenshot \
  --load-storage="${SESSION_FILE}" \
  --full-page \
  "${LEVEL_URL}" \
  "${OUTPUT_DIR}/screenshots/level-${LEVEL_SLUG}.png"
```

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
  "bms_platform": "Niagara N4",
  "extracted_at": "2026-03-16T14:30:00+11:00",
  "levels": [
    {
      "name": "Level 1",
      "screenshot": "screenshots/level-1.png",
      "zones": [
        { "name": "Zone A", "screenshot": "screenshots/level-1-zone-a.png" },
        { "name": "Zone B", "screenshot": "screenshots/level-1-zone-b.png" }
      ]
    },
    {
      "name": "Level 2",
      "screenshot": "screenshots/level-2.png",
      "zones": [
        { "name": "North Wing", "screenshot": "screenshots/level-2-north-wing.png" }
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

### 5d. Clean up temporary scripts

Remove the helper scripts and npm artifacts from the output directory:

```bash
rm -f "${OUTPUT_DIR}/discover.cjs" "${OUTPUT_DIR}/extract.cjs"
rm -f "${OUTPUT_DIR}/package.json" "${OUTPUT_DIR}/package-lock.json"
rm -rf "${OUTPUT_DIR}/node_modules"
```

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

### Playwright not installed
```bash
npx playwright install chromium
```

### Session expired
Delete the saved state and re-authenticate:
```bash
rm "${OUTPUT_DIR}/playwright-state.json"
```

### `networkidle` timeout
BMS pages with live data polling (temperatures, alarms) often never reach `networkidle`. Use `domcontentloaded` and add an explicit wait:
```javascript
await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30000 });
await page.waitForTimeout(5000);  // allow async rendering
```

### BMS content in iframes
Many BMS platforms (Allerton/Optergy, some Niagara setups) load page content in iframes. If `page.evaluate()` returns nothing useful:
1. List all frames: `page.frames().forEach(f => console.log(f.url()))`
2. Look for the content frame (e.g. `page?d=...`)
3. Either evaluate within that frame, or navigate directly to the iframe URL (e.g. replace `page?d=` with `page?dds=`)

### Nav tree not loading (SPA)
Some BMS platforms load the nav tree asynchronously. Use explicit waits:
```javascript
await page.waitForSelector('.nav-tree', { timeout: 10000 });
```

### Can't identify levels vs zones
Ask the user to clarify the hierarchy. Take a screenshot of the nav tree and ask:
> "I can see these nodes in the navigation. Which ones are levels and which are zones?"
