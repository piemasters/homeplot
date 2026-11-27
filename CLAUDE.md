# CLAUDE.md

HomePlot (working name) is a mobile-first map for finding where to live in England. Users set criteria, the map and a ranked list show how well each area matches, and users open street and property reports. The full blueprint is in [docs/blueprint/](docs/blueprint/README.md); `§N.N` references point to its files (see the index there).

**Current stage:** set by `NEXT_PUBLIC_APP_STAGE`: `prototype` (Stage 0) or `launch` (Stage 1). See §11.

## How to work

- Work phase by phase as listed in §12. Before each phase, read the files listed for it in the blueprint index. Don't start a phase until the previous phase's checks pass.
- Maintain `PROGRESS.md` in the repo root as a checklist.
- **Open decisions (§16) are not yours to make.** If work is blocked on one, stop and ask.
- **Stage 0 scope follows the demo script (§11.7).**
  - Build in full only what the script shows.
  - Build _(Stage 1)_ items only as far as their stage switch (§11.3) and the code path behind it.
  - Never fork code by stage or hard-code prototype shortcuts.

## Environments & data safety

- **No local Docker database.** Use `DATABASE_URL` (pooler, `prepare: false`) for the app and `DIRECT_DATABASE_URL` for migrations and ETL loads.
  - In **Stage 0** there is one Supabase project, used by real testers. Ask before any destructive data change.
  - In **Stage 1**, connect only to **staging**.
- **Never touch Stage 1 production.** Don't connect to it, apply migrations to it, or use live Stripe keys. Production changes go only through the release workflow.
- **Schema changes:** only through new files in `supabase/migrations/`, created with the Supabase CLI. Never edit an applied migration.
- **Stage 0 database size:** keep it within the §11.4 budget (≤ 400 MB).
- **Secrets:** never commit `.env*` files and never expose the Supabase service-role key or Stripe secret key to the client.
- **Downloaded data:** raw downloads go in a git-ignored `data/raw/` and staged GeoParquet in a git-ignored `data/stage/`. Never commit either.

## Data & scoring

- Use only the v1 datasets in §2. Ask before adding anything from §3.
- **Metrics come from `packages/shared/metrics.yaml`.** Never hard-code metric names, weights, thresholds or tag rules in TypeScript or Python. Regenerate the shared types after editing it.
- Use DuckDB for large ETL steps; don't load national datasets into pandas or geopandas.
- **ETL publishing:** never mark a data version current, or update `tiles.json`, unless the validation gate passes (§4).
- **One scoring function:** the map, the ranked list and "why this colour?" must use the same scoring code in `packages/shared`.
- **Small numbers:** every figure shown from a sample (prices, crime) follows the §6.1 minimum-sample rules and shows its basis.
- **Protected characteristics:** follow §6.4. Ethnicity, religion and country of birth never go in map files, criteria, tags or scoring.

## Features, tiers & access

- **Features and tiers come from `features.yaml` / `plans.yaml`** (§9.2).
  - Register every user-facing capability in `features.yaml`, with a description, before building it.
  - Gate by feature ID (`requireFeature`, `consumeQuota`, `<FeatureGate>`), never by plan name.
  - Never hard-code limits, prices or Stripe price IDs.
- **Access checks:** the server is authoritative for every paid action; client gates are UX only.
- **Free-tier features must be `cost: none`:** browser-side, static, or CDN-cached. Don't add per-user storage, email, PDF rendering or uncached queries to Free.
- **Rate limiting:** every new `/api/*` route goes behind the rate-limit middleware.

## Quality

- **Tests:** add or update tests with every change. CI must be green before a phase is complete.
  - Vitest covers shared logic.
  - pytest covers ETL scoring.
  - Playwright covers the core journey.
- **Performance:** stay within the §15 budgets. To change a budget, change §15 and the CI threshold in the same pull request, with a reason.
- **Accessibility:** new UI must be keyboard-usable and pass the axe checks.
- **Analytics:** send only events defined in `packages/shared/analytics/events.ts` (§14). No personal data in event properties.
- **SEO:** don't make new page types indexable without following §9.8. Never generate pages per Output Area.

## Reference material

`docs/reference` contain public examples for market understanding only. Don't reproduce their names, contracts, scores or designs.
