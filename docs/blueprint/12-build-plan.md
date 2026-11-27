# 12. Phased Build Plan

Each phase ends with its checks passing and `PROGRESS.md` updated.

**Stages:**
- **Phases 1–6 are Stage 0 (prototype)**, built with the §11.3 switches set to `prototype` values.
- Items marked _(Stage 1)_ are deferred until the §11.5 scale-up runbook.
- **Phases 7–10 are Stage 1.**

1. **Scaffold.**
   - Monorepo set up as in §4 (pnpm workspaces + Turborepo).
   - `apps/web`: Next.js (TypeScript, Tailwind, shadcn/ui) with the Supabase client (`@supabase/ssr`) and Drizzle using the pooled connection.
   - `packages/shared`: `metrics.yaml` with a first draft of the metrics, plus TypeScript generated from it.
   - `packages/shared`: `features.yaml` and `plans.yaml`, the entitlement resolver, `<FeatureGate>` / `requireFeature` stubs that allow everything in development, the registry CI checks (§9.3) and `docs/features.md` generation. Each later phase registers its features as it builds them.
   - `etl/`: Python project managed with uv, reading `metrics.yaml`.
   - Sentry and PostHog wired in (consent-gated); rate-limit middleware behind `RATE_LIMIT_ENABLED`.
   - Stage switches (§11.3) read from one typed config module.
   - Write `0001_init.sql`.
   - **Stage 0 environment:**
     - One **Supabase Free** project, linked with the Supabase CLI; migrations applied through the CLI.
     - Invite-only sign-up, with Resend SMTP for auth email.
     - Site deployed on **Netlify free**.
     - A public R2 bucket for map files.
     - The daily keep-awake GitHub Action.
   - _(Stage 1)_ Staging Supabase project, Preview vs Production variables, Stripe test mode, R2 staging bucket, spend caps and alerts on every service (§4, §11.5).
   - CI: lint, type-check, Vitest, pytest.
   - **Check:** the migration applies cleanly through the CLI, `turbo build` passes, CI is green, and the deployed prototype loads.
2. **ETL core (sample area).**
   - Write `etl/` with a `--bbox` option, defaulting to Purley/Croydon (`-0.18,51.30,-0.05,51.39`).
   - Ingest **real** data with DuckDB → GeoParquet → Postgres for OA boundaries, Census, Price Paid (via ONSPD) and flood zones.
   - Compute national and regional percentiles from **all of England** on the laptop, and apply the minimum-sample roll-up for prices (§6.1). Then load only the coverage area (§11.4) into Postgres.
   - The publish gate includes the database size budget (≤ 400 MB in Stage 0).
   - Build the publish gate (§4, ETL data validation) now, not later.
   - **Check:** the validation checks pass; pytest covers percentile and scoring maths.
3. **Tiles + mobile-first map.**
   - Protomaps background map clipped to the coverage bbox; a `tiles.json` manifest. In Stage 0, map files load straight from the public R2 bucket.
   - _(Stage 1)_ The PMTiles Worker (`services/tiles-worker`) and the full UK background map.
   - `etl/build_tiles.py` → `areas-{version}.pmtiles`.
   - Map page with search, a bottom-sheet area panel and layer toggles, with data layers drawn beneath the basemap labels.
   - "Not yet covered" mask and search messages outside `NEXT_PUBLIC_COVERAGE_BBOX`.
   - **Check:** the map renders on mobile and desktop viewports; tiles load from R2; the coverage mask shows.
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
   - Street and property report pages and `/reports/print/...` A4 print layouts. Every sampled figure shows its basis ("based on N sales").
   - With `PDF_MODE=browser`, a "Download PDF" button that opens the print layout and calls `window.print()`.
   - _(Stage 1)_ The PDF worker with versioned Storage caching (`PDF_MODE=worker`).
   - `/methodology` page generated from `metrics.yaml`; the "report a data issue" form, API and notification. _(Stage 1)_ Turnstile.
   - Pricing page generated from the registry. With `PAYMENTS_ENABLED=false` it shows "coming soon".
   - Testers get Premium through `entitlement_overrides`.
   - **Check:**
     - Browser "Save as PDF" output looks right in Chrome and Safari for a sample street and a sample address.
     - The Playwright end-to-end journey (§4) passes on mobile and desktop.
   - Analytics events (§14) and the willingness-to-pay prompt are in place; performance budgets (§15) are met.
   - **Stage 0 complete:** the investor demo script (§11.7) runs end to end on the deployed prototype on a phone; ready for tester invites.
**→ Run the §11.5 scale-up runbook steps 1–6 before phase 7.**

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
