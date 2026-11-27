# Project Blueprint: UK Neighbourhood & Street Intelligence Platform

## 1. Execution & Tooling Guidelines

**Environment:**
- You are running in Claude Code CLI. Create directories and files directly, run shell commands to verify migrations, Python scripts, and Next.js builds, and maintain a `PROGRESS.md` checklist in the root directory after completing each phase.

**Managed Services Only (No Local Docker):**
- Do not create a local Docker database setup. Connect directly to a managed **Supabase** Postgres + PostGIS instance using `DATABASE_URL` in `.env.local`.

**Supabase Ecosystem:**
- Use **Supabase Auth** (for Pro tier authentication).
- Use **Supabase Storage** (bucket: `street-reports` for caching generated PDF reports).

**Map Tile Hosting:**
- Configure MapLibre GL JS to load `.pmtiles` via `NEXT_PUBLIC_PMTILES_URL` pointing to **Cloudflare R2** (defaulting to `/tiles/liveability.pmtiles` in `public/tiles/` during local development).

**Architecture Constraint:**
- Keep map rendering on MapLibre GL JS native layers + `pmtiles`, structured cleanly so `@deck.gl/mapbox` (`MapboxOverlay` with `interleaved: true`) can be added later without refactoring map state.

---

## 2. Project Overview & Tech Stack

**Objective:**
Build a full-stack geospatial web application for England & Wales (similar to Liveable.co.uk) that helps homebuyers evaluate streets and neighbourhoods through interactive heatmaps, custom catchment drawing, place comparisons, and downloadable multi-page PDF Street Reports.

**Frontend:**
- Next.js 15 (App Router)
- TypeScript
- Tailwind CSS
- Shadcn/UI
- Recharts (demographic & property charts)

**Map & Drawing Engine:**
- `maplibre-gl`
- `pmtiles`
- `terra-draw` + `terra-draw-maplibre-gl-adapter` (supporting Freehand, Polygon, and Circle search modes)

**Tile Generation & Storage:**
- `tippecanoe` CLI to compile GeoJSON into `.pmtiles`
- Uploaded to Cloudflare R2

**Backend & Database:**
- Supabase (PostgreSQL 16 + PostGIS extension)
- Drizzle ORM (with raw SQL helpers for PostGIS spatial queries)
- Supabase Auth
- Supabase Storage

**Data Pipeline (ETL):**
- Python 3.11+ (`geopandas`, `pandas`, `duckdb`, `psycopg2-binary`, `shapely`, `boto3` for R2 uploads)

**PDF Generation:**
- Playwright (headless Chromium) rendering `/reports/print/[streetId]` to A4 PDF and caching in Supabase Storage

---

## 3. Environment Variables (`.env.local.example`)

Generate an `.env.local.example` file containing:

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
DATABASE_URL=postgresql://postgres:[PASSWORD]@db.[PROJECT].supabase.co:5432/postgres

# Cloudflare R2 (PMTiles Hosting)
NEXT_PUBLIC_PMTILES_URL=/tiles/liveability.pmtiles
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=map-tiles

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 4. Core UK Open Data Sources (England & Wales)

Design the Python ETL pipeline (`etl/`) to ingest and reproject datasets from `EPSG:27700` (British National Grid) to `EPSG:4326` (WGS84):

1. **Boundaries & Streets:**
   - ONS Output Areas (OA) & LSOA boundaries
   - OS Open Roads / OS Open USRN (street geometries)
   - OS Open Greenspace

2. **House Prices:**
   - HM Land Registry Price Paid Data (CSV)
   - Geocoded via ONS Postcode Directory (ONSPD)

3. **Crime:**
   - data.police.uk street-level crime CSVs

4. **Schools & Transport:**
   - DfE/Ofsted school ratings + coordinates
   - NaPTAN (National Public Transport Access Nodes) for rail/tube/bus stops

5. **Amenities:**
   - OpenStreetMap (OSM) POI extracts (pubs, restaurants, cafes, supermarkets, gyms)

6. **Demographics (ONS Census 2021 by OA):**
   - `TS054`: Tenure (Owned outright, Mortgage, Private rented, Social rented)
   - `TS066`: Economic activity (Full-time, Part-time, Retired, Unemployed)
   - `TS067`: Highest level of qualification (Degree+, A-Level, GCSE, None)
   - `TS062`: NS-SeC Occupation (Higher/Lower managerial, Intermediate, Routine)
   - `TS021`: Ethnic group
   - `TS012`: Country of birth
   - `TS030`: Religion

---

