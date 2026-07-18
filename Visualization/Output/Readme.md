# CANMOVES Output CSV Generation

This notebook processes CANMOVES raw output results and generates cleaned, summarized CSV files for emissions by model year, source type, fuel type, link type, and chart-ready summaries.

The main input is a `RAW_Results.csv` file where each row represents one road link and the emission/activity columns are repeated for each vehicle type. The notebook reshapes these wide columns into long-format tables, applies vehicle age and fuel distributions, standardizes pollutant names and units, and saves multiple output CSV files.

## Requirements

The code uses standard Python data tools:

```python
pandas
numpy
json
```

No specialized external libraries are required.

## Input Files

Update the file paths in the `#Parameters` cell before running the notebook.

| Input | Description |
|---|---|
| `RAW_Results.csv` | CANMOVES raw output file. Each row is one link, with repeated pollutant columns for each vehicle type. |
| `Age.csv` | Vehicle age distribution table. Each vehicle type has age fractions that are used to expand emissions by vehicle age/model year. |
| `canmovesRunSpecConfig.json` | Run configuration file. The notebook reads `selectedYear` and `linkCategories` from this file. |
| `Fuel Distribution_Basic.csv` | Fuel distribution table by vehicle subclass. Used to split tailpipe emissions by fuel type. |

## Main Outputs

The notebook generates the following CSV files:

| Output file | Description |
|---|---|
| `df_emission_model.csv` | Emissions by `linkID`, `modelYearID`, and `pollutantID`. |
| `df_emission_source.csv` | Emissions by `linkID`, `sourceTypeID`, and `pollutantID`. |
| `df_emission_fuel.csv` | Tailpipe emissions by `linkID`, `fuelTypeID`, and `pollutantID`. |
| `df_emission_total.csv` | Total emissions by `linkID`, `linkTypeID`, and `pollutantID`. |
| `df_emission_chart_total.csv` | Chart-ready table with `linkTypeID` as rows and pollutants as columns. |
| `df_emission_chart_by_fuel.csv` | Chart-ready table with `fuelTypeID` as rows and pollutants as columns. |
| `df_emission_chart_by_model.csv` | Chart-ready table with `modelYearID` as rows and pollutants as columns. |
| `df_emission_chart_by_source.csv` | Chart-ready table with `sourceTypeID` as rows and pollutants as columns. |

## Workflow Summary

### 1. Define pollutant and vehicle mappings

The notebook starts by defining the expected pollutant IDs and vehicle type mappings.

The `get_pollutant_ids()` function lists all raw pollutant columns expected in the CANMOVES output, such as energy, fuel consumption, electric plant emissions, tailpipe emissions, and particulate matter.

The `get_vehicle_types()` function maps vehicle type names between:

- the final standardized output name,
- the prefix used in `RAW_Results.csv`, and
- the corresponding column name in `Age.csv`.

For example, `LDV-Economy` in the raw output is standardized to `LDV-Eco`.

### 2. Clean and reshape raw output

The `prepare_raw_results()` function reads `RAW_Results.csv`, creates a traceable link ID using:

```python
link = inode + "-" + jnode
```

It then converts the raw wide-format file into a long-format table:

```text
link | vehicle_type | pollutantID | original_value
```

This function also standardizes pollutant names and units. In particular:

- values in `kWh` are converted to `Wh`,
- values in `kg` are converted to `g`,
- pollutant labels are renamed to consistent output names.

Examples:

```text
Energy (kWh)      -> Energy (Wh)
CO (kg)           -> Tailpipe CO (g)
NH3 (kg)          -> Tailpipe NH3 (g)
Sub 2.5 um Tailpipe PM (g) -> Tailpipe PM2.5 (g)
```

### 3. Expand emissions by age and model year

The `prepare_age_distribution()` function reads `Age.csv` and converts it into long format:

```text
vehicle_type | vehicle_age | age_fraction
```

The `expand_using_age_distribution()` function merges this age distribution with the raw long-format emissions and calculates:

```python
expanded_value = original_value * age_fraction
```

The `summarize_expanded_results_model()` function converts `vehicle_age` to `modelYearID` using the selected year from `canmovesRunSpecConfig.json`:

```python
modelYearID = selectedYear - vehicle_age
```

It then summarizes emissions by:

```text
linkID | modelYearID | pollutantID
```

### 4. Summarize by source type

The `summarize_expanded_results_source()` function summarizes the age-expanded emissions by vehicle/source type:

```text
linkID | sourceTypeID | pollutantID | emissionQuant
```

This output keeps vehicle classes separate while summing over age.

### 5. Expand tailpipe emissions by fuel type

The `prepare_fuel_distribution()` function reads the fuel distribution file and normalizes the fuel percentages for each vehicle subclass so fuel fractions sum to 1.

The fuel columns are:

```text
Percent Gasoline
Percent Diesel
Percent Nat. Gas
Percent Propane
Percent Methanol
percent Ethanol
```

If a subclass has no fuel distribution, the function uses the distribution from `Sub-class # = 1` within the same `Class #`.

The `expand_tailpipe_by_fuel_distribution()` function keeps only pollutants with `Tailpipe` in their name, merges them with fuel fractions, and calculates:

```python
fuel_expanded_value = original_value * fuel_fraction
```

The `summarize_tailpipe_fuel_by_vehicle_type()` function then summarizes these results as:

```text
linkID | fuelTypeID | pollutantID | emissionQuant
```

### 6. Summarize by link type

The `read_link_type_mapping()` function reads `linkCategories` from `canmovesRunSpecConfig.json`. Enabled link categories are mapped in order, so for example:

```text
1 -> Classified
2 -> Un-Classified
```

The `summarize_raw_results_by_link_type()` function joins this link type information with the raw emissions and summarizes emissions by:

```text
linkID | linkTypeID | pollutantID | emissionQuant
```

### 7. Generate chart-ready tables

The notebook also reshapes the long-format outputs into wide-format tables for plotting or dashboard use.

The reshaping functions are:

| Function | Rows | Columns |
|---|---|---|
| `reshape_link_type_summary_total()` | `linkTypeID` | `pollutantID` |
| `reshape_link_type_summary_fuel()` | `fuelTypeID` | `pollutantID` |
| `reshape_link_type_summary_model()` | `modelYearID` | `pollutantID` |
| `reshape_link_type_summary_source()` | `sourceTypeID` | `pollutantID` |

Each reshaped table contains pollutant names as column headers and summed `emissionQuant` values as data.

## How to Run

1. Place the required input files in the working directory or update the file paths in the parameter cell.
2. Run all notebook cells in order.
3. The final cell saves all output CSV files using the paths defined in the parameter cell.

Example parameter block:

```python
raw_csv_path = '/content/RAW_Results.csv'
age_csv_path = '/content/Age.csv'
config_json_path = '/content/canmovesRunSpecConfig.json'
fuel_distribution_csv_path = '/content/Fuel Distribution_Basic.csv'
```

## Notes and Assumptions

- The raw output file must include `inode` and `jnode` columns. These are used to create `linkID` values.
- The raw output columns are expected to follow the pattern `<vehicle type> <pollutantID>`.
- The age distribution file must include `Vehicle Age` and columns for each vehicle type.
- Fuel percentages are normalized internally before they are applied.
- Tailpipe-by-fuel outputs are generated only for pollutants whose names contain `Tailpipe`.
- Energy values are reported in `Wh`, and mass values are reported in `g` in the processed outputs.
