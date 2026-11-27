# 1. Product Summary

**Goal:** help people find where to live next by turning open data into a map that answers "which areas suit _me_?"

**Core loop:**

1. **Set criteria.** Hard filters (e.g. not in a flood zone, semis under £500k) plus importance sliders (schools, safety, green space, amenities, transport).
2. **See a colour-graded match map.** Every area is shaded by how well it matches.
3. **Tap an area.** See why it scored that way, recent sales and key facts.
4. **Download a report** for an area, street or property.

**Target users:** home buyers and renters comparing unfamiliar areas, especially people relocating, upsizing or moving for schools.

**Platforms:** mobile-first web app, released as a PWA, then wrapped with Capacitor for the App Store and Play Store (see §8).

**Pricing:** Free (no account, no running cost to us), Starter and Premium (account required). Features are assigned to tiers in config, not code (see §9).

**Delivery:** Stage 0 is a free-tier prototype for about 5 testers, used to secure funding. Stage 1 is the public launch. Same architecture, scaled up by configuration and plan upgrades (see §11).

**v1 scope:** England only.

**v1 non-goals:**

- Wales, Scotland and Northern Ireland
- Property listings
- Planning applications and development plans
- Commute-time isochrones

These are on the roadmap (§3).

**Principles:**

- **Easy data first.** v1 uses only free, openly licensed bulk downloads with no approval process.
- **Explainable scores.** Every colour and grade can be traced back to the numbers behind it.
- **No protected characteristics as criteria.** Users can never search, filter or score areas by ethnicity, religion or country of birth. See §6.4.
