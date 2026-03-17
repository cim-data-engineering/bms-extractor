# Output Format Reference

Detailed specifications for the JSON site model and Excel workbook output.

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

| equipment_name | level_select | zone_select | equipment_type_select | source_url |
|---------------|-------------|-------------|----------------------|------------|
| HR_CZN_AHU | Roof | Plantroom | Air Handling Units | https://... |
| Chiller_1 | Plantroom | Plantroom | Chiller | https://... |
| VAV_C1 | Ground | C1 | Variable Air Volume | https://... |

- One row per equipment from ALL pages (summary pages + floor plans)
- `level_select` and `zone_select` must reference values from Tab 1
- `equipment_type_select` must be from the master equipment types list

### Tab 3 — `equipment_types`

| equipment_types |
|----------------|
| Active Chilled Beams |
| Air Cooled Condensers |
| ... |

- Static reference tab: the 77 equipment type names from `references/equipment-types.md`
