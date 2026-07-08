# Feature 24 — Networking / Interceptors / Env Switching (طبقة الشبكة)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Shared (HTTP infra)
> **Flutter:** `lib/core/databases/api/` (DioConsumer + interceptors) + `AppEnvironmentManager` · **Laravel:** cors.php + Sanctum
> **One-line:** Dio-based HTTP layer with a dynamic base URL (prod `dashboard.hawdaj.net/api/` vs test `test.dashboard.hawdaj.net/api/`) switched at runtime via a **hidden 5-tap dev backdoor shipped in production** (password `dev123`). Interceptor chain = Language → Location(**dead/commented**) → Token(Bearer) → Chucker(opt) → PrettyDioLogger(**always on, logs tokens/PII**). Token in secure storage, but **401 is detected and silently ignored** (no refresh, no logout). Errors → `Either<Failure,T>` via `response.data['message']`. **No explicit timeouts, no retry.** Backend CORS = `allowed_origins:['*']` (credentials false). A **hardcoded `accept-tokenapi` key** sits in source.

---

## 1. Feature Name & Identity
- **Name:** Networking / Interceptors / Env switching — the shared HTTP client all features use.
- **Core:** `DioConsumer` (implements `ApiConsumer`), interceptor chain, `AppEnvironmentManager` (env/base-URL), failure mapping.

## 2. Business Goal & Rules
- **Goal:** One configured Dio client: auth + locale headers, base-URL per environment, uniform error→Failure mapping.
- **Rules:**
  - Base URL from `AppEnvironmentManager` (persisted `environment` pref, default production).
  - Every request gets locale headers + Bearer token (if present).
  - Errors mapped to `Failure`/`ServerFailure` via envelope `message`.
  - Env switch gated by hidden gesture + `dev123` password; persists.

## 3. User Flow (dev/infra)
1. App boot → `AppEnvironmentManager().init()` reads `environment` pref → sets base URL.
2. Every feature request flows through DioConsumer + interceptors.
3. (Dev) 5 taps top-right → `EnvSwitchScreen` → password `dev123` → switch prod/test + toggle Chucker → `refreshBaseUrl()`.

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| `env_switch_screen.dart` | 21 (pw), 38–48 (switch) | Dev env switch UI (`dev123`) |
| `environment_switcher_floating_button.dart` | 40 (always on), 57–59 | Invisible 5-tap trigger, wrapped in auth pages |
| `NoInternetView` | — | Offline screen (from connectivity listener) |

## 5. Widgets
- Invisible env-switch floating button (transparent, top-right). NoInternetView. Chucker inspection UI (opt).

## 6. Cubits / Blocs
| Cubit | File | Role |
|-------|------|------|
| ConnectivityCubit | `core/managers/connectivity_cubit/` | emits connected/disconnected; main.dart:172–184 → NoInternetView. **No retry/queue** |

## 7. State Flow
- Connectivity: stream → status → navigate offline screen. Requests: synchronous per-call, no queue; failures return Left(Failure).

## 8. Models
- `Failure` / `ServerFailure` ([`failure.dart`](lib/core/errors/failure.dart), [`exceptions.dart`](lib/core/errors/exceptions.dart:7–51)). Envelope `{code/status, message, data}` (parsed loosely via `message`).

## 9. Repositories
- All feature repos depend on `ApiConsumer` (DioConsumer). Methods get/post/put/delete return `Either<Failure,T>` via extension.

## 10. Services / Helpers
| Component | File:Lines | Role |
|-----------|-----------|------|
| DioConsumer | `dio_consumer.dart:15–39` | BaseOptions (baseUrl dynamic, Content-Type/Accept json, **hardcoded `accept-tokenapi` 22**, spoofed UA, followRedirects false, **no timeouts**), interceptor registration 26–39, dual-Dio for FormData (50–87) |
| AppEnvironmentManager | `app_environment_manager.dart:19–53` | init reads `environment` pref (default production), setProduction/setTest, refreshBaseUrl |
| api_consumer_extension | `api_consumer_extension.dart:12–59` | DioException → Failure via `response.data['message']` |
| StorageService | `storage_service.dart:29–36` | secure token (FlutterSecureStorage), prefs for env |
| AuthManager | `auth_manager.dart:22–32,48–50` | getToken/saveSecure/logout |

## 11. API Endpoints
- **Base URLs:** prod `https://dashboard.hawdaj.net/api/`, test `https://test.dashboard.hawdaj.net/api/`. `end_points.dart:8` reads from AppEnvironmentManager.
- All feature endpoints resolve relative to this base.

## 12. Request / Response Models
- Request headers: Content-Type/Accept json, `accept-tokenapi` (hardcoded), `accept-language`+`locale`, `Authorization: Bearer` (if token), spoofed User-Agent.
- Response envelope: `{code/status, message, data}`; client reads `data` on success, `message` on error.

