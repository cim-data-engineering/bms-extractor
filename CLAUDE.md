# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

BMS Extractor is a Claude Code skill for extracting building hierarchy data (levels and zones) from BMS (Building Management System) web interfaces. It uses Playwright for browser automation, making it compatible with both Claude Code CLI and CoWork (no `--chrome` flag required).

The primary use case is commissioning new sites in the Peak platform — extracting the spatial hierarchy from the BMS as the first onboarding step.

## Skill Architecture

```
skills/bms-extractor/
├── SKILL.md              # Skill definition
└── references/           # Platform-specific BMS UI patterns
    └── bms-ui-patterns.md
```

## How to Use

Run Claude Code and invoke the skill naturally:
- "Extract levels and zones from the BMS at https://bms.example.com"
- "Map the BMS hierarchy for 99 Elizabeth St"

The skill will:
1. Launch a browser for the user to authenticate
2. Screenshot and navigate the BMS nav tree
3. Extract levels and zones
4. Output `site_model.json`, `levels_and_zones.tsv`, and `manifest.json`

## Output

```
bms-extract/<site-name>/
├── site_model.json           # Structured hierarchy (JSON)
├── levels_and_zones.tsv      # Level + zones table for Peak import
├── manifest.json             # Extraction metadata
├── playwright-state.json     # Saved browser session (reusable)
└── screenshots/              # Evidence screenshots per level/zone
```

## Building Hierarchy Model

- **Site** (building) > **Level** (floor) > **Zone** (space/room)
- Levels can be physical floors or virtual spaces (e.g., "Plantroom", "Roof")
- Zones can be physical spaces on a level or virtual/system groupings
- Many zone names are equipment names (e.g., "VAV 01", "FCU-1")
- Basement level numbers are stored as positive numbers (B1=1, B2=2)

## CoWork Deployment

This repo is structured as a Claude Code plugin. To install in CoWork or another Claude Code environment:

```
/plugin marketplace add <path-or-url>
/plugin install bms-extractor@bms-extractor --scope project
```

## Prerequisites

- Playwright: `npx playwright install chromium`
- BMS URL accessible (VPN if required)
