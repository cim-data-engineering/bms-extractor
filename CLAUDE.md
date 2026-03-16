# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

BMS Extractor is a Claude Code skill for extracting HVAC building hierarchy data (levels, equipment, and zones) from BMS (Building Management System) web interfaces. It uses built-in browser tools (Navigate, Click, Take screenshot, Read page, Execute JavaScript) for browser automation.

The primary use case is commissioning new sites in the Peak platform — extracting the spatial hierarchy from the BMS as the first onboarding step.

## Skill Architecture

```
skills/bms-extractor/
├── SKILL.md              # Skill definition
└── references/           # Platform-specific BMS UI patterns (main branch only)
    └── bms-ui-patterns.md
```

## How to Use

Run Claude Code and invoke the skill naturally:
- "Extract levels and zones from the BMS at https://bms.example.com"
- "Map the BMS hierarchy for 99 Elizabeth St"

The skill will:
1. Navigate to the BMS and check if already authenticated
2. If login needed, prompt the user to log in via the Chrome browser tab
3. Show available levels, ask user what to extract
4. Extract HVAC equipment per level, then resolve zone names
5. Output `site_model.json`, `levels_and_zones.csv`, and `manifest.json`

## Output

```
bms-extract/<site-name>/
├── site_model.json           # Structured hierarchy with equipment + zones (JSON)
├── levels_and_zones.csv      # Level + zone + equipment table for Peak import
├── manifest.json             # Extraction metadata
└── pages/                    # Saved page source (HTML) per level
```

## Building Hierarchy Model

- **Site** (building) > **Level** (floor) > **Zone** (space/area) > **Equipment** (VAV, FCU, etc.)
- Equipment serves zones — a VAV is not a zone, it serves one
- Multiple equipment with the same zone ID serve the same zone (group them)
- Zone names come from: BMS descriptions, equipment naming conventions, or manual input
- Levels can be physical floors or virtual spaces (e.g., "Plantroom", "Roof")
- Basement level numbers are stored as positive numbers (B1=1, B2=2)

## Building the .skill file

To build the uploadable `.skill` file for CoWork:

```bash
cd skills/bms-extractor && zip -r ../../bms-extractor.skill SKILL.md && cd -
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
