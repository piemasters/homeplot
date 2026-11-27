# 11. Delivery Stages: Funding Prototype → Launch

The product is delivered in two stages. **Stage 0** is a low-cost prototype for about 5 testers, used to demonstrate the product and secure funding. **Stage 1** is the public launch described in the rest of this blueprint.

The architecture is the **same in both stages**: same code, schema, map files and feature registry. Moving to Stage 1 means upgrading plans, loading more data and switching on postponed services. It doesn't mean rewriting.

## 11.1 Stage 0 goals
- A demo-ready product for investors and about 5 invited testers.
- It covers the headline experience:
  - criteria → colour map
  - ranked list, "near places" pins, England/region toggle
  - property boundaries
  - area panel with "why this colour?"
  - street and property reports
  - methodology page
  - pricing page showing the planned tiers
- Evidence for the pitch:
  - PostHog usage (sessions, criteria used, reports opened)
  - structured tester feedback
  - a short screen-recorded demo
- **Target running cost: about £0/month**, plus a domain (~£10/year).

## 11.2 Stage 0 infrastructure (free tiers)

| Area | Stage 0 (prototype) | Limits to watch | Stage 1 (launch) |
|---|---|---|---|
| Database & auth | **Supabase Free**, 1 project | 500 MB database; **projects pause after about 7 days without activity**; built-in auth email is heavily rate-limited | Supabase Pro (prod) + a separate staging project |
| App hosting | **Netlify free** (or Cloudflare Workers free via OpenNext). Vercel Hobby is non-commercial only. | Free-plan build minutes and bandwidth (check current limits) | Vercel Pro, or a Netlify paid plan (§11.5) |
| Map files | **R2 free tier**; the browser reads `.pmtiles` straight from a public bucket (no Worker) | 10 GB storage; 1M write / 10M read operations a month | Protomaps PMTiles Worker in front of R2 |
| Background map | Protomaps extract **clipped to the prototype coverage area** (`pmtiles extract --bbox`) | Small file | Full UK extract |
| Coverage | **About 5–8 adjacent boroughs or districts**, grown while within the 400 MB database budget (§11.4) | Database size | All of England |
| PDFs | **Browser print-to-PDF** from `/reports/print/...` ("Save as PDF") | Output varies a little by browser | Playwright PDF worker + Storage cache |
| Payments | **Off.** Pricing page displays tiers; testers get Premium through `entitlement_overrides` | n/a | Stripe live (§9.6) |
| Sign-up | **Invite-only** (Supabase sign-ups disabled; testers invited from the dashboard) | n/a | Open sign-up |
| Auth email | **Resend free tier** as custom SMTP, so the email codes actually arrive | ~3k emails/month | Resend paid tier as volume needs |
| Monitoring | Sentry free, PostHog free (EU) | Event quotas | Same, upgraded if needed |
| Rate limiting | Middleware present; Upstash free or disabled (`RATE_LIMIT_ENABLED=false`) | n/a | Upstash on |
| Environments | **One** (prototype = production); local dev points at it | Be careful with data changes | Local / staging / production (§4) |
| ETL | **Run on a laptop**; upload with `etl/upload_r2.py` and `psql`/COPY | Manual | Scheduled GitHub Actions |
| CI | GitHub Actions: lint, type-check, tests. **Free and unmetered on standard runners while the repo is public** (§11.9). | n/a while public | Private repo: the paid plan's minutes allowance, plus migration deploys and scheduled ETL |
| Bot protection, SEO, PWA, apps, alerts | **Not built** | n/a | Built in Stage 1 phases |

**Keep Supabase awake:** schedule a daily GitHub Actions job that makes one lightweight query, so the project doesn't pause between demos. Before any investor demo, still check that it's up.

## 11.3 Stage switches (configuration, not code forks)
Everything that differs between stages is controlled by environment variables (§10), so no code is deleted or rewritten at scale-up:

| Variable | Stage 0 | Stage 1 |
|---|---|---|
| `NEXT_PUBLIC_APP_STAGE` | `prototype` (shows a small "Preview" badge) | `launch` |
| `NEXT_PUBLIC_COVERAGE_BBOX` | Prototype coverage bbox | unset (England) |
| `NEXT_PUBLIC_TILES_BASE_URL` | Public R2 bucket URL (direct `.pmtiles`) | PMTiles Worker URL |
| `PDF_MODE` | `browser` (print-to-PDF button) | `worker` |
| `PAYMENTS_ENABLED` | `false` (pricing page shows "coming soon") | `true` |
| `SIGNUP_MODE` | `invite` | `open` |
| `RATE_LIMIT_ENABLED` | `false` or `true` | `true` |

**Coverage behaviour:**
- Outside `NEXT_PUBLIC_COVERAGE_BBOX` the map shows a light "not yet covered" mask.
- Search results outside it say so instead of failing.
- The ranked list and pins only consider covered areas.

