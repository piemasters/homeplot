# Project Blueprint: UK Home-Finding Map

> **Working name: HomePlot.** Check Companies House, the UK IPO trademark register and `.co.uk` / `.com` / .app domains before choosing.

`response.json` and `Liveable-street-report-*.pdf` in this folder are **reference material only**, for understanding the market. Do not copy their API contracts, badge or persona names, scores or report layout. Every name, contract and design in this product is our own.

---

## 1. Product Summary

**Goal:** help people find where to live next by turning open data into a map that answers "which areas suit _me_?"

**Core loop:**

1. **Set criteria.** Hard filters (e.g. not in a flood zone, semis under £500k) plus importance sliders (schools, safety, green space, amenities, transport).
2. **See a colour-graded match map.** Every area is shaded by how well it matches.
3. **Tap an area.** See why it scored that way, recent sales and key facts.
4. **Download a report** for an area, street or property.

**Target users:** home buyers and renters comparing unfamiliar areas, especially people relocating, upsizing or moving for schools.

**Platforms:** mobile-first web app, released as a PWA, then wrapped with Capacitor for the App Store and Play Store (see §8).

**Pricing:** Free (no account, no running cost to us), Starter and Premium (account required). Features are assigned to tiers in config, not code (see §9).

**v1 scope:** England only.

**v1 non-goals:**

- Wales, Scotland and Northern Ireland
- Property listings
- Planning applications and development plans
- Commute-time isochrones

These are on the roadmap (§3).

**Principles:**

- **Easy data first.** v1 uses only free, openly licensed bulk downloads with no approval process.
- **Explainable scores.** Every colour and grade can be traced back to the numbers behind it.
- **No protected characteristics as criteria.** Users can never search, filter or score areas by ethnicity, religion or country of birth. See §6.4.

---

## 2. v1 Data Sources (easy tier only)

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

---

## 3. Roadmap Data (v1.5 and later)

| Item                                                                        | Why it's deferred                                                                                                              | Target                                            |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| **EPC register** (energy rating, floor area, £/m², property age band, UPRN) | Free, but needs registration/API key; licence conditions restrict some uses                                                    | **v1.5**: first add-on, enriches property reports |
| OS Open UPRN                                                                | Easy, but only useful once EPC links addresses to UPRNs                                                                        | v1.5                                              |
| DfE school performance (KS2/KS4)                                            | Easy, but needs careful presentation                                                                                           | v1.5                                              |
| **Commute isochrones**                                                      | Needs a routing engine (r5 / OpenTripPlanner with GTFS + NaPTAN) or the paid TravelTime API                                    | v2                                                |
| Walking-network distances                                                   | Needs pgRouting or precomputed isochrones                                                                                      | v2                                                |
| Road & rail noise (Defra strategic noise maps)                              | Easy download; lower priority                                                                                                  | v2                                                |
| Broadband & mobile coverage (Ofcom Connected Nations)                       | Postcode-level CSVs; joining them needs care                                                                                   | v2                                                |
| Air quality, radon                                                          | Mixed formats and licences                                                                                                     | v2+                                               |
| Wales (Estyn, NRW flood, WIMD), then Scotland                               | Different sources for every theme                                                                                              | v2+                                               |
| **Planning applications & future development plans**                        | No complete national dataset; planning.data.gov.uk coverage is growing but patchy, and scraping per council isn't worth it yet | Watch; v3                                         |

---

## 4. Tech Stack

**Frontend (web and app):**

- Next.js, latest stable (App Router), TypeScript
- Tailwind CSS
- shadcn/ui
- Recharts for charts
- Mobile-first layouts throughout

**Map:**

- `maplibre-gl` + `pmtiles`
- `terra-draw` + its MapLibre adapter. On touch devices, circle/radius and polygon modes come first; freehand is desktop-first.
- Structure map state so `@deck.gl/mapbox` (`MapboxOverlay`, `interleaved: true`) can be added later without refactoring.

**Basemap:** a self-hosted **Protomaps** UK extract (OSM-derived, `.pmtiles`) on R2, using the `@protomaps/basemaps` style with light and dark flavours.
- There are no per-view fees and no vendor API key.
- Your data layers sit between the basemap's land-use layers and its labels, so street and place names stay readable over the shading.
- Fallback if self-hosting is ever a burden: MapTiler or Stadia (paid per view).

**Tiles:** `tippecanoe` → `.pmtiles`, stored on Cloudflare R2. Use `public/tiles/` in local development. There are three tilesets:

- `basemap-{date}.pmtiles`: Protomaps UK extract, refreshed every few months.
- `areas-{dataVersion}.pmtiles`: OA/LSOA/MSOA scores for the match engine. Rebuilt with each data version.
- `plots-{yyyymm}.pmtiles`: INSPIRE property boundaries, z16 only, attributes `inspire_id` and `area_m2`. It will be several GB nationally and is rebuilt monthly. Build it per region and merge with `tile-join`.

**Tile serving:**
- Serve tiles through the **Protomaps PMTiles Cloudflare Worker**. It exposes ordinary `/{tileset}/{z}/{x}/{y}.mvt` URLs and caches each tile at Cloudflare's edge.
- Pointing the browser straight at the R2 bucket makes it fetch byte ranges of one huge file, which caches less well.
- Tileset file names include their version, so they're served with `Cache-Control: immutable`.
- A small `tiles.json` manifest (short cache) tells the app which versions are current.

**Hosting:** **Vercel** for the Next.js app: preview deployments per pull request, and the simplest Next.js support. Cloudflare (Worker + R2) hosts the tiles. Keep everything in the UK/EU regions (Vercel `lhr1`, Supabase London) to stay close to the database and to users.

**Backend:**

- Supabase: Postgres + PostGIS, Auth, Storage. Plan for the **Pro tier**; national data exceeds the free tier's limits.
- **Connections:** the app connects through the **Supavisor pooler in transaction mode** (port 6543), with `postgres.js` set to `prepare: false` (required in transaction mode). Use the direct connection (port 5432) only for migrations and ETL loads. Without the pooler, serverless functions run out of database connections under load.
- Drizzle ORM, with raw SQL helpers for PostGIS queries.
- **Rate limiting:** Upstash Redis (`@upstash/ratelimit`) in Next.js middleware on every `/api/*` route. It limits by IP for anonymous users and by user ID once signed in, with stricter limits on report and polygon-search routes so the data can't be scraped.

