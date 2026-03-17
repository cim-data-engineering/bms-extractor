# BMS Extractor

A Claude Desktop skill that extracts building hierarchy data (levels, zones, and equipment) from BMS (Building Management System) web interfaces. Used for commissioning new sites in the Peak platform.

## Installation

### Prerequisites

- Access to the BMS URL (connect to VPN first if required)
- A Claude account (Pro or Team plan recommended for higher-capability models)

### Setup steps

1. **Download this repo** — on the GitHub page, click **Code > Download ZIP** and unzip it locally

2. **Install Claude Desktop** — download from [claude.ai/download](https://claude.ai/download)

3. **Install the Chrome extension** — search for **"Claude in Chrome (Beta)"** in the Chrome Web Store and install it

4. **Sign in to both** — make sure you are signed in with the same Claude account in both Claude Desktop and the Chrome extension

5. **Open CoWork** — in Claude Desktop, navigate to the **CoWork** tab (not Chat)

6. **Upload the skill** — click **Customize > Skills > + > Upload skill**, then select the downloaded zip file

7. **Enable Chrome connector** — click **Customize > Connectors** and enable **"Claude in Chrome"** under Desktop

8. **Start a new task** — go to the **CoWork** tab and click **New task**

9. **Select a working folder** — choose a local folder where output files will be saved

10. **Select model settings** — choose the highest available model (e.g. Opus) and toggle **Extended thinking** on

11. **Run the skill** — type `/bms-extractor` followed by the BMS URL to begin extraction

## Output

The skill produces three files in a `bms-extract/<site-name>/` directory inside your working folder:

| File | Description |
|------|-------------|
| `{site_name}_assetregister.xlsx` | Excel workbook with 3 tabs: **levels_and_zones**, **equipment_list**, **equipment_types** |
| `{site_name}_sitemodel.json` | Structured building hierarchy with equipment and zones |
| `{site_name}_manifest.json` | Extraction metadata (timestamps, source URL, counts) |

### Building hierarchy

```
Site (building)
└── Level (floor)
    └── Zone (space/area)
        └── Equipment (VAV, FCU, AHU, Chiller, etc.)
```
