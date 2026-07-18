# CANMOVES Input Generation

This notebook generates CANMOVES-ready activity input tables from an EMME traffic file. It reads link-level traffic activity, fleet classification settings, fuel-distribution files, age-distribution files, and run-configuration metadata, then produces activity mapping tables by link, source type, fuel type, and model year.

The workflow is intended for use in Google Colab, but the functions only depend on standard Python packages plus `pandas` and `numpy`.

## Requirements

```python
import pandas as pd
import numpy as np
import json
```

## Input files

The notebook expects the following input files:

| File | Purpose |
|---|---|
| `EMME.csv` | Main traffic input file with link-level activity data. |
| `canmovesScaleDefinition.json` | Defines the fleet classification type. If `fleetClassification == "basic5"`, the Basic fuel-distribution file is used; otherwise the EPA fuel-distribution file is used. |
| `canmovesRunSpecConfig.json` | Provides run settings such as `selectedYear`, which is used to convert vehicle age to model year. |
| `Fuel Distribution_Basic.csv` | Fuel and subclass distribution file for the Basic 5-class fleet structure. |
| `Fuel Distribution_EPA.csv` | Fuel and subclass distribution file for the EPA fleet structure. |
| `Age.csv` | Vehicle age distribution by source type. |

## Expected EMME columns

The EMME file should include link identifiers and link length:

```text
inode
jnode
Length_km
```

It should also include vehicle activity columns for some or all of the following vehicle classes:

```text
LDV
LDT
MDV
HDV
Bus
```

The expected activity variables are:

```text
Volume
Speed_kph
ColdStart_pct
```

Bus may also include:

```text
DwellTime_s
```

The `MDV` columns are optional. If they are not present, the code skips them automatically.

## Main workflow

The notebook follows these steps:

1. Read the EMME input file.
2. Add total traffic activity columns:
   - `Total_Volume`
   - `Total_Speed_kph`
   - `Total_ColdStart_pct`
   - `Total_VKT`
3. Calculate VKT for each vehicle class:
   - `LDV_VKT`
   - `LDT_VKT`
   - `MDV_VKT`
   - `HDV_VKT`
   - `Bus_VKT`
4. Select the correct fuel-distribution file based on `fleetClassification`.
5. Break down class-level VKT into CANMOVES source-type VKT.
6. Break down source-type VKT by fuel type.
7. Break down source-type VKT by vehicle age and convert vehicle age to model year.
8. Generate activity map tables and summary tables.
9. Save all generated outputs as CSV files.

## Vehicle and source-type handling

The code maps broad EMME vehicle classes into CANMOVES source types.

The output source types are:

```text
LDV-mini
LDV-Eco
LDV-Large
LDT1
LDT2
LDT3
LDT4
HDV2b3
HDV3
HDV4
HDV5
HDV6
HDV7
HDV8a
HDV8b
Bus - SS
Bus - SL
Bus - TO
Bus - TN
Bus - TL
Bus - TS
```

The subclass split is based on the `Percent of Class` column in the selected fuel-distribution file.

For `basic5`, the class mapping is:

| Class # | Vehicle class |
|---:|---|
| 1 | LDV |
| 2 | LDT |
| 3 | MDV |
| 4 | HDV |
| 5 | Bus |

For the EPA fleet structure, the class mapping is:

| Class # | Vehicle class |
|---:|---|
| 1 | LDV |
| 2 | LDT |
| 3 | HDV |
| 4 | Bus |

## Fuel-distribution handling

The fuel-distribution file is normalized by row so that the fuel fractions for each subclass sum to 1.

The fuel columns used are:

```text
Percent Gasoline
Percent Diesel
Percent Nat. Gas
Percent Propane
Percent Methanol
percent Ethanol
```

They are converted to these fuel labels:

```text
Gasoline
Diesel
Nat. Gas
Propane
Methanol
Ethanol
```

If a subclass has zero fuel distribution, the code uses the distribution from `Sub-class # == 1` within the same `Class #`. This keeps total activity preserved while allowing subclasses without explicit fuel distributions to inherit the base distribution for their class.

## Age and model-year handling

The `Age.csv` file is converted to long format:

