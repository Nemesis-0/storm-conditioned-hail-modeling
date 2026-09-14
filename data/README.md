# Data Layout and Access

This repository does **not** distribute the raw meteorological datasets used in the analysis.

The notebooks expect locally available NOAA hail observations, NOAA MRMS radar data, ERA5 reanalysis fields, and a small set of legacy artifacts from the earlier summer workflow.

The `data/` directory is therefore used as a **local data interface**, not as a repository for committed raw data.

## Expected local structure

A working local setup follows the structure below:

```text
data/
├── README.md
│
├── hail_records/
│   └── Combined US Hail Data.csv
│
├── mrms/
│   ├── validation/
│   │   ├── active_20240418/
│   │   ├── active_20240508/
│   │   └── active_20240526/
│   │
│   └── expanded_active/
│       ├── expand_active_20240210_07/
│       ├── expand_active_20240502_21/
│       ├── expand_active_20240520_01/
│       ├── expand_active_20240523_10/
│       ├── expand_active_20240609_07/
│       └── expand_active_20240613_23/
│
├── era5/
│   ├── pilot_2024/
│   │   ├── active_20240418_single.nc
│   │   ├── active_20240418_pressure.nc
│   │   ├── active_20240508_single.nc
│   │   ├── active_20240508_pressure.nc
│   │   ├── active_20240526_single.nc
│   │   └── active_20240526_pressure.nc
│   │
│   └── quick_expanded/
│       ├── expand_active_20240210_07_single.nc
│       ├── expand_active_20240210_07_pressure.nc
│       ├── expand_active_20240502_21_single.nc
│       ├── expand_active_20240502_21_pressure.nc
│       ├── expand_active_20240520_01_single.nc
│       ├── expand_active_20240520_01_pressure.nc
│       ├── expand_active_20240523_10_single.nc
│       ├── expand_active_20240523_10_pressure.nc
│       ├── expand_active_20240609_07_single.nc
│       ├── expand_active_20240609_07_pressure.nc
│       ├── expand_active_20240613_23_single.nc
│       └── expand_active_20240613_23_pressure.nc
│
└── summer_pipeline/
    └── local legacy artifacts used by the sample-frame audit
```

The exact contents of `summer_pipeline/` depend on the earlier workflow and are intentionally not distributed as part of this research branch. Notebook 01 documents how those artifacts are interpreted.

## NOAA hail observations

The notebooks use a locally prepared hail-report table:

```text
data/hail_records/Combined US Hail Data.csv
```

The analysis expects the file to contain, at minimum, fields corresponding to:

- hail-event time;
- event time-zone information;
- latitude;
- longitude.

The current workflow converts reported local event times to UTC before temporal matching.

Hail observations are used **only after** the radar-based storm state has been independently constructed.

This ordering is important to the storm-first design:

```text
MRMS storm state
        ↓
NOAA hail overlay
```

rather than defining storm candidates from hail reports themselves.

The raw hail-report source file is not committed to this repository.

## NOAA MRMS radar data

The radar analysis uses the NOAA MRMS composite-reflectivity product:

```text
MergedReflectivityQCComposite_00.50
```

Local files are stored as compressed GRIB2 files:

```text
*.grib2.gz
```

The final storm-proxy implementation:

- treats both `-99` and `-999` as unavailable values;
- aggregates the native MRMS grid to nominal 0.25-degree cells;
- requires at least 80% valid fine-pixel coverage within a scan-cell;
- uses a 35-dBZ reflectivity threshold;
- divides reflectivity support by the full nominal 625-pixel coarse cell;
- defines strict spatial support as at least 10% of the coarse cell;
- requires persistence across at least 5 scans;
- requires at least 80% usable scans within the radar window.

These choices are documented in more detail in:

```text
docs/methodology.md
```

and in Notebooks 02–05.

### Cross-midnight windows

Radar windows may cross UTC midnight.

For example, the `expand_active_20240613_23` period requires scans from both:

```text
2024-06-13
2024-06-14
```

The corrected Notebook 05 MRMS helper therefore queries every UTC calendar date touched by the requested interval.

A valid local dataset must contain all scans needed by the corresponding pre-origin and future windows.

## ERA5 reanalysis

ERA5 fields are stored locally as NetCDF files.

Two files are used for each development period:

```text
<period_id>_single.nc
<period_id>_pressure.nc
```

The final compact predictor set uses the following ERA5 variables.

### Single-level fields

- 2-m temperature
- 2-m dew point
- CAPE

### Pressure-level fields

The final model uses fields required to derive:

- 500-hPa temperature;
- 850–500-hPa wind shear;
- 850–300-hPa wind shear.

ERA5 fields are aligned by **valid time**.

They are retrospective reanalysis products and should not be interpreted as operationally available real-time forecast predictors.

## Development periods

Notebook 05 uses nine event-enriched 2024 periods.

### Primary periods

```text
active_20240418
active_20240508
active_20240526
```

### Expanded periods

```text
expand_active_20240210_07
expand_active_20240502_21
expand_active_20240520_01
expand_active_20240523_10
expand_active_20240609_07
expand_active_20240613_23
```

These periods were selected for methodological development.

They do **not** constitute a representative climatological sample.

## Local symlinks

On the development machine, some entries under `data/` may be symbolic links to larger datasets stored elsewhere.

That is acceptable for local use.

Symlinks and raw-data directories are local-only implementation details and should not be committed as substitutes for the underlying public datasets.

A public clone of this repository is therefore expected to contain only:

```text
data/README.md
```

until the user places or links the required datasets locally.

## Raw data and Git

Large meteorological files should remain outside version control.

At minimum, the repository `.gitignore` should exclude:

```text
*.grib2
*.grib2.gz
*.nc
```

as well as local raw-data directories and system-generated files such as:

```text
.DS_Store
.ipynb_checkpoints/
__pycache__/
```

Derived research tables under:

```text
outputs/tables/
```

may be committed when they are small and are intended as part of the public research artifact.

## Reproducing the notebooks

After preparing the local datasets, notebooks should be run in numerical order:

```text
01_summer_sample_frame_audit.ipynb
02_mrms_storm_proxy_feasibility.ipynb
03_storm_first_sample_construction.ipynb
04_direct_vs_storm_conditioned_modeling.ipynb
05_temporally_aligned_hierarchical_modeling.ipynb
```

Not every notebook regenerates every upstream dataset.

In particular:

- Notebook 01 audits legacy sample-construction artifacts;
- Notebook 02 evaluates the MRMS storm proxy;
- Notebook 03 constructs the retrospective storm-first sample;
- Notebook 04 evaluates the retrospective direct-versus-conditioned formulation;
- Notebook 05 builds the corrected temporally ordered development panel and performs the final same-\(X\) Direct-versus-Hierarchical comparison.

The public repository is designed to document the complete methodology and derived results while keeping large raw meteorological datasets outside Git.