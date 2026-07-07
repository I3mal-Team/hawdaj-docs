# Web ↔ Mobile Comparison (hawdaj-frontend vs hawdaj Flutter)

> Compares the Angular 16 SSR web frontend (`/Users/mac/hawdaj-frontend`) against the Flutter app (`hawdaj/Untitled`) + shared Laravel API (`hawdaj-api`).
> Both consume the **same backend** (`{code,message,data}` envelope, Sanctum Bearer). Differences are mostly in *which* endpoints, *SEO/SSR*, *401 handling*, *social login mechanism*, and a **web-only AI assistant**.

---

## 1. Feature Comparison Matrix

Legend: ✅ full · 🟡 partial · ❌ none · N/A not applicable to that platform.

| # | Feature | Mobile | Web | Notes |
|---|---------|:-----:|:---:|-------|
| 1 | [[01_Authentication]] | ✅ | ✅ | Web: social via `/social/callback` (Firebase popup) + **401→auto-logout**; mobile native SDK + **401 ignored** |
| 2 | [[02_Home]] | ✅ (1 call) | ✅ (8+ calls) | Web splits home into granular endpoints (travelers, saudi-opening, saudi-places, services, sliders, top-destinations) |
| 3 | [[03_Trips]] v1 | 🟡 legacy | 🟡 legacy (commented) | both keep v1 around |
| 4 | [[03_Trips]] v2 enhanced | ✅ | ✅ | web adds **PDF/print export** + vehicleType; same `/v2/trips/*` API |
| 5 | [[04_Places]] | ✅ | ✅ | web adds **ChatGPT** + related-places + map data |
| 6 | [[05_Stores]] | ✅ | ✅ | web adds **ChatGPT**; same stats/related/favorite |
| 7 | [[06_Restaurants]] (Zad) | ✅ | ✅ | web geolocation "nearest" via headers; menus/offers/food-cats same |
| 8 | [[07_Tasneef]] unified | ✅ (PlacesCubit type-switch) | 🟡 (via `/global-search` viewAs) | different mechanism — see API diffs |
| 9 | [[08_Exploration]] search+map | ✅ (SearchController) | ✅ (`/global-search`,`global-map-data`,`places-data-for-map`) | different endpoints; web Google Maps |
| 10 | [[09_Events]] | ✅ | ✅ | same `/events` + rates |
| 11 | [[10_Swalef]] | ✅ | ✅ | same `/swalefs` + featured/categories/reviews |
| 12 | [[11_TourGuides]] | ✅ | ✅ | same guides + store-guide/update-guide-photo |
| 13 | [[12_Landmarks]] | ✅ | ✅ | web under `/settings/landmarks`; same `landmark/store` |
| 14 | [[13_FavoritesSaved]] | ✅ | ✅ | web `/favorites` + per-entity `*/favorite`; `/settings/my-favourites` |
| 15 | [[14_RatesReviews]] | ✅ | ✅ | same `/rates` polymorphic |
| 16 | [[15_MyProperties]] | ✅ | ✅ | web `/settings/my-properties` v2 + `/Profile/my-properties` v1; custom form validators |
| 17 | [[16_UserStories]] | ✅ (grid) | ✅ (posts + **comments**) | web `stories/addCommentToStory`; `/settings/posts` |
| 18 | [[17_Notifications]] FCM | 🟡 (disabled client) | 🟡 (config only) | both effectively off; web has firebaseConfig, no messaging service |
| 19 | [[18_Taxonomy]] | ✅ | ✅ | same regions/cities/categories/prices/languages |
| 20 | [[19_SplashOnboarding]] | ✅ | ❌ | web has spinner only, no onboarding |
| 21 | [[20_ForceUpdate]] | ✅ (Firebase) | ❌ | N/A for web (auto-updates) |
| 22 | [[21_ProfileSettings]] | ✅ | ✅ | web v2 `/settings/*`; **working contact + subscribe forms** (mobile contact is display-only) |
| 23 | [[22_Localization]] | ✅ (4 locales, RTL) | ✅ (4 locales, RTL, **+SEO/hreflang/canonical**) | web adds full SEO meta per locale |
| 24 | [[23_MediaGallery]] | ✅ | 🟡 | web upload via FormData + update-photo; gallery in detail pages |
| 25 | [[24_Networking]] | ✅ (Dio) | ✅ (HttpClient) | both: Bearer+locale+geo+api-key. Web **401→logout**; mobile **401 ignored**. Web adds SSR caching/state interceptors |
| 26 | [[25_Realtime]] | ❌ (scaffold, off) | ❌ | neither wired |
| 27 | [[26_DashboardAdmin]] | N/A | ❌ | admin is Laravel Blade, not this web app |
| 28 | [[27_RolesPermissions]] | N/A | 🟡 (implicit user/guide/owner via profile) | web shows role-conditional settings; no granular authz (API has none anyway) |
| 29 | [[28_CarModule]] | N/A | 🟡 (vehicleType in trip v2 only) | not the internal CarModel module; just a trip vehicle field |
| 30 | [[29_ReportModule]] | N/A | ❌ | internal admin only |
| 31 | [[30_VisitorTracking]] | 🟡 (views_num broken) | ✅ geo headers each request | web sends lat/lng every call (3-min refresh) |
| 32 | [[32_AiAssistant]] **(WEB-ONLY)** | ❌ | ✅ | ChatGPT assistant on Place/Store detail (`getChatGpt`/`addChatGpt`) |
| — | [[31_TranslationObservers]] | N/A backend | N/A | backend-only i18n mechanism (serves both) |

