# Feature 08 — Exploration (استكشاف — Global Search + Map)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/exploration) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 مُجمِّع بحث + خريطة. يبحث عبر **عدّة نماذج** (places/stores/zads/events) في نداء واحد ويعرضها كقائمة/ماركرات، ويوجّه لتفاصيل كل نوع (يعيد استخدام [[04_Places]]/[[05_Stores]]/[[06_Restaurants]]/events). مشابه لـ [[07_Tasneef]] لكن للبحث الحرّ والخريطة.

---

## 1. Feature Name
Exploration — بحث عام (global-search) عبر كل الأنواع + خريطة Google تفاعلية بماركرات دائرية للمحتوى، مع فلاتر (نوع/منطقة/مدينة) وموقع المستخدم.

## 2. Business Goal
اكتشاف حرّ: اكتب استعلامًا فترى نتائج مختلطة (أماكن/متاجر/مطاعم/فعاليات)، أو تصفّح خريطة بماركرات، وانتقل مباشرة لتفاصيل أي عنصر.

## 3. Business Rules
- البحث الافتراضي عبر 4 أنواع (events/stores/zads/places)؛ `viewAs` يقيّد الأنواع (CSV).
- كل نوع `active=1`. البحث عبر `whereTranslationLike(title/description, %search%)`.
- الترتيب بموقع → مسافة Haversine.
- الترقيم: **بعد الدمج في الذاكرة** (`LengthAwarePaginator(forPage)`) — لا على مستوى DB.
- ماركرات: عناصر بإحداثيات صالحة فقط، صور دائرية (56px، حدّ أبيض 2px)، تحميل دفعات (5 متزامن) + LRU cache (128).
- ماركر الموقع الحالي (أزرق) يُحفَظ عند تحديث الماركرات.
- debounce بحث 350ms.

## 4. User Flow
SearchView/MapView → اكتب → نتائج مختلطة (قائمة أو ماركرات) → tap عنصر/ماركر → bottom sheet (حفظ/موقع/المزيد) → تفاصيل النوع. فلاتر pills (events/places/stores/zads/all) + region/city dropdowns.

## 5. Flutter Screens
| Screen | File |
|---|---|
| SearchView (غلاف) | `presentation/view/search_view.dart` |
| SearchViewBody (شريط بحث + فلاتر + قائمة مصفّحة) | `.../view/widgets/search_view_body.dart:39-307` |
| MapView (خريطة + قائمة أفقية) | `.../view/map_view.dart:26-409` |

## 6. Widgets
Google Maps widget (`view/widgets/map_view.dart`)، ماركرات دائرية، `ItemDetailsWidget` (bottom sheet ماركر)، `MultiSelectPillsBar` (فلاتر أنواع)، `map_item_glodal_to_unified_place` (mapper)، region/city dropdowns، `TasneefPlacesItemCard` (مشترك مع Tasneef).

## 7. Cubits / Blocs
| Cubit | File | دور |
|---|---|---|
| GetSearchGlobalCubit | `.../get_search_global_cubit/*:10-84` | بحث مصفّح (PagingController)، updateFilters/refresh/_fetchPage |
| MapCubit | `.../map_cubit/map_cubit.dart:1-432` | ماركرات، setMarkersFromItems، addUserLocationMarker، صور دائرية+cache، changeViewAs، bottom sheet |
| SearchFilterCubit | `.../search_filter_cubit/*:8-94` | جسر (searchText/regionId/cityId) → يزامن GetSearchGlobalCubit |
| LocationCubit (core) | — | موقع المستخدم |

## 8. State Flow
كتابة → debounce 350ms → SearchFilterCubit.updateSearch → _syncWithSearchCubit → GetSearchGlobalCubit.updateFilters+refresh → PagingController → _fetchPage → repo.getSearchGlobal → fold → appendPage/LastPage. الخريطة: listener على pagingController → MapCubit.setMarkersFromItems → _addMarkers (دفعات) → MarkersUpdated.

## 9. Models
- `GlodalSearch` (code/message/data) → `Data` (pagination) → **`ItemGlodal`** (`item.dart:4-76`): id, slug, **type** (place/store/event/zad/restaurant/story), image, address, lat, long, featured, title, ratingsCount, rate, city, region, isFavorite.
- `PaginatedResponse<ItemGlodal>`، City/Region.
- mapper ItemGlodal → UnifiedPlaceModel (لإعادة استخدام TasneefPlacesItemCard).

