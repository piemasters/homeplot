# 9. Accounts, Pricing & Entitlements

## 9.1 Tiers

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
| Plain-English search _(planned, §3.1)_ | | e.g. 30 / month | e.g. 100 / month | metered |

**Possible later add-ons**, built on the same system with no code changes to gating:
- One-off "report packs" (e.g. 3 PDFs for £4.99) for any signed-in user.
- A Premium free trial (`trial_days` in `plans.yaml`).

## 9.2 Feature registry (single source of truth)

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

## 9.3 CI checks on the registry
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

## 9.4 Entitlements: how gating works
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

## 9.5 Authentication
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

## 9.6 Payments (Stripe)
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

## 9.7 Trust: methodology & data issues
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

## 9.8 Growth: SEO, rolled out gradually
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

## 9.9 Legal & compliance checklist (UK)
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