**Tallies:** Shared (both implement, full or partial): **~22**. Web-only feature: **1** (AI ChatGPT assistant) + web-only *capabilities* (SSR/SEO, PDF/print, granular home, newsletter/contact forms). Mobile-only: **4** genuine (Splash/Onboarding, ForceUpdate, plus mobile-shaped Media & the on-device push/realtime scaffolding). Backend/admin-only (neither client): Dashboard, Report, CarModel, internal VisitorTracking, TranslationObservers.

## 2. Missing Features
- **On Web, missing vs Mobile:** Splash & Onboarding ([[19_SplashOnboarding]]), Force Update ([[20_ForceUpdate]]), functional push/FCM ([[17_Notifications]] — config only), Realtime ([[25_Realtime]]), native offline/deeplink.
- **On Mobile, missing vs Web:** AI ChatGPT assistant ([[32_AiAssistant]]), SSR/SEO (meta/canonical/hreflang), PDF/print trip export, unified `/global-search` with `viewAs`, working newsletter/contact forms, social login via `/social/callback`.
- **Neither client:** Dashboard/Admin, Roles enforcement, Report, CarModel internal, internal Visitor/Guard ops (backend/[[26_DashboardAdmin]] only).

## 3. API Differences
| Area | Mobile | Web |
|------|--------|-----|
| Home | single `GET /home` (aggregated, 2h cache) | `home` + `get_top_destinations` + `get_travelers` + `get_saudi_opening` + `getSaudiPlaces` + `services` + `get_sliderPlaces` + `get_events` (granular) |
| Search | `SearchController` (multi-model in-memory) | `GET /global-search?viewAs=events\|places\|zads\|stores&region_id&city_id` |
| Map | in-app markers from search | `global-map-data`, `places/places-data-for-map` |
| Social login | native Firebase SDK | Firebase popup → `POST /social/callback` `{name,email,provider_id,provider_type}` |
| AI | none | `GET getChatGpt`, `POST addChatGpt` (Places/Stores) |
| Newsletter/Contact | none / display-only | `POST subscribe`, `POST contactus/send` |
| Stories | list + add + delete | + `stories/addCommentToStory` (comments) |
| Trips | `/v2/trips/*` | same `/v2/trips/*` + client PDF/print |
| 401 | ignored (stale auth) | interceptor → clear storage + logout + redirect |

Shared identical: places/stores/zads/events/swalefs/guides/landmark/favorites/rates/my-properties/regions/categories/prices/languages/auth core.

## 4. UI Differences
- **Web:** Bootstrap/PrimeNG/Material responsive desktop-first; modal auth (JoinUs) instead of route redirect; inline rating modals; taxonomy breadcrumbs; Google Maps; ChatGPT widget on detail pages; PDF/print button on trips; SSR = fast FCP + SEO.
- **Mobile:** native Flutter widgets, bottom-nav, infinite-scroll pagination, story grid, native pickers, RTL via Directionality; no SEO; splash + onboarding + force-update gates.
- **Both:** ar/en/ru/zh + RTL; same content cards conceptually.

## 5. Business Logic Differences
- **Home composition:** web assembles home client-side from many endpoints (more requests, more control, cache per-section); mobile trusts one cached aggregate. Web can show sections (travelers, saudi-opening) the mobile aggregate may not surface identically.
- **401/session:** web enforces session integrity (auto-logout on 401); mobile leaves user in stale-auth state — **web is stricter/safer here**.
- **Contact/Newsletter:** web actually POSTs (`contactus/send`, `subscribe`); mobile contact screen is display-only ([[21_ProfileSettings]]) → **inconsistent product behavior across platforms**.
- **AI assistant:** web-only value-add (place/store AI info) with no mobile parity → feature gap.
- **Trip export:** web lets users print/PDF a plan; mobile has its own share/email path.
- **Search model:** web `viewAs` global-search vs mobile Tasneef type-switch + SearchController — same data, different aggregation contracts (two code paths to maintain).
- **Roles:** web conditionally shows guide/owner settings from profile flags; neither enforces authz at API ([[27_RolesPermissions]]).

## 6. Recommendations
1. **Parity for AI assistant:** either add ChatGPT to mobile ([[32_AiAssistant]]) or document it as intentionally web-only; today it's an undocumented gap.
2. **Unify 401 handling:** adopt web's approach on mobile (refresh or logout on 401) — mobile currently ignores 401 ([[24_Networking]]).
3. **Reconcile contact/newsletter:** wire the mobile contact form to `contactus/send` (already exists) so behavior matches web ([[21_ProfileSettings]]); add rate-limit server-side (currently public/no-throttle).
4. **Converge search:** consider one search contract (either `global-search` or SearchController) to avoid two divergent code paths ([[08_Exploration]]).
5. **Home endpoint strategy:** decide aggregate (mobile) vs granular (web); document the intended contract to prevent drift; consider caching parity.
6. **FCM:** both platforms have push effectively off — if push is a product goal, wire web (firebase messaging) + mobile (firebase_messaging) together ([[17_Notifications]]).
7. **Social login consistency:** ensure `/social/callback` (web) and native mobile social produce the same user/token shape ([[01_Authentication]]).
8. **SEO stays web-only** (correct) — keep meta/hreflang/canonical maintained per locale as content grows.
9. **Security carry-over:** the shared-backend issues (rates unauth/PII, hardcoded api-key, no API authz) affect BOTH clients — fix at API ([[14_RatesReviews]], [[24_Networking]], [[27_RolesPermissions]]).

---

## Coverage Note
With the web frontend integrated, the knowledge base now spans **all three tiers** of the product: Flutter mobile, Laravel API/admin, and Angular web. See [[WEB_FRONTEND]] for web architecture and the per-feature **Web Frontend** sections in affected docs.
