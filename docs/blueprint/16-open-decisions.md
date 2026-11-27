# 16. Open Decisions

Unresolved choices in one place. When a decision is made:
1. Update its row (status ✅, the decision and the date).
2. Update the blueprint file it affects.
3. Delete the row once it no longer matters.

The coding agent must not make these decisions itself. If work is blocked on one, it asks.

| # | Decision | Options | Blocks | Needed by | Status |
|---|---|---|---|---|---|
| D1 | **Final product name & domain** | HomePlot (working name), others from the shortlist | Domain, email, every service account (§17), branding | Before creating service accounts | Open. Check Companies House, UK IPO, domains |
| D2 | **Business structure** | Sole trader / limited company | Stripe account type, Apple/Google developer enrolment (organisations need a D-U-N-S number), ICO registration, bank account | Before Stage 1 accounts; ideally before investor talks | Open |
| D3 | **Prototype coverage area** | ~5–8 boroughs/districts around Croydon/Sutton, or another area of your choice. **If experiment E4 shows it won't fit in 400 MB:** a smaller, more densely populated area (e.g. 2–3 inner boroughs) | ETL loading, coverage bbox, demo script | Phase 2 (after E4) | Open |
| D4 | **Stage 0 hosting** | Netlify free / Cloudflare Workers free (OpenNext) | Deployment setup | Phase 1a (after E7) | Open (Netlify suggested) |
| D5 | **Stage 1 hosting** | Netlify paid / Vercel Pro | Scale-up runbook step 2 | Scale-up | Open |
| D6 | **Ofsted scoring method** | Report-card areas weighted / legacy grades fallback / proximity only | Schools score | Phase 5 | Open |
| D7 | **Default weights & grade bands** | Values in `metrics.yaml` | Overall score, grades | Phase 5, then review with tester feedback | Draft in §6.1 |
| D8 | **Prices** | Placeholders: Starter £4.99/mo · £39/yr; Premium £9.99/mo · £79/yr | Pricing page, Stripe products | Display in phase 6; final before phase 7, informed by the willingness-to-pay results (§14) | Placeholder |
| D9 | **Final tier feature split** | `plans.yaml` | Phase 7 | Before phase 7 | Draft in §9.1 |
| D10 | **INSPIRE licence check** | Confirm the planned uses (display, plot area, reports) are allowed | Plots in a public product | Before Stage 1 launch; ideally before demos | Open |
| D11 | **iOS purchases** | Apple in-app purchase (RevenueCat) / web-only purchases | Capacitor apps | Phase 10 | Open |
| D12 | **Domain email provider** | Cloudflare Email Routing (free forwarding) / Google Workspace / Fastmail | Support and ops addresses | Before creating service accounts | Open |
| D13 | **When to add Wales** | After launch / later | Data roadmap | Post-launch | Deferred |
| D14 | **Plain-English search** | Build / don't build | §3.1 | Post-launch | Deferred |
| D15 | **Code licence** (public repo, §11.9) | **All rights reserved** (a `LICENSE` stating no reuse permitted; recommended for now) / source-available (e.g. PolyForm Noncommercial) / open source (e.g. MIT) | `LICENSE` file, what others may do with the code | Phase 0a | Open |
| D16 | **Target investor demo date** | A date that fits the milestone estimates (§12.2) and your available time | Scope cuts, milestone planning | Phase 0 | Open |
| D17 | **When to make the repo private** | Funding secured / runbook start / first commercial details (§11.9) | Runbook step 0, GitHub plan cost | Scale-up | Planned: at runbook step 0 |
