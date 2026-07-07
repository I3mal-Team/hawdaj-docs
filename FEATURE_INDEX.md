# Hawdaj — Feature Index

> نظام سياحي سعودي من ثلاث طبقات:
> - **Flutter (Mobile):** `/Users/mac/hawdaj/Untitled` (v5.10.23+40)
> - **Laravel API + Admin:** `/Users/mac/hawdaj-api` (Laravel 8.12, Sanctum)
> - **Web Frontend:** `/Users/mac/hawdaj-frontend` (Angular 16 SSR) — أُضيف في Phase 3. راجع `WEB_FRONTEND.md` + `WEB_MOBILE_COMPARISON.md`.

Status legend: ⬜ Not started · 🟡 In progress · ✅ Done (100%)

**Web column:** ✅ web full · 🟡 web partial · ❌ web none · N/A not-client. Full matrix in `WEB_MOBILE_COMPARISON.md`.

> ملفات حيّة تُحدَّث مع كل تغيير: `CHANGE_LOG.md` (سطر لكل تغيير) · `KNOWN_ISSUES.md` (أعطال تشغيلية K1..) · `DECISIONS/TECH_DEBT.md` (دَين D1..). راجع `CLAUDE.md §2` لسياسة المزامنة.

---

## Feature Inventory (مرتّبة من أعلى أولوية عمل إلى أدنى)

| # | Feature | Category | Priority | Status | Web | Doc |
|----|---------|----------|----------|--------|-----|-----|
| 01 | Authentication & Profile | Core | P0 — Critical | ✅ | ✅ (social→/social/callback, 401→logout) | `FEATURES/01_Authentication.md` |
| 02 | Home & Sliders (Discovery) | Core | P0 | ✅ | ✅ (8+ granular endpoints vs 1) | `FEATURES/02_Home.md` |
| 03 | Trips (v1 + v2 Enhanced) | Core (Flagship) | P0 | ✅ | ✅ (+PDF/print export) | `FEATURES/03_Trips.md` |
| 04 | Places | Core | P1 | ✅ | ✅ (+ChatGPT, map data) | `FEATURES/04_Places.md` |
| 05 | Stores | Core | P1 | ✅ | ✅ (+ChatGPT) | `FEATURES/05_Stores.md` |
| 06 | Restaurants (Zad / Menus / Offers) | Core | P1 | ✅ | ✅ | `FEATURES/06_Restaurants.md` |
| 07 | Tasneef (Unified Listing Hub) | Core | P1 | ✅ | 🟡 (via global-search viewAs) | `FEATURES/07_Tasneef.md` |
| 08 | Exploration / Global Search / Map | Core | P1 | ✅ | ✅ (global-search/global-map-data) | `FEATURES/08_Exploration.md` |
| 09 | Events | Core | P1 | ✅ | ✅ | `FEATURES/09_Events.md` |
| 10 | Swalef & Stories (Heritage content) | Core | P1 | ✅ | ✅ | `FEATURES/10_Swalef.md` |
| 11 | Tour Guides | Core | P1 | ✅ | ✅ | `FEATURES/11_TourGuides.md` |
| 12 | Landmarks | Core | P2 | ✅ | ✅ (/settings/landmarks) | `FEATURES/12_Landmarks.md` |
| 13 | Favorites & Saved | Supporting | P2 | ✅ | ✅ | `FEATURES/13_FavoritesSaved.md` |
| 14 | Rates & Reviews | Supporting | P2 | ✅ | ✅ | `FEATURES/14_RatesReviews.md` |
| 15 | My Properties (user submissions) | Supporting | P2 | ✅ | ✅ (v2 /settings/my-properties) | `FEATURES/15_MyProperties.md` |
| 16 | User Stories (last-day stories) | Supporting | P2 | ✅ | ✅ (+comments) | `FEATURES/16_UserStories.md` |
| 17 | Notifications (FCM push + Marketing) | Supporting | P2 | ✅ | 🟡 (config only) | `FEATURES/17_Notifications.md` |
| 18 | Taxonomy (Regions/Cities/Languages/Prices/Categories) | Shared | P2 | ✅ | ✅ | `FEATURES/18_Taxonomy.md` |
| 19 | Splash & Onboarding | Supporting | P3 | ✅ | ❌ | `FEATURES/19_SplashOnboarding.md` |
| 20 | Force Update | Supporting | P3 | ✅ | ❌ N/A | `FEATURES/20_ForceUpdate.md` |
| 21 | Profile Settings (password, contact-us, terms) | Supporting | P3 | ✅ | ✅ (+working contact/subscribe) | `FEATURES/21_ProfileSettings.md` |
| 22 | Localization & RTL (ar/en/ru/zh) | Shared | P3 | ✅ | ✅ (+SEO/hreflang/canonical) | `FEATURES/22_Localization.md` |
| 23 | Media / Gallery / Uploads | Shared | P3 | ✅ | 🟡 | `FEATURES/23_MediaGallery.md` |
| 24 | Networking / Interceptors / Env switching | Shared | P3 | ✅ | ✅ (HttpClient, 401→logout, SSR interceptors) | `FEATURES/24_Networking.md` |
| 25 | Real-time (Pusher / WebSockets) | Shared | P3 | ✅ | ❌ | `FEATURES/25_Realtime.md` |
| 26 | Dashboard / Admin CRUD & Content Moderation | Admin | P2 (backend-only) | ✅ | N/A | `FEATURES/26_DashboardAdmin.md` |
| 27 | Roles & Permissions (Spatie) | Admin | P2 (backend-only) | ✅ | 🟡 (implicit) | `FEATURES/27_RolesPermissions.md` |
| 28 | CarModel Module (fleet/logistics) | Admin / Internal | P4 (OUT OF SCOPE) | ✅ | N/A | `FEATURES/28_CarModule.md` |
| 29 | Report Module (analytics / PDF-Excel export) | Admin / Internal | P4 (OUT OF SCOPE) | ✅ | N/A | `FEATURES/29_ReportModule.md` |
| 30 | Visitor Tracking / Guard Check-in / Task Activity | Hidden / Internal | P4 | ✅ | 🟡 (geo headers each request) | `FEATURES/30_VisitorTracking.md` |
| 31 | Auto-Translation Observers (Google Translate) | Hidden / Internal | P3 | ✅ | N/A (backend) | `FEATURES/31_TranslationObservers.md` |
| 32 | AI ChatGPT Assistant (**WEB-ONLY**) | Web-only | P3 | ✅ | ✅ web / ❌ mobile | `FEATURES/32_AiAssistant.md` |
| 33 | Chat / Messaging (User ↔ Tour Guide) | Core | P2 | 🟡 PLANNED (design) | ⬜ لاحقًا | `FEATURES/ChatMessaging.md` |