## 11.4 Stage 0 data plan
- **National ranking, local hosting:** run the full ETL on **all of England** on the laptop (DuckDB handles it in minutes), so the national and regional percentiles are real. Then load **only the prototype coverage area** into Postgres and the map files. The demo's scores are genuine, and nothing is recomputed at launch.
- **Database budget: ≤ 400 MB** (headroom under the 500 MB limit). Biggest consumers, in order:
  1. Price Paid: keep 5 years in Stage 0, 10 at launch, and only the columns in the schema.
  2. OA geometries.
  3. Streets.
- Crime is stored aggregated only. INSPIRE plots never go in Postgres (§5).
- The ETL prints `pg_total_relation_size` per table after loading, and the publish gate **fails if the total is over the budget**.
- **Suggested coverage:** start with about 5–8 adjacent boroughs or districts around Croydon/Sutton (the sample area), then expand while under budget. A region the size of Greater London is likely to exceed 500 MB once 5 years of sales and indexes are included, so measure before committing.
- **Fallback if the budget doesn't fit** (experiment E4, §12.3): use a **smaller, more densely populated area** for the demo instead, e.g. 2–3 inner boroughs (Croydon + Sutton, or a compact inner-London cluster).
  - Dense areas have more sales per OA and more visible variation between neighbouring areas, so they often demo better anyway.
  - Price Paid can also be trimmed to 3 years in Stage 0.
  - The demo script (§11.7) is adjusted to the chosen area, e.g. the "Work" pin location.
- **Map files (R2 ≤ 10 GB):** `areas` (prototype coverage, all zooms), `plots` (coverage councils only), and the background map clipped to coverage. Each should stay well under 1 GB.

## 11.5 Scale-up runbook (after funding)
Steps are listed in order. Each one is a plan upgrade or a configuration change backed by code that already exists.

0. **Repository goes private** (§11.9):
   - Upgrade GitHub to a plan that keeps branch protection on private repos (Pro for a personal account, Team for an organisation; consider moving the repo into a GitHub organisation at the same time).
   - Re-check the branch protection rules after the switch. GitHub's free secret scanning stops for private repos; no paid replacement is planned, so follow the commit checks in §11.9.
   - Make sure the Actions minutes allowance covers CI plus the scheduled ETL.
   - Re-connect the hosting and Supabase GitHub integrations if they lose access.
1. **Supabase:**
   - Upgrade the existing project to **Pro**; it becomes production and keeps its data and users.
   - Create a **staging** project and apply all migrations with the Supabase CLI.
   - Turn on the spend cap.
2. **Hosting decision:** stay on Netlify (paid plan) or move to **Vercel Pro**.
   - Either way, split Preview and Production environment variables so previews use staging.
   - Set region and spend limits.
3. **Map files:**
   - Deploy the **PMTiles Worker** and create `tiles-staging` / `tiles-prod` buckets.
   - Switch `NEXT_PUBLIC_TILES_BASE_URL`.
   - Upload the **full UK background map**.
4. **National data:**
   - Load all of England (10 years of Price Paid) to staging, then production, through the publish gate.
   - Build the national `plots` file per region and merge with `tile-join`.
   - Move the ETL to a **scheduled GitHub Actions** run (larger runner for INSPIRE).
   - Unset `NEXT_PUBLIC_COVERAGE_BBOX`.
5. **Protection:** turn on Upstash rate limiting, add Turnstile on data-issue reports, add bot protection on the host, and set spend alerts on every service (§4).
6. **PDF worker:** deploy the container, set `PDF_MODE=worker`, and add the Storage cache.
7. **Compliance:** complete the §9.9 checklist (ICO fee, terms, privacy notice, processor agreements, cookie banner, subscription-rules decision).
8. **Payments:** create Stripe products and prices with lookup keys, the webhook and Stripe Tax; test end to end in test mode on staging, then set `PAYMENTS_ENABLED=true` (build phase 7).
9. **Open the doors:**
   - Set `SIGNUP_MODE=open`.
   - Move tester overrides either to an expiry date or to permanent grandfathered Premium (`entitlement_overrides.reason = 'prototype tester'`).
   - Remove the "Preview" badge (`NEXT_PUBLIC_APP_STAGE=launch`).
10. **Growth:** SEO stage 1 (§9.8), PWA (phase 8), then the Capacitor apps (phase 10).

Build phase 7 starts after steps 0–6.

**When to start the runbook:** any of these:
- funding is secured
- more than about 20 testers
- the database approaches 400 MB
- Supabase pausing or free-plan limits get in the way of demos
- you're ready to take payments

## 11.6 Cost ramp

