# Feature 19 — Splash & Onboarding (البداية والتعريف)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Supporting (startup)
> **Flutter:** `lib/features/splash/` + `main.dart` + `ForceUpdateIntermediateScreen` · **Laravel:** none dedicated (no bootstrap/config API)
> **One-line:** Startup pipeline is **`ForceUpdateIntermediateScreen → SplashView (2s) → OnboardingView | HomeView`**, decided purely by a `seenOnboarding` SharedPreferences bool. Onboarding is **3 hardcoded local-asset slides** (no CMS/backend). There is **no dedicated boot/config API** — the app instead fires `HomeCubit.fetchHome()` + `SlidersCubit.fetchSliders()` immediately at launch (blocking-ish network), the only startup "config" being the `/api/home` payload. Force-update gate reads Firebase Firestore `app_versions` with no offline fallback. Language defaults to Arabic with no first-run picker.

---

## 1. Feature Name & Identity
- **Name:** Splash & Onboarding — app boot, intro walkthrough, first-launch routing.
- **Scope:** startup order, force-update gate, splash delay, onboarding slides, locale bootstrap, splash→onboarding→home decision.
- **Backend:** stateless w.r.t. this feature — no onboarding content, no first-launch, no splash/config endpoint.

## 2. Business Goal & Rules
- **Goal:** Gate the app on version, show a first-run walkthrough once, then land the user in Home.
- **Rules:**
  - `seenOnboarding` false → OnboardingView; true → HomeView.
  - `seenOnboarding` set true on completing last onboarding slide.
  - Force-update: if latest > current AND mandatory → blocking dialog; if optional → dismissable (`ignored_update_version` remembered).
  - Language: saved `selected_language` or default `ar`.
  - No auth gate before onboarding (routes to Home which later handles logged-out state).

## 3. User Flow
1. App launches → `ForceUpdateIntermediateScreen` → `ForceUpdateCubit.checkForUpdate()` (Firebase Firestore).
2. → `SplashView` (2s timer, logo).
3. Reads `seenOnboarding`: false → `OnboardingView` (3 slides, next/استكشف); true → `HomeView`.
4. Onboarding last slide → set `seenOnboarding=true` → `kHomeView`.
5. Meanwhile Home/Sliders cubits already fetching `/api/home`.

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`splash_view.dart`](lib/features/splash/presentation/views/splash_view.dart) | 30–36 | 2s delay, reads `seenOnboarding`, routes |
| [`onboarding_view.dart`](lib/features/splash/presentation/views/onboarding_view.dart) | ~46,171 | PageView 3 slides, next/get-started, sets flag |
| `force_update_intermediate_screen.dart` | 25–28 | Initial route, triggers ForceUpdateCubit, spinner |

## 5. Widgets
- Onboarding PageView + indicator + next button (label "next" i18n or hardcoded "استكشف").
- Splash logo. Force-update loading spinner + update dialog (blocking/dismissable).

## 6. Cubits / Blocs
| Cubit | File | Role |
|-------|------|------|
| ForceUpdateCubit | `force_update_cubit.dart:17–76` | Firebase version check, mandatory/optional dialog, ignored-version tracking |
| LocaleCubit | `locale_cubit.dart:15–32` | load saved language / default ar |
| HomeCubit / SlidersCubit | (home feature) | fetchHome/fetchSliders fired at boot (main.dart:125–128) |

- No dedicated SplashCubit/OnboardingCubit (splash uses a timer, onboarding uses local state + prefs).

## 7. State Flow
- ForceUpdate: check → emit continue | show-dialog(mandatory/optional) | error.
- Splash: timer completes → read prefs → navigate.
- Onboarding: page index local; last page → prefs write → navigate.
- Locale: loadSavedLanguage async (not awaited before router init → possible flash).

## 8. Models
| Model | File:Lines | Notes |
|-------|-----------|-------|
| OnboardingModel | `splash/data/models/onboarding_model.dart:13–31` | **hardcoded** 3 slides (ar title/desc + `AppAssets.onboarding(1/2/3)` local images) |

## 9. Repositories
- None for splash/onboarding (all local). Force-update reads Firebase Firestore directly (no repo abstraction). Home data via home repo.

