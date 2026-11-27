# 3. Roadmap Data (v1.5 and later)

| Item                                                                        | Why it's deferred                                                                                                              | Target                                            |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| **EPC register** (energy rating, floor area, £/m², property age band, UPRN) | Free, but needs registration/API key; licence conditions restrict some uses                                                    | **v1.5**: first add-on, enriches property reports |
| OS Open UPRN                                                                | Easy, but only useful once EPC links addresses to UPRNs                                                                        | v1.5                                              |
| DfE school performance (KS2/KS4)                                            | Easy, but needs careful presentation                                                                                           | v1.5                                              |
| **Commute isochrones**                                                      | Needs a routing engine (r5 / OpenTripPlanner with GTFS + NaPTAN) or the paid TravelTime API                                    | v2                                                |
| Walking-network distances                                                   | Needs pgRouting or precomputed isochrones                                                                                      | v2                                                |
| Road & rail noise (Defra strategic noise maps)                              | Easy download; lower priority                                                                                                  | v2                                                |
| Broadband & mobile coverage (Ofcom Connected Nations)                       | Postcode-level CSVs; joining them needs care                                                                                   | v2                                                |
| Air quality, radon                                                          | Mixed formats and licences                                                                                                     | v2+                                               |
| Wales (Estyn, NRW flood, WIMD), then Scotland                               | Different sources for every theme                                                                                              | v2+                                               |
| **Planning applications & future development plans**                        | No complete national dataset; planning.data.gov.uk coverage is growing but patchy, and scraping per council isn't worth it yet | Watch; v3                                         |

## 3.1 Optional later features (not data)

**Plain-English search (optional, post-launch).**
- **What it does:** users type what they want, e.g. "quiet area, good primary school, semi under £500k, near Croydon", and the criteria form fills itself in. Users review and adjust it before anything changes on the map.
- **How it works:**
  - A server route sends the text to the latest **Claude Haiku** model via the Anthropic API, with the `Criteria` JSON schema from `packages/shared` as a structured-output / tool schema.
  - The response is validated with zod before use. Anything the schema can't express (e.g. "near the sea") is listed back to the user as "not supported yet", never guessed.
  - Place names in the text are geocoded with the existing search API.
  - Only the user's text is sent: no account details or history.
- **Cost:** it costs money per use, so the free-tier rule (§9.1) applies.
  - Register it in `features.yaml` as `plain_english_search` (`type: quota`, `cost: metered`, `status: planned`).
  - Give it a monthly allowance in Starter and Premium (e.g. 30 / 100), set in `plans.yaml`.
  - Cap the input length and rate-limit the route.
- **When:** after Stage 1 launch, once criteria usage data (§14) shows which phrasings people need. It's also good material for a later funding round.
