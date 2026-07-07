# Feature 22 — Localization & RTL (ar / en / ru / zh)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Shared (i18n mechanism)
> **Flutter:** `easy_localization` + `LocaleCubit` + `LanguageInterceptor` · **Laravel:** `ChangeLocale` middleware + `laravellocalization` config + Astrotomic
> **One-line:** 4 locales (ar/en/ru/zh) configured both sides; **ar is default+fallback-start** (fallback locale = en). RTL is auto via easy_localization + a redundant `locale=='ar'` manual check (only Arabic treated RTL); `EdgeInsetsDirectional` used correctly, but **one hardcoded `TextDirection.ltr` override** in a global wrapper. Locale is sent to backend via a **custom `locale` header (NOT standard Accept-Language)** — the middleware reads `Locale`/`locale` and **ignores `accept-language`**, defaulting to `ar`. RU/ZH translations are **~92% (Google-auto, ~40 keys short each)**; **15+ hardcoded Arabic strings** bypass i18n (incl. onboarding "استكشف"); **no locale-aware date/number/currency (no Hijri, no SAR)**.

---

## 1. Feature Name & Identity
- **Name:** Localization & RTL — the i18n mechanism (not the translatable data itself; that's [[18_Taxonomy]]/[[31_TranslationObservers]]).
- **Locales:** ar, en, ru, zh. Default/start = ar. Fallback = en (Flutter) / en (app.php) / ar (API middleware default).

## 2. Business Goal & Rules
- **Goal:** Multi-language UI + content with correct direction, persisted per user, propagated to backend for localized responses.
- **Rules:**
  - Supported locales fixed to ar/en/ru/zh both sides.
  - Arabic → RTL; others → LTR (by languageCode, not script).
  - Locale persisted in `selected_language` (SharedPreferences).
  - Every API request carries `accept-language` + `locale` headers.
  - Backend resolves content locale via `App::setLocale` from `locale` header (fallback ar).

## 3. User Flow
1. First launch → start locale ar (or saved).
2. User opens language bottom sheet → selects → `LocaleCubit` sets + persists → app refreshes (Home/Sliders/Regions) + direction flips.
3. Subsequent API calls send `locale` header → backend returns that-locale content.

## 4. Screens / Views
- Language bottom sheet (in `profile_view.dart:274–280`, see [[21_ProfileSettings]]). No dedicated localization screen; i18n is app-wide via `.tr()`.

## 5. Widgets
- All UI uses `.tr()` keys; `Directionality` auto from locale. Manual RTL checks in ~15 widgets (custom_text_field_upper_hint, home widgets details_section/rating_widget, etc.).

## 6. Cubits / Blocs
| Cubit | File:Lines | Role |
|-------|-----------|------|
| LocaleCubit | `core/locale/locale_cubit.dart:1–80` | supported ar/en/ru/zh (65–70), persist `selected_language` (12), `loadSavedLanguage()`, `isRTL` = `languageCode=='ar'` (60–62), `changeLanguage()` |

## 7. State Flow
- `loadSavedLanguage()` (main.dart:109) → set locale. `changeLanguage()` → EasyLocalization.setLocale + persist → BlocListener (main.dart:189–228) refreshes Home/Sliders/Regions with new locale.

## 8. Models
- No model; locale is a `Locale` value + string code. Translation content models are Astrotomic (backend).

## 9. Repositories
- None specific. `LanguageInterceptor` injects headers into all Dio requests.

## 10. Services / Helpers
- easy_localization (setup, `.tr()`, RTL).
- [`language_interceptor.dart`](lib/core/databases/language_interceptor.dart) `:1–26`: adds `accept-language: <lang>` (16) + `locale: <lang>` (17), fallback `ar` (19–20), from `selected_language` (14).
- `intl` for date/time (not locale-parameterized).
- Backend: `ChangeLocale` middleware, `App::setLocale`, Astrotomic trait.

## 11. API Endpoints
- No dedicated endpoint. Locale rides as headers on every request; backend resolves per-request.

## 12. Request / Response Models
- Request headers: `accept-language: ar|en|ru|zh`, `locale: ar|en|ru|zh`.
- Responses: translated `name`/`description`/etc. resolved by Astrotomic for the set locale.

## 13. Laravel Routes
- No route. `ChangeLocale` middleware registered globally (Kernel.php:25).

## 14. Laravel Controllers
- N/A (middleware-driven). Content controllers return translatable models resolved by active locale.

## 15. Laravel Services / Middleware
- [`ChangeLocale.php`](../../../hawdaj-api/app/Http/Middleware/ChangeLocale.php) `:1–34`: reads header `Locale`/`locale` (22); **does NOT read `Accept-Language`**; API default `ar` (27); `App::setLocale()` (31).
- `laravellocalization.php`: ar (45), en (46), ru (226), zh (293); `useAcceptLanguageHeader:true` (312) — **config set but middleware ignores it**.
- Astrotomic Translatable trait on 15+ models (`{model}_translations`, `locale` column).

## 16. Laravel Models
- Translatable models: Slider, Place, Store, Event, Guide, Region, City, Category(+variants), Offer, Swalef, Feature, Setting, Price… (`$translatedAttributes`, Translatable trait). See [[18_Taxonomy]].

## 17. Database Tables & Relationships
- `{model}_translations` tables, `(model_id, locale)` unique, `locale` indexed. Seeded with ar/en/ru/zh (ru/zh Google-auto via migrations/observers — see [[31_TranslationObservers]]).

## 18. Validation
- N/A (mechanism). Locale codes not validated server-side (unknown code → Astrotomic fallback).

## 19. Auth & Permissions
- Locale headers on all requests (auth-independent). Public.

## 20. Error Handling
- Missing translation → Astrotomic fallback locale. Missing `.tr()` key → key shown literally (easy_localization). Unknown locale header → default ar.

## 21. Edge Cases
- Flutter sends `accept-language` but middleware ignores it — works only because `locale` header also sent (single point: if `locale` header dropped, backend defaults ar regardless of user choice).
- RTL keyed on `=='ar'` — if an RTL locale added later (e.g. fa/he/ur), won't flip.
- `global_inner_pages_wrapper.dart` forces `TextDirection.ltr` → RTL content misrendered there.
- RU/ZH missing ~40 keys each → those show fallback (en) or key.
- Dates always en-US format regardless of locale (no ar-SA/ru-RU/zh-CN).
- Hardcoded Arabic strings never translate for en/ru/zh users.

## 22. Dependencies
- **Flutter:** easy_localization, intl, shared_preferences, flutter_bloc.
- **Laravel:** mcamara/laravel-localization, astrotomic/laravel-translatable.
- **Cross-feature:** [[18_Taxonomy]] (translatable data), [[31_TranslationObservers]] (auto ru/zh), [[24_Networking]] (interceptor), [[19_SplashOnboarding]] (start locale race), [[21_ProfileSettings]] (switch entry), every content feature (localized responses).

## 23. Files Involved
**Flutter:** main.dart (49,69–82 setup; 109 load; 189–228 refresh), locale_cubit.dart, language_interceptor.dart, assets/translations/{ar,en,ru,zh}.json, global_inner_pages_wrapper.dart (LTR override), ~15 widgets with manual RTL, date_time_utils/date_extensions/time_of_day_extension, onboarding_view.dart:172 (hardcoded).
**Laravel:** ChangeLocale.php, Kernel.php:25, config/laravellocalization.php, config/app.php (locale/fallback), Astrotomic models + translations.

## 24. Full Execution Flow (locale round-trip)
User selects lang → `LocaleCubit.changeLanguage` → EasyLocalization.setLocale + persist `selected_language` → BlocListener refreshes cubits → next Dio request → `LanguageInterceptor` adds `accept-language` + `locale` → Laravel `ChangeLocale` reads `locale` header → `App::setLocale(ar)` → Astrotomic queries `{model}_translations WHERE locale=ar` → localized JSON → Flutter renders with auto RTL (ar) / LTR (others).

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Language change refetches Home/Sliders/Regions | 🟢 Low | main.dart:189–228 |
| Astrotomic translation eager-load N+1 (per [[18_Taxonomy]]) | 🟡 Med | backend models |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| No sensitive data; locale headers benign | 🟢 OK | — |
| Unknown locale header silently defaults (no validation) | 🟢 Low | ChangeLocale |

## 27. Technical Debt
- Non-standard `locale` header instead of proper `Accept-Language` (middleware ignores the standard one it's sent).
- RTL logic hardcoded to `=='ar'` (not script/direction-based).
- One `TextDirection.ltr` override breaking RTL in a global wrapper.
- 15+ hardcoded Arabic strings bypass i18n (onboarding "استكشف", logout dialog, error messages, dev screens).
- RU/ZH ~8% incomplete (Google-auto).
- No locale-aware date/number/currency; no Hijri, no SAR.
- Redundant manual RTL checks alongside auto handling.

## 28. Improvement Opportunities
- Standardize on `Accept-Language` (make middleware read it; keep `locale` as fallback).
- Direction from locale script (support future RTL locales) — drop `=='ar'` hardcode.
- Remove the global LTR override; audit RTL rendering.
- Extract all hardcoded strings to .arb/.json; add lint/CI check for raw literals.
- Complete RU/ZH keys; human-review Google translations.
- Locale-aware `DateFormat(locale)`, number separators, SAR currency, optional Hijri.

---

## 32. Related Features
- **[[18_Taxonomy]]** — translatable master data.
- **[[31_TranslationObservers]]** — auto ru/zh translation on write.
- **[[24_Networking]]** — LanguageInterceptor in Dio stack.
- **[[19_SplashOnboarding]]** — start-locale + boot race.
- **[[21_ProfileSettings]]** — language switch entry.

## 33. How to Modify This Feature
- **Add a locale:** add to easy_localization supportedLocales + LocaleCubit + laravellocalization.php + translation JSON + `{model}_translations` seed; set RTL if needed (fix `isRTL`).
- **Fix backend header:** make `ChangeLocale` read `Accept-Language` (parse first tag) before `locale`.
- **Fix RTL override:** remove hardcoded `TextDirection.ltr` in global_inner_pages_wrapper.
- **Localize dates:** pass `context.locale` to `DateFormat`.
- **Extract hardcoded string:** replace literal with `'key'.tr()` + add to all 4 JSONs.

## 34. Regression Checklist
- [ ] Switch to each locale → UI strings change.
- [ ] Arabic → RTL layout; others → LTR.
- [ ] Locale persists across restart.
- [ ] API returns content in selected locale (check `locale` header).
- [ ] RU/ZH screens fall back gracefully for missing keys.
- [ ] No LTR-forced RTL breakage on wrapped pages.

## 35. Common Bugs
- Backend returns Arabic despite en/ru/zh selection (if `locale` header lost; Accept-Language ignored).
- Onboarding button always Arabic ("استكشف").
- Dates always English format.
- RU/ZH show English/keys for missing translations.
- RTL text misaligned inside global LTR wrapper.

## 36. Debug Guide
- **Wrong-language API content:** confirm `locale` header present (interceptor); note Accept-Language is ignored by middleware.
- **String not translating:** check it uses `.tr()` + key exists in all JSONs (not hardcoded).
- **RTL broken on a page:** check for `TextDirection.ltr` override / non-directional EdgeInsets.
- **Missing RU/ZH text:** diff key counts (ar 552 / en 550 / ru 513 / zh 511).
- **Date format wrong:** DateFormat lacks locale param.

## 37. Search Keywords
`easy_localization`, `EasyLocalization`, `LocaleCubit`, `selected_language`, `supportedLocales ar en ru zh`, `isRTL`, `Directionality`, `TextDirection.rtl`, `EdgeInsetsDirectional`, `LanguageInterceptor`, `accept-language`, `locale header`, `ChangeLocale`, `laravellocalization`, `Astrotomic translatable`, `استكشف hardcoded`, التعريب, اللغة, RTL, يمين لليسار.

## Web Frontend (Angular)
- **Same 4 locales** ar/en/ru/zh via **ngx-translate** (mobile uses easy_localization). Files `assets/i18n/{ar,en,ru,zh}.json`. RTL auto for Arabic (`app.component.ts:256–261`).
- **Lang in URL:** `/:lang/route` prefix (e.g. `/ar/places`) + localStorage; versioned loader for cache-busting. Header sent to backend: `locale` (interceptor) — same backend resolution ([[31_TranslationObservers]]).
- **WEB-ONLY: full SEO/i18n meta.** `MetadataService` sets title, og/twitter share images, **canonical** (`updateCanonicalLink:178`) and **hreflang** (`updateLinkRelAlternate:165`); per-locale tags in `meta-tags-pages.ts` for Home/Places/Events/Applications/TourGuides/Stores/Stories/Restaurants. Mobile has none of this. See [[WEB_FRONTEND]].

## 38. Future Improvements
- Standards-compliant Accept-Language negotiation + script-based direction.
- Full RU/ZH coverage + translation QA + CI hardcoded-string lint.
- Locale-aware dates/numbers/currency (Hijri + SAR) via a formatting service.
- Single source-of-truth locale config shared across Flutter + Laravel.
