# 6. Scoring, Match Engine & Tags

## 6.1 Sub-scores (ETL, `etl/scoring.py`)

Each sub-score is computed **twice**, as a percentile (0–100, higher is better):

- **national**: ranked against all England OAs
- **regional**: ranked against OAs in the same ONS region (London, South East, North West, ...)

National percentiles squash cities: nearly all of London ranks above 90 for transport, so the colours stop telling areas apart. Regional percentiles restore the contrast. Users choose with a toggle (§6.2).

Distances are measured from the OA's **population-weighted centroid**.

| Score       | Method                                                                                                                |
| ----------- | --------------------------------------------------------------------------------------------------------------------- |
| Schools     | Sum of state schools within 2 km (primary) / 3 km (secondary), each weighted by inspection outcome and distance decay |
| Safety      | Inverse of recorded crimes per 1,000 residents over 24 months, smoothed over neighbouring OAs                         |
| Transport   | Distance to nearest rail/tube/tram stop combined with bus-stop count within 400 m                                     |
| Amenities   | Count of everyday POIs within 800 m, by category with diminishing returns                                             |
| Green space | Green-space area within 500 m plus distance to the nearest park                                                       |
| Deprivation | Inverse IMD 2025 rank of the OA's LSOA                                                                                |

**Minimum sample sizes (small-numbers rule).** An OA has only about 125 households, so raw medians and rates are noisy. The thresholds live in `metrics.yaml`.

- **House prices:**
  - Use the OA median for a property type only if it has at least **8 sales** in the last 24 months.
  - Otherwise roll up to the LSOA, then the MSOA.
  - If the MSOA still has too few, store `null` and show "not enough sales".
  - Always store `n` and `level` with the median.
- **Crime:**
  - Compute the rate at OA level only if the 24-month count is at least **20**.
  - Otherwise use the LSOA's rate, then the MSOA's.
  - Record `crime_level` and the count.
- **Display:**
  - Every number shown to users that comes from a sample carries its basis, e.g. "Median £725k · based on 14 sales in the wider neighbourhood".
  - Values that come from a rolled-up area are labelled as such.
  - The price filter and "price vs quality" ignore `null` medians and never treat them as zero.
- **Testing:** the publish gate (§4) checks that no displayed median uses fewer than the minimum sample.

**Overall score:** weighted mean of the sub-scores using default weights. Grades: `A` 80–100, `B` 65–79, `C` 50–64, `D` 35–49, `E` 0–34.

**Price vs quality indicator:** compare an area's overall-score percentile with its median-price percentile (same property type, last 24 months).

- Shown as "more / similar / less for your money".
- Never presented as investment advice.

## 6.2 Match engine (headline feature, runs in the browser)

- **Tiles carry the numbers.** Each OA feature in the PMTiles includes:
  - the national **and** regional sub-score percentiles
  - `flood_zone`, `surface_water_risk` and `nearest_station_m`
  - median prices by type, with their sample size (`n`)
- At low zoom, LSOA/MSOA layers carry population-weighted averages; OA polygons appear from about zoom 12.
- **Criteria model** (TypeScript, also serialised into the URL):
  ```ts
  type Criteria = {
    compareWithin: "england" | "region"; // which percentile set to score with
    filters: {
      maxFloodZone?: 1 | 2 | 3; // e.g. 1 = exclude zones 2 & 3
      maxSurfaceWaterRisk?: 0 | 1 | 2 | 3;
      maxPrice?: { type: "D" | "S" | "T" | "F"; price: number };
      maxStationDistanceM?: number;
      excludeConservationAreas?: boolean;
      // "Near places I care about": each is a hard filter, max 3 pins
      nearPlaces?: { label: string; lat: number; lng: number; maxKm: number }[];
    };
    weights: Partial<
      Record<
        | "schools"
        | "safety"
        | "transport"
        | "amenities"
        | "greenspace"
        | "deprivation",
        0 | 1 | 2 | 3 // not important .. essential
      >
    >;
  };
  ```
- **Scoring:**
  - `match = Σ(weight × percentile) / Σ(weight)`.
  - Areas that fail a hard filter are greyed out.
  - Build the formula as a MapLibre style expression (`case` + `interpolate`), so changing criteria recolours the map instantly with no API call.
- **"Compare within" toggle (England / my region):**
  - It switches which percentile properties the expression reads.
  - "Region" uses each OA's own region, so areas near a regional border are each ranked within their own region.
  - The legend and "why this colour?" panel state which comparison is in use.
- **"Near places I care about":**
  - Users drop up to 3 pins (work, family, a school) by search or tap, each with a maximum straight-line distance.
  - It's a cheap stand-in for commute time until isochrones arrive (§3), and runs entirely in the browser (`cost: none`).
  - **Implementation:**
    - Preferred: MapLibre's `distance` expression, which measures from each feature to the pin, so the filter lives in the style like the others. Confirm the installed MapLibre version supports `distance` on polygon layers.
    - Fallback: compute distances from each OA's population-weighted centroid (a tile attribute) in a Web Worker, then apply the result with `feature-state`.
  - Pins appear on the map with labelled distance rings. "Why this colour?" shows the distance to each pin.
- **Display:**
  - A colour-blind-safe sequential palette (e.g. viridis) in 5 bands, with a legend.
  - Tapping an area opens a "why this colour?" panel showing each criterion's contribution.
- **Ranked results list (map + list):**
  - A list view, a tab in the bottom sheet on mobile and a side panel on desktop, shows the **top 20 matching areas** sorted by match %.
  - Scope is either "in the current view" or "within X km of a searched place / pin".
  - Each row shows:
    - area name (nearest place + street)
    - match %
    - the 2–3 strongest and weakest criteria
    - median price for the user's property type, with its sample size
  - Tapping a row flies the map to that area and opens its panel.
  - It's computed in the browser from the loaded tile features (`queryRenderedFeatures` / `querySourceFeatures`) using the same scoring function as the map, so it's free (`cost: none`).
  - The list updates when criteria or the view change.
  - It's also the **accessible alternative to the map**: keyboard-navigable, screen-reader-labelled, and meeting WCAG 2.2 AA alongside the rest of the app.
- **One scoring function:** the map expression, the ranked list and "why this colour?" all use the same scoring logic in `packages/shared`. Tests assert they agree for the same criteria and feature.
- **Sharing:** criteria live in the URL query string. Saving searches and alerts are paid features (§9).

## 6.3 Area tags (rule-based, our own wording)

Tags are computed in the ETL and stored in `oa_metrics.tags`. Each tag's rule lives in `packages/shared/metrics.yaml` (§4). Examples:

- `great-high-street`: amenities ≥ 75
- `green-and-leafy`: green space ≥ 70
- `car-free-friendly`: transport ≥ 80, and Census households without a car above the national median
- `family-area`: households with dependent children in the top 30%
- `mostly-owner-occupied`: owned outright + mortgage ≥ 65%
- `flood-zone-3`: a warning tag, not a lifestyle tag

**Avoid stigmatising labels** such as "high unemployment". Present that kind of context neutrally in reports instead.

## 6.4 Protected characteristics

- Ethnicity (`TS021`), religion (`TS030`) and country of birth (`TS012`) are **never** stored in tiles, used as criteria or tags, or used in scoring.
- They are excluded from v1 entirely. If added later, they are neutral, report-only context, and that decision needs a deliberate review.
