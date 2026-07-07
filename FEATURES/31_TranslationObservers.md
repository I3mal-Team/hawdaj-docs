# Feature 31 — Auto-Translation Observers (Google Translate)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Hidden / Internal (shared i18n infra) · **🏁 FINAL feature (31/31)**
> **Laravel:** `app/Observers/*` + `stichoza/google-translate-php` + Astrotomic · **Flutter:** NOT involved (pure backend)
> **One-line:** On every translatable model save, ~11 Observer classes **synchronously scrape free Google Translate** (`stichoza/google-translate-php`, no API key, unofficial) to translate Arabic (authoritative source) → en/ru/zh, storing into `{model}_translations`. **Zero error handling**: any network failure / rate-limit throws uncaught → the **save fails entirely** (all-or-nothing), and each save blocks the request 3–6s (6 HTTP calls for a 2-field model). No caching (re-translates unchanged text). Incomplete coverage: Swalef `mainCharacters` commented, GuideObserver Arabic commented, MenuObserver `updated()` empty, LanguageObserver disabled. Machine-translating heritage content (Swalef) risks quality/context loss.

---

## 1. Feature Name & Identity
- **Name:** Auto-Translation Observers — model-observer-driven machine translation into 4 locales.
- **Mechanism:** `stichoza/google-translate-php` v5.1 (unofficial HTML-scraping, **no API key, no auth**), aliased `GoogleTranslate` (config/app.php:231).
- **Storage:** Astrotomic `laravel-translatable` (`{model}_translations`).

## 2. Business Goal & Rules
- **Goal:** Auto-populate en/ru/zh translations from Arabic on content save (feeds [[18_Taxonomy]] + [[22_Localization]]).
- **Rules:**
  - Source locale ar → targets `locales('ar')` = [en, ru, zh] (App.php:139–145).
  - Observers fire on `saving`/`creating` (mostly synchronous, pre-save).
  - Place-family also generates English slug from translated title.

## 3. User Flow (admin/content save)
1. Admin/user saves translatable model (Arabic).
2. Observer stores ar translation, loops en/ru/zh → `GoogleTranslate::trans()` per field per locale (HTTP).
3. Stores each translation → save completes (after all HTTP calls).

## 4. Screens / Views
- None (backend infra). Effects visible via localized content ([[22_Localization]]).

## 5–7. Widgets / Cubits / State
- N/A. Flutter uninvolved.

## 8. Models (observed)
| Model | Observer | Hook | Fields |
|-------|----------|------|--------|
| Region, City, Price | NameObserver | saving | name |
| Feature | FeatureObserver | saving | name, description |
| Place, Event, Store, ZadElgadel | PlaceObserver | creating, saving | title, description, slug(en) |
| Swalef | SwalefObserver | saving | title, description, content, timePeriod, location, storyImportance, relevanceToPresent, source, audioStoryTitle; **mainCharacters commented (22,37,50)** |
| Category(+OfZad/OfSwalef/OfStore/OfApplication) | CategoryObserver | saving | name, notes |
| Offer | OfferObserver | saving | title, description |
| Application | ApplicationObserver | saving | title, description, slug(en) |
| Guide | GuideObserver | saving | name, nickName, description (**Arabic source commented 20–22**) |
| Menu | MenuObserver | created (**updated() empty 41**) | title, description |
| Slider | SliderObserver | saving | title |
| Language | LanguageObserver | **NOT registered (disabled, Language.php:157–159)** | name, description |

## 9. Repositories / 10. Services
- `GoogleTranslate::trans($text, $locale)` inline in observers (no service wrapper, no try/catch).
- `HasSettingTranslationUpdate` trait (17–20): manual Setting translation (all 4 locales), no error handling.
- CLI: `TranslateToChinese`, `TranslateToRussian` batch commands.
- Helper `locales($exclude)` (App.php:139–145), `default_lang()`.

## 11. API Endpoints
- None. Triggered by model events (dashboard/API content creation → observer fires).

