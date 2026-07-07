# Feature 18 — Taxonomy (Regions / Cities / Categories / Languages / Prices / Seasons)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Shared (master data)
> **Flutter:** `lib/core/managers/` + `lib/features/tasneef/data/models/` · **Laravel:** RegionController, CityController, CategoryController (+ Store/Zad/Food variants), PriceController, LanguageController
> **One-line:** Shared translatable (AR/EN/RU/ZH via Astrotomic) lookup data feeding every filter/classifier. Region→City and Category→SubCategory are hierarchical (self-ref `parent_id`); **categories are fragmented into 6 separate per-content-type models** (Category / CategoryOfStore / CategoryOfZad / FoodCategoryOfZad / CategoryOfSwalef / CategoryOfApplication). All lookup endpoints are **public**. Client caches taxonomy in **Cubit state only (no persistent cache → refetch across screens)**, and the Home screen uses **hardcoded dummy categories** instead of the API.

---

## 1. Feature Name & Identity
- **Name:** Taxonomy / master data — the reference tables content references for filtering & classification.
- **Members:** Regions, Cities, Categories (6 typed variants), Prices, Languages, Seasons (on Place), plus pivot links to content.
- **Shape:** translatable name/notes, hierarchical where applicable, seeded master data managed via dashboard.

## 2. Business Goal & Rules
- **Goal:** Provide consistent, localized dropdown/filter values across all content features.
- **Rules:**
  - Regions/Cities/Categories/Prices/Languages all translatable (locale-scoped name).
  - Region→City: `City.region_id` FK; cities fetched per region.
  - Category→SubCategory: self-referential `parent_id`; `whereNull(parent_id)` = roots.
  - Prices filtered by `show=1` (only active shown).
  - Languages have `active` + `order_id` + `image`.
  - Categories are per-content-type (not one shared table).
  - All lookup endpoints public (no auth).

## 3. User Flow
1. Filter/create screens request taxonomy (region picker, city picker, category chips, price filter).
2. Region dropdown → on select → CitiesCubit fetches that region's cities (cascade).
3. Category chips load parent categories per content type; sub-categories on demand.
4. Prices/Languages loaded for filters/guide forms.
- Home screen bypasses API and shows hardcoded `dummyCategories`.

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`region_drop_down_field.dart`](lib/core/views/region_drop_down_field.dart) | 31–189 | Region dropdown (RegionsCubit, Loading/Error/Success) |
| [`city_dropdown.dart`](lib/core/views/city_dropdown.dart) | 31–193 | City dropdown, depends on region → `fetchCitiesByRegion()` |
- Category chips + price filter live in Tasneef/filter UIs.

## 5. Widgets
- RegionDropdown, CityDropdown (cascading). Category chips (Tasneef). Price filter chips. Language multi-select (Tour Guides).

## 6. Cubits / Blocs
| Cubit | Location | Fetches | Cache |
|-------|----------|---------|-------|
| RegionsCubit | `core/managers/regions_cubit/` | GET /regions | state-only |
| CitiesCubit | `core/managers/cities_cubit/` | GET /regions/{id}/cities | state-only (cleared on re-enter) |
| CategoriesCubit | `features/tasneef/.../cubits/categories/` | /categories, /stores/categories, /zads/categories (dynamic by type) | state-only |
| SubCategoriesCubit | same | /zads/food-categories | state-only |
| FetchCategoryCubit | `features/trip/.../fetch_category_cubit/` | trip repo | state-only |
| FetchLanguagesCubit | `features/tourguide/.../fetch_languages_cubit/` | guide repo | state-only |
| FetchRegionCubit | `features/tourguide/.../fetch_region_cubit/` | guide repo (alt to core RegionsCubit) | state-only |
| FetchPricesCubit | `features/trip/.../fetch_prices_cubit/` | trip repo | state-only |

- **⚠️ Duplication:** two region-fetch cubits (core RegionsCubit + tourguide FetchRegionCubit).

