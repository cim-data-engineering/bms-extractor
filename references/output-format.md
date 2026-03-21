# Output Format Reference

Detailed specifications for the JSON site model and Excel workbook output.

### Two-phase output

Output is generated in two phases:

- **Phase 1 (after Part B):** 3-tab xlsx (`levels_and_zones`, `equipment_list`, `equipment_types`), sitemodel.json without `points_list`, manifest.json with `points_extracted: false`. These files are complete and usable immediately.
- **Phase 2 (after Part C, optional):** 4-tab xlsx (adds `points_list` tab), sitemodel.json updated with `points_list` array, manifest.json updated with `points_extracted: true` and point counts. `write_xlsx.py` overwrites the Phase 1 workbook cleanly — it creates a fresh workbook each run.

---

## {site_name}_sitemodel.json Structure

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
      "device_id": "NAE-01",
      "source_url": "https://bms.example.com/chw-plant"
    },
    {
      "equipment_name": "VAV_C1",
      "level_select": "Ground",
      "zone_select": "C1",
      "equipment_type_select": "Variable Air Volume",
      "device_id": "JACE-02",
      "source_url": "https://bms.example.com/floor/ground"
    }
  ],
  "points_list": [
    {
      "equipment_name": "HR_PZN_AHU",
      "graphic_point_name": "S/A Duct Pressure",
      "graphic_value": "23",
      "graphic_unit": "Pa",
      "underlying_point_name": "SA_StaticPressure"
    },
    {
      "equipment_name": "HR_PZN_AHU",
      "graphic_point_name": "SAF Call",
      "graphic_value": "Off",
      "graphic_unit": "",
      "underlying_point_name": "SAF_Call"
    }
  ]
}
```

---

## Excel Workbook Tabs

### Tab 1 — `levels_and_zones`

| level | equipments | zones | source_url |
|-------|-----------|-------|------------|
| Plantroom | default | Plantroom, All | default |
| Ground | VAV_C1, VAV_C2, FCU_G_1 | Plantroom, All, C1, C2, G_1 | https://... |

- `equipments`: comma-separated equipment names from the floor plan
- `zones`: always starts with "Plantroom, All", followed by unique zone IDs derived from equipment names
- First data row is always the Plantroom level with default values
- Levels without floor plan equipment (Roof, B1, etc.) get zones="Plantroom, All" with empty equipments

### Tab 2 — `equipment_list`

| equipment_name | level_select | zone_select | equipment_type_select | device_id | source_url |
|---------------|-------------|-------------|----------------------|-----------|------------|
| HR_CZN_AHU | Roof | Plantroom | Air Handling Units | NAE-01 | https://... |
| Chiller_1 | Plantroom | Plantroom | Chiller | NAE-01 | https://... |
| VAV_C1 | Ground | C1 | Variable Air Volume | JACE-02 | https://... |

- One row per equipment from ALL pages (summary pages + floor plans)
- `level_select` and `zone_select` must reference values from Tab 1
- `equipment_type_select` must be from the master equipment types list
- `device_id` is the controller/device identifier — best endeavour, blank if not available

### Tab 3 — `points_list` (optional — only if Part C was run)

| equipment_name | level_select | zone_select | equipment_type_select | device_id | graphic_point_name | graphic_value | graphic_unit | underlying_point_name |
|---|---|---|---|---|---|---|---|---|
| HR_PZN_AHU | Roof | HR-Centre | Air Handling Units | NAE-01 | S/A Duct Pressure | 23 | Pa | SA_StaticPressure |
| HR_PZN_AHU | Roof | HR-Centre | Air Handling Units | NAE-01 | SAF Call | Off | | SAF_Call |
| HR_PZN_AHU | Roof | HR-Centre | Air Handling Units | NAE-01 | Out Of Service | No | | *(no DOM binding)* |

- 9 columns: 5 equipment context columns looked up from `equipment_list` + 4 extraction columns from Part C
- `level_select`, `zone_select`, `equipment_type_select`, `device_id` are joined from `equipment_list` by matching `equipment_name`
- `graphic_point_name` = human-readable label from the BMS graphic
- `graphic_value` = displayed value (numeric or state string)
- `graphic_unit` = unit of measure if shown; blank for state values
- `underlying_point_name` = DOM-bound point/tag name; `*(no DOM binding)*` if none found
- JSON `points_list` stores only the 5 raw extraction fields; the 4 equipment context columns are added at xlsx write time

### Tab 4 — `equipment_types`

| equipment_types |
|----------------|
| Active Chilled Beams |
| Air Cooled Condensers |
| ... |

- Static reference tab: the 77 equipment type names from `references/equipment-types.md`
