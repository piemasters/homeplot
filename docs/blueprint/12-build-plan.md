# 12. Phased Build Plan

Each phase ends with its checks passing and `PROGRESS.md` updated.

**Stages:**
- **Phases 0–6 are Stage 0 (prototype)**, built with the §11.3 switches set to `prototype` values.
- Items marked _(Stage 1)_ are deferred until the §11.5 scale-up runbook.
- **Phases 7–10 are Stage 1.**

## 12.1 Working process
- **Every change goes through a branch and pull request.** Nothing is committed directly to `main`.
  - Branch per phase step, e.g. `phase-1a/map-skeleton` or `phase-4/ranked-list`. Small pull requests are better than one per phase.
  - CI must pass before merging (branch protection on `main`, §17 A3). Use squash merge, with commit messages in the conventional style (`feat:`, `fix:`, `docs:`, `chore:`).
  - The pull request description says which phase step it completes and how it was checked.
- **`main` is always deployable.** Netlify deploys `main` to the prototype site; pull requests get deploy previews.
- **The repo is public (§11.9).** Before every commit, check that nothing secret or private is included:
  - keys or `.env` files
  - tester data
  - investor material
  - third-party material (e.g. `docs/reference/`)
  - raw downloaded data

  Anything ever pushed to a public repo must be treated as public forever.
- After each step, update `PROGRESS.md` with what's done, any experiment results (§12.3) and any newly found decisions, which go into §16.

## 12.2 Milestones & rough timeline
Estimates are in **focused weeks** (about 30–40 productive hours using Claude Code). Scale them to the time you actually have; part-time roughly doubles them. Re-estimate at the end of each milestone. The target investor demo date is open decision D16.

| Milestone | Phases | Estimate | Done when |
|---|---|---|---|
| **M0: De-risked** | 0 | ~1 week | Experiment results recorded, design direction agreed, repo hygiene done |
| **M1: Working slice** | 1a | ~1–2 weeks | Real sample-area data on a deployed map, coloured by one criterion |
| **M2: Foundations + data core** | 1b, 2 | ~2 weeks | Registry, CI, monitoring, full ETL core with publish gate |
| **M3: Demo-able match map** | 3, 4 | ~3–4 weeks | Styled map, criteria, pins, region toggle, ranked list |
| **M4: Stage 0 complete** | 5, 6 | ~3–4 weeks | Demo script (§11.7) runs end to end on a phone; testers invited |
| **Stage 0 total** | | **~10–13 focused weeks** | |
| **M5: Public launch** | Runbook + 7–9 | ~6–10 weeks after funding | Payments live, England-wide data, SEO stage 1 |
| **M6: Apps** | 10 | ~3–4 weeks | Store builds approved |

**When behind schedule:** cut scope using the demo script's "built minimally" column (§11.7) before moving dates. Don't skip tests or the publish gate.

## 12.3 Phases