## 13. Laravel Routes / Config
- `config/cors.php`: paths `['api/*']`, **allowed_origins `['*']`**, methods `*`, headers `*`, `supports_credentials:false`.
- Sanctum bearer auth; `throttle:api` default (per-endpoint throttle largely absent — see [[21_ProfileSettings]] contactus, [[14_RatesReviews]]).

## 14. Laravel Controllers
- N/A (infra). Auth guard `auth('api')` used inline across controllers.

## 15. Interceptor Chain (order)
`dio_consumer.dart:26–39`:
1. **LanguageInterceptor** (27) — `language_interceptor.dart`: async reads `selected_language` each request → `accept-language`+`locale` (default ar).
2. **LocationInterceptor** (28) — `location_interceptor.dart`: **commented out** (would add X-Device-Latitude/Longitude). يبقى ميّتًا عمدًا — الباك `getLocationCoords()` لا يقرأ headers أصلًا. **(2026-07-08)** الموقع يصل الآن لـ`/home` عبر **query params `lat`/`lng`** من `HomeRepoImp` (لا عبر هذا الـinterceptor). راجع [[02_Home]] §13.
3. **TokenInterceptor** (29) — `token_interceptor.dart:8–40`: strips `accept-tokenapi` (14), async `AuthManager.getToken()` (17) → `Authorization: Bearer` (19), removes Content-Type for FormData (23–26). **onError 401 (34–40): does NOTHING** (refresh/logout commented).
4. **ChuckerDioInterceptor** (31) — conditional (`isChuckerEnabled` pref).
5. **PrettyDioLogger** (32–38) — **always enabled** (`enabled:true`), logs bodies+headers (**tokens/PII**).

## 16. Laravel Models
- N/A.

## 17. Database Tables & Relationships
- N/A (infra). Token in secure storage (client); env in SharedPreferences.

## 18. Validation
- N/A. Envelope assumes `message` present (crash risk if absent).

## 19. Auth & Permissions
- Bearer token from secure storage injected per request. **401 not acted on** → stale-auth UX. No token refresh flow. Logout clears token (but not fcm_token, per [[21_ProfileSettings]]).

## 20. Error Handling
- `ServerFailure.fromDioException` maps DioExceptionType + status (400/401/403/405/422/404/500) → messages. 401/403 just returns message. Non-Dio → `ServerFailure(e.toString())`.

## 21. Edge Cases
- 401 with invalid token → user stuck in logged-in UI (no logout).
- Error response missing `message` → parsing crash.
- No timeout tuning → 30s default hangs.
- Transient failure → immediate error (no retry).
- Offline → NoInternetView, but in-flight requests not queued/retried.
- Env switch persists → user could stay on test API unknowingly.
- Async locale + token lookups per request → minor overhead.

## 22. Dependencies
- **Flutter:** dio, pretty_dio_logger, chucker_flutter, flutter_secure_storage, shared_preferences, connectivity(+_plus), dartz, get_it.
- **Laravel:** fruitcake/laravel-cors (or built-in), Sanctum.
- **Cross-feature:** used by ALL features. [[22_Localization]] (LanguageInterceptor), [[01_Authentication]] (token/401), [[23_MediaGallery]] (FormData), [[17_Notifications]]/[[20_ForceUpdate]] (Firebase separate).

## 23. Files Involved
**Flutter:** dio_consumer, api_consumer(abstract)+extension, token_interceptor, language_interceptor, location_interceptor(dead), app_environment_manager, env_switch_screen, environment_switcher_floating_button, end_points, failure/exceptions, storage_service, auth_manager, connectivity_cubit, service_locator:56–57, main.dart:56/172–184.
**Laravel:** config/cors.php, Sanctum config, route middleware.

## 24. Full Execution Flow (request lifecycle)
Feature repo → `DioConsumer.get/post(path)` → BaseOptions(baseUrl from env, headers) → interceptors onRequest: Language(headers) → [Location dead] → Token(Bearer, strip accept-tokenapi) → [Chucker] → PrettyDioLogger(logs) → HTTP → onResponse/onError → api_consumer_extension maps DioException→Failure(`message`) → `Either<Failure,T>` → cubit.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| No explicit connect/receive timeout (30s default) | 🟡 Med | dio_consumer BaseOptions |
| No retry/backoff for transient failures | 🟡 Med | extension/interceptors |
| Async SharedPrefs read per request (language + token) | 🟢 Low | Language/Token interceptors |
| PrettyDioLogger always on (serialization overhead + noise) | 🟢 Low | dio_consumer:35 |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **Hardcoded `accept-tokenapi` key in source** | 🔴 CRITICAL | dio_consumer.dart:22 |
| **Env-switch dev backdoor shipped in prod** (5-tap + `dev123`, switch to test + Chucker) | 🟠 HIGH | environment_switcher_floating_button:40, env_switch_screen:21 |
| **PrettyDioLogger always on → tokens/PII in logs** | 🟠 HIGH | dio_consumer:35 |
| **401 silently ignored** (no logout/refresh → stale auth) | 🟠 HIGH | token_interceptor:34–40 |
| **CORS `allowed_origins:['*']`** (credentials false mitigates) | 🟡 Med | cors.php |
| No cert pinning | 🟢 Low | dio |
| Token in secure storage ✅ | OK | storage_service |