## 12. Request / Response
- No direct contract. Side effect: `{model}_translations` rows populated.

## 13. Routes
- N/A (event-driven).

## 14. Controllers
- Indirect: any create/update of a translatable model triggers observer. Direct call also in Dashboard UsersPlacesController:400–401.

## 15. Services / Providers
- Observers registered via `static::observe(X::class)` in each model's `boot()` (not a central provider). Some commented (Language).

## 16. Models
- All translatable content models + Astrotomic trait + `$translatedAttributes` + `$with=['translations']` (drives eager-load, N+1 per [[18_Taxonomy]]).

## 17. Database Tables
- `{model}_translations` (id, {model}_id, locale, {attrs}, unique(model_id,locale), FK cascade). Migrations 2023_06_15–17. Bulk backfill migrations 2023_06_22_* call `GoogleTranslate::trans` in loops (no error handling → migrate can hang/fail).

## 18. Validation
- None on translation output. Input is the model's own validation.

## 19. Auth & Permissions
- Inherits the triggering action's auth (dashboard/API). Observer itself no auth. No API key (scraping).

## 20. Error Handling
- **ZERO.** `GoogleTranslate::trans()` uncaught in all observers. Network timeout / 503 / rate-limit → exception → **save() fails** (model not saved). Partial/empty translations possible if lib returns empty.

## 21. Edge Cases
- Google unreachable/rate-limited → content creation fails outright.
- Rapid bulk creates → IP blocked (~100 req/day free tier).
- Unchanged text re-translated every save (no diff/cache).
- Commented fields (Swalef mainCharacters, Guide Arabic) → missing translations.
- MenuObserver updates don't re-translate.
- Language model never auto-translates.
- Heritage/cultural text (Swalef) machine-translated → quality/context loss.
- Migration backfill hangs if Google down.

## 22. Dependencies
- `stichoza/google-translate-php` ^5.1 (composer.json:40), astrotomic/laravel-translatable v11.
- **Cross-feature:** [[18_Taxonomy]] (auto-translates region/category/etc.), [[22_Localization]] (4-locale schema + serving), [[10_Swalef]] (mainCharacters gap), [[11_TourGuides]] (Guide observer bug), all translatable content ([[04_Places]]…[[09_Events]], Offers, Applications, Sliders).

## 23. Files Involved
`app/Observers/*` (NameObserver, FeatureObserver, PlaceObserver, SwalefObserver, CategoryObserver, OfferObserver, ApplicationObserver, GuideObserver, MenuObserver, SliderObserver, LanguageObserver[disabled]), model boot() observe() calls, App.php:139–152 (locales/default_lang), composer.json:40, config/app.php:231 (alias), HasSettingTranslationUpdate trait, Console TranslateToChinese/Russian, migrations 2023_06_15–17 (+2023_06_22_* bulk), UsersPlacesController:400–401.

## 24. Full Execution Flow (Place create)
`POST /places` (Arabic) → `PlaceObserver::creating()` → store ar title/description → `foreach locales('ar') [en,ru,zh]`: `GoogleTranslate::trans(title,$locale)` (HTTP ~500ms) + `trans(description,$locale)` (HTTP) → `translateOrNew($locale)` → set fields → generate en slug → `$model->save()` completes → response (~3–6s). Any `trans()` throw → whole save fails.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| **Synchronous blocking translation** (3–6s/save, external API in request cycle) | 🔴 HIGH | all observers |
| **No caching** (re-translate unchanged text) | 🟡 Med | observers |
| Bulk create → hours + rate-limit (1000 Places = 6000 calls) | 🔴 HIGH | observers/migrations |
| Migration backfill blocking network loops | 🟡 Med | 2023_06_22_* |

## 26. Security / Reliability
| Issue | Severity | Location |
|-------|----------|----------|
| **No error handling → save fails on Google outage** (all-or-nothing) | 🔴 HIGH | all observers |
| **Free-tier scraping** (unofficial, breakable, IP-bannable, no SLA) | 🔴 HIGH | stichoza lib |
| No API key (scraping) — no cost but fragile | 🟡 Med | config |
| Content creation coupled to external 3rd-party availability | 🔴 HIGH | request cycle |