0. **Prepare and de-risk.**

   **0a. Repo hygiene (in a public repo).**
   - `.gitignore` covers `.env*` (allowing `.env*.example`), `data/`, `docs/reference/`, `node_modules/`, `.turbo/`, `.next/`, `dist/`, `.netlify/`, `.vercel/`, `__pycache__/`, `.venv/`, `.DS_Store`, `*.pmtiles` (outside `public/tiles` fixtures) and `.history/`.
   - Add a `LICENSE` file according to D15 (§16).
   - Turn on GitHub **secret scanning with push protection** (free while public), **Dependabot alerts**, and **branch protection** on `main` (pull request required, CI required, no force-push).
   - Add a short `README.md` saying what the project is and that it's a work in progress, with a link to the blueprint.

   **0b. Experiments** (about 1–2 days each, in throwaway branches or `experiments/`). Each records its result and decision in `PROGRESS.md` and updates the affected blueprint file.

   | # | Question | How | Pass if | If it fails |
   |---|---|---|---|---|
   | E1 | Does MapLibre's `distance` expression work on area shapes in the installed version? | A minimal page with OA polygons and one pin | Correct distances for polygons; filter re-evaluates on pin move | Use the Web Worker + `feature-state` fallback (§6.2) |
   | E2 | Is expression-based colouring fast enough on a phone? | ~5–10k sample-area OA features, a 6-criterion expression, measured on a mid-range Android | Recolour ≤ 100 ms, smooth panning (§15) | Fewer attributes in the map files, LSOA polygons at mid zoom, or `feature-state` colouring |
   | E3 | Can the deployed site read `.pmtiles` straight from R2? | Public bucket + CORS + a range-request test from the Netlify preview | Tiles load, 206 responses, no CORS errors | Deploy the PMTiles Worker early on the free Workers plan |
   | E4 | Does the planned coverage fit within 400 MB? | Load the coverage area's tables (5 years of Price Paid, OA geometry, streets, crime aggregates) into Supabase Free and measure `pg_total_relation_size` | ≤ 400 MB with indexes | **Use a smaller, more densely populated area for the demo** (e.g. 2–3 inner boroughs such as Croydon + Sutton, or a compact inner-London cluster), which gives more sales per OA and more visible variation. Or trim Price Paid to 3 years. Update D3. |
   | E5 | Are the INSPIRE plot tiles a reasonable size? | Convert 2 councils' GML → `ogr2ogr` → tippecanoe at z16 | Max tile ≤ 500 KB; outlines look right | Simplify geometry slightly, raise the minimum zoom to 17, or drop attributes |
   | E6 | Does the Ofsted report-card data exist in the downloadable files? | Inspect the latest Ofsted management-information and GIAS downloads | Usable report-card fields with dates | Score on legacy grades + proximity; decide D6 |
   | E7 | Does Next.js (App Router, features we use) run well on Netlify free? | Deploy a skeleton with one server route, one cached route and middleware | Builds and runs; caching headers honoured | Try Cloudflare Workers via OpenNext; decide D4 |

   **0c. Design** (§18). Can run in parallel with 1a; must be finished before phase 3.
   - Mobile wireframes and the brand basics.
   - **Check:** design direction agreed and recorded in §18.

1. **Working slice, then foundations.**

   **1a. Working slice** (the first visible milestone, M1). The thinnest end-to-end path, all real:
   - Monorepo skeleton (pnpm + Turborepo) with `apps/web` (Next.js, Tailwind, shadcn/ui), `packages/shared` and `etl/` (uv).
   - `0001_init.sql` with **only** the tables the slice needs (`data_versions`, `output_areas`, `oa_metrics`), applied with the Supabase CLI to the Stage 0 Supabase Free project. Later tables come in later migrations.
   - A minimal ETL: OA boundaries + **one metric** (e.g. green space or median price) for the sample area, with national percentiles computed from England.
   - `areas` map file uploaded to R2; an unstyled map page that colours OAs by that metric and shows its value on tap.
   - Deployed on Netlify from `main` through a pull request.
   - **Check:** the deployed URL works on a phone, showing real data coloured on the map.

   **1b. Foundations** (around the working slice).
   - `packages/shared`: `metrics.yaml` with the full first draft of metrics plus generated TypeScript; `features.yaml`, `plans.yaml`, the entitlement resolver, and `<FeatureGate>` / `requireFeature` stubs that allow everything in development; the registry CI checks (§9.3) and `docs/features.md` generation. Each later phase registers its features as it builds them.
   - `etl/sources.yaml` manifest (§2.1) with the downloader reading from it.
   - Sentry and PostHog (consent-gated); rate-limit middleware behind `RATE_LIMIT_ENABLED`.
   - Stage switches (§11.3) read from one typed config module.
   - **Stage 0 environment:** invite-only sign-up with Resend SMTP for auth email, and the daily keep-awake GitHub Action.
   - _(Stage 1)_ Staging Supabase project, Preview vs Production variables, Stripe test mode, R2 staging bucket, spend caps and alerts on every service (§4, §11.5).
   - CI: lint, type-check, Vitest, pytest, `size-limit`; Lighthouse CI added in phase 3.
   - **Check:** `turbo build` passes, CI is green and required on `main`, Sentry receives a test error, and PostHog receives a test event after consent.
