# 8. Mobile Strategy

**One codebase:** the Next.js web app, designed mobile-first.

**UI rules:**

- A **bottom sheet** (peek / half / full) for area details; a side panel on desktop.
- The criteria editor is a full-screen sheet on mobile.
- Tap targets are at least 44 px.
- On touch, drawing tools default to circle/radius and polygon.
- Downloading a PDF opens the native share sheet when available.

**Stage 1: PWA (launch)**

- Web app manifest, icons, offline shell and "add to home screen".
- No app-store review is needed.

**Stage 2: Capacitor (after product-market fit)**

- Wrap the hosted web app for the App Store and Play Store.
- To pass Apple's "minimum functionality" rule (guideline 4.2), add native features:
  - Push notifications for saved-search alerts
  - Geolocation ("areas near me")
  - Native share sheet for reports
  - Offline saved reports
- **In-app purchases:** selling Starter/Premium or reports _inside_ the iOS app generally requires Apple In-App Purchase (15–30% fee). Decide before Stage 2: either use IAP in the apps (e.g. via RevenueCat) or keep purchases web-only. Either way, entitlements don't change: `subscriptions.source` records the store, and the same plans apply (§9.4).
- **Sign in with Apple:** required on iOS if Google or other social logins are offered.

SEO pages (`/area/[slug]`, `/street/[slug]`) stay server-rendered for the web. The app uses the same routes.
