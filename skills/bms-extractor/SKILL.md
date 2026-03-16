---
name: bms-extractor
description: Extracts building hierarchy data (levels, zones, and equipment) from a BMS (Building Management System) web interface using built-in browser tools (Navigate, Click, Read page, Execute JavaScript). Produces an Excel workbook (.xlsx) with levels/zones, equipment list, and equipment types tabs for Peak platform import, plus a JSON site model with page source for verification. Triggers for "extract levels from BMS", "get zones from BMS", "map BMS levels and zones", "BMS site structure extraction", or "extract BMS hierarchy for Peak import".
---

# BMS Extractor

Extracts **building hierarchy data** (levels, zones, and all equipment) from a BMS web interface using built-in browser tools, producing an Excel workbook (.xlsx) ready for Peak platform import and a JSON site model with page source evidence.

**Extract all equipment types** — HVAC, electrical, hydraulic, fire, vertical transport, and any other equipment visible in the BMS that matches the equipment types reference list.

---

## Workflow Overview

```
1. SETUP      → Collect BMS URL + site name, create output directory
2. AUTH       → Navigate to BMS, check if already logged in, handle login if needed
3. SCOPE      → Ask user for guidance on where to find levels/zones and equipment
4. DISCOVER   → Identify nav tree pattern using user-provided URLs
5. EXTRACT    → Two-pass: floor plans (levels & zones), then full equipment list
6. OUTPUT     → Write {site_name}.xlsx, site_model.json, manifest.json
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
2. **Read the page** to check for login forms vs dashboard content
3. If it shows the BMS dashboard/home page, the user is already authenticated → skip to Step 3
4. Only if it shows a login page, proceed with the login flow below

### Login flow

If login is needed, tell the user:

> "The BMS login page is showing. Please log in using the Chrome browser — you can find it in your browser tabs. Let me know once you're on the dashboard."

Wait for the user to confirm, then read the page again to verify.

**Note:** The browser tab doesn't always pop up automatically — the user may need to find it themselves.

**CRITICAL: Never write credentials to output files.**

---

## Browser Tab Setup

**CRITICAL:** Before attempting JavaScript execution on a BMS page, always ensure you have a working tab.

### Correct sequence:
1. Call `tabs_context_mcp(createIfEmpty: true)` to initialize the tab group
2. Call `tabs_create_mcp` to create a **new, clean tab** — use this tab ID for all interaction work
3. Navigate to the BMS URL on the new tab
4. Wait 5 seconds for the page to fully render
5. Read page — this should now work reliably

### If page reads fail on a tab:
- Create another new tab with `tabs_create_mcp` and retry

---

## Step 3: SCOPE — User-Guided URL Discovery

After authentication, **ask the user for guidance** before exploring. Do not start extracting immediately.

Present this prompt to the user:

> "I'm authenticated. To extract efficiently, can you help me with:
> 1. **Levels & zones** — what URL(s) should I look at for floor plans or level navigation? (e.g., floor plan graphics, level summary page)
> 2. **Equipment list** — what URL(s) should I look at for equipment summaries? (e.g., AHU summary, chiller plant, ventilation pages)
>
> You can share links or describe where to find them in the BMS navigation."

Wait for the user to respond with URL pointers or navigation guidance before proceeding to Step 4.

---

## Step 4: DISCOVER — Identify Navigation Structure

Using the URLs or guidance from Step 3, explore the BMS interface:

1. **Navigate** to the user-provided URLs
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
- **Equipment summary pages** — where to find comprehensive equipment lists

---

## Step 5: EXTRACT — Two-Pass Extraction

Extraction has two parts: floor plans (levels & zones) and full equipment discovery.

### Part A: Floor Plan Extraction (builds levels_and_zones tab)

For each level in scope:
1. **Navigate** to the level's floor plan / graphics page using URLs from user guidance
2. **Read page** + **Execute JavaScript** to extract equipment names and types:
   ```javascript
   document.documentElement.outerHTML
   ```
3. **Save page source** (HTML) to `${OUTPUT_DIR}/pages/level-<slug>.html`
4. **Record `source_url`** — the page URL where equipment was found
5. **Extract equipment names** from the page — look for all equipment labels on the graphic
6. **Derive zone IDs** — strip the equipment type prefix to get the zone ID:
   - `VAV_P1` → zone `P1`
   - `FCU_G_1` → zone `G_1`
   - `VAV_C2` → zone `C2`
   - `PAU_11_1` → zone `11_1`
7. **Build zone list** — always prepend "Plantroom, All" to each level's zone list, then append unique zone IDs

**Zone derivation rule:** Equipment names typically follow `{TYPE}_{ZoneID}` or `{TYPE}_{Level}_{ZoneID}` patterns. Strip the known equipment type prefix (VAV, FCU, AHU, PAU, etc.) to get the zone identifier. Multiple equipment with the same zone ID serve the same zone — deduplicate.

**Always include these default rows:**
- A "Plantroom" level with equipments=`default`, zones=`Plantroom, All`, source_url=`default`
- Any level without floor plan equipment (e.g., Roof, B1) still gets zones=`Plantroom, All`

### Part B: Full Equipment List (builds equipment_list tab)

Browse summary/plant/overview pages using URLs from user guidance to find ALL equipment:

1. **Navigate** to each equipment summary page (AHU summary, chiller plant, ventilation, FCU summary, etc.)
2. **Read page** + **Execute JavaScript** to extract all equipment names
3. For each equipment item:
   - **`equipment_name`** — the BMS name (e.g., `Chiller_1`, `HR_CZN_AHU`, `CPEF`)
   - **`level_select`** — assign to a level from Part A, or create a new level (e.g., "Plantroom", "Roof", "B1")
   - **`zone_select`** — assign to a zone from the level's zone list (default "Plantroom" or "All" for plant equipment)
   - **`equipment_type_select`** — classify using the equipment types reference in `references/equipment-types.md`
   - **`source_url`** — the page URL where the equipment was found
4. **Floor-plan equipment also goes into equipment_list** — every VAV, FCU, PAU found in Part A gets a row here too

### Saving page source

For every graphics/summary page visited, **always save the page source**:

```bash
cat > "${OUTPUT_DIR}/pages/level-1.html" << 'HTMLEOF'
... page source from Execute JavaScript ...
HTMLEOF
```

**Note on screenshots:** In CoWork, browser screenshots cannot be saved directly to the filesystem — the Chrome extension is sandboxed. Rely on saved page source (HTML) as the persistent artifact.

### Build the structure as you go

Maintain a running JSON structure:

```json
{
  "site": "99 Elizabeth St",
  "bms_platform": "detected platform",
  "extracted_at": "2026-03-16T14:30:00+11:00",
  "levels": [
    {
      "name": "Plantroom",
      "equipments": "default",
      "zones": "Plantroom, All",
      "source_url": "default"
    },
    {
      "name": "Ground",
      "source_url": "https://bms.example.com/floor/ground",
      "equipments": ["VAV_C1", "VAV_C2", "FCU_G_1"],
      "zones": "Plantroom, All, C1, C2, G_1"
    }
  ],
  "equipment_list": [
    {
      "equipment_name": "Chiller_1",
      "level_select": "Plantroom",
      "zone_select": "Plantroom",
      "equipment_type_select": "Chiller",
      "source_url": "https://bms.example.com/chw-plant"
    },
    {
      "equipment_name": "VAV_C1",
      "level_select": "Ground",
      "zone_select": "C1",
      "equipment_type_select": "Variable Air Volume",
      "source_url": "https://bms.example.com/floor/ground"
    }
  ]
}
```

### Progress reporting

After each level/page, report progress:
> "Extracted Ground floor: found 3 VAVs, 3 FCUs, 1 PAU → 6 zones. Moving to Level 1..."

---

## Step 6: OUTPUT — Write Files

### 6a. `{site_name}.xlsx` — Excel Workbook

Use Python with openpyxl to write an Excel workbook with 3 tabs. Install if needed:

```bash
pip install openpyxl
```

**Tab 1 — `levels_and_zones`**

| level | equipments | zones | source_url |
|-------|-----------|-------|------------|
| Plantroom | default | Plantroom, All | default |
| Ground | VAV_C1, VAV_C2, FCU_G_1 | Plantroom, All, C1, C2, G_1 | https://... |

- `equipments`: comma-separated equipment names from the floor plan
- `zones`: always starts with "Plantroom, All", followed by unique zone IDs derived from equipment names
- First data row is always the Plantroom level with default values
- Levels without floor plan equipment (Roof, B1, etc.) get zones="Plantroom, All" with empty equipments

**Tab 2 — `equipment_list`**

| equipment_name | level_select | zone_select | equipment_type_select | source_url |
|---------------|-------------|-------------|----------------------|------------|
| HR_CZN_AHU | Roof | Plantroom | Air Handling Units | https://... |
| Chiller_1 | Plantroom | Plantroom | Chiller | https://... |
| VAV_C1 | Ground | C1 | Variable Air Volume | https://... |

- One row per equipment from ALL pages (summary pages + floor plans)
- `level_select` and `zone_select` must reference values from Tab 1
- `equipment_type_select` must be from the master equipment types list

**Tab 3 — `equipment_types`**

| equipment_types |
|----------------|
| Active Chilled Beams |
| Air Cooled Condensers |
| ... |

- Static reference tab: the 77 equipment type names from `references/equipment-types.md`

**Python script to write xlsx:**

```python
import openpyxl

