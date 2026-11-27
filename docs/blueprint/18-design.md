# 18. Design

The investor demo (§11.7) is judged as much on look and feel as on data, so design is its own step: **phase 0c**, finished before phase 3 (§12.3). Keep it lightweight. The aim is a clear, consistent direction, not pixel-perfect mockups of every screen.

## 18.1 Deliverables (before phase 3)

**Mobile wireframes first**, then the desktop adaptations, for:

1. **Map home:** search bar, entry to the criteria sheet, legend, coverage mask, first-visit hint.
2. **Criteria sheet** (full-screen on mobile):
   - hard filters (flood, price by type, station distance, conservation areas)
   - importance sliders (0–3)
   - "near places I care about" pins
   - "compare within England / my region" toggle
3. **Map with matches:** 5-band legend; greyed-out areas that fail a filter; pins with distance rings.
4. **Bottom sheet** in its three heights (peek / half / full), with tabs for **Area** and **Ranked list**.
5. **Ranked list row** and the **area panel**, including "why this colour?" and the "based on N sales" labels.
6. **Plot card**, shown when a boundary is tapped at zoom 16+.
7. **Street report**, on screen and as the **A4 print layout**.
8. **Methodology**, **pricing** (Free / Starter / Premium, "coming soon" in Stage 0) and **sign-in** (email code).
9. **States:** loading, empty ("no areas match: loosen a filter"), not covered, offline and error.

**Brand basics:**
- **Name:** uses the working name until D1 (§16) is decided; keep the logo easy to swap.
- **Logo:** a simple wordmark plus a square icon that works at 16 px (favicon) and 512 px (PWA / app icon).
- **UI colour palette:** primary, neutral greys, success, warning and danger, all as shadcn/Tailwind theme tokens.
- **Typography:** one variable font, self-hosted and subset (§15).
- **Voice:** plain English, neutral about places. Never stigmatise an area (§6.3).
  - Use consistent terms from a short glossary: "match %", "area", "plot", "indicative boundary", "based on N sales".

## 18.2 Colour rules: brand vs data
- **The data palette is reserved.** Match bands use a colour-blind-safe sequential palette (viridis, 5 bands: dark purple → blue → teal → green → yellow). Areas that fail a filter use a light neutral grey with a subtle hatch.
- **Brand and UI colours must not clash with the data.**
  - Pick a primary colour outside the viridis hues, e.g. a warm coral/orange or a deep ink navy, so buttons, pins and highlights never look like match bands.
  - Pins and the selected-area outline use the brand primary.
  - Flood warnings use the danger/warning tokens, never a viridis colour.
- **Background map:** a light, low-saturation style so the data layers carry the colour. Dark mode is optional later (§4 background-map flavours).
- **Colour is never the only signal.** The legend has labelled bands, and the panel states the match % in text. Filtered-out areas have a hatch as well as grey. Contrast meets WCAG 2.2 AA (§6.2).
- The data palette and band breaks live in `packages/shared` (generated from `metrics.yaml` legend settings), not in the theme, so the map, legend, list and reports all match.

## 18.3 Layout & interaction rules
These add to §8.

- Thumb-reachable primary actions (bottom of the screen on mobile). Tap targets ≥ 44 px.
- The map is always visible behind the bottom sheet at peek and half height; the criteria sheet covers it fully.
- Criteria changes apply live, with no "Apply" button. Recolouring is instant (§15).
- Motion is short and functional; respect `prefers-reduced-motion`.
- Reports are designed print-first (A4 margins, page breaks between sections), then adapted for screen.

## 18.4 Tools & where things live
- **Tools:**
  - **Figma**, on the free plan (§17 A10).
  - Or pen sketches turned into **HTML mockups** built with shadcn components. Claude Code can produce these quickly, and they become the real components.
- **Where things live:**
  - Wireframe exports and decisions: `docs/design/` (PNG exports plus `design-decisions.md`, which records palette, font and key layout choices with dates).
  - Logo and icon source files: `apps/web/public/brand/`.
  - Theme tokens: in the web app's Tailwind/shadcn theme. The data palette stays in `packages/shared`.

## 18.5 Check (end of phase 0c)
- Walk through the demo script (§11.7) on the phone-sized wireframes. Every step has a screen and the flow makes sense.
- The brand primary is visibly distinct from all 5 viridis bands and the filtered-out grey.
- Record the agreed direction in `docs/design/design-decisions.md`.
