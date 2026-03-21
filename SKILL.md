---
name: bms-extractor
description: Extracts building hierarchy (levels, zones, equipment) from BMS web interfaces. Produces an Excel workbook (.xlsx) for Peak platform import .Triggers for "extract from BMS", "map BMS hierarchy", "BMS site structure", or "extract BMS for Peak".
---

## Safety: Read-Only Operations

**CRITICAL: This skill is strictly READ-ONLY. BMS systems control real HVAC equipment in occupied buildings — accidental writes can affect occupant comfort, energy usage, or equipment safety.**

### Prohibited actions
- Clicking "Write", "Override", "Set", "Apply", "Command", or "Acknowledge" buttons
- Changing setpoints, schedules, modes, or any equipment parameters
- Submitting forms that modify BMS state
- Acknowledging or dismissing alarms
- Executing JavaScript that mutates state (e.g., POST requests, form submissions, element value changes)

### Allowed actions
- **Navigate** to pages and URLs
- **Click** to expand/collapse tree nodes and navigate between views
- **Read page** to extract displayed content
- **Execute JavaScript** for DOM reading only (e.g., `document.querySelectorAll`, `outerHTML`, `textContent`)

If a page shows editable controls (sliders, input fields, dropdowns), extract the displayed values but **never modify them**.

---

## Workflow Overview

```
1. SETUP      → Collect BMS URL + site name, create output directory
2. AUTH       → Navigate to BMS, handle login if needed
3. SCOPE      → Ask user for URL guidance (floor plans, equipment pages)
4. DISCOVER   → Identify nav tree pattern using user-provided URLs
5. EXTRACT    → Two-pass: floor plans (levels & zones), then full equipment list
6. OUTPUT     → Write xlsx (3 tabs), sitemodel.json, manifest.json
7. POINTS     → (Optional) Extract BMS points, then update output files (4 tabs)
```

### Progress checklist

Use this to track extraction progress:

- [ ] SETUP — output directory created
- [ ] AUTH — authenticated and on BMS dashboard
- [ ] SCOPE — user provided URL guidance
- [ ] DISCOVER — nav pattern identified
- [ ] EXTRACT Part A — floor plans done, levels & zones captured
- [ ] EXTRACT Part B — full equipment list complete
- [ ] OUTPUT — xlsx (3 tabs) + JSON files written
- [ ] POINTS Part C — (optional) points extracted, output updated (4 tabs)

---

## Step 1: SETUP

Ask the user for:
1. **BMS URL** — the web interface URL
2. **Site name** — human-readable name (e.g. "99 Elizabeth St")

Create the output directory: `bms-extract/<site-name-kebab>/`

---

## Step 2: AUTH

**CRITICAL: Never ask the user to type credentials into the chat. Never write credentials to output files.**

1. Navigate to the BMS URL
2. Read the page — if it shows the BMS dashboard, skip to Step 3
3. If it shows a login page, tell the user:
   > "The BMS login page is showing. Please log in using the Chrome browser tab. Let me know once you're on the dashboard."
4. Wait for confirmation, then re-read to verify

**Note:** The browser tab doesn't always pop up automatically — the user may need to find it in their browser tabs.

### Browser tab setup

Before executing JavaScript on a BMS page:
1. Call `tabs_context_mcp(createIfEmpty: true)` to initialize
2. Call `tabs_create_mcp` to create a new tab — use this tab ID for all work
3. Navigate to the BMS URL, wait 5 seconds for render
4. If page reads fail, create another new tab and retry

---

## Step 3: SCOPE — User-Guided URL Discovery

After authentication, **ask the user before exploring**:

> "I'm authenticated. To extract efficiently, can you help me with:
> 1. **Levels & zones** — what URL(s) should I look at for floor plans or level navigation?
> 2. **Equipment list** — what URL(s) should I look at for equipment summaries? (e.g., AHU summary, chiller plant)
>
> You can share links or describe where to find them in the BMS navigation."

Wait for the user to respond before proceeding.

---

## Step 4: DISCOVER — Identify Navigation Structure

Navigate to user-provided URLs. Read pages and use Execute JavaScript to extract DOM elements, especially from iframes.

**Read `references/bms-ui-patterns.md`** for platform-specific guidance on navigation, hierarchy, and common gotchas.

### What to identify

- **Nav pattern** — sidebar tree, top tabs, button grid, or breadcrumb-based
- **Level/zone structure** — how levels and zones appear in the hierarchy
- **Direct URLs** — whether each level/zone has a navigable URL
- **Equipment summary pages** — where to find comprehensive equipment lists

**Iframe handling:** Many BMS platforms load content in iframes. If Read page returns limited content, navigate directly to the iframe URL.

---

## Step 5: EXTRACT — Two-Pass Extraction

**Remember: read and extract only. Do not click controls that modify setpoints, schedules, or equipment state.**

### Part A: Floor Plan Extraction (builds levels_and_zones)

For each level:
1. Navigate to the level's floor plan / graphics page
2. Read page + Execute JavaScript (`document.documentElement.outerHTML`) to extract equipment names
3. Record `source_url` — the page URL where equipment was found
5. Extract equipment names — look for all equipment labels on the graphic
6. **Derive zone IDs** — strip the equipment type prefix:
   - `VAV_P1` → zone `P1`, `FCU_G_1` → zone `G_1`, `PAU_11_1` → zone `11_1`
7. Build zone list — always prepend "Plantroom, All", then append unique zone IDs

**Zone derivation rule:** Equipment names follow `{TYPE}_{ZoneID}` or `{TYPE}_{Level}_{ZoneID}`. Strip the known type prefix to get the zone. Deduplicate across equipment sharing the same zone.