```text
vehicle_type | vehicle_age | age_fraction
```

Age fractions are normalized by vehicle type. The model year is calculated using:

```text
modelYearID = selectedYear - vehicle_age
```

The `selectedYear` value is read from `canmovesRunSpecConfig.json`.

## Output files

The notebook writes the following CSV files:

| Output file | Description |
|---|---|
| `df_activity_map_total_volume.csv` | Total traffic volume by `linkID`. |
| `df_activity_map_total_speed.csv` | Total weighted-average speed by `linkID`. |
| `df_activity_map_total_coldstart.csv` | Total weighted-average cold-start percentage by `linkID`. |
| `df_activity_map_total_vkt.csv` | Total VKT by `linkID`. |
| `df_activity_map_fuel.csv` | VKT activity by `linkID` and `fuelTypeID`. |
| `df_activity_map_source.csv` | VKT activity by `linkID` and `sourceTypeID`. |
| `df_activity_map_model.csv` | VKT activity by `linkID` and `modelYearID`. |
| `df_activity_chart_by_model_per_fuel.csv` | Fuel activity summary reshaped with fuel types as columns. |
| `df_activity_chart_by_model_per_source.csv` | Source-type activity summary reshaped with source types as columns and model years as rows. |
| `df_activity_chart_by_source_per_fuel.csv` | Fuel activity summary reshaped with fuel types as columns and source types as rows. |
| `df_activity_chart_total_per_source.csv` | Total activity by source type. |
| `df_activity_chart_total_per_model.csv` | Total activity by model year. |
| `df_activity_chart_total_per_fuel.csv` | Total activity by fuel type. |

## Key functions

| Function | Purpose |
|---|---|
| `read_emme_file()` | Reads the EMME CSV file and removes empty rows. |
| `get_existing_vehicle_types()` | Detects which vehicle classes exist in the EMME file. |
| `add_total_vehicle_columns()` | Adds total volume, weighted speed, and weighted cold-start columns. |
| `add_vkt_columns()` | Calculates VKT for each available vehicle class and total vehicles. |
| `prepare_emme_input()` | Runs the initial EMME preparation steps. |
| `read_fleet_classification()` | Reads `fleetClassification` from the scale-definition JSON file. |
| `select_fuel_distribution_path()` | Selects the Basic or EPA fuel-distribution file. |
| `prepare_subclass_vkt_distribution()` | Converts `Percent of Class` into subclass VKT fractions. |
| `breakdown_vkt_to_subclass_columns()` | Breaks class-level VKT into source-type VKT columns. |
| `extract_total_emme_dataframes()` | Creates separate total activity dataframes for volume, speed, cold start, and VKT. |
| `extract_subclass_vkt_dataframe()` | Extracts source-type VKT columns with `linkID`. |
| `prepare_fuel_distribution()` | Normalizes fuel fractions by source type. |
| `expand_subclass_vkt_by_fuel()` | Breaks source-type VKT into fuel-type activity. |
| `prepare_age_distribution()` | Converts age distribution data to long format. |
| `expand_subclass_vkt_by_age()` | Breaks source-type VKT into model-year activity. |
| `generate_activity_summary_dataframes()` | Creates total summaries by source type, model year, and fuel type. |

## How to run

Update the paths in the parameter section of the notebook:

```python
emme_csv_path = "/content/EMME.csv"
scale_definition_json_path = "/content/canmovesScaleDefinition.json"
config_json_path = "/content/canmovesRunSpecConfig.json"
basic_fuel_csv_path = "/content/Fuel Distribution_Basic.csv"
epa_fuel_csv_path = "/content/Fuel Distribution_EPA.csv"
age_csv_path = "/content/Age.csv"
```

Then run all notebook cells from top to bottom. The final cell writes all output dataframes to CSV.

## Notes

- `linkID` is created as `inode-jnode`.
- Total speed and total cold-start percentage are volume-weighted averages.
- `Activity` represents VKT-based activity after source, fuel, or model-year breakdown.
- The code is designed to keep the data in simple pandas dataframes so that the outputs can be used directly by downstream CANMOVES processing or reporting workflows.
