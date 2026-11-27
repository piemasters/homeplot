# 10. Environment Variables (`.env.local.example`)

Each environment (§4) has its own values. Vercel Preview variables point at **staging** services and Stripe test mode; only Vercel Production variables point at production.

```env
# Environment: local | staging | production
NEXT_PUBLIC_APP_ENV=local

# Delivery stage switches (§11.3). Stage 0 values shown.
NEXT_PUBLIC_APP_STAGE=prototype          # prototype | launch
NEXT_PUBLIC_COVERAGE_BBOX=-0.25,51.28,0.0,51.45   # unset = all of England
PDF_MODE=browser                         # browser | worker
PAYMENTS_ENABLED=false
SIGNUP_MODE=invite                       # invite | open
RATE_LIMIT_ENABLED=false

# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
# App runtime: Supavisor pooler, transaction mode (use prepare: false)
DATABASE_URL=postgresql://postgres.[PROJECT]:[PASSWORD]@aws-0-eu-west-2.pooler.supabase.com:6543/postgres
# Migrations and ETL loads only: direct connection
DIRECT_DATABASE_URL=postgresql://postgres:[PASSWORD]@db.[PROJECT].supabase.co:5432/postgres

# Tiles (Cloudflare R2 + PMTiles Worker). The app reads tiles.json for current tileset versions.
NEXT_PUBLIC_TILES_BASE_URL=http://localhost:3000/tiles
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=map-tiles

# PDF worker
PDF_WORKER_URL=
PDF_WORKER_SECRET=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Rate limiting (Upstash Redis)
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=

# Bot protection (Cloudflare Turnstile) for data-issue reports
NEXT_PUBLIC_TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=

# Notifications (new data-issue reports, ops alerts)
OPS_NOTIFY_WEBHOOK_URL=

# Monitoring
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=https://eu.i.posthog.com

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```