**Default rows:**
- A "Plantroom" level with equipments=`default`, zones=`Plantroom, All`, source_url=`default`
- Levels without floor plan equipment still get zones=`Plantroom, All`

### Part B: Full Equipment List (builds equipment_list)

Browse summary/plant/overview pages to find ALL equipment:

1. Navigate to each equipment summary page
2. Read page + Execute JavaScript to extract equipment names
3. For each item, record:
   - **`equipment_name`** — BMS name (e.g., `Chiller_1`, `HR_CZN_AHU`)
   - **`level_select`** — assign to a level from Part A, or create new (e.g., "Plantroom", "Roof")
   - **`zone_select`** — assign to a zone from the level's zone list
   - **`equipment_type_select`** — classify using `references/equipment-types.md`
   - **`device_id`** — controller/device identifier, often found in the page URL or page content (e.g., `NAE-01`, `JACE-8000-1`). Comma-separate if multiple. Best endeavour — skip if not readily available.
   - **`source_url`** — the page URL where found
4. Floor-plan equipment from Part A also gets rows here

### Build the structure as you go

Maintain a running JSON site model during extraction. See `references/output-format.md` for the full schema and example.

### Progress reporting

After each level/page:
> "Extracted Ground floor: found 3 VAVs, 3 FCUs, 1 PAU → 6 zones. Moving to Level 1..."

---

## Step 6: OUTPUT — Write Files

### 6a. `{site_name}_sitemodel.json`

Write the JSON structure built during extraction (without `points_list` — that is added in Step 7 if Part C runs).

### 6b. `{site_name}_assetregister.xlsx` — Excel Workbook

Generate the workbook:

```bash
pip install openpyxl -q
python scripts/write_xlsx.py <output_dir>
```

This produces a 3-tab workbook (`levels_and_zones`, `equipment_list`, `equipment_types`). See `references/output-format.md` for column specs and examples.

### 6c. `{site_name}_manifest.json`

Write extraction metadata (source URL, site name, platform, timestamps, counts). Set `points_extracted: false`.

### Extraction summary and Part C prompt

Report to the user and offer Part C:

> **Extraction complete for [Site Name]**
> - Levels: N | Zones: Z | Equipment: M
> - Output: `bms-extract/<site-name>/`
> - Files: `{site_name}_assetregister.xlsx`, `{site_name}_sitemodel.json`, `{site_name}_manifest.json`
>
> Would you like me to also extract BMS points (sensors, setpoints, actuators) from equipment graphics?
> - **All** — extract points for every equipment
> - **Selected** — extract for specific equipment (list them)
> - **Skip** — done, no points extraction needed

---

## Step 7: POINTS — Optional Part C (BMS Points Extraction)

If the user chose "Skip", stop here — files from Step 6 are the final deliverables.

If the user chose to extract points:

### Part C orchestration

1. **Read `references/points-extraction.md`** for the full extraction technique (includes guidance on navigating to equipment graphics pages)
2. Build the equipment work list from the **completed `equipment_list`** (all or user-selected subset)
3. For each equipment:
   a. **Navigate using `source_url`** from the equipment's row in `equipment_list`:
      - If the URL loads a **direct equipment graphics page** → proceed to extraction
      - If the URL loads a **summary/list page** (multiple equipment listed) → find the specific equipment entry and click through to its detail/graphics page
      - If no drill-down is possible or no graphics page exists → log and skip this equipment
   b. Follow the 6-step extraction technique from `references/points-extraction.md`
   c. Append extracted rows to the `points_list` array in the site model
   d. Report progress: `"Extracted 12 points for VAV_C1 (3/25 equipment done)"`
4. **Save intermediate results** every 10 equipment — write the current `points_list` to `{site_name}_sitemodel.json`

### After points extraction — update output files

Re-run output generation to incorporate the extracted points:

1. **Update `{site_name}_sitemodel.json`** — now includes the `points_list` array
2. **Re-run `write_xlsx.py`** — overwrites the 3-tab workbook with a 4-tab version (adds `points_list` tab between `equipment_list` and `equipment_types`)
   ```bash
   python scripts/write_xlsx.py <output_dir>
   ```
3. **Update `{site_name}_manifest.json`** — set `points_extracted: true`, add point counts

Report to the user:
> **Points extraction complete for [Site Name]**
> - Points: P total across E equipment
> - Updated files: `{site_name}_assetregister.xlsx` (now 4 tabs), `{site_name}_sitemodel.json`

### Error handling

- **Canvas-only graphics** — use screenshot fallback per the extraction technique
- **Navigation failure** — log the equipment name with error, skip, and continue
- **Auth timeout** — if a page returns a login screen, pause and ask the user to re-authenticate
- **Empty graphic** — if a page yields 0 points, note it and move on
- **Summary page with no drill-down** — log the equipment, skip, and continue

### Progress tracking

Maintain a running count:
> "Points extraction: 15/25 equipment complete. 187 points extracted so far."

---

## Troubleshooting

### Browser tab not visible
Tell the user to look for the Chrome browser tab.

### JS errors ("Cannot access chrome-extension:// URL")
Create a fresh tab with `tabs_create_mcp` and use that tab ID.

### Network blocked
If the BMS IP/domain is blocked, add it under settings → "Additional allowed domains".

### Tool errors (Click, Execute JavaScript)
Navigate to a page first and wait for load. If errors persist, create a new tab and retry.

### BMS content in iframes
If Read page returns limited content, check for iframes and navigate directly to the iframe URL.

### Can't identify levels vs zones
Ask the user to clarify:
> "I can see these nodes in the navigation. Which ones are levels and which are zones?"