## 10. Repositories
- **Flutter** `exploration_repo.dart` + `exploration_repo_imp.dart:15-48`: `getSearchGlobal(page, search, regionId, cityId, viewAs)` → GET `global-search` (per_page=50) → PaginatedResponse<ItemGlodal>.
- **Laravel:** لا repository — SearchController.

## 11. Services
لا service مخصّص. يعيد استخدام `getLocationCoords`/`applyDistanceSorting`. الخريطة: معالجة صور canvas + LRU cache client-side.

## 12. API Endpoints
| Method | Endpoint | Auth |
|---|---|---|
| GET | `global-search?search=&region_id=&city_id=&viewAs=&page=&per_page=` | عام |
| GET | `global-map-data` | عام |
| GET | `places/places-data-for-map` | عام |

## 13. Request Models
لا body. query: search، region_id، city_id، viewAs (CSV أنواع)، page، per_page(50).

## 14. Response Models
`SearchCollection` مصفّح → items عبر `SearchResource` (flat، بحقل `type`): id, slug, type, image(+small/medium), address, lat, long, featured, title, ratings_count, rate, city, region, is_favorite.

## 15. Laravel Routes
`routes/api.php`: `GET global-search → SearchController@search` · `GET global-map-data → SearchController@global_map_data` · `GET places/places-data-for-map → PlaceController@placesDataForMap`. عامة.

## 16. Controllers
**SearchController** (`app/Http/Controllers/Api/SearchController.php:42-275`): `search` (:45-89، `getViewAsTypes` + 4 استعلامات منفصلة searchEvents/Stores/Zads/Places → merge → **LengthAwarePaginator في الذاكرة** → SearchCollection)، مساعِدات `applySearchFilter:244-252` (whereTranslationLike). `global_map_data` (ماركرات). `PlaceController@placesDataForMap` (أماكن بإحداثيات).

## 17. Service Classes (Laravel)
لا service. منطق التجميع inline في SearchController (4 دوال بحث خاصة بالنوع).

## 18. Models (Laravel)
يعيد استخدام Place/Store/ZadElgadel/Event (بحث + إحداثيات). SearchResource يوحّد الإخراج بحقل type. `isFavorite($id, $type)` per-user.

## 19. Database Tables
يعيد استخدام جداول الكيانات (+translations للبحث). لا جدول خاص بالبحث. الخريطة تقرأ lat/long من places/stores/zad_elgadels/events.

## 20. Relationships
كل نتيجة تحمل city/region (eager). لا علاقة بحث مخصّصة. الماركرات تعتمد lat/long + image + slug/id للتنقّل.

## 21. Validation Rules
لا تحقّق على search (متساهل). viewAs يُفكَّك CSV. region_id/city_id اختيارية.

## 22. Authorization & Permissions
عام بالكامل. is_favorite/is_saved per-user. لا حماية على البحث/الخريطة.

## 23. Error Handling
- **Flutter:** pagingController.error؛ ماركرات: تُسقط العناصر بلا صورة/إحداثيات بصمت.
- **Laravel:** الغلاف ApiModalController. whereTranslationLike parameterized (يخفّف الحقن).

## 24. Edge Cases
- استعلام فارغ → يعيد الكل (أو حسب viewAs).
- بلا موقع → ترتيب order_id.
- عنصر بلا إحداثيات → لا ماركر.
- عنصر بلا صورة → يُسقَط من الخريطة.
- نتائج ضخمة → دمج كامل ثم ترقيم بالذاكرة (كلفة).
- نوع غير معروف في navigation → fallback places.
- أذونات موقع مرفوضة → openAppSettings/snackbar.
- zad ↔ restaurant alias في التنقّل.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, infinite_scroll_pagination, **google_maps_flutter**, geolocator (LocationCubit), http (صور ماركر)، dart:ui (canvas). Laravel: astrotomic/translatable، Haversine (raw), LengthAwarePaginator.