## 5. Database Schema (`supabase/migrations/0001_init_postgis.sql`)

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

-- 1. Census Output Areas (OAs) with pre-computed metrics & percentiles
CREATE TABLE output_areas (
    oa_code VARCHAR(12) PRIMARY KEY,
    lsoa_code VARCHAR(12),
    geom GEOMETRY(MultiPolygon, 4326),
    liveability_score NUMERIC(5,2),
    grade CHAR(1), -- 'A', 'B', 'C', 'D'
    amenity_percentile NUMERIC(5,2),
    school_percentile NUMERIC(5,2),
    transport_percentile NUMERIC(5,2),
    crime_percentile NUMERIC(5,2),
    greenspace_percentile NUMERIC(5,2),
    demographics JSONB,
    archetypes JSONB
);
CREATE INDEX idx_oa_geom ON output_areas USING GIST (geom);

-- 2. Named Streets (Linked to intersecting OA)
CREATE TABLE streets (
    id SERIAL PRIMARY KEY,
    usrn VARCHAR(20),
    name VARCHAR(255) NOT NULL,
    town VARCHAR(100),
    postcode_sector VARCHAR(10),
    oa_code VARCHAR(12) REFERENCES output_areas(oa_code),
    geom GEOMETRY(MultiLineString, 4326),
    centroid GEOMETRY(Point, 4326)
);
CREATE INDEX idx_streets_geom ON streets USING GIST (geom);
CREATE INDEX idx_streets_centroid ON streets USING GIST (centroid);

-- 3. Towns & Suburbs (For area search & nearby comparisons)
CREATE TABLE places (
    id SERIAL PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    slug VARCHAR(150) UNIQUE NOT NULL,
    place_type VARCHAR(50), -- 'town', 'suburb', 'village'
    geom GEOMETRY(Point, 4326),
    avg_liveability_score NUMERIC(5,2),
    total_coverage_percentile NUMERIC(5,2),
    amenity_percentile_rank NUMERIC(5,2),
    school_percentile_rank NUMERIC(5,2),
    transport_percentile_score NUMERIC(5,2),
    labels JSONB
);
CREATE INDEX idx_places_geom ON places USING GIST (geom);

-- 4. Land Registry Property Sales
CREATE TABLE property_sales (
    transaction_id VARCHAR(50) PRIMARY KEY,
    price INTEGER NOT NULL,
    transfer_date DATE NOT NULL,
    postcode VARCHAR(10),
    property_type CHAR(1), -- 'D' Detached, 'S' Semi, 'T' Terraced, 'F' Flat
    paon VARCHAR(100),
    saon VARCHAR(100),
    street VARCHAR(255),
    geom GEOMETRY(Point, 4326)
);
CREATE INDEX idx_sales_geom ON property_sales USING GIST (geom);
CREATE INDEX idx_sales_date ON property_sales (transfer_date);

