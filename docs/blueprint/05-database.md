# 5. Database Schema (`supabase/migrations/0001_init.sql`)

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
