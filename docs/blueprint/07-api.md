# 7. API Endpoints

All responses use real JSON types (numbers as numbers), empty arrays rather than `null`, and flat structures. Every response includes `dataVersion`.

## `GET /api/search?q=`

Search postcodes (postcodes.io / ONSPD), places and streets. Returns up to 10 results, each with `type`, `label`, `lat`, `lng`, `bbox` and `slug`.

## `GET /api/areas/nearby?lat=&lng=&limit=20`

Nearby places with the population-weighted average of their area scores:

```json
{
  "dataVersion": 12,
  "origin": { "lat": 51.3444, "lng": -0.1098 },
  "places": [
    {
      "id": 10648,
      "name": "Purley",
      "slug": "purley",
      "type": "suburb",
      "lat": 51.3388,
      "lng": -0.1159,
      "distanceKm": 0.75,
      "overallScore": 59.3,
      "grade": "C",
      "scores": {
        "schools": 79.4,
        "safety": 61.2,
        "transport": 84.0,
        "amenities": 84.2,
        "greenspace": 70.1
      },
      "tags": ["great-high-street", "green-and-leafy"]
    }
  ]
}
```

## `GET /api/areas/[oaCode]`

Full panel data for one OA:

- scores
- the explanation inputs used by "why this colour?"
- flood and conservation status
- median prices
- the latest 5 sales
- nearest schools and stations

## `POST /api/search/polygon`

- **Body:** `{ "geometry": GeoJSON Polygon, "criteria"?: Criteria }`, where the polygon comes from terra-draw (circle, polygon or freehand).
- **Returns:**
  - population-weighted averages for the drawn area
  - its best-matching OAs
  - the streets and places inside it
  - its sales summary

## `GET /api/reports/street/[slug]` and `GET /api/reports/property?postcode=&paon=&saon=` or `?lat=&lng=&inspireId=&areaM2=`

The property report can be opened from an address search or from a tapped plot. For a tapped plot:

- The location comes from the tap.
- Plot area is included.
- Sales are matched from the nearest postcode. Exact address matching waits for EPC/UPRN in v1.5.

Report data:

- grade and summary
- match against criteria (if passed)
- sales table (previous vs latest 12 months, by type)
- recent sales
- crime summary
- nearby schools
- flood status at the point
- demographics (non-protected)
- price vs quality

The property report adds that address's own sale history from Price Paid. EPC data is added in v1.5.

## `GET /api/reports/[kind]/[id]/pdf`

Checks Supabase Storage for `reports/{kind}/{id}-{dataVersion}.pdf`. If it's missing:

1. Ask the PDF worker to render `/reports/print/[kind]/[id]`.
2. Store the result.
3. Return a signed URL.

Requires the `pdf_reports` entitlement:
- `consumeQuota` runs before rendering, keyed on `{kind}:{id}:{dataVersion}`, so re-downloading uses no extra quota.
- When the quota is used up, the route returns 403 with an `upgrade` payload.

## Gating & public caching (all routes)
- Free routes (`/api/search`, `/api/areas/*`) are public and CDN-cached with `s-maxage` keyed to the data version.
- Paid routes require a session and call `requireFeature` / `consumeQuota`.
- Paid responses are never publicly cached.

## Account & billing
- `GET /api/me/entitlements`: plan, resolved features, quota used/remaining, and the reason a feature is unavailable (`not_signed_in` | `plan` | `quota`).
- `POST /api/billing/checkout`: `{ lookupKey }` → Stripe Checkout URL. Requires sign-in.
- `POST /api/billing/portal`: Stripe Customer Portal URL.
- `POST /api/webhooks/stripe`: signature-verified, idempotent; updates `subscriptions`. Not rate-limited by IP.
- `GET /api/me/export`: a JSON export of the user's data.
- `DELETE /api/me`: cancels the subscription and deletes the account and its data.
- `GET /api/plans`: plan names, features and current Stripe prices for the pricing page. Public, cached.

## Trust
- `POST /api/data-issues`: `{ oaCode?, lat?, lng?, metricId, message, contactEmail?, turnstileToken }`.
  - Verifies the Turnstile token, then stores the report and sends a notification.
  - Anonymous allowed; strictly rate-limited.
