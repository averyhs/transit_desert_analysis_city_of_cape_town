# Methods: Transit Equity and Access Analysis - City of Cape Town

This notebook documents the complete methodological workflow used to identify transit equity and access in the City of Cape Town. It is intended to ensure the analysis is transparent and reproducible. 

---

## 1. Study Area

The City of Cape Town (CoCT) metropolitan municipality forms the study area. The municipal boundary was obtained from [GeoBoundaries](https://www.geoboundaries.org/index.html#getdata). All spatial data were clipped to this boundary for subsequent analysis.


## 2. Data Sources and Acquisition

### 2.1 Census/Household Survey Data
- **Source:** [City of Cape Town Open Data Portal - 2024 Household Survey](https://odp-cctegis.opendata.arcgis.com/datasets/75ebd83593e24f1390cceb07ab234bb6_0/explore) (Demography and Statistics)
- **Acquired:** 2024
- **Format:** CSV
- **Records:** 8,318 households, 140 fields
- **Purpose:** Derive indicators of transit need (income/poverty proxy, vehicle access, pension receipt, household composition)

### 2.2 Public Transport Infrastructure
- **MyCiTi bus stops:** [CoCT Open Data Portal - Integrated Rapid Transit (IRT) system MyCiTi Bus Stops](https://odp-cctegis.opendata.arcgis.com/datasets/19684ea3f08b4c95ab4fe7099ed75e81_10/explore) (downloaded 24 September 2026)
- **Railway stations:** [CoCT Open Data Portal - Railway Stations](https://odp-cctegis.opendata.arcgis.com/datasets/e35733985e48458cbc8836dc640b4ee9_6/explore) (downloaded 24 September 2026)
- **Minibus taxi routes:** [CoCT Open Data Portal - Taxi Routes](https://odp-cctegis.opendata.arcgis.com/datasets/2efbfd905a974d50be57c8641a2ea171_11/explore) (downloaded 24 September 2026)
- **Golden Arrow bus stops:** [Western Cape Government Spatial Data Warehouse - Transportation](https://gis.westerncape.gov.za/server2/rest/services/SpatialDataWarehouse) (downloaded 24 September 2026)

All infrastructure datasets were downloaded in geospatial formats (SHP/GeoJSON), clipped to the CoCT municipal boundary, and reprojected for distance-based analysis.


### 2.3 Other Spatial Data
- **Suburb boundaries:** Extracted from OpenStreetMap (via QuickOSM queries for place=suburb) within the study area. Where polygon representations were missing but point locations existed, polygons were manually digitised for the purposes of this analysis using OSM labels and Google Maps as reference.
- **Base map:** OpenStreetMap basemap for cartographic context.
- **Municipal boundary:** City of Cape Town extracted from GeoBoundaries ADM2 for South Africa.

---


## 3. Data Preparation

### 3.1 Coordinate Reference System (CRS)
All spatial layers were reprojected to **Hartebeesthoek94 Lo19 E-N (ESRI:102562)** to enable accurate Euclidean distance calculations in metres. Source data were primarily in WGS84 (EPSG:4326).

### 3.2 Suburb Data Cleaning and Alignment
- The 2024 Household Survey records suburbs as named entities (some combined with slashes, e.g. "De Waterkant/Green Point/Sea Point").
- OSM suburb names were normalised (whitespace trimmed) and compared against survey suburb names to identify mismatches and missing geometries.
- Combined suburbs in the survey were matched to corresponding spatial features; where necessary, multiple OSM polygons were merged in QGIS to align with the survey naming convention.
- Suburbs present in the survey but missing from OSM polygon/point coverage were manually digitised (guided by OSM labels and Google Maps). A small set of survey entries were excluded from spatial mapping where they could not be reliably located for the analysis.

### 3.3 Transport Infrastructure Processing
- **MyCiTi bus stops, railway stations:** Point layers clipped to CoCT municipal boundary and reprojected.
- **Golden Arrow bus stops:** Retrieved from Western Cape Government Spatial Data Warehouse. Data quality notes: on-board survey tracking produces dense point clusters along routes in places; these points do not consistently correspond one-to-one with official stops. This introduces uncertainty into the Golden Arrow coverage estimates.
- **Minibus taxi routes:** Line layer clipped and reprojected.


---
## 4. Defining Walking Accessibility (Supply)

A 500 m walking distance threshold was adopted (approximately 0.25 miles), consistent with common transit accessibility practice. Buffers were computed as follows:
- **Point-based stops (MyCiTi, Golden Arrow, railway stations):** 500 m Euclidean buffer around each stop point. Overlapping buffers were dissolved to create a single continuous access area per mode.
- **Route-based (minibus taxis):** 500 m buffer around taxi route lines, dissolved to represent areas within walking distance of the taxi network.

The dissolved buffer layers were used to calculate coverage relative to suburb polygons.

### Limitations noted
- Golden Arrow stop locations carry known positional uncertainty (section 3.3).
- Euclidean (as-the-crow-flies) buffers do not account for pedestrian network connectivity, barriers (e.g. highways, rail lines, fenced areas), or safety considerations. These may overestimate actual walking access in some locations.


---
## 5. Measuring Transit Coverage per Suburb

Coverage was computed as the percentage of each suburb polygon area intersected by the mode-specific 500 m access buffer:

1. **Overlap analysis:** For each transport mode (MyCiTi, Golden Arrow, taxis, trains), Vector Analysis → Overlap Analysis was used to determine the proportion of suburb area covered by the dissolved buffer.
2. **Joining to suburbs:** The resulting overlap percentages were joined to the suburb layer attribute table (as fields `myciti_bus_coverage_pc`, `ga_bus_coverage_pc`, `taxi_coverage_pc`, `train_coverage_pc`).
3. **Combined measures:**
   - **Bus + Train:** Buffers for all bus stops (MyCiTi ∪ Golden Arrow) and train stations were merged and dissolved; overlap percentages computed and joined as `bus_and_train_coverage_pc`.
   - **All public transit:** Buffers for all modes (buses ∪ trains ∪ taxis) were merged and dissolved; overlap percentages computed and joined as `all_public_transit_coverage_pc`.
4. **Persistence:** The suburb layer with all joined coverage fields was exported as `suburbs_with_public_transit_coverage.gpkg`.

Coverage values range from 0% (no part of suburb within 500 m of the mode) to 100% (entire suburb within 500 m).


---
## 6. Measuring Transit Need per Suburb

Transit need was operationalised as a weighted composite of four indicators derived from the 2024 Household Survey, following the project specification:

$$
\text{Transit Need} = 0.4 \times (\text{Poverty}) + 0.4 \times (\text{No Vehicle}) + 0.1 \times (\text{Elderly}) + 0.1 \times (\text{Under 18})
$$

### 6.1 Indicator definitions

- **Poverty:** Percentage of surveyed households in the suburb with estimated per-adult monthly income ≤ R1,109 (reference: South African poverty lines ~R1,109–R1,603 per individual/month in 2024). Household income (`hhincome`) was provided in ranges and included "No income" and "Don't know" as values. Range midpoints were assigned; "No income" was set to 0; non-numeric/unusable values ("Don't know", "1", "0") were treated as missing. Adult population per household was calculated as `hhadults = hhsize - hhchildren`. A household was classified as below poverty line if `(hhincome_num / hhadults) <= 1109`. Percentages use valid responses as the denominator.

- **No Vehicle:** Percentage of surveyed households without a private vehicle, derived from `hhasset_vehicle` ("No" → 1, "Yes" → 0).

- **Elderly:** Percentage of surveyed households receiving pension income (`income_pension` = "Yes"), used as a proxy for elderly residents. "Refused to answer" was treated as missing.

- **Under 18:** Percentage of total surveyed population in the suburb under 18, computed as `(sum of hhchildren) / (sum of hhsize)`.

### 6.2 Aggregation
Household-level indicators were aggregated to suburb level. Survey weights were not re-applied beyond the provided survey design; results reflect sample proportions within each suburb.

### 6.3 Output
Aggregated need indicators and the composite score were joined to suburb geometries and exported as `suburbs_with_transit_need_score.gpkg`.



---
## 7. Bivariate Classification (Need vs Coverage)

To identify areas of potential oversupply/undersupply, suburbs were classified using a 3×3 bivariate scheme based on:
- **x-axis:** Transit need (composite score), split into terciles (low, medium, high). 
- **y-axis:** Transit coverage (percentage of suburb area within 500 m of the selected transport mode/combination), split into terciles (low, medium, high).

The classification was applied independently for each transport mode/layout (MyCiTi, Golden Arrow, taxis, trains, bus+train, all public transit). It is noted that because terciles are computed per layer’s distribution, colour class boundaries differ between maps and cannot be directly compared across modes. The style was defined in QGIS and saved as QML for consistent application.

Bivariate choropleth styling used the following palette (from the project styling):
```
#64acbe  #627f8c  #574249
#b0d5df  #ad9ea5  #985356
#e8e8e8  #e4acac  #c85a5a
```



---
## 8. Cartographic Outputs

Map layouts were produced in QGIS 3.x for each scenario:
- `layout_transit_need.pdf`: Spatial distribution of transit need scores
- `layout_stops_and_buffer_train_and_bus.pdf`: Transport stops and 500 m buffers (trains and buses)
- `layout_stops_and_buffer_taxi.pdf`: Taxi routes and 500 m buffers
- `layout_transit_coverage_and_need_train.pdf`: Bivariate (need vs train coverage)
- `layout_transit_coverage_and_need_myciti_bus.pdf`: Bivariate (need vs MyCiTi coverage)
- `layout_transit_coverage_and_need_ga_bus.pdf`: Bivariate (need vs Golden Arrow coverage)
- `layout_transit_coverage_and_need_taxi.pdf`: Bivariate (need vs taxi coverage)
- `layout_transit_coverage_and_need_all.pdf`: Bivariate (need vs all public transit coverage)

For each layout, appropriate map themes were created and applied to ensure correct layers appeared in the layout for each mode.



---
## 9. Software and Tools

- **QGIS 4.x**: Spatial data processing, buffering, overlap analysis, joins, styling, and map layout/export
- **Python (pandas, numpy)**: Tabular data cleaning, transformation, and aggregation of survey data
- **Jupyter Notebooks**: Methodological documentation and step-by-step processing
- **QuickOSM (QGIS plugin)**: Extraction of OSM suburb data

---
## 10. References and acknowledgements

Data:
- City of Cape Town Open Data Portal. 2024 Household Survey. Available: https://odp-cctegis.opendata.arcgis.com/datasets/75ebd83593e24f1390cceb07ab234bb6_0/explore
- City of Cape Town Open Data Portal. MyCiTi Bus Stops. Available: https://odp-cctegis.opendata.arcgis.com/datasets/19684ea3f08b4c95ab4fe7099ed75e81_10/explore
- City of Cape Town Open Data Portal. Railway Stations. Available: https://odp-cctegis.opendata.arcgis.com/datasets/e35733985e48458cbc8836dc640b4ee9_6/explore
- City of Cape Town Open Data Portal. Taxi Routes. Available: https://odp-cctegis.opendata.arcgis.com/datasets/2efbfd905a974d50be57c8641a2ea171_11/explore
- Western Cape Government Spatial Data Warehouse. Transportation - Golden Arrow Bus Services Stops. Available: https://gis.westerncape.gov.za/server2/rest/services/SpatialDataWarehouse
- GeoBoundaries. Administrative boundaries (ADM2). Available: https://www.geoboundaries.org/index.html#getdata
- Statistics South Africa. 2024 Poverty Lines. Available: https://www.statssa.gov.za/publications/P03101/P031012024.pdf

Guides:
- Learn GIS. GIS Portfolio Projects. Available: https://learn-gis-chi.vercel.app/projects/
- Joshua Stevens. Bivariate Choropleth Maps: A How-to Guide. Available: https://www.joshuastevens.net/cartography/make-a-bivariate-choropleth-map/
- GeoAwesome. Understanding Bivariate Maps: A How-to Guide. Available: https://geoawesome.com/understanding-bivariate-maps-a-how-to-guide/