## 7. State Flow
- RegionsCubit: Loading → Success(regions)/Error, no persistence.
- CitiesCubit: `fetchCitiesByRegion(regionId)` → Loading → Success/Error; `clear()` resets on screen re-enter → refetch.
- CategoriesCubit: `loadMainCategories(type)` switches endpoint by content type.

## 8. Models
| Model | Location | Fields | Notes |
|-------|----------|--------|-------|
| RegionModel | `tasneef/data/models/region_model.dart:1–18` | id, name | simple |
| CityModel | `tasneef/data/models/` | id, name | **region_id not exposed** |
| CategoryModel | `tasneef/data/models/` | id, icon, name, notes | no parent_id/translations |
| PriceModel | `tasneef/data/models/` | id, name | — |
| LanguageModel | `home/data/model/guide_model/` | id, name (+) | guide languages |
| ExploreCategoryModel | `home/data/model/explore_category_model.dart:22–80` | id, name, imagePath, routes, type, manor | **hardcoded dummyCategories** (not API) |

## 9. Repositories
- Taxonomy fetched via `TasneefRepository` (categories), core managers (regions/cities), and per-feature repos (trip/guide) — no single unified taxonomy repo.

## 10. Services / Helpers
- Astrotomic Translatable (backend) resolves locale name.
- `NameObserver` auto-fills/translates name on some models (see [[31_TranslationObservers]]).
- No client persistent cache layer.

## 11. API Endpoints (all PUBLIC)
| Method | Endpoint | Controller |
|--------|----------|-----------|
| GET | `regions` | RegionController@index (`Region::all()`) |
| GET | `regions/{id}` | RegionController@show |
| GET | `regions/{id}/cities` | RegionController@citiesByRegionId (cascade) |
| GET | `cities` | CityController@index (`City::all()` — all at once) |
| GET | `cities/{id}` | CityController@show |
| GET | `categories` | CategoryController@index (`whereNull parent_id`) |
| GET | `categories/{id}/sub-categories` | CategoryController@getSubCategories |
| GET | `prices` | PriceController@index (`show=1`) |
| GET | `languages` | LanguageController@index |
| GET | `stores/categories` | StoreCategoryController@index |
| GET | `zads/categories` | ZadCategoryController@index (search) |
| GET | `zads/food-categories` | ZadFoodCategoryController@index |

`routes/api.php:83–97`. Flutter EndPoints: `end_points.dart:53–155` (regions, citiesByRegion, categoriesPlaces, storesCategories, zadsCategories, zadsFoodCategories, swalefsCategories, applicationsCategories, prices, languages).

## 12. Request / Response Models
```json
Region/City: {"id":1,"name":"الرياض"}
Category:    {"id":1,"order_id":0,"icon":"https://...","name":"Places"}
Price:       {"id":1,"order_id":1,"name":"Low"}
Language:    {"id":1,"order_id":1,"name":"Arabic"}
```

## 13. Laravel Routes
`routes/api.php:83–97` — all taxonomy GETs public, no auth guard.

## 14. Laravel Controllers
- RegionController: `index()` (`Region::all()`, N+1 translations), `show()`, `citiesByRegionId()` (`City::where region_id`).
- CityController: `index()` (all), `show()`.
- CategoryController: `index()` (roots), `getSubCategories($id)`.
- PriceController: `index()` (`show=1`).
- LanguageController: `index()` (all).
- StoreCategoryController / ZadCategoryController / ZadFoodCategoryController: parent-only lists.

## 15. Laravel Services
None dedicated. Translatable trait + observers handle localization.