## 27. Technical Debt
- Dev backdoor + hardcoded password + no build flavors (same code all builds).
- Hardcoded API key + committed base URLs.
- 401 handling stubbed (no refresh, no forced logout).
- Always-on verbose logging.
- No timeouts/retry/connectivity-aware queuing.
- LocationInterceptor dead code.
- Envelope parsing assumes `message`.

## 28. Improvement Opportunities
- Remove/flavor-gate env switch (debug builds only); move key to secure config/remote; rotate exposed key.
- Implement 401 → token refresh or forced logout+redirect.
- Disable PrettyDioLogger in release (kReleaseMode) or redact auth/PII.
- Set explicit connect/receive/send timeouts + retry interceptor (backoff) + connectivity-aware retry.
- Tighten CORS to known origins.
- Robust envelope parsing (null-safe) + typed error codes.
- Add cert pinning for high-value endpoints.

---

## 32. Related Features
- **ALL features** consume this layer.
- **[[01_Authentication]]** — token lifecycle + 401.
- **[[22_Localization]]** — LanguageInterceptor.
- **[[23_MediaGallery]]** — FormData dual-Dio.
- **[[17_Notifications]] [[20_ForceUpdate]]** — Firebase (separate transport).
- **[[30_VisitorTracking]]** — Location headers (dead interceptor) / IP.

## 33. How to Modify This Feature
- **Change base URL:** AppEnvironmentManager setProduction/setTest + end_points.dart.
- **Add a header globally:** add interceptor or BaseOptions header in dio_consumer.
- **Handle 401 properly:** implement onError in token_interceptor (refresh or `AuthManager.logout()` + navigate login).
- **Add timeouts:** set connectTimeout/receiveTimeout in BaseOptions.
- **Disable backdoor in prod:** gate `_shouldShowButton` on `kDebugMode`.

## 34. Regression Checklist
- [ ] Requests hit correct base URL per env.
- [ ] Bearer token attached when logged in.
- [ ] Locale headers present.
- [ ] Error responses map to Failure (no crash on missing message).
- [ ] Offline → NoInternetView.
- [ ] (After fix) 401 → logout/refresh.
- [ ] (After fix) logger off in release.

## 35. Common Bugs
- Stuck logged-in with invalid token (401 ignored).
- App using test API after accidental switch (persisted).
- Crash on error response lacking `message`.
- Long hangs on flaky network (no timeout).
- Tokens visible in device logs.

## 36. Debug Guide
- **Wrong API data:** check `environment` pref (prod/test) + refreshBaseUrl.
- **Auth failing:** verify token in secure storage + Bearer injected; note 401 not handled.
- **Header missing:** confirm interceptor order + not stripped for FormData.
- **Timeout:** none set — add to diagnose hangs.
- **Env stuck on test:** clear `environment` pref.

## 37. Search Keywords
`Dio`, `DioConsumer`, `ApiConsumer`, `BaseOptions`, `TokenInterceptor`, `LanguageInterceptor`, `LocationInterceptor`, `AppEnvironmentManager`, `env_switch_screen`, `environment_switcher_floating_button`, `dev123`, `accept-tokenapi`, `PrettyDioLogger`, `ServerFailure`, `Either Failure`, `401 handling`, `refreshBaseUrl`, `dashboard.hawdaj.net`, CORS, الشبكة, البيئة.

## Web Frontend (Angular)
- **Stack:** Angular `HttpClient` (vs mobile Dio). Same base URLs (prod `dashboard.hawdaj.net/api`, test `test.dashboard.hawdaj.net/api`), same `{code,message,data}` envelope, same static `accept-tokenapi` key + `Authorization: Bearer` + `locale` + **geolocation headers** (lat/lng/accuracy, refreshed every 3 min via `LocationLatLongFacade`).
- **401 handling DIFFERS (web stricter):** `ErrorInterceptor` (`error-interceptor.service.ts:52–58`) → clear all localStorage + POST `/logout` + redirect `/home`. **Mobile ignores 401** — recommend adopting web behavior on mobile.
- **Web-only interceptors:** `CachingInterceptor` (HTTP cache) + `ServerStateInterceptor` (SSR server→client state transfer). Errors surfaced via PrimeNG `MessageService`/`AlertsService.openToast`.
- **Env switching:** environment files (dev/prod/staging) at build time — no runtime dev backdoor like mobile's `dev123` env-switch. See [[WEB_FRONTEND]] · [[WEB_MOBILE_COMPARISON]].

## 38. Future Improvements
- Build-flavor separation (dev/prod), remove backdoor + hardcoded key/password from release.
- Proper auth-refresh/401 lifecycle + secure release logging.
- Timeouts + retry/backoff + offline queue.
- Tightened CORS + cert pinning + typed error envelope contract shared with backend.