## 26. Files Involved
**Flutter:** `lib/features/exploration/**` (data/repo، data/model/glodal_search/*، manager/{get_search_global,map,search_filter}_cubit، view/{search_view,map_view} + widgets)، `lib/features/tasneef/.../tasneef_places_item_card` (مشترك)، LocationCubit، `end_points.dart`.
**Laravel:** `app/Http/Controllers/Api/SearchController.php`، `PlaceController@placesDataForMap`، `app/Http/Resources/Search/{SearchResource,SearchCollection}.php`، `app/Helper/App.php`، `routes/api.php`.

## 27. Complete Execution Flow
**البحث:** كتابة → debounce → SearchFilterCubit → GetSearchGlobalCubit.refresh → _fetchPage → repo.getSearchGlobal → GET `global-search` → SearchController@search (viewAs → 4 استعلامات → merge → LengthAwarePaginator → SearchResource) → PaginatedResponse<ItemGlodal> → appendPage → PagedSliverList (TasneefPlacesItemCard) → tap → push(route by type).
**الخريطة:** initState → LocationCubit.fetchCurrentLocation → animateCamera + addUserLocationMarker → listener على pagingController → setMarkersFromItems → _addMarkers (صور دائرية canvas، دفعات 5، LRU cache) → MarkersUpdated → GoogleMap → tap ماركر → ItemDetailsWidget (حفظ/موقع/المزيد).

## 28. Performance Considerations
- 🔴 **N استعلامات منفصلة** (4 نماذج) + **دمج وترقيم بالذاكرة** (لا DB pagination) — كلفة عالية لنتائج كبيرة.
- 🔴 **whereTranslationLike `%..%`** = full scan بلا فهرس (لا full-text/trigram).
- 🟠 **Haversine لكل صف** قبل الترقيم.
- 🟠 صور الماركر: http.get لكل ماركر (خفّفها LRU cache 128 + دفعات 5).
- 🟠 لا **clustering** للماركرات (مئات الماركرات = بطء).
- 🟡 per_page=50 + دمج 4 أنواع → payload كبير.

## 29. Security Considerations
- عام (مقبول للبحث)، لكن يمكن استغلاله للـscraping.
- whereTranslationLike parameterized → الحقن مُخفّف.
- 🟡 `type`/`viewAs` من العميل بحرية (قد يطلب أنواعًا داخلية).
- يرث تسريب email (لكن SearchResource لا يُخرج ratings الخام — يُخرج rate/ratings_count فقط ✅).

## 30. Technical Debt
- 4 دوال بحث متكرّرة في SearchController (يمكن UNION/موحّد).
- ترقيم بالذاكرة (يجب DB-level UNION).
- switch navigation منتشر (مطابق Tasneef، 3 مواضع).
- ItemGlodal منفصل عن UnifiedPlaceModel (mapper وسيط).
- لا clustering/autocomplete/saved searches.
- صور ماركر تُبنى client-side (كلفة) بدل server thumbnails.

## 31. Improvement Opportunities
- استعلام UNION ALL على مستوى DB + ترقيم حقيقي.
- full-text/trigram index للبحث بدل LIKE.
- spatial index لـHaversine.
- marker clustering + server thumbnails جاهزة.
- autocomplete/suggestions + saved searches.
- توحيد ItemGlodal مع UnifiedPlaceModel.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 04/05/06/09/10 | البحث يشملها ويوجّه لتفاصيلها |
| 07 Tasneef | يشارك TasneefPlacesItemCard + switch navigation |
| 01 Auth | is_favorite/is_saved + الحفظ من bottom sheet |
| 13 Favorites/Saved | حفظ من الخريطة |
| 18 Taxonomy | فلاتر region/city |
| Location (core) | موقع المستخدم + الترتيب الجغرافي |
> تغيير SearchResource/ItemGlodal يؤثّر على البحث والخريطة معًا.

## 33. How to Modify This Feature
- **نوع بحث جديد:** SearchController (دالة searchX + merge) + viewAs + navigation switch + ItemGlodal.type.
- **فلتر جديد:** SearchFilterCubit + query params + SearchController.
- **تحسين الأداء:** UNION في SearchController، فهرسة، clustering في MapCubit.
- **APIs:** global-search, global-map-data, places-data-for-map.
- **المخاطر:** N استعلامات، ترقيم بالذاكرة، LIKE بطيء، switch منتشر، كلفة صور الماركر.
- **اختبارات:** بحث (فارغ/نتائج/خاص)، فلاتر أنواع/منطقة، خريطة (ماركرات/موقع/tap)، navigation بالنوع، أداء نتائج كبيرة.

## 34. Regression Checklist
- [ ] كتابة → نتائج مختلطة بعد debounce.
- [ ] فلاتر pills (events/places/stores/zads/all) تقيّد الأنواع.
- [ ] region → clear city cascade.
- [ ] ترقيم لانهائي للنتائج.
- [ ] خريطة: موقع المستخدم (أزرق) + ماركرات دائرية.
- [ ] tap ماركر → bottom sheet (حفظ/موقع/المزيد).
- [ ] navigation صحيح لكل نوع.
- [ ] أذونات موقع (مرفوض/دائم).
- [ ] تبديل نوع الخريطة (normal/satellite/terrain/hybrid).
- [ ] لغة/RTL.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «بحث بطيء» | N استعلامات + LIKE + ترقيم ذاكرة | `SearchController@search` |
| «ماركر مفقود» | بلا صورة/إحداثيات | `MapCubit._addMarkers` |
| «موقع المستخدم يختفي» | استبدال الماركرات | `setMarkersFromItems` (حفظ/استعادة) |
| «navigation خاطئ» | switch type | `search_view_body`/`ItemDetailsWidget` |
| «صور ماركر بطيئة» | http لكل ماركر | `_getCircularMarkerIconFromUrl` cache |
| «city لا تُحدَّث» | cascade region | `SearchFilterCubit.updateRegion` |
| «نتائج مكرّرة» | merge/pagination | `SearchController` forPage |

## 36. Debug Guide
1. راقب `GET global-search?...` (تحقّق viewAs/search/region/city).
2. تتبّع debounce → SearchFilterCubit → GetSearchGlobalCubit → PagingController.
3. للخريطة: راقب listener → setMarkersFromItems → _addMarkers (دفعات) + cache.
4. للأداء: عدّ استعلامات SearchController (DB::listen) + زمن LIKE.
5. للتنقّل: تأكّد switch(item.type) صحيح.
6. للموقع: تتبّع LocationCubit + أذونات Geolocator.

## 37. Search Keywords
`exploration`, `استكشاف`, `global-search`, `global_map_data`, `SearchController`, `SearchResource`, `SearchCollection`, `search`, `map`, `MapView`, `SearchView`, `GetSearchGlobalCubit`, `MapCubit`, `SearchFilterCubit`, `ItemGlodal`, `GlodalSearch`, `viewAs`, `setMarkersFromItems`, `addUserLocationMarker`, `_getCircularMarkerIconFromUrl`, `MultiSelectPillsBar`, `ItemDetailsWidget`, `places-data-for-map`, `whereTranslationLike`, `LengthAwarePaginator`, `applyDistanceSorting`, `google_maps_flutter`, `marker clustering`, `debounce search` → **08_Exploration.md**

## Web Frontend (Angular)
- **Different search contract:** web uses `GET /global-search?viewAs=events|places|zads|stores&region_id&city_id&search&page&per_page` (`home.service.ts:157–179`). Mobile uses `SearchController` multi-model aggregation + Tasneef type-switch ([[07_Tasneef]]). **Two divergent code paths for the same intent.**
- **Map:** web `global-map-data` (152–156) + `places/places-data-for-map` (138–150) via `@angular/google-maps`. Mobile builds markers in-app (LRU cache + batched fetch).
- **Components:** `QuickGlobalSearch` v1/v2/mobile, `SearchResultComponent`. See [[WEB_MOBILE_COMPARISON]] (recommend converging on one search contract).

## 38. Future Improvements
- UNION ALL + DB pagination بدل دمج الذاكرة.
- full-text/trigram + spatial index.
- marker clustering + server thumbnails.
- autocomplete/suggestions/saved searches.
- توحيد ItemGlodal↔UnifiedPlaceModel وخريطة type→config للتنقّل.
- عرض المسافة في bottom sheet.

---
**تدفق مرجعي:** UI(بحث/خريطة) → Cubit(Search/Map/Filter) → Repo → API(global-search/map-data) → SearchController(4 نماذج merge) → Models → DB → SearchResource → ItemGlodal → قائمة/ماركرات → navigate by type. **مُجمِّع بحث+خريطة يعيد استخدام كيانات المحتوى.**