**ETL:** Python 3.12+.
- **DuckDB with its spatial extension is the main engine.** It handles the large joins, aggregations and spatial work: 30M Price Paid rows, 24M INSPIRE polygons, crime to OA. It runs on a laptop in minutes.
- **Intermediate outputs are GeoParquet** in `data/stage/`.
- `geopandas`/`shapely` handle only small or awkward steps.
- Other libraries: `pyogrio`/`ogr2ogr` to read GML and GeoPackage, `osmium` for OSM, `psycopg` (COPY) to load Postgres, `boto3` for R2 uploads.

**Single source of truth for metrics: `packages/shared/metrics.yaml`.** Every metric is defined once with:
- `id`, label and description
- direction (higher is better or worse)
- default weight, criteria-slider settings and legend breaks
- tag rules (e.g. `green-and-leafy: greenspace >= 70`)
- source dataset and its attribution

The Python ETL reads this file to compute scores and tags. A build step generates TypeScript types and constants from it for the frontend, the expression builder and the reports. Adding a metric means editing this file plus its ETL step, nothing else. A CI check fails if the generated TypeScript is out of date.

**Repository layout (pnpm workspaces + Turborepo monorepo):**

```
apps/web/              Next.js app (web, PWA, Capacitor target)
services/pdf-worker/   Playwright PDF renderer (container)
services/tiles-worker/ Protomaps PMTiles Cloudflare Worker
packages/shared/       metrics.yaml, features.yaml, plans.yaml, generated types,
                       Criteria type, MapLibre expression builder,
                       entitlement resolver, API response types
etl/                   Python ETL (uv-managed), reads packages/shared/metrics.yaml
supabase/migrations/   SQL migrations
docs/                  generated docs (features.md), written guides
```

**Monitoring:**
- **Sentry** for errors and performance (web app, PDF worker, ETL runs).
- **PostHog** for product analytics: which criteria and weights people use, search → report conversion, and feature flags.
- Analytics only loads after cookie consent (UK GDPR / PECR).

**Testing:**
- **Vitest:** `packages/shared`, especially the expression builder (given criteria X and a feature with properties Y, the match score is Z) and URL serialisation of criteria.
- **Vitest, entitlements:** the resolver (plan inheritance, overrides, expiry), the registry checks in §9.3, and table-driven tests asserting each tier's access to each feature.
- **Integration:** the Stripe webhook handler (replayed fixture events, idempotency) and the `consume_quota` function (concurrent calls never exceed the quota).
- **pytest:** ETL scoring, tag rules and percentile maths, run on small fixed fixture datasets.
- **Vitest, scoring agreement:** the map expression, the ranked list and "why this colour?" give the same match % for the same criteria and feature, in both "compare within" modes and with near-place pins.
- **Playwright end-to-end tests:**
  - **The core journey:** search a postcode → set criteria (including a pin) → map recolours and the ranked list updates → open a list row → zoom in and tap a plot → open the property report → download the PDF. Run at mobile and desktop viewport sizes.
  - **Accessibility:** automated checks with `@axe-core/playwright` on the map page, list, reports and pricing page.
- CI (GitHub Actions) runs lint, type-check, Vitest, pytest and the Playwright run on every pull request.

**ETL data validation (publish gate):** each ETL run writes the new data to staging tables and a new tile version, then checks it before anything goes live:
- Row counts are within ±10% of the previous version, per dataset.
- No score column, national or regional, is empty, constant or out of its 0–100 range.
- No median price or OA-level crime rate breaks the minimum-sample rules (§6.1).
- Every OA has scores, and the national OA count matches.
- Spot-check fixtures: known areas land in their expected score bands.

Only if every check passes does it set `data_versions.is_current` and update `tiles.json`. On failure, the live version stays as it is and the run alerts via Sentry.

**PDF generation:** a **separate worker**, either a small container (Fly.io / Railway / Cloud Run) running Playwright, or Cloudflare Browser Rendering. It renders `/reports/print/...` to A4 and caches the result in Supabase Storage. Do not run Chromium inside serverless Next.js functions.

**Accounts & payments:** Supabase Auth and Stripe, with feature gating driven by `features.yaml` / `plans.yaml`. See §9; for app-store rules see §8.

**Scheduled refresh:** GitHub Actions runs a monthly ETL for Price Paid, police, schools, NaPTAN, OSM, flood and INSPIRE data. The national INSPIRE tile build may need a larger runner than GitHub's default. Each run writes a new `data_versions` row.

**Environments:**

| | Local | Staging | Production |
|---|---|---|---|
| App | `next dev` | Vercel **preview** deployments (every PR) and the `staging` branch | Vercel production (`main`) |
| Database | Staging Supabase project (no local Docker) | **Separate Supabase project** `homeplot-staging`, sample-area data plus a few regions | Supabase project `homeplot-prod`, full England |
| Stripe | Test mode (Stripe CLI forwards webhooks) | **Test mode** | Live mode |
| Tiles | `public/tiles/` | R2 bucket `tiles-staging` | R2 bucket `tiles-prod` |
| Email/push | Logged only | Sent to an allowlist only | Live |

- **Previews never connect to production:** Vercel's Preview environment variables point at staging, and Production variables are only set in production.
- **Migrations:** managed with the **Supabase CLI** (`supabase/migrations/`).
  - CI applies them to staging on merge to `staging`.
  - Production gets them on release, through a manual-approval GitHub Actions job.
  - Never edit production's schema in the dashboard.
- **ETL:** runs against staging first. The publish gate must pass there before a production run.
- **Backups:** user tables (accounts, subscriptions, saved searches) rely on Supabase's daily backups. Consider point-in-time recovery once revenue justifies it. Data tables can always be rebuilt from the ETL.

**Running costs & budgets.** Rough fixed costs at launch are **£60–100/month**:

| Service | Plan | Approx. |
|---|---|---|
| Supabase (prod) | Pro + small compute | ~£20–40 |
| Supabase (staging) | Free, or Pro if limits bite | £0–20 |
| Vercel | Pro (1 seat) | ~£16 |
| Cloudflare R2 + Workers | Paid Workers plan; R2 storage of tens of GB, no egress fees | ~£5–10 |
| PDF worker | Small container, scales to zero | ~£0–10 |
| Sentry, PostHog, Upstash | Free tiers | £0 |
| Domain, email sending (e.g. Resend) | | ~£5 |

- **Spend caps and alerts on every service:**
  - Vercel spend management with a hard limit.
  - Supabase spend cap on.
  - Cloudflare billing notifications.
  - Upstash budget limit.
  - Sentry and PostHog event quotas.
- **Main surprise risk:** bots crawling server-rendered pages. Mitigate with static generation and CDN caching for SEO pages (§9.8), Vercel bot protection or firewall rules, and rate limiting.
- A monthly cost review goes in `PROGRESS.md`/ops notes. Paid tiers should cover fixed costs at roughly 20 Starter subscribers.

