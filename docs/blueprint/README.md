# HomePlot Blueprint

> **Working name: HomePlot** (final name is open decision D1, §16).

A mobile-first map for finding where to live in England.

1. Users set criteria (hard filters plus importance sliders).
2. The map is coloured by how well each area matches, with a ranked list alongside.
3. Users tap an area to see why it scored that way.
4. Users open a street or property report.

It's built on free, openly licensed data. It's delivered as a free-tier **Stage 0 prototype** for about 5 testers to secure funding, then a **Stage 1 public launch** with Free / Starter / Premium tiers.

Section numbers (`§N.N`) are used throughout and map to the files below.

| §   | File                                                                       | Contents                                                                            |
| --- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1   | [01-product.md](01-product.md)                                             | Goal, core loop, users, scope, principles                                           |
| 2   | [02-data-sources.md](02-data-sources.md)                                   | v1 datasets, Census tables, data notes, INSPIRE plots                               |
| 3   | [03-roadmap.md](03-roadmap.md)                                             | Deferred data; optional later features (plain-English search)                       |
| 4   | [04-tech-stack.md](04-tech-stack.md)                                       | Stack, map files, hosting, ETL, monorepo, monitoring, tests, environments, costs    |
| 5   | [05-database.md](05-database.md)                                           | Schema and row-level security                                                       |
| 6   | [06-scoring-match-engine.md](06-scoring-match-engine.md)                   | Scores, minimum samples, match engine, ranked list, tags, protected characteristics |
| 7   | [07-api.md](07-api.md)                                                     | API routes and contracts                                                            |
| 8   | [08-mobile.md](08-mobile.md)                                               | Mobile UI, PWA, Capacitor                                                           |
| 9   | [09-accounts-pricing-entitlements.md](09-accounts-pricing-entitlements.md) | Tiers, feature registry, entitlements, auth, Stripe, trust, SEO, legal              |
| 10  | [10-environment-variables.md](10-environment-variables.md)                 | `.env` reference                                                                    |
| 11  | [11-delivery-stages.md](11-delivery-stages.md)                             | Stage 0 prototype, stage switches, scale-up runbook, costs, demo script             |
| 12  | [12-build-plan.md](12-build-plan.md)                                       | Phases 1–10 with checks                                                             |
| 13  | [../../CLAUDE.md](../../CLAUDE.md)                                         | Rules for the coding agent                                                          |
| 14  | [14-metrics-analytics.md](14-metrics-analytics.md)                         | Pitch metrics, event taxonomy, willingness-to-pay prompt, tester feedback           |
| 15  | [15-performance-budgets.md](15-performance-budgets.md)                     | Speed and size budgets, CI checks                                                   |
| 16  | [16-open-decisions.md](16-open-decisions.md)                               | Unresolved choices                                                                  |
| 17  | [17-service-accounts.md](17-service-accounts.md)                           | Every external account, what it needs and what it produces                          |
| 18  | [18-design.md](18-design.md)                                               | Design deliverables, brand vs data colours, layout rules                            |

## What to read for each phase

Always read `CLAUDE.md`, §12 (the phase itself, including §12.1 working process) and §16 (open decisions) first.

| Phase                     | Also read                                                       |
| ------------------------- | --------------------------------------------------------------- |
| 0a. Repo hygiene          | §11.9, §17.2 (A3)                                               |
| 0b. Experiments           | §6.2, §11.2–11.4, §15, §2 (INSPIRE, schools notes)              |
| 0c. Design                | §18, §8, §11.7, §6.2                                            |
| 1a. Working slice         | §2.1, §4, §5, §11.2–11.3, §17.2                                 |
| 1b. Foundations           | §2.1, §4, §9.2–9.4, §10, §11.3, §14, §15                        |
| 2. ETL core               | §2, §4 (ETL, publish gate), §5, §6.1, §11.4                     |
| 3. Map                    | §4 (map files, background map), §8, §18, §11.3, §15             |
| 4. Match engine        | §6, §14, §15                                              |
| 5. Remaining data      | §2, §5, §6                                                |
| 6. Reports             | §7, §9.7, §11.7, §14                                      |
| 7. Accounts & payments | §9, §7, §17.3                                             |
| 8. PWA                 | §8                                                        |
| 9. National scale-up   | §2, §4, §9.8, §11.5                                       |
| 10. Apps               | §8, §17.4                                                 |
