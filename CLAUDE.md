# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

BMS Extractor is a Claude Code skill for extracting building hierarchy data (levels, zones, and all equipment) from BMS (Building Management System) web interfaces. It uses built-in browser tools (Navigate, Click, Read page, Execute JavaScript) for browser automation.

The primary use case is commissioning new sites in the Peak platform — extracting the spatial hierarchy and equipment inventory from the BMS as the first onboarding step.

## Skill Architecture

```
skills/bms-extractor/
├── SKILL.md              # Skill definition
└── references/           # Reference data (bundled into .skill zip)
    ├── bms-ui-patterns.md    # Platform-specific BMS UI patterns
    └── equipment-types.md    # Master list of 77 equipment types for Peak

examples/
└── final_output.xlsx     # Golden example of expected xlsx output format (not bundled into .skill)
```

## How to Use

Run Claude Code and invoke the skill naturally:
- "Extract levels and zones from the BMS at https://bms.example.com"
- "Map the BMS hierarchy for 99 Elizabeth St"

The skill will:
1. Navigate to the BMS and check if already authenticated
2. If login needed, prompt the user to log in via the Chrome browser tab
3. Ask the user for URL guidance (floor plans, equipment summary pages)
4. Two-pass extraction: floor plans (levels & zones), then full equipment list
5. Output `{site_name}.xlsx` (3 tabs), `site_model.json`, and `manifest.json`

## Output

```
bms-extract/<site-name>/
├── {site_name}.xlsx          # Excel workbook with 3 tabs (levels_and_zones, equipment_list, equipment_types)
├── site_model.json           # Structured hierarchy with equipment + zones (JSON)
├── manifest.json             # Extraction metadata
└── pages/                    # Saved page source (HTML) per level/page
```

## Building Hierarchy Model

- **Site** (building) > **Level** (floor) > **Zone** (space/area) > **Equipment** (VAV, FCU, AHU, Chiller, etc.)
- Equipment serves zones — a VAV is not a zone, it serves one
- Zone IDs are derived by stripping the equipment type prefix (e.g., `VAV_P1` → zone `P1`)
- Every level auto-includes "Plantroom" and "All" as default zones
- Levels can be physical floors or virtual spaces (e.g., "Plantroom", "Roof")
- Basement level numbers are stored as positive numbers (B1=1, B2=2)

## Building the .skill file

To build the uploadable `.skill` file for CoWork:

```bash
cd skills/bms-extractor && zip -r ../../bms-extractor.skill SKILL.md references/ && cd -
```

This creates `bms-extractor.skill` in the repo root. Upload it via CoWork → Customize → Skills → **+** button.

**IMPORTANT:** The zip must have `SKILL.md` at the top level, not nested in subdirectories.

## CoWork Deployment

To install/update the skill in CoWork:

1. In the conversation window, click the **+** button
2. Click **Connectors** then **Manage Connectors**
3. Select **Skills** on the right-hand side
4. To install: click **+** → **Upload skill** → select the `.skill` file
5. To update: click the existing skill → **⋯** (ellipsis) → **Replace** → select the new `.skill` file

**Note:** When replacing, the `.skill` file sometimes appears greyed out on the first attempt. Close and retry — it usually works the second time.

## Prerequisites

- BMS URL accessible (VPN if required)
