# 2. v1 Data Sources (easy tier only)

All of these are free bulk downloads under the Open Government Licence (OGL) or the Open Database License (ODbL, for OSM). The exception is HM Land Registry's INSPIRE polygons, which are free but come with their own licence (see the boundaries note below). Source data arrives in `EPSG:27700` (British National Grid) or `EPSG:4326`. The ETL reprojects everything to `EPSG:4326` for storage and tiles, and uses `EPSG:27700` for distance and area calculations.

| Theme              | Dataset                                                                                                                             | Format               | Refresh    |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | -------------------- | ---------- |
| Areas              | ONS Output Area / LSOA / MSOA 2021 boundaries (Open Geography Portal, generalised clipped) and OA population-weighted centroids     | GeoPackage / GeoJSON | Static     |
| Postcodes & search | ONS Postcode Directory (ONSPD), plus postcodes.io for live geocoding                                                                | CSV / free API       | Quarterly  |
| House prices       | HM Land Registry Price Paid Data, last ~10 years, category A (standard) sales only                                                  | CSV                  | Monthly    |
| Crime              | data.police.uk bulk archive, rolling 24 months, aggregated to OA × month × category                                                 | CSV                  | Monthly    |
| Schools            | DfE _Get Information About Schools_ (location, phase, type, status) plus the Ofsted monthly management-information CSV              | CSV                  | Monthly    |
| Transport          | NaPTAN (rail, tube, tram/metro and bus stops)                                                                                       | CSV                  | Monthly    |
| Amenities          | OpenStreetMap England extract (Geofabrik) → POIs: supermarkets, cafés, restaurants, pubs, gyms, GPs, pharmacies, parks, playgrounds | PBF                  | Monthly    |
| Green space        | OS Open Greenspace                                                                                                                  | GeoPackage           | ~6-monthly |
| Streets            | OS Open Roads (named road links)                                                                                                    | GeoPackage           | ~6-monthly |
| Flood              | Environment Agency _Flood Map for Planning_: Flood Zones 2 & 3; _Risk of Flooding from Surface Water_ (extent bands)                | GeoPackage / SHP     | Quarterly  |
| Deprivation        | English Indices of Deprivation 2025 (LSOA)                                                                                          | CSV                  | Static     |
| Demographics       | Census 2021 by OA (Nomis bulk)                                                                                                      | CSV                  | Static     |
| Conservation areas | planning.data.gov.uk conservation-area dataset                                                                                      | GeoJSON              | Monthly    |
| Property boundaries | HM Land Registry _INSPIRE Index Polygons_ (registered freehold extents), one zip per local authority                              | GML                  | Monthly    |

**Census 2021 tables used:**

- `TS001` Usual residents (population, used for weighting)
- `TS003` Household composition (families with children, one-person households)
- `TS044` Accommodation type (detached, semi, terrace, flat)
- `TS045` Car or van availability
- `TS054` Tenure (owned outright, mortgage, private rented, social rented)
- `TS066` Economic activity
- `TS067` Highest qualification
- `TS062` NS-SeC (occupation class)

**Data notes:**

- **Schools.** Ofsted replaced single-word judgements (Sept 2024) with multi-area report cards (from Nov 2025). The v1 school score:
  - is based on distance to state primary and secondary schools,
  - weights each school by its latest report-card outcome where one exists, and otherwise by its last graded judgement,
  - is flagged in the code for review as report-card coverage grows.
  - DfE KS2/KS4 performance data is a later addition (§3).
- **Crime.** police.uk locations are anonymised and snapped to "on or near" points, and some forces have upload gaps. Show crime as rates relative to population, label it "recorded crime", and add a coverage warning wherever a force's data is missing.
- **Transport.** The v1 score combines walking distance to the nearest rail, tube or tram stop with bus-stop density within 400 m. It stands in for commute time until isochrones arrive (§3).
- **Distance.** Use straight-line distance in v1 and label it "as the crow flies". Walking-network distance is on the roadmap.
- **Property boundaries (INSPIRE).**
  - **What the data is:** roughly 24M polygons for England & Wales.
    - Each polygon is the indicative extent of a _registered freehold_. They are "general boundaries", not legal ones.
    - Flats and leaseholds share their freehold's polygon.
    - Unregistered land (roughly 10–15% of England) has no polygon.
    - The free data carries only an INSPIRE ID: no title number, address or owner.
  - **Licence:** it's HMLR's INSPIRE licence, not plain OGL.
    - The attribution wording it requires (HM Land Registry and Ordnance Survey Crown copyright) must be shown on the map and in reports.
    - Read the current terms before launch to check the planned uses (display, plot-area calculation, reports) are allowed.
  - **Pipeline:** download the per-council zips (England only in v1), convert them with `ogr2ogr`, and compute `area_m2` in EPSG:27700. The polygons go into **their own PMTiles file**, `plots-{yyyymm}.pmtiles`.
  - **Display:** outlines only, shown from **zoom 16** (tile maxzoom 16, overzoomed beyond that). The layer is hidden below zoom 16 so the map stays fast and tiles stay small.
  - **Tap a plot** to highlight it and show its area in m² and acres, with a "property report for this plot" action.
  - **Not loaded into Postgres in v1.** The report takes the tapped point and plot ID, and the area is read from the tile attributes.
  - **UI wording:** always label plots as "indicative boundaries". Never call them legal boundaries or ownership.
