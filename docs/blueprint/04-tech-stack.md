# 4. Tech Stack

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

**Hosting (Stage 1; for the Stage 0 prototype on free tiers, see §11):** **Vercel** for the Next.js app: preview deployments per pull request, and the simplest Next.js support. Cloudflare (Worker + R2) hosts the tiles. Keep everything in the UK/EU regions (Vercel `lhr1`, Supabase London) to stay close to the database and to users.

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
- `in_tiles`: whether it's written to the map files (§15 file-size budget)
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
