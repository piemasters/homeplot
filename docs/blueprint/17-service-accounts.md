# 17. Service Accounts & What Each Needs

Every external account the project needs: when it's needed, what it costs, what you need ready to sign up, what to configure, and which keys it produces.

Costs are approximate as of late 2026. **Check current pricing and terms when signing up.**

## 17.1 Before you start

**Have these ready:**
- **Final name and domain decided** (D1 in §16). Almost every account below asks for the product name, website URL and a support email.
- **A domain email address** for all service logins, e.g. `ops@<domain>`, plus `support@<domain>` for users. Decide the provider first (D12).
  - Use one shared ops address rather than a personal email, so accounts can be handed over to co-founders or staff later.
- **A password manager**, e.g. Bitwarden or 1Password, with three vaults: `homeplot-stage0`, `homeplot-staging` and `homeplot-prod`.
  - Every key and password listed below goes in it, never in chat, email or the repo.
- **An authenticator app or hardware key:** turn on **two-factor authentication on every account**, and save the recovery codes in the password manager.
- **A payment card.** Some free tiers still need a card on file (e.g. Cloudflare R2).
- **For Stage 1:** a business structure decision (D2) and a business bank account.

**Setting up each account:**
1. Sign up with the ops email.
2. Turn on two-factor authentication.
3. Choose the UK/EU region where offered.
4. Store each key in the password manager.
5. Add the key to the right environment (`.env.local`, the host's environment variables, or GitHub Actions secrets) under the variable names in §10.

## 17.2 Stage 0: prototype (needed for phases 0–6)

| # | Service | Purpose | Cost |
|---|---|---|---|
| A1 | Domain registrar | Domain name | ~£10–15/yr for `.co.uk` (`.app` / `.com` more) |
| A2 | Cloudflare | DNS, R2 map-file storage, email routing (optional) | Free (R2 free tier needs a card on file) |
| A3 | GitHub | Code, CI, scheduled jobs | Free |
| A4 | Supabase | Database, auth, storage | Free plan |
| A5 | Netlify | Hosting the web app | Free plan |
| A6 | Resend | Sign-in emails (custom SMTP for Supabase) | Free tier |
| A7 | Sentry | Error tracking | Free (Developer) plan |
| A8 | PostHog | Product analytics | Free tier |
| A9 | Upstash (optional in Stage 0) | Rate limiting | Free tier |
| A10 | Figma (optional) | Wireframes and brand (§18) | Free plan |

### A1. Domain registrar
- **Suggested registrar:** Cloudflare Registrar (sells at cost and keeps DNS in one place). It doesn't sell `.co.uk`; for that, use any registrar and point the nameservers at Cloudflare.
- **Sign-up needs:** name, address, email and payment card. UK `.co.uk` domains publish registrant details unless you opt out (individuals can opt out).
- **Configure:** auto-renew **on**, and registrar lock **on**.
- **Produces:** the domain. Nameservers point to Cloudflare (A2).

### A2. Cloudflare
- **Sign-up needs:** ops email; a payment card (needed to enable R2 even on the free tier).
- **Configure:**
  - Add the domain (DNS).
  - **R2:**
    - Create the bucket `tiles-stage0` with public access through a custom subdomain, e.g. `tiles.<domain>`.
    - **CORS:** allow `GET` and `HEAD` from the app's origins, and expose the `Range`, `Content-Range`, `ETag` and `Content-Length` headers. PMTiles needs range requests.
  - **API token:** R2 read/write for the upload script.
  - Optional: **Email Routing** to forward `ops@` and `support@` to your inbox for free.
- **Produces:** `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, and the public bucket URL for `NEXT_PUBLIC_TILES_BASE_URL`.

### A3. GitHub
- **Sign-up needs:** an account with two-factor authentication.
- **Configure:**
  - The repo is **public in Stage 0** (§11.9) and goes private at scale-up (runbook step 0).
  - **Branch protection / ruleset on `main`:** pull request required, CI status checks required, no force-pushes, no deletions (§12.1).
  - **Secret scanning with push protection** on while the repo is public (free), plus **Dependabot alerts** and security updates.
  - **Actions secrets** for the keep-awake job and CI: `DATABASE_URL`, Supabase CLI access token, and later the Stage 1 deployment secrets.
  - Repo settings: squash merge only, auto-delete branches after merge.
- **Produces:** the repo, CI and the scheduled keep-awake job (§11.2).
- **At scale-up:** a paid plan (Pro for a personal account, Team for an organisation) keeps branch protection on a private repo. Secret scanning stops when the repo goes private; no paid replacement (§11.9).

### A10. Figma (optional)
- **Sign-up needs:** ops email.
- **Configure:** a team or project `homeplot` on the free plan; export agreed wireframes to `docs/design/` (§18.4).
- **Produces:** nothing secret. Design files only.

### A4. Supabase
- **Sign-up needs:** GitHub login or ops email.
- **Configure:**
  - **Organisation** `homeplot`; **project** `homeplot` in the **London (eu-west-2)** region.
  - A strong **database password**, stored in the password manager.
  - Enable the **PostGIS** extension (the migration does this).
  - **Auth settings:**
    - Site URL = the app URL; add the redirect URLs (localhost, the Netlify URL, the custom domain).
    - **Disable new sign-ups** (invite-only in Stage 0).
    - Email OTP (6-digit code) enabled.
    - **Custom SMTP** set to the Resend details (A6).
    - Email templates branded with the product name.
  - **Storage:** none needed in Stage 0 (PDFs come from the browser).
  - Install the **Supabase CLI** locally and `supabase link` it to the project.
  - Create a **personal access token** for the CLI and CI.
- **Produces:**
  - `NEXT_PUBLIC_SUPABASE_URL`
  - the public key (`NEXT_PUBLIC_SUPABASE_ANON_KEY`; newer projects call it the *publishable* key)
  - the server key (`SUPABASE_SERVICE_ROLE_KEY`; the *secret* key on newer projects). **Server only, never in the browser.**
  - `DATABASE_URL` (pooler, port 6543)
  - `DIRECT_DATABASE_URL` (port 5432)
  - the CLI access token
- **Note:** free projects pause after about 7 days without activity (§11.2).

### A5. Netlify
- **Sign-up needs:** GitHub login.
- **Configure:**
  - Import the repo. Base directory is `apps/web`; Netlify's Next.js support is automatic.
  - Add the **environment variables** from §10 with the Stage 0 values.
  - Add the custom domain through Cloudflare DNS (a CNAME to Netlify), with HTTPS on.
  - Turn off deploy previews for forks.
- **Produces:** the site URL.
- **Check:** the free plan's current build and bandwidth limits, and confirm its terms allow commercial use.

### A6. Resend
- **Sign-up needs:** ops email.
- **Configure:**
  - Add a sending domain (e.g. `mail.<domain>`) and add its **SPF, DKIM and DMARC** DNS records in Cloudflare, then wait for verification.
  - Create SMTP credentials and enter them in Supabase Auth's SMTP settings. Sender: `HomePlot <no-reply@mail.<domain>>`.
- **Produces:** SMTP host, port, user and password (stored in Supabase, not in the app). An API key is only needed later, for alert emails.

### A7. Sentry
- **Sign-up needs:** ops email. Choose the **EU data region** at sign-up; it can't be changed later.
- **Configure:**
  - Organisation, plus projects: `web` (Next.js), `etl` (Python), and in Stage 1 `pdf-worker`.
  - Alert rules: new issue → email.
  - An auth token for uploading source maps from CI.
- **Produces:** `NEXT_PUBLIC_SENTRY_DSN` (per project) and `SENTRY_AUTH_TOKEN`.

### A8. PostHog
- **Sign-up needs:** ops email. Choose **EU Cloud**.
- **Configure:**
  - Project `homeplot`.
  - Turn off autocapture of text and inputs, so only the typed events in §14 are sent.
  - Turn session replay off by default; if used, mask all inputs.
  - Create the "Pitch" dashboard (§14.2).
- **Produces:** `NEXT_PUBLIC_POSTHOG_KEY` and `NEXT_PUBLIC_POSTHOG_HOST` (`https://eu.i.posthog.com`).

### A9. Upstash (optional in Stage 0)
- **Sign-up needs:** ops email or GitHub login.
- **Configure:** a Redis database in an EU region close to the host.
- **Produces:** `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN`.

## 17.3 Stage 1: launch (from the scale-up runbook, §11.5)

| # | Service | Purpose | Cost |
|---|---|---|---|
| B1 | Supabase Pro + staging project | Production database; separate staging | ~$25/mo + compute; staging free or Pro |
| B2 | Vercel Pro (if moving from Netlify, D5) | Hosting | ~$20/mo per seat |
| B3 | Cloudflare Workers paid + Turnstile | Map-file Worker, bot protection | ~$5/mo; Turnstile free |
| B4 | PDF worker host: Fly.io, Railway or Google Cloud Run | Playwright container | ~£0–10/mo, scales to zero |
| B5 | Stripe | Payments, tax, invoices | No monthly fee; per-transaction fees |
| B6 | Google Cloud (OAuth only) | "Sign in with Google" | Free |
| B7 | Apple Developer Program | "Sign in with Apple", then the iOS app | ~£79/yr (US$99) |
| B8 | Google Search Console + Bing Webmaster Tools | SEO monitoring, sitemaps | Free |
| B9 | Slack or Discord (optional) | Ops notifications webhook | Free |
| B10 | GitHub paid plan (Pro or Team) | Keep branch protection once the repo is private (§11.9) | ~$4/user/mo |

### B1. Supabase (Pro and staging)
- **Configure:**
  - Upgrade the existing project to Pro (it becomes **production**). Turn on the **spend cap**.
  - Create a second project, `homeplot-staging` (London), and apply the migrations with the CLI.
  - Storage bucket `reports` (private) for PDFs.
  - Auth: enable **Google** and **Apple** providers (B6, B7); turn sign-up **on**.
  - Add the **custom access token hook** (§9.5).
- **Produces:** a second set of keys and URLs for staging; store them in the `homeplot-staging` vault.

### B2. Vercel (if chosen)
- **Sign-up needs:** GitHub login and a payment card.
- **Configure:**
  - Import the repo (root `apps/web`).
  - Function region `lhr1`.
  - **Preview** variables → staging services and Stripe test keys; **Production** variables → production.
  - Spend management with a hard limit.
  - Bot protection / firewall rules.
  - Custom domain via Cloudflare DNS.

### B3. Cloudflare Workers paid plan and Turnstile
- **Configure:**
  - Deploy `services/tiles-worker` with `wrangler` (production and staging buckets `tiles-prod`, `tiles-staging`) on the route `tiles.<domain>`.
  - Billing notifications.
  - **Turnstile:** add a site widget for the domain.
- **Produces:** the Worker URL (→ `NEXT_PUBLIC_TILES_BASE_URL`), `NEXT_PUBLIC_TURNSTILE_SITE_KEY` and `TURNSTILE_SECRET_KEY`.

### B4. PDF worker host
- **Sign-up needs:** ops email and a payment card.
- **Configure:**
  - Deploy the `services/pdf-worker` container in a London or EU region, scaling to zero, with a memory limit of about 1–2 GB.
  - Set a shared secret.
  - Spend alert.
- **Produces:** `PDF_WORKER_URL` and `PDF_WORKER_SECRET`.

### B5. Stripe
- **Sign-up needs:**
  - Legal business details: sole trader name and address, or the company name, number and registered address.
  - **Identity verification** of the owner (photo ID).
  - A **UK business bank account** for payouts.
  - A public website with a **product description, prices, terms, privacy, refund and cancellation policy, and contact details**. Stripe reviews these.
  - A statement descriptor (e.g. `HOMEPLOT`).
  - A support email and phone.
- **Configure:**
  - **Products:** Starter and Premium, each with monthly and yearly **Prices** in GBP, with **lookup keys** `starter_monthly`, `starter_yearly`, `premium_monthly`, `premium_yearly`. Set these up in **test mode first**, then live.
  - **Customer Portal:** allow plan switching, cancellation (at period end), payment-method updates and invoice history.
  - **Stripe Tax:** turn it on and add the UK VAT registration once registered.
  - **Webhook endpoint** `https://<domain>/api/webhooks/stripe` with the events from §9.6, set up in both test and live mode.
  - Branding: logo and colours.
- **Produces:** `STRIPE_SECRET_KEY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` and `STRIPE_WEBHOOK_SECRET`, one set for test (staging) and one for live (production). Install the **Stripe CLI** locally for testing webhooks.

### B6. Google Cloud: "Sign in with Google"
- **Sign-up needs:** a Google account (the ops address) and a Cloud project `homeplot`.
- **Configure:**
  - **OAuth consent screen:**
    - App name, support email, logo.
    - **Privacy policy and terms URLs.**
    - Authorised domain = your domain.
    - Scopes: `openid`, `email`, `profile` only.
    - Publish the app; it may go through Google's verification.
  - **OAuth client (Web):** the authorised redirect URI is the Supabase callback URL (`https://<project>.supabase.co/auth/v1/callback`), one for each Supabase project.
- **Produces:** a client ID and client secret, entered in the Supabase Auth Google provider (not in the app's environment).

### B7. Apple Developer Program
- **Sign-up needs:**
  - An Apple ID with two-factor authentication.
  - **As an organisation:** legal entity name, a **D-U-N-S number** (free from Dun & Bradstreet, but it can take days to weeks, so apply early), a website, and authority to sign for the company.
  - **As an individual:** your legal name appears as the seller in the App Store.
  - Annual fee.
- **Configure (Sign in with Apple, for web):**
  - An App ID with Sign in with Apple enabled.
  - A **Services ID** (the web client ID) with the domain and the Supabase callback URL as return URL.
  - A **Sign in with Apple key (.p8)**, plus the Key ID and Team ID.
  - Generate the client secret (a JWT) for Supabase. **It expires after at most 6 months.** Put a calendar reminder to rotate it, or sign-ins break.
- **Later (phase 10):** App Store Connect app record, push notification key, in-app purchase products (if D11 is IAP), privacy "nutrition label", and an in-app account deletion flow (already built, §9.5).
- **Produces:** Services ID, Team ID, Key ID and the .p8 key (in the password manager), entered in the Supabase Apple provider.

### B8. Google Search Console and Bing Webmaster Tools
- **Configure:** verify the domain with a DNS TXT record (Cloudflare), submit the sitemaps (§9.8), and watch coverage and indexing.

### B9. Ops notifications (optional)
- **Configure:** an incoming webhook to a private channel.
- **Produces:** `OPS_NOTIFY_WEBHOOK_URL`.

### B10. GitHub paid plan (when the repo goes private)
- **When:** scale-up runbook step 0 (§11.5).
- **Configure:** upgrade to Pro (personal) or Team (organisation; consider creating a `homeplot` GitHub organisation and transferring the repo). Then re-check branch protection, Actions minutes and the hosting/Supabase GitHub app access. Free secret scanning stops for private repos; no paid add-on (§11.9).
- **Produces:** nothing new; existing secrets carry over.

## 17.4 Later (phase 10 and beyond)

| # | Service | When | Needs |
|---|---|---|---|
| C1 | **Google Play Console** | Phase 10 (Android app) | One-off ~US$25 fee. Identity verification; **organisation accounts need a D-U-N-S number**. Newer personal accounts must run a closed test with testers before going to production (check current rules). |
| C2 | **App Store Connect** | Phase 10 | Comes with B7. App listing, screenshots, privacy details, review notes and a demo account for Apple reviewers. |
| C3 | **RevenueCat** | Phase 10, only if D11 = in-app purchases | Free up to a revenue threshold. Connect App Store and Play credentials; set its webhook to update `subscriptions` (`source = app_store / play_store`). |
| C4 | **EPC Register** (data) | v1.5 | Free registration with email; produces an API key (`EPC_API_KEY`). Read and follow the licence conditions. |
| C5 | **Anthropic API** | Only if plain-English search is built (§3.1) | Console account, billing and a spend limit; produces `ANTHROPIC_API_KEY` (server only). |
| C6 | **Routing / commute time** (TravelTime API, or self-hosted r5) | v2 isochrones | API key and plan, or a container host. |

## 17.5 Data sources: no accounts needed (v1)

These are downloaded directly, so there are no keys to manage. Record the **licence and attribution** for each on the attribution page (§9.9).

| Source | Notes |
|---|---|
| ONS Open Geography Portal (boundaries, ONSPD) | OGL |
| Nomis (Census 2021 bulk) | No account needed for bulk downloads |
| HM Land Registry Price Paid | OGL; attribution wording required |
| HM Land Registry INSPIRE Index Polygons | Accept the licence on download; **confirm the uses (D10)** |
| data.police.uk | OGL |
| DfE Get Information About Schools; Ofsted management information | OGL |
| NaPTAN | OGL |
| Geofabrik (OpenStreetMap extracts) | ODbL; credit © OpenStreetMap contributors |
| OS Open Greenspace, OS Open Roads | OGL; no OS Data Hub account needed for downloads |
| Environment Agency flood data (Defra Data Services Platform) | OGL; some datasets have their own terms |
| English Indices of Deprivation 2025 | OGL |
| planning.data.gov.uk | OGL |
| postcodes.io | Free public API, no key. Respect its fair use, or self-host. |
| Protomaps background map builds | ODbL (OpenStreetMap-derived); credit required |

## 17.6 Business & legal registrations

| # | Registration | When | Needs | Cost |
|---|---|---|---|---|
| L1 | **Companies House** (if a limited company, D2) | Before Stage 1 accounts | Company name, registered office address, director and shareholder details, SIC code | ~£50 online |
| L2 | **HMRC** | Sole trader: Self Assessment registration. Company: Corporation Tax. **VAT** once over the threshold | National Insurance number / company details | Free |
| L3 | **Business bank account** | Before Stripe live | ID and business details | Usually free tiers available |
| L4 | **ICO data protection fee** | Before taking payments (arguably once holding testers' emails; check the ICO self-assessment) | Organisation details, data protection contact | ~£50/yr (tier 1) |
| L5 | **D-U-N-S number** | As soon as D2 is decided (needed for Apple and Google organisation accounts) | Company details | Free |
| L6 | **UK IPO trademark** (optional but recommended) | Once the name is final | Name, classes (e.g. 9 software, 42 SaaS, 36 real-estate information) | ~£170 for one class online, plus ~£50 per extra class |

## 17.7 Account inventory
Keep `docs/ops/accounts.md` (no secrets in it) with, for each account:
- service and URL
- login email
- owner
- two-factor method
- plan and monthly cost
- renewal date
- which keys it produced, and where they're stored (password manager item and environment variable names)
- the date keys were last rotated

Review it quarterly, along with the cost review (§4).