-- 5. Recorded Crimes
CREATE TABLE crimes (
    id SERIAL PRIMARY KEY,
    month VARCHAR(7),
    category VARCHAR(100),
    location_desc VARCHAR(255),
    geom GEOMETRY(Point, 4326)
);
CREATE INDEX idx_crimes_geom ON crimes USING GIST (geom);
```

---

## 6. Scoring, Tagging & Persona Rules (Python ETL)

Implement deterministic scoring functions in `etl/scoring.py`:

1. **Sub-Scores (0–100 Percentile Ranks):**
   - National percentile ranks across all OAs for Amenities (count within 800m)
   - Schools (distance-weighted Ofsted rating)
   - Transport (distance to rail/tube/bus)
   - Crime (inverse incident density within 400m)
   - Greenspace (area within 500m)

2. **Overall Liveability Score & Grade:**
   - Weighted average of sub-scores mapped to grades: `A` (80–100), `B` (60–79), `C` (40–59), `D` (0–39)

3. **Liveable Value Score:**
   - Compare the street's `liveability_score` percentile against its median property price percentile within a 400m (~5-minute walk) buffer

4. **Rule-Based Lifestyle & Demographic Badges:**
   - `🍸 Dinner & Drinks`: `amenity_percentile >= 75`
   - `🌳 Parklife`: `greenspace_percentile >= 70`
   - `🚶‍♀️ No Car Needed`: `transport_percentile >= 80`
   - `👶 Young Families` / `🏠 Homeowner Families` / `Settled Professionals`: Deterministic threshold checks on Census household composition & tenure (`TS054`, `TS062`, `TS067`)

---

## 7. API Endpoints Specification

### `GET /api/places/nearby?lat={lat}&lng={lng}&limit=20`
Returns nearby towns/suburbs sorted by PostGIS distance (`ST_DistanceSphere`), matching this exact JSON contract:

```json
{
  "places": [
    {
      "id": 10648,
      "name": "Purley",
      "slug": "purley",
      "place_type": "suburb",
      "osm_latitude": "51.3388112",
      "osm_longitude": "-0.1158589",
      "avg_liveability_score": 59.34,
      "total_coverage_percentile": "72.53",
      "amenity_percentile_rank": "84.20",
      "school_percentile_rank": "79.43",
      "transport_percentile_score": "83.98",
      "labels": {
        "labels": {
          "lifestyle": [{ "name": "🍸 Dinner & Drinks", "value": true }],
          "demographic": [{ "name": "👶 Young Families", "value": true }]
        }
      },
      "distance_km": "0.75"
    }
  ],
  "count": 20,
  "coordinates": { "lng": -0.1098, "lat": 51.3443 }
}
```

### `POST /api/search/polygon`
Accepts a GeoJSON Polygon generated by **TerraDraw** (`{ "geometry": { "type": "Polygon", "coordinates": [...] } }`) and executes a PostGIS `ST_Within` / `ST_Intersects` query to return matching streets, places, and average area scores inside the user's custom drawn catchment area.

### `GET /api/streets/[id]/report-data`
Executes spatial queries within a **400m radius (~5-min walk)** of `streets.centroid` to return:
- Overall grade (`A`–`D`)
- Category bars (Home Ownership, Employment, Education, Professional Occupations)
- Quick Summary bullet points
- Top 3 Neighbour Profiles
- Property Insights:
  - Most expensive sale within 400m
  - Most common property type
  - % price difference vs. wider area
- YoY Property Sales Table (`Detached`, `Semi-Detached`, `Terraced`, `Flat/Maisonette`) comparing previous 12 months vs. latest 12 months median price and count
- 10 most recent individual sales
- Full Census 2021 demographic distributions

### `GET /api/streets/[id]/pdf`
Checks Supabase Storage (`street-reports/{id}.pdf`) for a cached PDF. If absent, launches Playwright to render `/reports/print/[id]` to an A4 multi-page PDF, uploads to Supabase Storage, and streams the file back.

---

## 8. Phased Build Instructions

Execute sequentially. Verify builds/tests at the end of each phase and update `PROGRESS.md`:

### Phase 1: Project Scaffolding & Supabase Schema
- Initialise Next.js 15 (TypeScript, App Router, Tailwind CSS, Shadcn/UI) and configure Supabase client (@supabase/ssr) + Drizzle ORM.
- Write `supabase/migrations/0001_init_postgis.sql` and verify schema structure.

### Phase 2: Python ETL, Sample Seeder & PMTiles Pipeline
- Create `etl/` with `ingest.py`, `scoring.py`, and `build_tiles.py`.
- Include a `--sample-bbox` mode (defaulting to Purley/Croydon: -0.18, 51.30, -0.05, 51.39) plus a realistic synthetic/sample seed script (`etl/seed_sample.py`) so the app can be tested immediately before ingesting full UK archives.
- Output `public/tiles/liveability.pmtiles` via Tippecanoe and include an R2 upload script (`etl/upload_r2.py`).

### Phase 3: Interactive Map, TerraDraw & Area Search (`/map`)
- Build a full-screen MapLibre GL JS component (`components/map/LiveabilityMap.tsx`) registered with pmtiles protocol.
- Add layer toggles (Overall Score, Crime, Schools, Transport, Amenities) with colour-coded OA fill layers interleaved cleanly below map labels.
- Integrate TerraDraw (TerraDrawMapLibreGLAdapter) with floating toolbar controls for Freehand Draw, Circle Radius, Polygon, and Clear, wired to `POST /api/search/polygon`.
- Build a responsive slide-over drawer displaying selected street details, nearby places (`/api/places/nearby`), and a "Download Street Report (PDF)" CTA.

### Phase 4: Street Report View & Playwright PDF Export
- Build `/reports/print/[id]` matching a clean, multi-page A4 print layout (Grade badge, Quick Summary, Neighbour Profile cards, Liveable Value Score gauge, YoY Sales table, Recent Sales list, and Recharts horizontal demographic bars).
- Implement `/api/streets/[id]/pdf` using Playwright and Supabase Storage caching.

---

**Begin now with Phase 1: scaffold the directory structure, Next.js configuration, Supabase/Drizzle setup, and `supabase/migrations/0001_init_postgis.sql`.**