## 27. Technical Debt
- Synchronous external dependency in save path (fragile, slow).
- Zero error handling / retry / fallback.
- No caching / dedupe.
- Incomplete coverage (Swalef mainCharacters, Guide ar, Menu updates, Language disabled).
- Unofficial scraping library (breakage risk).
- Inline duplication across 11 observers (no shared translation service).
- Machine translation of cultural heritage (quality).

## 28. Improvement Opportunities
- **Queue translation** (job after save) so content saves instantly + retries on failure; never block/fail the save.
- Wrap in try/catch + logging + fallback (leave untranslated, backfill later).
- Cache/dedupe by source-text hash; skip unchanged.
- Migrate to official Google Cloud Translation API (SLA, rate limits, key) — or keep stichoza only for non-critical.
- Central `TranslationService` (DRY) + complete coverage (uncomment/fix Swalef/Guide/Menu/Language).
- Human review for heritage content (Swalef) — don't machine-translate blindly.

---

## 32. Related Features
- **[[22_Localization]]** — serves these translations (4 locales, RTL).
- **[[18_Taxonomy]]** — region/category/etc. auto-translated.
- **[[10_Swalef]]** — mainCharacters gap + heritage quality.
- **[[11_TourGuides]]** — GuideObserver Arabic-commented bug.
- **[[04_Places]]…[[09_Events]]** + Offers/Applications/Sliders — all observed.

## 33. How to Modify This Feature
- **Add a translated field/model:** add `$translatedAttributes` + observer (or extend existing) with the field in the locale loop; register `static::observe()` in boot().
- **Fix a gap:** uncomment Swalef mainCharacters / Guide ar / Menu updated() / register LanguageObserver.
- **Make resilient:** wrap trans() in try/catch; move to a queued job; add caching.
- **Switch provider:** replace stichoza with Google Cloud Translate (add key/config).

## 34. Regression Checklist
- [ ] Create Arabic content → en/ru/zh translations populated.
- [ ] Google down → decide: save succeeds (queued) vs fails (current).
- [ ] Update re-translates (except Menu — currently no).
- [ ] Bulk create doesn't rate-limit/ban.
- [ ] Heritage content quality acceptable/reviewed.

## 35. Common Bugs
- Content save fails when Google unreachable/rate-limited.
- Slow saves (3–6s).
- Missing translations for commented fields.
- Menu updates keep stale translations.
- IP banned after bulk ops.
- Machine-translated cultural text reads poorly.

## 36. Debug Guide
- **Save failing:** check if observer's GoogleTranslate call threw (network/rate-limit) — no try/catch.
- **Missing translation:** field commented in observer? Language disabled? Menu update?
- **Slow create:** count HTTP calls (fields × 3 locales).
- **Rate-limited:** too many rapid saves → Google IP block.
- **Migration hangs:** 2023_06_22_* bulk translate loops.

## 37. Search Keywords
`Observer`, `GoogleTranslate`, `stichoza/google-translate-php`, `NameObserver`, `PlaceObserver`, `SwalefObserver`, `CategoryObserver`, `GuideObserver`, `MenuObserver`, `LanguageObserver`, `translateOrNew`, `locales()`, `translatedAttributes`, `{model}_translations`, `saving creating hook`, `auto translate ar en ru zh`, الترجمة التلقائية, المراقبون.

## 38. Future Improvements
- Queued, retryable, cached translation via official API; complete + DRY coverage; human review for heritage; decouple content save from external translation availability.

---

## 🏁 Knowledge Base Complete
This is the **final feature (31/31)**. All features analyzed. See `AI/FEATURE_INDEX.md`, `AI/PROJECT_PROGRESS.md`, and the cross-cutting index files (`FEATURE_GRAPH.md`, `SEARCH_INDEX.md`, `API_INDEX.md`, `DATABASE_INDEX.md`).
