# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Research code for the paper *Gallaher et al. (2024): Sustainability trade-offs across modeled floating solar waterscapes of the Northeastern United States.* It models floating photovoltaic (FPV) solar potential across 14 Northeastern US states + DC using a two-stage pipeline: GIS-based suitability screening followed by energy simulation.

## Running the Pipeline

Scripts must be executed in order. There is no build system or test suite — each script is a standalone processing step.

**Stage 1 — Pre-Processing (requires ArcGIS Pro + `arcpy`):**
```
Pre_Processing/FPV_NID_Preprocessing.py   # Clean & geocode NID dam data
Pre_Processing/NIW_Data_Cleaning.py        # Clean National Wetlands Inventory per state
Pre_Processing/Local_Roads.py              # Extract/merge DOT local roads by FIPS
Pre_Processing/NIW_NID_Join.py             # Spatial join: NHD waterbodies + NID + TNC + roads/transmission
```

**Stage 2 — Energy Analysis (requires PySAM + NREL API key):**
```
Pre_Processing/PySAM_weatherFiles.py       # Download NREL PSM3 weather files per location
FPV_Analysis/Solar_Estimation.py           # Run PVWatts v8 for each waterbody, output annual kWh
```

## Configuration Before Running

**All file paths are placeholders (`" "`) that must be filled in before running any script.** Look for comments like `## path to...` or `## set working directory`.

Key paths to configure per script:
- `FPV_NID_Preprocessing.py`: NID raw CSV path, ArcGIS geodatabase workspace
- `NIW_Data_Cleaning.py`: results folder from previous step, state wetland feature classes
- `Local_Roads.py`: local roads GIS data folder, study area boundary file, output folder
- `NIW_NID_Join.py`: workspace geodatabase, output location, local roads geodatabase path
- `PySAM_weatherFiles.py`: NREL API key + email, weather file storage directory, input CSV path
- `Solar_Estimation.py`: input CSV with locations + system sizes, SAM config JSON path, weather files folder, output CSV path

## Architecture

### Data Flow

```
NID (dams)       ──┐
NHD (waterbodies)──┤  FPV_NID_Preprocessing.py  →  Northeast_NID_Proj (GIS)
                   │  NIW_NID_Join.py             →  Northeast_Selection_2 (final suitable waterbodies)
NIW (wetlands)   ──┤  NIW_Data_Cleaning.py        →  Northeast_Wetlands (GIS, used as exclusion mask)
Local Roads      ──┤  Local_Roads.py              →  Northeast_local_Roads (GIS)
TNC Lakes/Ponds  ──┘
        │
        ▼ (CSV with lat/lon + system_size_kw + weather_file IDs)
        │
PySAM_weatherFiles.py  →  NREL PSM3 .csv weather files (one per location)
        │
Solar_Estimation.py    →  output CSV with Year1_Energy_kWh per waterbody
```

### Suitability Criteria (encoded in `NIW_NID_Join.py`)
- Waterbodies ≥ 2.5 acres (NHD)
- Excludes Swamp/Marsh, Estuary, Playa types
- Within 0.06 miles of a NID dam record
- Within 1 mile of transmission line or substation
- Within 0.5 miles of a local road

### Energy Simulation (`SAM_config.json`)
Template parameters for PVWatts v8: 100 MW system, fixed-tilt array (11°, south-facing), DC:AC ratio 1.15, ~14% losses. `solar_resource_file` and `system_capacity` are overridden per location at runtime in `Solar_Estimation.py`.

## Key Dependencies

- **arcpy** (ArcGIS Pro Python environment) — required for all `Pre_Processing/` scripts
- **PySAM** — Python wrapper for NREL System Advisor Model; `pip install NREL-PySAM`
- **pandas**, **numpy** — data manipulation
- **NREL API key** — register at https://developer.nrel.gov to use `PySAM_weatherFiles.py`

## Coordinate Reference Systems

- Raw inputs: WGS 1984 (EPSG:4326)
- NID/NHD processing: NAD 1983 UTM Zone 18N (EPSG:26918)
- Wetlands processing: USA Contiguous Albers Equal Area Conic (ESRI:102004)

## Study Area

14 states + DC: Maine, New Hampshire, Vermont, New York, Massachusetts, Rhode Island, Connecticut, New Jersey, Pennsylvania, Delaware, Maryland, West Virginia, Virginia, District of Columbia.
