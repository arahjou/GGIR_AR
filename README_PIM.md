# PIM (Proportional Integrating Measure) Support for GGIR

This document describes the additions made to GGIR to support pre-processed PIM data from actigraphy devices.

## Overview

Some actigraphy devices provide pre-processed activity metrics (like PIM) instead of raw accelerometer data (x, y, z). This feature allows GGIR to accept CSV files containing PIM data and process them through parts 2-6, bypassing the raw data processing in part 1.

## What Was Changed

### 1. R/convertEpochData.R

**Device Information (lines 144-150):**
```r
} else if (params_general[["dataFormat"]] == "pim_csv") {
  deviceName = "PIM_Device"
  monn = "pim"
  monc = 93
  dformc = 93
  epSizeShort = 0 # extract from file
}
```

**Data Reading Logic (lines 320-341):**
- Reads CSV files using `data.table::fread()`
- Auto-detects timestamp column (first column or column starting with "time")
- Auto-detects PIM column (case-insensitive search for "pim")
- Calculates epoch size automatically from timestamp differences
- Renames PIM column to `ExtAct` for GGIR compatibility

**Multi-column Support (line 467):**
- Added `"pim_csv"` to the `morethan1` list to support CSV files with additional columns

### 2. R/check_params.R

**Format Validation (line 505):**
- Added `"pim_csv"` to the list of external epoch data formats
- Disables raw-data metrics (ENMO, angle, MAD, etc.) since they cannot be calculated from PIM
- Sets `acc.metric = "ExtAct"`

### 3. inst/testfiles/pim_testfile.csv

Sample test file with 120 rows of 15-second epoch PIM data for testing purposes.

### 4. tests/testthat/test_convertExtEpochData.R

Added test case for `pim_csv` format validation.

## How to Use

### Input File Format

Your CSV file should have at minimum two columns:
1. **Timestamp column** - named "Timestamp", "Time", or similar (or just be the first column)
2. **PIM column** - named "PIM" (case-insensitive)

Example CSV format:
```csv
Timestamp,PIM
2024-01-15 08:00:00,125
2024-01-15 08:00:15,130
2024-01-15 08:00:30,142
2024-01-15 08:00:45,118
...
```

The epoch size is automatically detected from the time difference between consecutive rows.

### Basic Usage

```r
library(GGIR)

GGIR(
  datadir = "/path/to/your/pim/csv/files",
  outputdir = "/path/to/output",
  dataFormat = "pim_csv",
  extEpochData_timeformat = "%Y-%m-%d %H:%M:%S",  # Adjust to match your timestamp format
  mode = 2:5  # Skip part 1, run parts 2-5
)
```

### Important Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `datadir` | Directory containing your PIM CSV files | `"/data/pim_files"` |
| `outputdir` | Directory for GGIR output | `"/output"` |
| `dataFormat` | Must be set to `"pim_csv"` | `"pim_csv"` |
| `extEpochData_timeformat` | Format string for parsing timestamps | `"%Y-%m-%d %H:%M:%S"` |
| `mode` | Which GGIR parts to run (skip part 1) | `2:5` or `c(2,4,5)` |
| `desiredtz` | Timezone for timestamps | `"America/New_York"` |

### Common Timestamp Formats

| Format | Example | Format String |
|--------|---------|---------------|
| ISO 8601 | 2024-01-15 08:00:00 | `"%Y-%m-%d %H:%M:%S"` |
| US format | 01/15/2024 08:00:00 | `"%m/%d/%Y %H:%M:%S"` |
| EU format | 15/01/2024 08:00:00 | `"%d/%m/%Y %H:%M:%S"` |
| With milliseconds | 2024-01-15 08:00:00.000 | `"%Y-%m-%d %H:%M:%OS"` |

### Complete Example

```r
library(GGIR)

GGIR(
  # Input/Output
  datadir = "/Users/researcher/study/pim_data",
  outputdir = "/Users/researcher/study/output",

  # PIM-specific settings
  dataFormat = "pim_csv",
  extEpochData_timeformat = "%d/%m/%Y %H:%M:%S",

  # Processing settings
  mode = 2:5,
  desiredtz = "Europe/London",

  # Window sizes (in seconds)
  windowsizes = c(15, 900, 3600),  # 15-sec epochs, 15-min windows, 1-hour non-wear

  # Output settings
  do.report = c(2, 4, 5),
  visualreport = TRUE,

  # Verbosity
  verbose = TRUE
)
```

### Using convertEpochData Directly

For more control, you can call `convertEpochData()` directly:

```r
library(GGIR)

params_general <- load_params()$params_general
params_general[["dataFormat"]] <- "pim_csv"
params_general[["extEpochData_timeformat"]] <- "%Y-%m-%d %H:%M:%S"
params_general[["desiredtz"]] <- "UTC"
params_general[["windowsizes"]] <- c(15, 900, 3600)

convertEpochData(
  datadir = "/path/to/pim/files",
  metadatadir = "/path/to/output",
  params_general = params_general,
  verbose = TRUE
)
```

## Output

After processing, GGIR will create:

```
output_directory/
├── config.csv                    # Configuration used
├── meta/
│   └── basic/
│       └── meta_yourfile.csv.RData  # Processed milestone data
└── results/
    ├── QC/
    │   └── data_quality_report.csv
    ├── part2_summary.csv
    ├── part2_daysummary.csv
    ├── part4_nightsummary_sleep_cleaned.csv
    ├── part5_daysummary_*.csv
    └── ...
```

## Limitations

1. **No raw metrics**: Since PIM is already a processed metric, raw-data metrics like ENMO, MAD, and angle cannot be calculated.

2. **Sleep detection**: The vanHees2015 sleep algorithm (which requires angle data) is not available. Use alternative sleep detection methods.

3. **Single metric**: Currently only the PIM column is used as the activity metric (`ExtAct`).

## Troubleshooting

### "NA" timestamps
Ensure your `extEpochData_timeformat` matches your actual timestamp format exactly.

### "No files to process"
Check that your CSV files have the `.csv` extension and are in the specified `datadir`.

### Non-wear detection issues
The non-wear detection looks for periods with zero or near-zero activity. Adjust `windowsizes[3]` if needed.

## Technical Details

- **Monitor code**: 93
- **Format code**: 93
- **Internal metric name**: `ExtAct`
- **Supported file extension**: `.csv`

## Version

This feature was added to GGIR development version. For the official GGIR package, check if this feature has been merged.