## 16. Laravel Models
| Model | Table | Translatable | Hierarchy | Relations |
|-------|-------|--------------|-----------|-----------|
| Region | regions | name | — | hasMany City |
| City | cities | name | region_id | belongsTo Region |
| Category | categories | name, notes | parent_id self-ref | hasMany childes / belongsTo parent |
| CategoryOfStore | category_of_stores | name, notes | parent_id | hasMany childes |
| CategoryOfZad | category_of_zads | name, notes | parent_id | belongsToMany ZadElgadel (pivot category_zad_elgadels) |
| FoodCategoryOfZad | food_category_of_zads | name, notes | parent_id | belongsToMany ZadElgadel (food_category_zad_elgadels) |
| Price | prices | name | — | `show` tinyint |
| Language | languages | name, description | — | active, order_id, image |

- Files: Region.php:13–36, City.php:13–39, Category.php:14–51, Price.php:12–25, Language.php:11–39.

## 17. Database Tables & Relationships
- Base tables + `*_translations` (Astrotomic): RegionTranslation, CityTranslation, CategoryTranslation, CategoryOfStoreTranslation, CategoryOfZadTranslation, FoodCategoryOfZadTranslation, PriceTranslation, LanguageTranslation.
- Translations: `(entity_id, locale)` unique, `locale` indexed (e.g. `2023_06_17_000000_create_category_translations_table.php`).
- Content links: `region_id`/`city_id`/`category_id` FKs on content; Zad categories via pivot tables.
- Regions/Cities use soft deletes; no `active` column.

## 18. Validation
- Lookups are read-only GETs; no request validation. Mutations happen in dashboard (out of API scope). Prices gated by `show=1`.

## 19. Auth & Permissions
- **All taxonomy endpoints public** (no auth) — appropriate for lookup data.
- No per-region/category active filtering beyond soft-delete (regions/cities) and price `show`.

## 20. Error Handling
- Flutter cubits: Loading/Error/Success; dropdowns render error/empty states.
- Laravel: standard collection responses; missing id → 404 on show.

## 21. Edge Cases
- Region with no cities → empty city dropdown.
- Category with no children → empty sub list.
- Home dummy categories drift from real backend categories (stale/incorrect).
- CitiesCubit cleared on re-enter → refetch same region's cities repeatedly.
- Inactive region/category not filtered (only soft-deleted excluded).
- `City::all()` with 100s of cities → heavy payload if called without region.

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dio, easy_localization (locale header).
- **Laravel:** Astrotomic Translatable, Eloquent, seeders.
- **Cross-feature:** consumed by nearly every content feature — [[04_Places]], [[05_Stores]], [[06_Restaurants]], [[07_Tasneef]], [[08_Exploration]], [[09_Events]], [[03_Trips]], [[11_TourGuides]], [[12_Landmarks]]. Localization via [[22_Localization]]. Auto-name via [[31_TranslationObservers]].

## 23. Files Involved
**Flutter:** core/managers (regions_cubit, cities_cubit), tasneef cubits (categories, sub-categories) + models (region/city/category/price), trip cubits (fetch_category/prices), tourguide cubits (fetch_region/languages), core/views (region_drop_down_field, city_dropdown), explore_category_model (dummy), end_points:53–155, service_locator:87–89.
**Laravel:** Region/City/Category/CategoryOfStore/CategoryOfZad/FoodCategoryOfZad/Price/Language models + translations, Region/City/Category/Price/Language + Store/Zad/Food category controllers, resources (RegionResource, CategoryResource, PriceResource…), migrations (+translations), seeders (RegionSeeder 8 regions, CitySeeder, CategorySeeder), routes/api.php:83–97.

## 24. Full Execution Flow (UI → DB → UI)
**Cascade region→city:** RegionDropdown → RegionsCubit → GET `/regions` → `Region::all()` (+translations) → RegionModel[] → user selects region → CityDropdown triggers `CitiesCubit.fetchCitiesByRegion(id)` → GET `/regions/{id}/cities` → `City::where(region_id)` → CityModel[] → dropdown.
**Category by type:** CategoriesCubit `loadMainCategories('store')` → GET `/stores/categories` → CategoryOfStore roots → chips.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| N+1 on translations (`Region::all()` loads all 4 locales/record) | 🔴 HIGH | RegionController@index + `$with=['translations']` |
| `City::all()` unpaginated | 🟡 Med | CityController@index |
| No client persistent cache → refetch every screen | 🟡 Med | all taxonomy cubits (state-only) |
| Hardcoded home categories | 🔴 High (staleness) | explore_category_model.dart:22–80 |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| Public lookups (by design) — no sensitive data | 🟢 OK | routes |
| No active filter on regions/cities/categories (only soft-delete/price show) | 🟢 Low | controllers |

