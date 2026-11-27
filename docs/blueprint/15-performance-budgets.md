# 15. Performance Budgets

The product is a map used mostly on phones, so it has to feel instant on an ordinary phone, not just on a developer's laptop. These budgets are checked in CI so performance can't quietly slip as metrics and features are added.

## 15.1 Reference device
A mid-range Android phone on 4G: Lighthouse's default mobile throttling (simulated slow 4G, 4× CPU slowdown). Test manually on a real mid-range Android and an older iPhone before each demo or release.

## 15.2 Budgets

| What | Budget | How it's measured |
|---|---|---|
| Map page: first data layer visible | **≤ 3 s** | Playwright timing (map `idle` after first areas tile) under throttling |
| Content pages (area, street, methodology, pricing): LCP | ≤ 2.5 s | Lighthouse CI |
| Interaction responsiveness (INP) | ≤ 200 ms | Lighthouse CI / PostHog web vitals |
| Layout shift (CLS) | ≤ 0.1 | Lighthouse CI |
| Criteria change → map recoloured | ≤ 100 ms | Vitest benchmark of the expression builder + manual check |
| Criteria/view change → ranked list updated | ≤ 200 ms for the top 20 of the loaded features | Vitest benchmark |
| Map page JavaScript (gzipped, first load) | ≤ 450 KB including MapLibre | `size-limit` in CI |
| Other pages' JavaScript (gzipped) | ≤ 150 KB | `size-limit` in CI |
| Areas map file: single tile | ≤ 200 KB max, ≤ 80 KB average | ETL publish gate (§4) |
| Plots map file: single tile | ≤ 500 KB max | ETL publish gate |
| API: area panel (CDN-cached) | p95 ≤ 300 ms | Sentry performance |
| API: polygon search, report data | p95 ≤ 1 s | Sentry performance |
| PDF generation (worker, Stage 1) | p95 ≤ 8 s | Sentry performance |

## 15.3 How the budgets are kept
- **Map file attributes:** only metrics with `in_tiles: true` in `metrics.yaml` go into the areas file.
  - Percentiles are rounded to integers (0–100) and median prices to the nearest £1k.
  - Sample sizes are capped at 99 ("99+").
  - Adding a metric to the map files is a deliberate choice, and the tile-size check shows its cost.
- **Lazy loading:** the drawing tools (`terra-draw`), Recharts, report components and the criteria editor's advanced options load only when first used.
- **Fonts and images:** one variable font, self-hosted and subset. Images through Next's image handling.
- **Map style:** a minimal set of background-map layers. Data layers use simple fills, with no per-feature images.
- **CI gates:**
  - Lighthouse CI runs on the map page, one place page and the pricing page.
  - `size-limit` checks bundle sizes.
  - The ETL publish gate checks tile sizes, using per-tile size statistics from `pmtiles` / `tippecanoe` output.
  - Failing any budget fails the build or the publish.
- **Budgets can change**, but only deliberately: edit this file and the CI thresholds in the same pull request, with the reason.