## 10. Services / Helpers
- `StorageService` (SharedPreferences) — `seenOnboarding`, `selected_language`, `ignored_update_version`.
- `AppEnvironmentManager().init()` (env/base-url).
- `PackageInfo.fromPlatform()` (current version).
- Firebase Firestore `app_versions/latest_version`.

## 11. API Endpoints
| Method | Endpoint | Role |
|--------|----------|------|
| GET | `home` | de-facto boot config (places/events/zads/guides/services/mapData); 2h cache; location-aware |
- **No** `app-config`/`settings`/`onboarding`/`splash` endpoint. Force-update source is Firebase, not Laravel.

## 12. Request / Response Models
- `/api/home` returns mixed home payload (see [[02_Home]]) incl. `services` (Settings group=main_services). No onboarding/splash config in any response.
- Firebase doc: `{latest_version, isMandatory, ...}`.

## 13. Laravel Routes
- `GET /api/home` → HomeController@index (routes/api.php:63). No other boot-related route.

## 14. Laravel Controllers
- HomeController (`:28–40`, fetchHomeData `:209–221`): aggregates home + services; 2h cache; distance sorting if location. Serves as the only startup data call. No onboarding logic.

## 15. Laravel Services
- None for this feature. Settings table provides `main_services` mixed into home.

## 16. Laravel Models
- None specific. Settings model supplies services block. No onboarding/app-version model (version lives in Firebase).

## 17. Database Tables & Relationships
- No dedicated tables. Firebase Firestore `app_versions` holds version gate (external to MySQL). Settings table for services.

## 18. Validation
- N/A (read-only startup). Version compare in client.

## 19. Auth & Permissions
- `/api/home` public. Force-update Firebase read public. Onboarding shown regardless of auth. No auth gate at boot (Home handles logged-out later).

## 20. Error Handling
- **Weak:** main.dart boot steps (Firebase init, env load, service locator) have **no try/catch** → single failure crashes pre-UI.
- ForceUpdateCubit emits error → continues (but may hang on slow Firebase).
- HomeCubit fetch failure at boot handled by home state, but no offline default.

## 21. Edge Cases
- `seenOnboarding` persists across reinstall (prefs not cleared) → user never re-sees onboarding; no "replay" in settings.
- Offline at boot: force-update Firebase unreachable → spinner/blocked; home fetch fails silently.
- Locale loads after router init → brief wrong-language flash.
- Force-update blocking dialog with no network → stuck.
- No first-run language selection.

## 22. Dependencies
- **Flutter:** easy_localization, firebase_core, cloud_firestore (force update), package_info_plus, geolocator, shared_preferences, flutter_bloc, get_it, flutter_dotenv.
- **Laravel:** HomeController, Settings.
- **Cross-feature:** [[20_ForceUpdate]] (gate), [[22_Localization]] (locale bootstrap), [[02_Home]] (boot data), [[24_Networking]] (env/base-url init), [[30_VisitorTracking]] (IP on first home call).

## 23. Files Involved
**Flutter:** main.dart (47–83 boot, 105–168 cubits, 125–128 fetch, 196–210 locale refresh), app_router.dart (30–37 initial route), splash_view, onboarding_view, onboarding_model, force_update_intermediate_screen, force_update_cubit, locale_cubit, routes.dart:831 (nextPage kSplashView), StorageService, AppEnvironmentManager, AppAssets.
**Laravel:** HomeController, routes/api.php:63, Settings model.

## 24. Full Execution Flow (boot order)
1. `WidgetsFlutterBinding.ensureInitialized()` → `EasyLocalization.ensureInitialized()` → `dotenv.load()` → Firebase init → `AppEnvironmentManager.init()` → `setupServiceLocator()` → social auth init → `StorageService.init()` (main.dart:47–62).
2. Cubits instantiated (LocaleCubit first, 109); `LocaleCubit.loadSavedLanguage()` (async, unawaited).
3. Router initialLocation = `/kForceUpdateIntermediateScreen` → `ForceUpdateCubit.checkForUpdate()` (Firebase) → dialog or continue → nextPage `kSplashView`.
4. `SplashView` 2s timer → read `seenOnboarding` → OnboardingView | HomeView.
5. `HomeCubit.fetchHome()` + `SlidersCubit.fetchSliders()` fire → GET `/api/home` (background).
6. Final screen renders.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Boot fires `fetchHome`/`fetchSliders` immediately (network at splash) | 🟡 Med | main.dart:125–128 |
| Force-update Firebase read blocks first screen, no timeout/offline | 🟡 Med | force_update_cubit |
| No offline cache for boot data | 🟡 Med | HomeController/home cubit |
| 2s fixed splash delay regardless of readiness | 🟢 Low | splash_view |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| Boot steps no try/catch → crash reveals stack / poor UX | 🟢 Low | main.dart |
| No sensitive data at boot | 🟢 OK | — |