## 27. Technical Debt
- **6 duplicated category models** (identical translatable+hierarchy) — no shared trait/polymorphic base.
- Duplicate region-fetch cubits (core vs tourguide).
- Home dummy categories vs API categories (two sources of truth).
- No locale scoping on translation eager-load.
- No client cache/TTL; taxonomy refetched constantly.
- CityModel drops region_id.

## 28. Improvement Opportunities
- Locale-scope translation loads (Accept-Language) to kill N+1.
- Consolidate category models via shared trait or polymorphic `categories(type)` table.
- Client persistent cache (Hive/SharedPreferences) with TTL for regions/categories/prices/languages.
- Replace Home dummy categories with API-driven fetch.
- Add `active` filter to lookups; paginate/scope `cities`.
- Unify region cubits.

---

## 32. Related Features
- **[[04_Places]] [[05_Stores]] [[06_Restaurants]] [[09_Events]] [[07_Tasneef]] [[08_Exploration]] [[03_Trips]] [[11_TourGuides]] [[12_Landmarks]]** — all consume region/city/category/price taxonomy.
- **[[22_Localization]]** — locale drives translated names.
- **[[31_TranslationObservers]]** — auto-translate category/region names.
- **[[02_Home]]** — hardcoded dummy categories anomaly.

## 33. How to Modify This Feature
- **Add a category type:** create model+translations+controller+route+resource (copy Category pattern); add endpoint + CategoriesCubit case in Flutter.
- **Add a region/city:** dashboard seed/CRUD (dashboard scope); appears via public GET automatically.
- **Fix N+1:** scope `translations` by app locale in controllers or add a locale query scope.
- **Add caching:** wrap taxonomy repos with Hive-backed cache + TTL; invalidate on version bump.

## 34. Regression Checklist
- [ ] Regions load; selecting one loads its cities.
- [ ] Category roots load per content type; sub-categories load on demand.
- [ ] Prices show only `show=1`.
- [ ] Languages load for guide form.
- [ ] Locale switch returns translated names.
- [ ] City dropdown empty until region chosen.
- [ ] (After cache) taxonomy not refetched every screen.

## 35. Common Bugs
- Home categories stale/wrong (dummy hardcoded).
- Slow region load (N+1 translations).
- Cities refetched repeatedly (no cache).
- Wrong-locale names if translation missing (fallback).
- Empty city list when region has none.

## 36. Debug Guide
- **Names wrong language:** check Accept-Language header + translation row for locale.
- **Categories don't match backend:** Home uses dummyCategories — check explore_category_model.
- **Slow /regions:** query-log translation eager-load count.
- **Cities empty:** confirm region selected → fetchCitiesByRegion called.
- **Missing category type:** verify endpoint mapping in end_points + CategoriesCubit switch.

## 37. Search Keywords
`region`, `Region`, `city`, `City`, `regions/{id}/cities`, `category`, `Category`, `CategoryOfStore`, `CategoryOfZad`, `FoodCategoryOfZad`, `parent_id`, `price`, `prices show=1`, `language`, `RegionsCubit`, `CitiesCubit`, `CategoriesCubit`, `ExploreCategoryModel dummyCategories`, `Astrotomic translations`, المناطق, المدن, التصنيفات, الأسعار, اللغات.

## 38. Future Improvements
- Single localized taxonomy bundle endpoint (all lookups in one cached call) + versioned client cache.
- Polymorphic/unified category system across content types.
- Admin-driven Home categories (kill dummy).
- Locale-scoped, indexed translation queries; active flags everywhere.