| Stage | What's running | Approx. monthly |
|---|---|---|
| **Stage 0: prototype** | Free tiers above, domain | **~£1** (domain amortised) |
| **Stage 0.5: private beta** (optional) | Supabase Pro (prod only), hosting free/paid, R2 still free, browser PDFs | ~£20–40 |
| **Stage 1: public launch** | Full setup in §4: staging + production, Worker, PDF worker, Stripe, private GitHub repo on a paid plan | ~£65–110 (plus Stripe fees per transaction) |

## 11.7 Investor demo script (defines Stage 0 scope)

The demo comes first; Stage 0 builds what it shows. It runs about 4 minutes on a phone, mirrored to a screen, with a desktop fallback.

1. **The problem** (spoken): "You're moving somewhere you don't know. Today that means 15 browser tabs."
2. **Set criteria:** a family with a primary-age child, budget £550k for a semi, no flood risk, and a pin at "Work: East Croydon" within 5 km. The map recolours instantly and the ranked list updates.
3. **Compare within England vs my region:** show how the colours change. The scores are honest and comparable.
4. **Open the top result from the list:** "why this colour?" shows each criterion's contribution, and the median semi price says "based on 14 sales".
5. **Zoom to a street:** plot boundaries appear; tap one to see its area.
6. **Street report:** scroll it, then "Download PDF" (browser print).
7. **Trust:** a quick look at the methodology page.
8. **Business model:** the pricing page (Free / Starter / Premium), then the tester numbers from PostHog (§14).

**Scope rule:** in Stage 0, only what the script shows is built in full. Everything else is built only as far as its code path or switch, so it can be finished in Stage 1 without rework.

| Built in full in Stage 0 | Built minimally in Stage 0 |
|---|---|
| Map, criteria panel, match engine, pins, England/region toggle, ranked list | Property report: a plot card (area, nearest-postcode sales, flood status), with the full report in Stage 1 |
| Area panel with "why this colour?" | Data-issue reporting: a simple form, stored only (no Turnstile, no notifications) |
| Plot boundaries and plot tap | Entitlements: resolver + overrides + gates; no account page beyond sign-out |
| Street report + browser PDF | Accounts: email-code sign-in, invite-only |
| Methodology page (generated) | Search: postcodes + places (streets in Stage 1) |
| Pricing page (display only) + willingness-to-pay prompt (§14) | |

Rehearse the script against the deployed prototype before every investor meeting, with Supabase confirmed awake.

## 11.8 Rules during Stage 0 (to keep scale-up cheap)
- Use **migrations from day one**, through the Supabase CLI, even with one environment. Never change the schema in the dashboard.
- Keep the **feature registry and entitlement checks** in place even though payments are off. Testers' access comes from overrides, not special-case code.
- **Never hard-code the coverage area**, the map file location or the PDF mode; always read the stage switches.
- Keep CI and tests running. They're free and protect the demo.
- Even with 5 testers, hold minimal personal data: a short privacy note in the invite and on the site, and testers' emails only.

## 11.9 Repository visibility: public in Stage 0, private at scale-up
**Stage 0: build in public (deliberate).** The repo, including this blueprint (pricing, roadmap, demo script, pitch metrics, open decisions), is public. The accepted trade-off:
- **Benefits:** free unmetered CI, free secret scanning with push protection, free branch protection, and a visible build history to show investors and future hires.
- **Cost:** competitors can read the plan.
- **Mitigation:** the advantage is in execution, data quality and speed, not in the document.

**Never commit, even though the repo is public:**
- secrets or `.env` files
- tester names, emails or feedback (`docs/research/` stays anonymised: "Tester A")
- investor names, terms or pitch decks
- third-party material (e.g. `docs/reference/`, which is git-ignored)
- raw or staged data

Anything pushed while public **stays public forever** (clones, forks, caches), even after the repo is made private or history is rewritten.

**Code licence:** the `LICENSE` file follows D15 (§16). Being public doesn't make the code open source: without a licence that grants rights, others may read it but not reuse it.

**Going private (scale-up runbook step 0, §11.5).** Make the repo private when the first of these happens:
- funding is secured
- the runbook starts
- real customer or commercial details start to appear in issues, pull requests or docs

**What changes when it goes private:**

| Feature | Public (Stage 0) | Private (Stage 1) |
|---|---|---|
| GitHub Actions | Free, unmetered on standard runners | Monthly minutes allowance on the paid plan; larger runners billed |
| Branch protection / rulesets | Free | Needs GitHub Pro (personal) or Team (organisation) |
| Secret scanning + push protection | Free | **Not used** (paid only). Rely on the commit checks below. |
| Dependabot alerts | Free | Free |

**Secret safety after going private:** no paid scanning. Rely on the strict `.gitignore`, only committing `.env*.example` files with empty values, and checking every diff before committing (the list above). If a secret is ever committed, rotate the key straight away; deleting the commit isn't enough.