---

## Categorization

- **Core Features:** 01–11 (buyer-facing discovery + planning + auth). Trips = flagship.
- **Supporting Features:** 12–21 (favorites, rates, submissions, notifications, splash, settings).
- **Shared Features:** 18, 22, 23, 24, 25, 31 (taxonomy, i18n, media, networking, real-time, translation).
- **Admin-related:** 26, 27, 28, 29 (Laravel dashboard/web session only — not exposed to mobile).
- **Hidden / Internal:** 30, 31, plus `monarx-analyzer.php` (⚠️ malicious RCE artifact — flagged for deletion, NOT a feature).
- **Legacy Features:** Trips v1 (superseded by v2), `hydrated_bloc` (declared, unused), `firebase_auth`/Firestore (declared, unused).
- **Deprecated / Dead:** `LocationInterceptor` (commented out), legacy `trips/*` endpoints coexisting with `v2/trips/*`.

---

## Notes on scope questions (pending user confirmation)
- **CarModel & Report modules** appear to be enterprise/logistics logic unrelated to tourism — flagged P4 pending confirmation whether in-scope.
- **Payments:** none exist in any repo.

---

## Web Frontend (Phase 3 — Angular 16 SSR)
- Repo `/Users/mac/hawdaj-frontend`. Architecture: `WEB_FRONTEND.md`. Web↔Mobile matrix + recommendations: `WEB_MOBILE_COMPARISON.md`.
- **Web implements ~22/31** shared features (full/partial). **Web-only feature:** #32 AI ChatGPT Assistant. **Web-only capabilities:** SSR/SEO (meta/hreflang/canonical), PDF/print trip export, granular home endpoints, unified `/global-search` + map, working newsletter/contact forms, social via `/social/callback`, 401→auto-logout.
- **Web lacks (vs mobile):** Splash/Onboarding (#19), Force Update (#20), functional FCM (#17), Realtime (#25).
- **Neither client (admin/backend only):** Dashboard (#26), Report (#29), CarModel internal (#28), internal Visitor/Guard ops (#30-B), Roles enforcement (#27), Translation observers (#31).
- Affected feature docs carry a **"Web Frontend" section**: 01, 02, 03, 04, 05, 08, 21, 22, 24.