2. **ETL core (sample area).**
   - Expand `etl/` with a `--bbox` option, defaulting to the coverage area decided in D3 (informed by E4).
   - Ingest **real** data with DuckDB → GeoParquet → Postgres for OA boundaries, Census, Price Paid (via ONSPD) and flood zones, all downloaded through `sources.yaml`.
   - Compute national and regional percentiles from **all of England** on the laptop, and apply the minimum-sample roll-up for prices (§6.1). Then load only the coverage area (§11.4) into Postgres.
   - Build the publish gate (§4, ETL data validation) now, not later, including the database size budget (≤ 400 MB in Stage 0).
   - **Check:** the validation checks pass; pytest covers percentile and scoring maths.
3. **Tiles + mobile-first map.**
   - Apply the design (§18): theme, layout, bottom sheet, typography.
   - Protomaps background map clipped to the coverage bbox; a `tiles.json` manifest. In Stage 0, map files load straight from the public R2 bucket (unless E3 failed).
   - _(Stage 1)_ The PMTiles Worker (`services/tiles-worker`) and the full UK background map.
   - `etl/build_tiles.py` → `areas-{version}.pmtiles`.
   - Map page with search, a bottom-sheet area panel and layer toggles, with data layers drawn beneath the background map's labels.
   - "Not yet covered" mask and search messages outside `NEXT_PUBLIC_COVERAGE_BBOX`.
   - **Check:** the map renders on mobile and desktop viewports; tiles load from R2; the coverage mask shows; Lighthouse CI is within the §15 budgets.
4. **Match engine.**
   - Criteria UI, expression builder, legend, "why this colour?" and URL state.
   - "Compare within" toggle; "near places I care about" pins, using the approach chosen in E1.
   - Ranked results list using the shared scoring function, keyboard and screen-reader accessible.
   - **Check:**
     - Unit tests for the expression builder and the scoring-agreement tests pass.
     - Changing criteria recolours the map and re-ranks the list without network requests.
     - axe checks pass on the map page.
5. **Remaining v1 data.**
   - Crime, schools (method per E6/D6), NaPTAN, OSM amenities, green space, IMD, conservation areas; full scoring and tags.
   - INSPIRE plots for the coverage councils → `plots-{yyyymm}.pmtiles`, an outline layer from zoom 16 (or as set by E5), and tap-to-select a plot.
   - **Check:** score distributions look sensible (no empty or constant columns); plots render only at the chosen zoom and tapping one shows its area.
6. **Reports.**
   - Street and property report pages and `/reports/print/...` A4 print layouts. Every sampled figure shows its basis ("based on N sales").
   - With `PDF_MODE=browser`, a "Download PDF" button that opens the print layout and calls `window.print()`.
   - _(Stage 1)_ The PDF worker with versioned Storage caching (`PDF_MODE=worker`).
   - `/methodology` and attribution pages generated from `metrics.yaml` and `sources.yaml`; the "report a data issue" form, API and notification. _(Stage 1)_ Turnstile.
   - Pricing page generated from the registry. With `PAYMENTS_ENABLED=false` it shows "coming soon".
   - Testers get Premium through `entitlement_overrides`.
   - Analytics events (§14) and the willingness-to-pay prompt are in place; performance budgets (§15) are met.
   - **Check:**
     - Browser "Save as PDF" output looks right in Chrome and Safari for a sample street and a sample address.
     - The Playwright end-to-end journey (§4) passes on mobile and desktop.
   - **Stage 0 complete:** the investor demo script (§11.7) runs end to end on the deployed prototype on a phone; ready for tester invites.

**→ Run the §11.5 scale-up runbook steps 0–6 before phase 7.**

7. **Accounts, tiers & payments.** _(Stage 1)_
   - Supabase Auth: email-code sign-in and the profile/subscription trigger already exist from Stage 0. Add Google and Apple sign-in, the custom access token hook, and open sign-up.
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
