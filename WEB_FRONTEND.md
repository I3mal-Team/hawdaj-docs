# Web Frontend — Architecture & Knowledge (hawdaj-frontend)

> **Repo:** `/Users/mac/hawdaj-frontend` · Same product as the Flutter app (`hawdaj/Untitled`) + Laravel API (`hawdaj-api`).
> **Role:** Public, SEO-first, user-facing web app (discovery + trip planning + profile). NOT the admin dashboard (that's Laravel Blade, [[26_DashboardAdmin]]).
> **One-line:** Angular 16 SSR (NgUniversal + Express) SPA, service+RxJS state (no NgRx), path-based routing with `/:lang/` prefix, HttpClient + interceptors (Bearer token + locale + geo + static api-key), ngx-translate (ar/en/ru/zh + RTL), Firebase social login (Google/Twitter → `/social/callback`), full SEO/meta/hreflang. Consumes the same Laravel API as mobile but via **more granular home endpoints**, a **global-search** stack, and a **web-only AI ChatGPT assistant** ([[32_AiAssistant]]).

---

## 1. Architecture
- **Framework:** Angular **16.2.0** (standalone components + lazy `loadComponent`), TypeScript 5.1.3.
- **Rendering:** **SSR** via `@nguniversal/express-engine` 16.2 + `@angular/platform-server`; prerender for SEO; `initialNavigation: 'enabledBlocking'`; `ServerStateInterceptor` transfers server→client state; `isPlatformBrowser()` guards throughout.
- **UI libs:** Angular Material 16.2 + PrimeNG 16.3 + Bootstrap 5.3 + `@angular/cdk`.
- **Maps:** `@angular/google-maps` 16.2. **Charts/geo:** amCharts 5.6.
- **Firebase:** `@angular/fire` 16 + firebase 9.16 (social auth; FCM SDK present but **messaging not wired**).
- **PDF:** trip print via raw-HTML Blob + `window.print()` (`html2canvas`/`jspdf` present but the active trip export uses browser print).
- **Pattern:** feature `components/` + domain-driven `domains/` (trip v2, profile-settings v2) + global `services/`.

## 2. Folder Structure
```
src/app/
  app-routing.module.ts        # main router (~1–324), /:lang/ prefix
  app.module.ts / app.component.ts
  components/                   # feature components (v1 + refactored v2)
    authentication/ home-page/ places/ stores/ resturants/ stories/
    events/ tour-guides/ applications/ personal-profile/ my-trips/
    create-trip/ errors/ join-us/
  domains/                      # domain-driven (newer)
    trip/          # v2 enhanced planner (services/routes/components + trip-pdf)
    profile-settings/  # v2 profile: personal-info, posts, tour-guide-info,
                       # landmarks, favourites, my-properties + form validators
  services/                     # global services (auth, home, places, stores,
    restaurants, stories, events, trips[v1], profile) + interceptors/
  modules/shared/configs/endPoints.ts   # endpoint root paths (~1–107)
  modules/shared/services/metadata.service.ts  # SEO
  core/ (LocationLatLongFacade geolocation)
  services/meta-tags-pages.ts   # per-locale SEO tags
src/assets/i18n/{ar,en,ru,zh}.json
src/environments/{environment,.dev,.prod,.staging}.ts
```

## 3. State Management
- **No NgRx/Redux.** Pure **RxJS service state** (BehaviorSubjects in singleton services: PublicService, PlacesService, etc.).
- **Persistence:** `localStorage` (token, userLoginData, userData, profileData, language, trip step/save state, nav history).
- Taxonomy (regions/categories) loaded once on app init (`app.component.ts:277–305`).

## 4. Routing
- Path-based (no hash), **language prefix** `/:lang/route` (e.g. `/ar/places`, `/en/home`). SPA + SSR/prerender.
- Key routes: `/home`, `/places`(+`/details/:id`), `/stores`, `/restaurants`, `/events`(+`/event-details/:id`), `/stories`, `/tour-guides`(+`/:id`), `/applications`, `/search-result`, `/Profile` + `/settings/*` (v2: personal-info/posts/tour-guide-info/landmarks/my-favourites/my-properties), `/v1-trips`, `/v2-trips`, legacy `/my-trips` (commented). Errors 404/500.
- **Guards:** `AuthGuard` (CanActivate) on `/Profile`, `/settings`, `/my-trips` — `isLoggedIn()`? allow : redirect `/home` (permissive). Also `AuthService.checkIsLogin()` opens a **JoinUs modal** instead of redirect for gated actions.

## 5. API Layer
- **Angular HttpClient** + endpoint roots in `endPoints.ts`. Base URL per environment.
- **Envelope:** `{code, message, data}` (same as mobile).
- **Interceptors** (`services/interceptors/`):
  - `HttpInterceptorService` (~1–125): `Authorization: Bearer {token}`, `locale`, static `accept-tokenapi` key, **geolocation headers** (lat/lng/accuracy/source, refreshed every 3 min via `LocationLatLongFacade`).
  - `ErrorInterceptor` (27–63): **401 → full logout** (clears all localStorage 91–97, POST `/logout`, toast, redirect `/home`). Other codes commented. Errors surfaced via PrimeNG `MessageService`/`AlertsService.openToast`. **← divergence from mobile, which ignores 401.**
  - `CachingInterceptor` (HTTP response cache), `ServerStateInterceptor` (SSR state transfer).

## 6. Authentication
- **Email/password:** `/login`, `/register`, `/logout`, `/forgot-password`, `/verify-email-code`, `/reset-password`, `/update-password`, `/update-profile`, `/get-profile-page` (`auth.service.ts`).
- **Social:** Firebase popup (Google/Twitter) → FormData `{name,email,provider_id,provider_type}` → **POST `/social/callback`** → store `data` in localStorage → `getProfileData()` (`auth-firebase.service.ts:57–152`). Mobile uses native SDKs — web uses browser popup + `/social/callback`.
- **Token:** JWT Bearer in `localStorage`; attached by interceptor. Auto-logout on 401.

## 7. Shared Components
- Header (lang switcher, notifications icon, auth status), Footer (links + subscribe), JoinUs auth modal (PrimeNG DynamicDialog), toast alerts (PrimeNG MessageService), breadcrumbs (taxonomy-aware), reusable cards (places/stores/restaurants/events), Google Maps, global search widgets (`QuickGlobalSearch` v1/v2/mobile), ChatGPT widget ([[32_AiAssistant]]).

## 8. Localization / SEO
- **ngx-translate** 15, locales ar/en/ru/zh, files `assets/i18n/*.json` (ar 54KB / en 41KB / ru 63KB / zh 39KB), RTL auto for Arabic (`app.component.ts:256–261`), lang in URL + localStorage, versioned loader for cache-busting.
- **SEO (web-only vs mobile):** `MetadataService` (title, meta name/property, og:image/twitter share, canonical `updateCanonicalLink`, **hreflang** `updateLinkRelAlternate`), per-locale tags in `meta-tags-pages.ts` for Home/Places/Events/Applications/TourGuides/Stores/Stories/Restaurants. See [[22_Localization]] Web section.

## 9. Environment Configuration
- `environments/{environment,.dev,.prod,.staging}.ts`: `production`, `apiUrl` (prod `https://dashboard.hawdaj.net/api`, test `https://test.dashboard.hawdaj.net/api`), `publicUrl`, `imageBaseUrl`, `firebaseConfig`.
- Same backend as mobile ([[24_Networking]]).

## 10. Build Process
- Scripts: `ng serve` (dev SPA), `build:ssr` / `build:ssr-dev` / `build:ssr-staging` (SSR prod builds), `dev:ssr` (SSR dev), `prerender` (static), `compress` (gzip), `vercel-build`.
- **Output:** SPA `dist/hawdaj/browser/`, SSR server `dist/hawdaj/server/main.js`, prerendered HTML. Vercel-ready.

## 11. Web-Only Endpoints (not called by mobile)
`getChatGpt`, `addChatGpt` ([[32_AiAssistant]]), `social/callback`, `global-search`, `global-map-data`, `places/places-data-for-map`, `get_travelers`, `get_saudi_opening`, `getSaudiPlaces`, `get_top_destinations`, `get_sliderPlaces`, `services`, `subscribe`, `contactus/send` (web contact form — note mobile has this endpoint but its UI is display-only per [[21_ProfileSettings]]).

## 12. Feature Coverage Snapshot (vs 31 mobile/backend features)
Full detail in [[WEB_MOBILE_COMPARISON]]. Summary: web implements ~23/31 (fully or partial). Web-only: **AI ChatGPT assistant**, SSR/SEO, PDF/print export, unified global-search+map, newsletter/contact forms. Web lacks: Splash/Onboarding, ForceUpdate, FCM push (config only), Realtime, admin Dashboard/Roles/CarModel/Report/VisitorTracking-internal (backend/admin, never client).

## 13. Cross-links
Related feature docs updated with a **"Web Frontend" section**: [[01_Authentication]], [[02_Home]], [[03_Trips]], [[04_Places]], [[05_Stores]], [[08_Exploration]], [[21_ProfileSettings]], [[22_Localization]], [[24_Networking]]. New web-only doc: [[32_AiAssistant]]. Comparison matrix: [[WEB_MOBILE_COMPARISON]].
