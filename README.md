# Transit Desert Analysis - City of Cape Town

## Project Overview

This project analyses transit deserts in the City of Cape Town by comparing public transport accessibility (supply) against indicators of transit need. Transit deserts are areas with high transit need and low transit access, highlighting where improved public transport would most benefit vulnerable populations.

The analysis uses data from:
- **City of Cape Town (CoCT) Open Data Portal**: 2024 Household Survey, MyCiTi bus stops, railway stations, and minibus taxi routes
- **OpenStreetMap**: Suburb boundaries and base map data
- **GeoBoundaries**: Municipal boundary data

## Repository Structure

```
transit_desert_analysis_city_of_cape_town/
├── data/                          # Input and processed spatial/tabular data
├── design/                        # Design assets and styling
├── notebooks/                     # Additional notebooks
├── notebooks (Copy)/              # Development notebooks (data collection, map prep)
├── outputs/                       # Map layouts (PDF exports)
├── transit_desert_cape_town.qgz   # QGIS project
└── README.md                     # Project documentation
```

### Key Data Files

- **2024_Household_Survey_-4393438653176099798.csv**: CoCT 2024 Household Survey data (8,318 households, 140 fields)
- **suburbs_with_transit_need_score.gpkg**: Suburb polygons with computed transit need indicators
- **suburbs_with_public_transit_coverage.gpkg**: Suburb polygons with transit coverage percentages by mode
- **coverage_*.gpkg**: Coverage analysis for each transport mode (MyCiTi, Golden Arrow, taxi, train, combined)
- **Transport infrastructure (clipped/reprojected)**: Bus stops, train stations, and taxi routes for Cape Town municipality

## Transit Need Index

Transit need is calculated at the suburb level using four weighted indicators from the 2024 Household Survey:

$$
\text{Transit Need} = 0.4 \times \text{Poverty} + 0.4 \times \text{No Vehicle} + 0.1 \times \text{Elderly} + 0.1 \times \text{Under 18}
$$

Where each component represents the percentage of households/population in the suburb meeting the criterion:

- **Poverty** (40%): Households with per-adult monthly income below the South African upper-bound poverty line (R1,109 per adult/month, based on 2024 StatsSA poverty lines)
- **No Vehicle** (40%): Households without access to a private vehicle (`hhasset_vehicle = "No"`)
- **Elderly** (10%): Households with at least one pension income recipient (`income_pension = "Yes"`)
- **Under 18** (10%): Percentage of population under 18 years (`hhchildren` / `hhsize`)

## Transit Accessibility (Supply)

Accessibility is measured as the percentage of each suburb's area within walking distance of public transport infrastructure. A walking distance of **500 m (0.5 km)** is used throughout this analysis.

Transport modes considered:
- **MyCiTi buses**: Formal rapid transit bus stops (from CoCT Open Data Portal)
- **Golden Arrow buses**: Scheduled bus stops (from Western Cape Government Spatial Data Warehouse)
- **Minibus taxis**: Taxi route network (routes buffered to represent stop access along routes)
- **Trains**: Railway stations (from CoCT Open Data Portal)

Combined measures include:
- **Bus + Train coverage**: Areas within 500 m of any bus stop (MyCiTi or Golden Arrow) or train station
- **All public transit coverage**: Areas within 500 m of any public transport access point (bus, train, or taxi)

## Bivariate Analysis

Bivariate choropleth maps classify suburbs into 3×3 categories based on transit need and coverage (both split into low/medium/high terciles). This visualisation identifies transit deserts (high need, low coverage) and areas of mismatch between need and supply.

## Limitations

- **Golden Arrow bus stops**: Point locations from on-board survey tracking may not represent all official stop locations, potentially affecting coverage estimates.
- **Network connectivity**: Buffers do not account for physical barriers (highways, fenced areas, unsafe crossings) that may prevent actual walking access.
- **Need indicators**: The 'elderly' proxy (households receiving pension income) may not capture all elderly residents; poverty is estimated using household income divided by number of adults.
- **Spatial aggregation**: Analysis is conducted at suburb level; within-suburb variation in need and access is not captured.

## Outputs

The `outputs/` directory contains PDF map layouts showing:
- Transit need distribution across Cape Town suburbs
- Bivariate analysis for each transport mode (MyCiTi, Golden Arrow, taxis, trains)
- Combined coverage (bus+train, all public transit) against transit need
- Overview maps showing transport stop locations and buffers

## Reproducibility

The analysis workflow is documented in the Jupyter notebooks:
1. `data_collection_and_preparation.ipynb`: Data acquisition, cleaning, and preprocessing (survey data cleaning, spatial data processing, identification of missing suburbs)
2. `map_prep.ipynb`: Spatial analysis, computation of coverage buffers and overlap percentages, transit need calculation, and QGIS map preparation

All processing steps, data sources with URLs and access dates, and methodological decisions are recorded in these notebooks to support reproducibility.

## Requirements

- **QGIS 3.x** for spatial analysis and cartography
- **Python 3.x** with pandas, numpy, geopandas (for notebook analysis)
- CRS used: Hartebeesthoek94 Lo19 E-N (ESRI:102562) for distance-based calculations; WGS84 (EPSG:4326) for source data