---

## 5. Database Schema (`supabase/migrations/0001_init.sql`)

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

-- Every ETL run is recorded; reports and caches key off the version.
CREATE TABLE data_versions (
    id            SERIAL PRIMARY KEY,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    sources       JSONB NOT NULL,   -- { "price_paid": "2026-10", "police": "2026-09", ... }
    is_current    BOOLEAN NOT NULL DEFAULT false
);

-- Census Output Areas: the unit of scoring and map shading.
CREATE TABLE output_areas (
    oa_code       VARCHAR(9) PRIMARY KEY,
    lsoa_code     VARCHAR(9) NOT NULL,
    msoa_code     VARCHAR(9) NOT NULL,
    lad_code      VARCHAR(9) NOT NULL,
    region_code   VARCHAR(9) NOT NULL,   -- ONS English region (London, South East, ...)
    geom          GEOMETRY(MultiPolygon, 4326) NOT NULL,
    pop_centroid  GEOMETRY(Point, 4326) NOT NULL,
    population    INTEGER
);
CREATE INDEX ON output_areas USING GIST (geom);
CREATE INDEX ON output_areas (lsoa_code);

-- Metric percentiles (0–100, higher = better) per OA, both national (England)
-- and regional (within the OA's ONS region). See §6.1.
CREATE TABLE oa_metrics (
    oa_code              VARCHAR(9) REFERENCES output_areas(oa_code),
    data_version_id      INTEGER REFERENCES data_versions(id),
    schools_pct          REAL,
    safety_pct           REAL,   -- inverse recorded-crime rate
    transport_pct        REAL,
    amenities_pct        REAL,
    greenspace_pct       REAL,
    deprivation_pct      REAL,   -- inverse IMD rank
    schools_pct_region     REAL,
    safety_pct_region      REAL,
    transport_pct_region   REAL,
    amenities_pct_region   REAL,
    greenspace_pct_region  REAL,
    deprivation_pct_region REAL,
    crime_count_24m      INTEGER,   -- raw count behind safety_pct, for "based on N"
    crime_level          VARCHAR(4),-- geography the rate was computed at: 'oa' | 'lsoa' | 'msoa'
    overall_score        REAL,   -- default-weighted, 0–100
    grade                CHAR(1),-- 'A'..'E'
    flood_zone           SMALLINT,  -- max of 1/2/3 covering >10% of OA
    surface_water_risk   SMALLINT,  -- 0 none .. 3 high
    in_conservation_area BOOLEAN,
    nearest_station_m    INTEGER,
    median_price         JSONB,  -- per type, with sample size and the geography used:
                                 -- { "D": { "median": 725000, "n": 14, "level": "lsoa" }, "F": null, ... }
                                 -- null when even the MSOA has too few sales (§6.1)
    demographics         JSONB,  -- tenure, economic activity, qualifications, NS-SeC, households, cars
    tags                 TEXT[] NOT NULL DEFAULT '{}',
    PRIMARY KEY (oa_code, data_version_id)
);

CREATE TABLE streets (
    id            SERIAL PRIMARY KEY,
    os_road_id    VARCHAR(40),
    name          VARCHAR(255) NOT NULL,
    town          VARCHAR(100),
    slug          VARCHAR(255) UNIQUE NOT NULL,
    geom          GEOMETRY(MultiLineString, 4326) NOT NULL,
    centroid      GEOMETRY(Point, 4326) NOT NULL
);
CREATE INDEX ON streets USING GIST (geom);
CREATE INDEX ON streets USING GIST (centroid);

-- Streets often cross several OAs.
CREATE TABLE street_output_areas (
    street_id     INTEGER REFERENCES streets(id) ON DELETE CASCADE,
    oa_code       VARCHAR(9) REFERENCES output_areas(oa_code),
    length_share  REAL NOT NULL,   -- fraction of street length in this OA
    PRIMARY KEY (street_id, oa_code)
);

-- Towns, suburbs and villages (OSM place nodes) for search, SEO pages and "nearby".
CREATE TABLE places (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(150) NOT NULL,
    slug          VARCHAR(150) UNIQUE NOT NULL,
    place_type    VARCHAR(20) NOT NULL,  -- 'town' | 'suburb' | 'village'
    geom          GEOMETRY(Point, 4326) NOT NULL
);
CREATE INDEX ON places USING GIST (geom);

CREATE TABLE property_sales (
    transaction_id  UUID PRIMARY KEY,
    price           INTEGER NOT NULL,
    transfer_date   DATE NOT NULL,
    postcode        VARCHAR(8),
    property_type   CHAR(1),    -- D, S, T, F (O excluded)
    new_build       BOOLEAN,
    tenure          CHAR(1),    -- F freehold, L leasehold
    paon            VARCHAR(100),
    saon            VARCHAR(100),
    street          VARCHAR(255),
    oa_code         VARCHAR(9),
    geom            GEOMETRY(Point, 4326)  -- postcode centroid via ONSPD
);
CREATE INDEX ON property_sales USING GIST (geom);
CREATE INDEX ON property_sales (postcode, paon, saon);
CREATE INDEX ON property_sales (transfer_date);

-- Crime is aggregated, never queried as raw points.
CREATE TABLE crime_oa_month (
    oa_code       VARCHAR(9) REFERENCES output_areas(oa_code),
    month         DATE NOT NULL,         -- first of month
    category      VARCHAR(60) NOT NULL,
    count         INTEGER NOT NULL,
    PRIMARY KEY (oa_code, month, category)
);

CREATE TABLE schools (
    urn           INTEGER PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    phase         VARCHAR(30),           -- primary, secondary, all-through
    type          VARCHAR(60),
    is_state      BOOLEAN NOT NULL,
    ofsted        JSONB,                 -- latest report card or legacy grade + date
    geom          GEOMETRY(Point, 4326) NOT NULL
);
CREATE INDEX ON schools USING GIST (geom);

CREATE TABLE transport_stops (
    atco_code     VARCHAR(20) PRIMARY KEY,
    name          VARCHAR(255),
    stop_type     VARCHAR(10) NOT NULL,  -- rail, tube, tram, bus
    geom          GEOMETRY(Point, 4326) NOT NULL
);
CREATE INDEX ON transport_stops USING GIST (geom);

-- Subdivided polygons for fast point-in-polygon lookups in property reports.
CREATE TABLE flood_areas (
    id            SERIAL PRIMARY KEY,
    kind          VARCHAR(20) NOT NULL,  -- 'zone2', 'zone3', 'sw_low', 'sw_med', 'sw_high'
    geom          GEOMETRY(Polygon, 4326) NOT NULL
);
CREATE INDEX ON flood_areas USING GIST (geom);

-- ---------- Accounts, billing & entitlements (§9) ----------

CREATE TABLE profiles (
    user_id          UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    display_name     VARCHAR(100),
    marketing_opt_in BOOLEAN NOT NULL DEFAULT false,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- One row per user; plan_id matches a key in plans.yaml.
CREATE TABLE subscriptions (
    user_id                UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    plan_id                VARCHAR(30) NOT NULL DEFAULT 'free',
    status                 VARCHAR(20) NOT NULL DEFAULT 'active', -- active | trialing | past_due | canceled
    source                 VARCHAR(20) NOT NULL DEFAULT 'stripe', -- stripe | app_store | play_store
    stripe_customer_id     VARCHAR(50) UNIQUE,
    stripe_subscription_id VARCHAR(50) UNIQUE,
    current_period_end     TIMESTAMPTZ,
    cancel_at_period_end   BOOLEAN NOT NULL DEFAULT false,
    updated_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Per-user exceptions: beta access, comp accounts, promos, grandfathering.
CREATE TABLE entitlement_overrides (
    id          SERIAL PRIMARY KEY,
    user_id     UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    feature_id  VARCHAR(50) NOT NULL,      -- key in features.yaml
    value       JSONB NOT NULL,            -- true/false or a number
    reason      VARCHAR(200) NOT NULL,
    expires_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON entitlement_overrides (user_id);

-- Quota usage per period. consume_quota() checks and increments atomically.
CREATE TABLE usage_counters (
    user_id      UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    feature_id   VARCHAR(50) NOT NULL,
    period_start DATE NOT NULL,
    used         INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (user_id, feature_id, period_start)
);

-- Which items have been counted this period, so re-downloads don't use quota.
CREATE TABLE usage_events (
    user_id      UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    feature_id   VARCHAR(50) NOT NULL,
    period_start DATE NOT NULL,
    item_key     VARCHAR(200) NOT NULL,    -- e.g. 'street:high-street-purley:12'
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, feature_id, period_start, item_key)
);

-- Stripe webhook idempotency.
CREATE TABLE stripe_events (
    id           VARCHAR(100) PRIMARY KEY,
    type         VARCHAR(100) NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE saved_searches (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    name          VARCHAR(100),
    criteria      JSONB NOT NULL,
    alerts        BOOLEAN NOT NULL DEFAULT false,
    read_only     BOOLEAN NOT NULL DEFAULT false,  -- set when over limit after a downgrade
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- "Report a data issue" submissions (§9.7). Anonymous allowed.
CREATE TABLE data_issue_reports (
    id              SERIAL PRIMARY KEY,
    user_id         UUID REFERENCES auth.users(id) ON DELETE SET NULL,
    oa_code         VARCHAR(9),
    location        GEOMETRY(Point, 4326),
    metric_id       VARCHAR(50),           -- key in metrics.yaml, or 'other'
    message         VARCHAR(2000) NOT NULL,
    contact_email   VARCHAR(255),          -- optional, for anonymous reporters
    data_version_id INTEGER REFERENCES data_versions(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'new', -- new | confirmed | fixed | rejected
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Row-level security:**
- Enable it on every user table.
- Users can read their own `profiles`, `subscriptions`, `entitlement_overrides`, `usage_counters` and `saved_searches` rows.
- Users can write only `profiles` and `saved_searches`; everything else is written by the service role (server and Stripe webhook).
- `data_issue_reports` is insert-only, through the API route (service role); there's no public read.
- A trigger on `auth.users` insert creates the `profiles` row and a `subscriptions` row with `plan_id = 'free'`.

INSPIRE property boundaries live only in `plots-{yyyymm}.pmtiles` in v1, with no Postgres table. Add a table (`inspire_id`, `area_m2`, `geom`) only when a feature needs server-side plot queries, such as "plots over 0.5 acres in this area".

---

## 6. Scoring, Match Engine & Tags

### 6.1 Sub-scores (ETL, `etl/scoring.py`)

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

### 6.2 Match engine (headline feature, runs in the browser)

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

### 6.3 Area tags (rule-based, our own wording)

Tags are computed in the ETL and stored in `oa_metrics.tags`. Each tag's rule lives in `packages/shared/metrics.yaml` (§4). Examples:

- `great-high-street`: amenities ≥ 75
- `green-and-leafy`: green space ≥ 70
- `car-free-friendly`: transport ≥ 80, and Census households without a car above the national median
- `family-area`: households with dependent children in the top 30%
- `mostly-owner-occupied`: owned outright + mortgage ≥ 65%
- `flood-zone-3`: a warning tag, not a lifestyle tag

**Avoid stigmatising labels** such as "high unemployment". Present that kind of context neutrally in reports instead.

### 6.4 Protected characteristics

- Ethnicity (`TS021`), religion (`TS030`) and country of birth (`TS012`) are **never** stored in tiles, used as criteria or tags, or used in scoring.
- They are excluded from v1 entirely. If added later, they are neutral, report-only context, and that decision needs a deliberate review.

---

## 7. API Endpoints

All responses use real JSON types (numbers as numbers), empty arrays rather than `null`, and flat structures. Every response includes `dataVersion`.

### `GET /api/search?q=`

Search postcodes (postcodes.io / ONSPD), places and streets. Returns up to 10 results, each with `type`, `label`, `lat`, `lng`, `bbox` and `slug`.

### `GET /api/areas/nearby?lat=&lng=&limit=20`

Nearby places with the population-weighted average of their area scores:

```json
{
  "dataVersion": 12,
  "origin": { "lat": 51.3444, "lng": -0.1098 },
  "places": [
    {
      "id": 10648,
      "name": "Purley",
      "slug": "purley",
      "type": "suburb",
      "lat": 51.3388,
      "lng": -0.1159,
      "distanceKm": 0.75,
      "overallScore": 59.3,
      "grade": "C",
      "scores": {
        "schools": 79.4,
        "safety": 61.2,
        "transport": 84.0,
        "amenities": 84.2,
        "greenspace": 70.1
      },
      "tags": ["great-high-street", "green-and-leafy"]
    }
  ]
}
```

### `GET /api/areas/[oaCode]`

Full panel data for one OA:

- scores
- the explanation inputs used by "why this colour?"
- flood and conservation status
- median prices
- the latest 5 sales
- nearest schools and stations

### `POST /api/search/polygon`

- **Body:** `{ "geometry": GeoJSON Polygon, "criteria"?: Criteria }`, where the polygon comes from terra-draw (circle, polygon or freehand).
- **Returns:**
  - population-weighted averages for the drawn area
  - its best-matching OAs
  - the streets and places inside it
  - its sales summary

### `GET /api/reports/street/[slug]` and `GET /api/reports/property?postcode=&paon=&saon=` or `?lat=&lng=&inspireId=&areaM2=`

The property report can be opened from an address search or from a tapped plot. For a tapped plot:

- The location comes from the tap.
- Plot area is included.
- Sales are matched from the nearest postcode. Exact address matching waits for EPC/UPRN in v1.5.

Report data:

- grade and summary
- match against criteria (if passed)
- sales table (previous vs latest 12 months, by type)
- recent sales
- crime summary
- nearby schools
- flood status at the point
- demographics (non-protected)
- price vs quality

The property report adds that address's own sale history from Price Paid. EPC data is added in v1.5.

### `GET /api/reports/[kind]/[id]/pdf`

Checks Supabase Storage for `reports/{kind}/{id}-{dataVersion}.pdf`. If it's missing:

1. Ask the PDF worker to render `/reports/print/[kind]/[id]`.
2. Store the result.
3. Return a signed URL.

Requires the `pdf_reports` entitlement:
- `consumeQuota` runs before rendering, keyed on `{kind}:{id}:{dataVersion}`, so re-downloading uses no extra quota.
- When the quota is used up, the route returns 403 with an `upgrade` payload.

### Gating & public caching (all routes)
- Free routes (`/api/search`, `/api/areas/*`) are public and CDN-cached with `s-maxage` keyed to the data version.
- Paid routes require a session and call `requireFeature` / `consumeQuota`.
- Paid responses are never publicly cached.

### Account & billing
- `GET /api/me/entitlements`: plan, resolved features, quota used/remaining, and the reason a feature is unavailable (`not_signed_in` | `plan` | `quota`).
- `POST /api/billing/checkout`: `{ lookupKey }` → Stripe Checkout URL. Requires sign-in.
- `POST /api/billing/portal`: Stripe Customer Portal URL.
- `POST /api/webhooks/stripe`: signature-verified, idempotent; updates `subscriptions`. Not rate-limited by IP.
- `GET /api/me/export`: a JSON export of the user's data.
- `DELETE /api/me`: cancels the subscription and deletes the account and its data.
- `GET /api/plans`: plan names, features and current Stripe prices for the pricing page. Public, cached.

### Trust
- `POST /api/data-issues`: `{ oaCode?, lat?, lng?, metricId, message, contactEmail?, turnstileToken }`.
  - Verifies the Turnstile token, then stores the report and sends a notification.
  - Anonymous allowed; strictly rate-limited.

---

## 8. Mobile Strategy

**One codebase:** the Next.js web app, designed mobile-first.

**UI rules:**

- A **bottom sheet** (peek / half / full) for area details; a side panel on desktop.
- The criteria editor is a full-screen sheet on mobile.
- Tap targets are at least 44 px.
- On touch, drawing tools default to circle/radius and polygon.
- Downloading a PDF opens the native share sheet when available.

**Stage 1: PWA (launch)**

- Web app manifest, icons, offline shell and "add to home screen".
- No app-store review is needed.

**Stage 2: Capacitor (after product-market fit)**

- Wrap the hosted web app for the App Store and Play Store.
- To pass Apple's "minimum functionality" rule (guideline 4.2), add native features:
  - Push notifications for saved-search alerts
  - Geolocation ("areas near me")
  - Native share sheet for reports
  - Offline saved reports
- **In-app purchases:** selling Starter/Premium or reports _inside_ the iOS app generally requires Apple In-App Purchase (15–30% fee). Decide before Stage 2: either use IAP in the apps (e.g. via RevenueCat) or keep purchases web-only. Either way, entitlements don't change: `subscriptions.source` records the store, and the same plans apply (§9.4).
- **Sign in with Apple:** required on iOS if Google or other social logins are offered.

SEO pages (`/area/[slug]`, `/street/[slug]`) stay server-rendered for the web. The app uses the same routes.

---

## 9. Accounts, Pricing & Entitlements

### 9.1 Tiers

| | **Free** | **Starter** | **Premium** |
|---|---|---|---|
| Account | Not required | Required | Required |
| Price (placeholder, validate before launch) | £0 | £4.99/mo or £39/yr | £9.99/mo or £79/yr |
| Who it's for | Browsing and shortlisting | An active move | A serious search across several areas |

**Free tier rule: Free features must not cost money per use.**
- Each one runs in the browser (match engine, map layers), comes from static tiles on R2/CDN, or uses an API response cached at the CDN and keyed by data version.
- Nothing in Free may call the PDF worker, send email, store per-user data, or make an uncached database query that grows with traffic.
- CI enforces this rule (§9.3).

**Starting feature assignment.** This table is generated from the registry and shown here for reference; the YAML files are the source of truth:

| Feature | Free | Starter | Premium | Cost class |
|---|---|---|---|---|
| Map, all data layers, search | ✓ | ✓ | ✓ | none |
| Match engine (criteria → colour map), share via URL | ✓ | ✓ | ✓ | none |
| Ranked results list (top 20 matching areas) | ✓ | ✓ | ✓ | none |
| "Near places I care about" pins (up to 3) | ✓ | ✓ | ✓ | none |
| Compare within England / my region | ✓ | ✓ | ✓ | none |
| Property boundaries (z16+) and plot area | ✓ | ✓ | ✓ | none |
| Area panel with "why this colour?" | ✓ | ✓ | ✓ | none (CDN-cached) |
| Area & street summary pages (on screen) | ✓ | ✓ | ✓ | none (CDN-cached) |
| Full street & property report (on screen) | | ✓ | ✓ | low |
| PDF report downloads | | 5 / month | 30 / month | metered |
| Saved searches | | 5 | 25 | low |
| Saved-search alerts (email; push in apps) | | | ✓ | metered |
| Draw-an-area (polygon) search | | ✓ | ✓ | low |
| Compare areas side by side | | 2 areas | 4 areas | low |
| Export polygon-search results (CSV) | | | ✓ | low |
| Early access to new datasets (e.g. commute times) | | | ✓ | varies |

**Possible later add-ons**, built on the same system with no code changes to gating:
- One-off "report packs" (e.g. 3 PDFs for £4.99) for any signed-in user.
- A Premium free trial (`trial_days` in `plans.yaml`).

### 9.2 Feature registry (single source of truth)

Like `metrics.yaml`, features and tiers live in YAML in `packages/shared/`, and everything else is generated from them.

**`features.yaml`: every feature, documented once.**

```yaml
pdf_reports:
  name: PDF report downloads
  description: Download a street or property report as an A4 PDF.
  category: reports            # map | search | reports | saved | alerts | data
  type: quota                  # boolean | quota | limit
  unit: report
  period: month                # quota only; calendar month, Europe/London
  cost: metered                # none | low | metered — see free-tier rule
  status: live                 # planned | beta | live | retired
  upgrade_copy: Download this report as a PDF to keep or share.
saved_searches:
  name: Saved searches
  description: Save criteria and return to them later.
  category: saved
  type: limit                  # max number held at once
  unit: search
  cost: low
  status: live
match_engine:
  name: Match map
  description: Colour the map by how well each area matches your criteria.
  category: map
  type: boolean
  cost: none
  status: live
```

**`plans.yaml`: which features each tier gets.**

```yaml
free:
  name: Free
  requires_account: false
  features: { match_engine: true, map_layers: true, plot_boundaries: true, area_panel: true }
starter:
  name: Starter
  requires_account: true
  inherits: free
  stripe_lookup_keys: [starter_monthly, starter_yearly]
  features: { full_reports: true, pdf_reports: 5, saved_searches: 5, polygon_search: true, compare_areas: 2 }
premium:
  name: Premium
  requires_account: true
  inherits: starter
  stripe_lookup_keys: [premium_monthly, premium_yearly]
  trial_days: 0
  features: { pdf_reports: 30, saved_searches: 25, alerts: true, compare_areas: 4, export_csv: true, early_access: true }
```

**To add, move or remove a feature:**
- Edit the two YAML files and wrap the feature's code in a gate (§9.4).
- To change which tier gets an existing feature, edit `plans.yaml` only: no code change, no Stripe change.

**Generated from the YAML on build:**
- TypeScript types: a `FeatureId` union and a typed `PLANS` constant. A gate that names a feature which doesn't exist is a compile error.
- `docs/features.md`: every feature with its description, category, status, cost class, and value per tier.
- Pricing page and comparison table data, plus upgrade-prompt copy.

### 9.3 CI checks on the registry
- Every feature has a name, description, type, cost and status.
- `plans.yaml` only references features that exist, and values match the feature type (boolean, or a number for quota/limit).
- **Every Free feature has `cost: none`.** This enforces the free-tier rule.
- Every `status: live` feature is assigned to at least one tier.
- Every feature ID used in code exists in `features.yaml`, and no live feature is defined but never used in code.
- The generated `docs/features.md` and types are up to date.

**Site functions outside the registry.** These are always available to everyone and never sold as features, so they aren't in `features.yaml` and the free-tier cost rule doesn't apply to them:

- sign-in
- account export and deletion
- "report a data issue"
- the methodology and legal pages

They're still rate-limited. Their running cost is negligible.

### 9.4 Entitlements: how gating works
- **Gate by feature, never by tier name.** Code asks "can this user use `pdf_reports`?", never `if (plan === 'premium')`. Moving a feature between tiers then needs no code change.
- **How a user's entitlements are worked out**, in order:
  1. Their plan, from `subscriptions`, or `free` if they're not signed in or have no active subscription.
  2. That plan's features, including everything it inherits.
  3. Their `entitlement_overrides`, if any (beta testers, complimentary accounts, promotions, grandfathering), each with an optional expiry.
- **Server (authoritative):**
  - `getEntitlements(userId)`, cached for about 60 s per user.
  - `requireFeature(user, id)` returns 403 with an `upgrade` payload.
  - `consumeQuota(user, id, key)` calls a Postgres function that checks and increments the counter atomically. Re-downloading the same report (same `key`) in the same period doesn't use quota again.
- **Client (UX only):**
  - A `useEntitlement(id)` hook.
  - A `<FeatureGate feature="..." fallback={<UpgradePrompt feature="..." />}>` component.
  - Upgrade prompts read their copy from the registry.
  - The client never decides access on its own; the server always re-checks.
- **Limits vs quotas:**
  - `limit` features (saved searches, compared areas) are checked when something is created.
  - `quota` features (PDFs) are counted per period, in `usage_counters`.
- **Downgrades:**
  - Saved searches over the new limit stay but become read-only.
  - Alerts pause.
  - Unused quota doesn't carry over.
  - Nothing is deleted.
- **Store-agnostic:** `subscriptions.source` records where a subscription came from (`stripe` | `app_store` | `play_store`), so app-store subscriptions (§8) map onto the same plans and entitlements later.

### 9.5 Authentication
- **Supabase Auth**, with three sign-in methods:
  - **email one-time code** (6 digits). This works better than magic links inside the PWA and the Capacitor apps.
  - **Google**
  - **Sign in with Apple**: offered everywhere, and required on iOS if Google is offered.
  - No passwords in v1.
- **Signing up is optional on Free.** Hitting a paid feature shows the upgrade prompt, then sign-in, then checkout. Map position and criteria are kept in the URL so the user comes back to exactly where they were.
- A `profiles` row is created by trigger on sign-up; `subscriptions` defaults to `free`.
- **JWT claim:** a Supabase *custom access token hook* adds `plan` to the session token so the client can render instantly. Paid actions are still checked on the server against the database.
- **Security:**
  - Row-level security on every user table: users read only their own rows.
  - Only the server and the Stripe webhook write to `subscriptions`, `usage_counters` and `entitlement_overrides`, using the service role.
  - The service-role key is never sent to the client.
- **Account management:** an in-app account page with:
  - plan and usage
  - Stripe Customer Portal link
  - **data export**
  - **account deletion**, which cancels the Stripe subscription, then deletes saved searches, usage and profile. Both UK GDPR and Apple require deletion to be in the app.
- **Capacitor:** OAuth opens the system browser and returns via a deep link (Capacitor `App` + `Browser` plugins). Email-code sign-in works inside the WebView.

### 9.6 Payments (Stripe)
- **Stripe owns prices.**
  - Each plan is a Stripe Product with monthly and yearly Prices, identified by **lookup keys** (`starter_monthly`, ...), never hard-coded price IDs.
  - The pricing page fetches current prices from Stripe (cached), so changing a price needs no deploy.
- **Checkout:** Stripe Checkout (hosted).
  - Upgrades, downgrades, cancellation, payment methods and invoices go through the Stripe Customer Portal.
  - Prices are in GBP; Stripe Tax handles UK VAT.
- **Webhooks:** `/api/webhooks/stripe` handles `checkout.session.completed` and `customer.subscription.created|updated|deleted`, plus `invoice.payment_failed`. It:
  - verifies the signature,
  - is idempotent (a `stripe_events` table records processed IDs),
  - updates `subscriptions` and clears that user's entitlement cache.
- **Failed payments:** Stripe retries automatically; the account keeps access until the subscription is cancelled, then drops to Free.
- **Cancellation rights for digital content:**
  - At checkout, and before any PDF downloads immediately after purchase, the user must tick a box agreeing to immediate supply and acknowledging they lose the 14-day cancellation right for that content (Consumer Contracts Regulations 2013).
  - Store the consent with a timestamp.
  - The subscription itself can still be cancelled at any time in the Customer Portal.

### 9.7 Trust: methodology & data issues
- **Methodology page (`/methodology`)**, generated from `metrics.yaml` and so always in step with the scoring. It shows:
  - every metric: what it measures, its source and refresh date, how it's calculated, its thresholds and sample-size rules
  - the national vs regional comparison
  - the known limitations, written plainly: crime location snapping, INSPIRE plots being indicative, straight-line distances, the state of Ofsted report-card data
  - links to each dataset's licence
- "Why this colour?", reports and the ranked list link to the relevant section of the page.
- **"Report a data issue" button:**
  - It appears in the area panel and in reports, and opens a short form (metric, message, optional email). Anonymous reports are allowed.
  - Submissions are stored in `data_issue_reports` with the OA, location and data version, so they can be reproduced.
  - It's protected by Cloudflare Turnstile and rate-limited.
  - New reports send a notification (email or Slack webhook).
  - Confirmed issues become ETL fixes or regression test fixtures.

### 9.8 Growth: SEO, rolled out gradually
- **Stage 1 (launch):** index **place pages only** (about 20k towns, suburbs and villages, `/area/[slug]`). Each must have substantial unique content:
  - scores and tags
  - price table with sample sizes
  - schools and stations
  - flood summary
  - nearby places
  - links to criteria-match searches ("family-friendly areas near Croydon")
- **Stage 2:** index street pages (`/street/[slug]`) **only where there's enough data** to say something distinct: at least N sales in 24 months, a named street of real length, and not a duplicate of its neighbours. All other streets render for users but are `noindex`.
  - Ship in batches through the sitemap.
  - Watch Search Console coverage and indexing before expanding.
- **Engineering:**
  - Pages are statically generated or cached with ISR, keyed to the data version, so bots don't cause server rendering on every request.
  - Sitemaps are split by type and region.
  - Canonical URLs throughout.
  - Known bad bots are blocked at the edge.
- **Never** auto-generate pages for every Output Area.

### 9.9 Legal & compliance checklist (UK)
Every item must be done before taking payments.

- **ICO data protection fee:** register and pay it. It's almost certainly required once the service holds user accounts.
- **UK GDPR:**
  - Privacy notice, record of processing, and a data processing agreement with each processor (Supabase, Vercel, Stripe, Sentry, PostHog, Upstash, Cloudflare).
  - Export and deletion are already built (§9.5).
  - Keep data in the UK/EU where possible.
- **PECR:** cookie banner. Analytics and session replay only after consent; marketing email is opt-in only.
- **Consumer Contracts Regulations 2013:** waiver consent for immediate digital content (§9.6), plus clear pre-contract information: price, renewal and how to cancel.
- **Digital Markets, Competition and Consumers Act 2024, subscription contract rules:**
  - Check whether they're in force at launch.
  - If they are: pre-contract information, reminder notices before renewal and after a free trial, and cancellation as easy as sign-up.
  - Build reminder emails so they can be switched on.
- **Terms of service:**
  - Disclaimer: "for information only, not property, financial or investment advice".
  - Limitation of liability for data accuracy.
  - Acceptable use, including a ban on scraping.
- **Data attribution page:** OGL, ODbL / © OpenStreetMap contributors, Crown copyright, Royal Mail & ONS for postcode data, and the HM Land Registry / Ordnance Survey wording the INSPIRE licence requires.
- **Accessibility statement:** target WCAG 2.2 AA.
- **Business basics:** company or sole-trader setup, VAT registration once over the threshold (Stripe Tax already handles collection), and a trademark/name check for the final name.

---

## 10. Environment Variables (`.env.local.example`)

Each environment (§4) has its own values. Vercel Preview variables point at **staging** services and Stripe test mode; only Vercel Production variables point at production.

```env
# Environment: local | staging | production
NEXT_PUBLIC_APP_ENV=local

# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
# App runtime: Supavisor pooler, transaction mode (use prepare: false)
DATABASE_URL=postgresql://postgres.[PROJECT]:[PASSWORD]@aws-0-eu-west-2.pooler.supabase.com:6543/postgres
# Migrations and ETL loads only: direct connection
DIRECT_DATABASE_URL=postgresql://postgres:[PASSWORD]@db.[PROJECT].supabase.co:5432/postgres

# Tiles (Cloudflare R2 + PMTiles Worker). The app reads tiles.json for current tileset versions.
NEXT_PUBLIC_TILES_BASE_URL=http://localhost:3000/tiles
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=map-tiles

# PDF worker
PDF_WORKER_URL=
PDF_WORKER_SECRET=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Rate limiting (Upstash Redis)
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=

# Bot protection (Cloudflare Turnstile) for data-issue reports
NEXT_PUBLIC_TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=

# Notifications (new data-issue reports, ops alerts)
OPS_NOTIFY_WEBHOOK_URL=

# Monitoring
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=https://eu.i.posthog.com

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 11. Phased Build Plan

Each phase ends with its checks passing and `PROGRESS.md` updated.

1. **Scaffold.**
   - Monorepo set up as in §4 (pnpm workspaces + Turborepo).
   - `apps/web`: Next.js (TypeScript, Tailwind, shadcn/ui) with the Supabase client (`@supabase/ssr`) and Drizzle using the pooled connection.
   - `packages/shared`: `metrics.yaml` with a first draft of the metrics, plus TypeScript generated from it.
   - `packages/shared`: `features.yaml` and `plans.yaml`, the entitlement resolver, `<FeatureGate>` / `requireFeature` stubs that allow everything in development, the registry CI checks (§9.3) and `docs/features.md` generation. Each later phase registers its features as it builds them.
   - `etl/`: Python project managed with uv, reading `metrics.yaml`.
   - Sentry and PostHog wired in (consent-gated); Upstash rate-limit middleware.
   - Write `0001_init.sql`.
   - **Environments:** create the staging Supabase project, link the Supabase CLI, configure Vercel Preview vs Production variables, use Stripe test mode, and create the R2 staging bucket.
   - **Spend caps and alerts** on every service (§4).
   - CI: lint, type-check, Vitest, pytest; migrations apply to staging automatically.
   - **Check:** the migration applies cleanly to staging through CI, `turbo build` passes, CI is green, and a preview deployment talks to staging only.
2. **ETL core (sample area).**
   - Write `etl/` with a `--bbox` option, defaulting to Purley/Croydon (`-0.18,51.30,-0.05,51.39`).
   - Ingest **real** data with DuckDB → GeoParquet → Postgres for OA boundaries, Census, Price Paid (via ONSPD) and flood zones.
   - Compute national and regional percentiles, and apply the minimum-sample roll-up for prices (§6.1). A bbox can't produce real national or regional percentiles, so rank sample-area OAs within the bbox and flag the result as provisional until phase 9.
   - Build the publish gate (§4, ETL data validation) now, not later.
   - **Check:** the validation checks pass; pytest covers percentile and scoring maths.
3. **Tiles + mobile-first map.**
   - Protomaps UK basemap extract and the PMTiles Worker (`services/tiles-worker`); a `tiles.json` manifest.
   - `etl/build_tiles.py` → `areas-{version}.pmtiles`.
   - Map page with search, a bottom-sheet area panel and layer toggles, with data layers drawn beneath the basemap labels.
   - **Check:** the map renders on mobile and desktop viewports, and tiles load through the Worker with cache hits on reload.
4. **Match engine.**
   - Criteria UI, expression builder, legend, "why this colour?" and URL state.
   - "Compare within" toggle, "near places I care about" pins (check MapLibre `distance` support first; otherwise use the Web Worker fallback).
   - Ranked results list using the shared scoring function, keyboard and screen-reader accessible.
   - **Check:**
     - Unit tests for the expression builder and the scoring-agreement tests pass.
     - Changing criteria recolours the map and re-ranks the list without network requests.
     - axe checks pass on the map page.
5. **Remaining v1 data.**
   - Crime, schools, NaPTAN, OSM amenities, green space, IMD, conservation areas; full scoring and tags.
   - INSPIRE plots for the sample-area councils (Croydon, Sutton) → `plots-{yyyymm}.pmtiles`, a z16+ outline layer, and tap-to-select a plot.
   - **Check:** score distributions look sensible (no empty or constant columns); plots render only at z16+ and tapping one shows its area.
6. **Reports.**
   - Street and property report pages, `/reports/print/...` A4 layouts and the PDF worker with versioned caching. Every sampled figure shows its basis ("based on N sales").
   - `/methodology` page generated from `metrics.yaml`; the "report a data issue" form, API, Turnstile and notification.
   - **Check:** a generated PDF for a sample street and a sample address; the Playwright end-to-end journey (§4) passes on mobile and desktop.
7. **Accounts, tiers & payments.**
   - Supabase Auth (email code, Google, Apple), the profile/subscription trigger and the custom access token hook.
   - Account page with export and deletion.
   - Real entitlement enforcement: `consume_quota`, overrides, downgrade handling.
   - Saved searches.
   - Stripe products/prices with lookup keys, Checkout, Customer Portal, webhook and Stripe Tax.
   - Pricing page generated from the registry.
   - Checkout consent for immediate digital content (§9.6); legal pages and the cookie banner (§9.9).
   - **Before going live with payments:** complete the §9.9 checklist (ICO fee, terms, privacy notice, processor agreements, and a decision on the subscription rules).
   - **Check:**
     - Table-driven tests pass for every tier × feature.
     - Webhook replay tests pass.
     - A Playwright journey passes: free user → hits PDF gate → signs in → Stripe test checkout → downloads PDF → quota decrements.
     - CI confirms every Free feature is `cost: none`.
8. **PWA.**
   - Manifest, offline shell, install prompt.
9. **National scale-up.**
   - Full England ETL (including all INSPIRE councils) with real national and regional percentiles, tile-size tuning, scheduled monthly refresh in GitHub Actions (staging, then production).
   - SEO stage 1: statically generated, CDN-cached **place pages only**, with sitemaps and Search Console. Street pages render for users but stay `noindex` until stage 2 (§9.8).
   - Review spend against the budget in §4.
10. **Capacitor apps.**
    - iOS/Android shells, push alerts, geolocation, share sheet; settle the in-app-purchase decision first.

---

## 12. Agent Execution Notes (for Claude Code)

- Work phase by phase as listed in §11. Don't start a phase until the previous phase's checks pass.
- Maintain `PROGRESS.md` in the repo root as a checklist.
- **No local Docker database.** Connect to the **staging** Supabase project: `DATABASE_URL` (pooler, `prepare: false`) for the app, `DIRECT_DATABASE_URL` for migrations and ETL loads.
- **Never touch production.** Don't connect to production, apply migrations to it, or use live Stripe keys. Production changes go only through the release workflow.
- **Schema changes:** only through new files in `supabase/migrations/`, created with the Supabase CLI. Never edit an applied migration.
- **One scoring function:** the map, the ranked list and "why this colour?" must use the same scoring code in `packages/shared`.
- **Small numbers:** every figure shown from a sample (prices, crime) must follow the §6.1 minimum-sample rules and show its basis.
- **Accessibility:** new UI must be keyboard-usable and pass the axe checks.
- **SEO:** don't make new page types indexable without following §9.8. Never generate pages per Output Area.
- Don't commit `.env*` files or downloaded data. Put raw downloads in a git-ignored `data/raw/` and staged GeoParquet in a git-ignored `data/stage/`.
- **Metrics come from `packages/shared/metrics.yaml`.** Never hard-code metric names, weights, thresholds or tag rules in TypeScript or Python. Regenerate the shared types after editing it.
- Use DuckDB for large ETL steps; don't load national datasets into pandas or geopandas.
- **ETL publishing:** never mark a data version current, or update `tiles.json`, unless the validation gate passes.
- **Tests:** add or update tests with every change. Vitest covers shared logic, pytest covers ETL scoring, and Playwright covers the core journey. CI must be green before a phase is complete.
- Every new `/api/*` route goes behind the rate-limit middleware.
- **Features and tiers come from `features.yaml` / `plans.yaml`.**
  - Every user-facing capability is registered in `features.yaml`, with a description, before it's built.
  - Gate by feature ID (`requireFeature`, `consumeQuota`, `<FeatureGate>`), never by plan name.
  - Never hard-code limits, prices or Stripe price IDs.
- **Access checks:** the server is authoritative for every paid action; client gates are UX only.
- **Free-tier features must be `cost: none`:** browser-side, static, or CDN-cached. Don't add per-user storage, email, PDF rendering or uncached queries to Free.
- **Secrets:** never expose the Supabase service-role key or Stripe secret key to the client.
- Use only the v1 datasets in §2. Ask before adding anything from §3.
- Follow §6.4: no protected characteristics in tiles, criteria, tags or scoring.
- Don't reproduce Liveable's names, contracts or designs from the reference files.