wb = openpyxl.Workbook()

# Tab 1: levels_and_zones
ws1 = wb.active
ws1.title = "levels_and_zones"
ws1.append(["level", "equipments", "zones", "source_url"])
for level in levels_data:
    ws1.append([level["name"], level["equipments"], level["zones"], level["source_url"]])

# Tab 2: equipment_list
ws2 = wb.create_sheet("equipment_list")
ws2.append(["equipment_name", "level_select", "zone_select", "equipment_type_select", "source_url"])
for eq in equipment_data:
    ws2.append([eq["equipment_name"], eq["level_select"], eq["zone_select"], eq["equipment_type_select"], eq["source_url"]])

# Tab 3: equipment_types
ws3 = wb.create_sheet("equipment_types")
ws3.append(["equipment_types"])
for et in equipment_types_list:
    ws3.append([et])

wb.save(f"{OUTPUT_DIR}/{site_name}.xlsx")
```

### 6b. `site_model.json`

Write the complete JSON structure built during extraction (includes `source_url` fields).

### 6c. `manifest.json`

Write extraction metadata (source URL, site name, platform, timestamps, counts).

### Final summary

Report to the user:
> **Extraction complete for [Site Name]**
> - Levels: N
> - Total equipment: M
> - Total zones: Z
> - Output: `bms-extract/<site-name>/`
> - Files: `{site_name}.xlsx`, `site_model.json`, `manifest.json`
> - Page sources: `pages/` (P files)

---

## Troubleshooting

### Browser tab not visible
The browser doesn't always pop up automatically. Tell the user to look for the Chrome browser tab.

### JS errors ("Cannot access chrome-extension:// URL")
Create a fresh tab with `tabs_create_mcp` and use that tab ID. Do not reuse the default tab for JavaScript execution.

### Network blocked
If the BMS IP/domain is blocked, add it under settings → "Additional allowed domains".

### Tool errors (Click, Execute JavaScript)
Always Navigate to a page first and wait for it to load before using other browser tools. If errors persist, create a new tab and retry.

### BMS content in iframes
If Read page or Execute JavaScript returns limited content, check for iframes and navigate directly to the iframe URL.

### Can't identify levels vs zones
Ask the user to clarify. Read the page and ask:
> "I can see these nodes in the navigation. Which ones are levels and which are zones?"