## 27. Technical Debt
- Onboarding hardcoded (no CMS) — content change = app release.
- `seenOnboarding` never resettable; survives reinstall; no replay entry point.
- No dedicated boot/config API — startup config smuggled inside `/api/home`.
- Force-update source split (Firebase) from rest of backend (Laravel).
- Locale load races router init.
- No offline/boot-failure handling.
- Fixed 2s splash.

## 28. Improvement Opportunities
- Await locale load before first render; show splash until boot truly ready (readiness-based, not fixed 2s).
- Add a lightweight `/api/bootstrap` (config, min-version, feature flags, onboarding slides) → single boot call, cacheable, offline fallback.
- Serve onboarding slides from backend (CMS) with local fallback.
- Add "replay onboarding" + first-run language picker.
- Wrap boot in try/catch with graceful error screen + retry; add offline cache.
- Move force-update to Laravel (or keep Firebase but add timeout + offline continue).

---

## 32. Related Features
- **[[20_ForceUpdate]]** — the first boot gate.
- **[[22_Localization]]** — locale bootstrap + default ar.
- **[[02_Home]]** — boot data (`/api/home`) + landing screen.
- **[[24_Networking]]** — env/base-url init at boot.
- **[[01_Authentication]]** — post-onboarding auth handling in Home.

## 33. How to Modify This Feature
- **Change onboarding slides:** edit `onboarding_model.dart:13–31` + `AppAssets.onboarding(n)` (requires rebuild). To make dynamic: fetch from a new endpoint.
- **Reset onboarding for user:** add a settings toggle that clears `seenOnboarding` pref.
- **Adjust boot order:** main.dart:47–83; move blocking calls off the splash path.
- **Change force-update behavior:** force_update_cubit + Firebase `app_versions` doc.

## 34. Regression Checklist
- [ ] Fresh install → onboarding shows once.
- [ ] After completing → Home; relaunch skips onboarding.
- [ ] Mandatory update → blocking dialog; optional → dismissable + remembered.
- [ ] Language persists across restart.
- [ ] Offline boot → graceful (no infinite spinner).
- [ ] No wrong-language flash on first frame.

## 35. Common Bugs
- Onboarding never re-appears after reinstall (prefs persisted).
- Splash hangs when Firebase/home unreachable.
- Wrong-language flash before locale loads.
- App crash pre-UI if a boot init throws.
- Stuck on force-update screen offline.

## 36. Debug Guide
- **Onboarding skipped/stuck:** inspect `seenOnboarding` pref value.
- **Splash hang:** check Firebase reachability + home fetch; add logging around checkForUpdate.
- **Language flash:** confirm loadSavedLanguage timing vs first render.
- **Boot crash:** wrap/inspect main.dart init steps.
- **Update dialog wrong:** check Firestore `app_versions/latest_version` + isMandatory + `ignored_update_version`.

## 37. Search Keywords
`splash`, `SplashView`, `onboarding`, `OnboardingView`, `onboarding_model`, `seenOnboarding`, `ForceUpdateCubit`, `force_update_intermediate_screen`, `app_versions`, `LocaleCubit loadSavedLanguage`, `selected_language`, `kSplashView`, `kForceUpdateIntermediateScreen`, `fetchHome boot`, البداية, التعريف, شاشة البداية.

## 38. Future Improvements
- Backend `/bootstrap` (config + min version + feature flags + onboarding CMS) with offline cache.
- Readiness-based splash, awaited locale, graceful boot error/retry.
- First-run language picker + replayable onboarding.
- Unified version-gate on Laravel with timeout/offline continue.
