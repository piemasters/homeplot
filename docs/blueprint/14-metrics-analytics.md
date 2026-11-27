# 14. Metrics & Analytics Events

Decide up front what to show investors and what drives product decisions, and define the tracking events now so the data exists from the first tester session. PostHog, EU cloud, loaded only after consent (§4).

## 14.1 Pitch metrics (Stage 0, about 5 testers)

| Metric | Definition | Why investors care |
|---|---|---|
| **Activation** | % of testers who, in their first session, change at least one criterion **and** open an area (`criteria_changed` + `area_opened`) | People understand and use the core idea |
| **Return visits** | % of testers active in 2+ separate weeks | It's useful beyond novelty |
| **Depth** | Median areas opened and reports viewed per tester per week | It supports a real decision |
| **Feature pull** | Share of sessions using pins, the region toggle, the ranked list and plots | What to build next |
| **Willingness to pay** | Answers to the willingness-to-pay prompt (§14.3), plus interview answers | It validates the tiers and prices |
| **Qualitative** | Quotes from tester interviews | Story for the deck |

With 5 testers, these numbers are directional. Present them next to the qualitative evidence, never as statistically meaningful.

## 14.2 Event taxonomy
- **Event naming:** `object_action`, snake_case, past tense.
- **Definitions:** in `packages/shared/analytics/events.ts` as a typed map: event name → property types. The app can only send events defined there, and adding an event means adding it to the map.
- **Privacy:** no personal data in properties. Never send emails, full postcodes, exact coordinates or free text. Locations are reduced to the **postcode district** (e.g. `CR8`) or OA code.

| Event | Key properties |
|---|---|
| `search_performed` | `result_type` (postcode, place, street), `result_count` |
| `criteria_changed` | `changed` (list of criteria IDs), `weights` (id → 0–3), `filters_active` (list), `compare_within`, `pins_count` |
| `results_list_opened` | `scope` (view, near_place), `result_count` |
| `area_opened` | `source` (map, list, search), `oa_code`, `match_pct`, `rank` (if from list) |
| `why_colour_opened` | `oa_code` |
| `compare_within_toggled` | `to` (england, region) |
| `pin_added` | `pins_count`, `max_km` |
| `plot_selected` | `area_m2_band` (e.g. `<200`, `200–500`, ...) |
| `report_viewed` | `kind` (street, property), `entry` (panel, list, search) |
| `pdf_downloaded` | `kind`, `mode` (browser, worker) |
| `methodology_viewed` | `section` |
| `pricing_viewed` | `source` |
| `upgrade_prompt_shown` | `feature_id`, `reason` (not_signed_in, plan, quota) |
| `wtp_answered` | `price_shown`, `answer` (yes, maybe, no), `tier` |
| `data_issue_reported` | `metric_id` |
| `signed_in` | `method` (email_code, google, apple) |
| `checkout_started` / `subscription_started` | `plan_id`, `interval` _(Stage 1)_ |

**Identity:**
- Anonymous visitors have a PostHog anonymous ID; after sign-in it's linked to the Supabase user ID (`posthog.identify`).
- Person properties: `plan_id` and `stage` only.

**Dashboards:** one PostHog dashboard, "Pitch", showing:
- the §14.1 metrics
- a funnel: `search_performed` → `criteria_changed` → `area_opened` → `report_viewed`
- feature usage

## 14.3 Willingness-to-pay prompt (Stage 0)
- **When:** shown once per tester, after their second `report_viewed`.
- **Question:** "Would you pay **£4.99/month** for full reports, PDFs and saved searches?" Answers: Yes / Maybe / No, plus optional free text, which is stored in Supabase rather than sent to PostHog.
- **Testing prices:** the price shown comes from the Starter price in `plans.yaml` (placeholder). To test other prices, show different testers different prices through a PostHog feature flag.
- **Scope:** Stage 0 only. It's switched off with `NEXT_PUBLIC_APP_STAGE=launch`.

## 14.4 Tester feedback loop
- **Invite email:** what to try, the privacy note, and how to give feedback.
- **After about a week:** a 5-question form (what you used it for, what was confusing, what's missing, would you recommend it, would you pay), then a 20-minute call.
- Keep notes in a single `docs/research/` file per tester, anonymised (Tester A, B, ...).
