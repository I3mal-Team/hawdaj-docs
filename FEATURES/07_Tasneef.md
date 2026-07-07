# Feature 07 — Tasneef (تصنيف — Unified Listing Hub)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core (Aggregator)
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/tasneef) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 **Tasneef = مُجمِّع (Aggregator/Hub)**، لا كيان بذاته. يعرض 7 أنواع محتوى ([[04_Places]], [[05_Stores]], [[06_Restaurants]], events, swalefs/stories, guides, **applications**) عبر **cubit + repo + model واحد** يعيد استخدام endpoints الكيانات (index الخاصة بكل controller). الفريد الوحيد: نوع **applications**.

---

## 1. Feature Name
Tasneef (تصنيف) — واجهة قوائم موحّدة تعرض كل أنواع المحتوى بفلاتر مشتركة (فئة/منطقة/مدينة/لغة/بحث/تاريخ/موقع/top_rated) وترقيم لانهائي، وتوجّه لتفاصيل كل نوع.

## 2. Business Goal
مركز الاكتشاف/التصفّح: DRY — cubit/repo/model واحد يخدم 7 أنواع، يقلّل التكرار، ويوجّه المستخدم من "View All" (Home) إلى القوائم المصنّفة ثم للتفاصيل.

## 3. Business Rules
- كل الأنواع `active=1`.
- تبديل endpoint حسب type: places/stores/zads/events/guides/applications/swalefs.
- فلاتر مشتركة: page, per_page, search, region_id, city_id, category_id, lat/lng.
- فلاتر خاصة بالنوع: zads→food_categories[]/categories(join)؛ guides→language_id(JSON_CONTAINS)+top_rated(افتراضي)+excludedId؛ events→daterange/date_from/date_to/address_type؛ stores→is_online؛ places→top_featured/top_visited.
- الترتيب: بموقع→مسافة؛ top_visited→views_num DESC؛ top_rated→AVG(rate) DESC (guides)؛ events→CASE (جارية→قادمة→منتهية) ثم date_from؛ وإلا order_id/id.
- applications: قوائم فقط (بلا شاشة تفاصيل)، روابط ios/android.

## 4. User Flow
Home "View All" / TasneefView (hub بـ7 بطاقات) → قائمة موحّدة (PlacesCubit) بفلاتر → بطاقة → تفاصيل النوع (route حسب type: place→slug، zad→{slug,id}، guide→id...).

## 5. Flutter Screens
| Screen | File |
|---|---|
| TasneefView (hub، 7 بطاقات) | `presentation/views/tasneef_view.dart:11-82` |
| TasneefPlacesListView (عامة: places/stores/zads/events/guides) | `.../tasneef_places_list_view.dart:19-535` |
| TasneefAppsListView (GridView تطبيقات) | `.../tasneef_apps_list_view.dart:12-107` |
| TasneefTourGuidesListView (topRated افتراضي) | `.../tasneef_tour_guides_list_view.dart:14-334` |
| TasneefStoriesListView (GridView سوالف) | `.../tasneef_stories_list_view.dart:14-128` |
| TasneefSearchFilterWidget (فلاتر عامة) | `.../tasneef_search_filter_widget.dart` |

## 6. Widgets
`tasneef_card` (بطاقة نوع بالـhub)، `tasneef_places_item_card:13-100` (بطاقة موحّدة places/stores/zads)، `tasneef_apps_item_card` (روابط ios/android)، `tasneef_tour_guides_item_card`، `tasneef_stories_item_card`، `rating_widget`، فلاتر: `region_filter_widget`/`city_filter_widget`/`store_type_filter`/`zad_food_category`/`date_range_picker` + `widgets/filtering/*` (per-type).

## 7. Cubits / Blocs
**PlacesCubit** (`cubits/places_cubit/places_cubit.dart:7-338`) — الموحّد:
- `getItemsByType(type, page, perPage, categoryId, topFeatured, regionId, cityId, languageId, topRated, search, dateFrom, dateTo, lat, lng, foodCategories)` — يبدّل endpoint حسب type.
- wrappers: getPlaces/getStores/getZads/getEvents/getStories/getApps/getGuides.
- `loadMoreItems()` (ترقيم), `refreshItems()`.
- States (`places_state.dart:1-81`): Initial/Loading/**LoadingMore**/Success(places,currentPage,totalPages,total,hasNextPage)/Error.

## 8. State Flow
`getItemsByType` → إن page==1: Loading؛ page>1: LoadingMore → repo.getItems → fold → Success (إلحاق `_allPlaces` إن page>1) / Error. refresh يمسح ويعيد لـ page 1. تمرير قرب النهاية (200px) → loadMoreItems(page+1).

## 9. Models
- **UnifiedResponseModel** (`unified_response_model.dart:4-33`): يغلّف `LaravelPaginationModel<UnifiedPlaceModel>` (code/message/data).
- **UnifiedPlaceModel** (`unified_place_model.dart:12-646`): نموذج واحد لكل الأنواع؛ حواسّ نوع (`isPlace/isStore/isEvent/isStory/isApp/isGuide`, :231-236)، مساعِدات `locationText`(:266-304)/`priceText`(:307-330) تختلف بالنوع، حقول خاصة (iosLink, androidLink, experience, regions, languages, audioStoryLink...).
- `LaravelPaginationModel` (عام)، CategoryModel/RegionModel/CityModel/PriceModel/GalleryModel/RatingModel، `FilterModel:14-156` (key/name/widget + factories).

## 10. Repositories
- **Contract** `domain/repositories/tasneef_repository.dart:7-39`: `getItems(endpoint, ...16 param..., type) → Either<Failure,UnifiedResponseModel>`.
- **Impl** `data/repositories/tasneef_repository_impl.dart:9-68`: `getItems` + **`_buildQueryParams(:70-133)`** type-aware (zads→`categories`؛ غيره→`category_id`؛ +region/city/lat/lng/top_rated/top_featured/date_from/date_to/is_online/food_categories[]/address_type).

## 11. Services
لا service. يعيد استخدام مساعِدات App.php (distance/listing) عبر كل controller.

## 12. API Endpoints (يعيد استخدامها من الكيانات)
| type | Endpoint | Controller |
|---|---|---|
| places | GET `places` | PlaceController@index |
| stores | GET `stores` | StoreController@index |
| zads | GET `zads` | ZadController@index |
| events | GET `events` | EventController@index |
| guides | GET `guides` | GuideController@index |
| stories | GET `swalefs` | SwalefController@index |
| **applications** | GET `applications` | **ApplicationsController@index** (فريد Tasneef) |

## 13. Request Models
لا body. query params موحّدة تُبنى في `_buildQueryParams` حسب النوع (§3/§7).

## 14. Response Models
غلاف موحّد `{code,message,data:{current_page,items[],last_page,per_page,total}}` — لكن كل نوع يستخدم Collection الخاص به (PlaceCollection/StoreCollection/ZadCollection/EventCollection/GuideCollection/ApplicationCollection). Flutter يوحّدها في `UnifiedPlaceModel`.

## 15. Laravel Routes
`routes/api.php`: `GET places/stores/zads/events/guides/swalefs/applications` — كلها index الخاصة بالـcontroller. `applications → ApplicationsController@index`. عامة.

## 16. Controllers
يعيد استخدام index المُوثّقة: PlaceController (:26-115)، StoreController (:35-116)، ZadController (:14-75)، EventController (:12-91، ترتيب CASE زمني)، GuideController (:17-69، top_rated AVG + language JSON_CONTAINS + excludedId). **ApplicationsController@index** (:12-37): active + search + category(JSON_CONTAINS) + `applyListingOrder` + paginate → ApplicationCollection.

## 17. Service Classes (Laravel)
لا service. النمط المشترك: `where(active,1)` + eager (حيث ينطبق) + فلاتر + distance/listing + paginate + Collection.

## 18. Models (Laravel)
يعيد استخدام Place/Store/ZadElgadel/Event/Guide/Swalef. **الفريد: Application** (`app/Models/Application.php:16-56`): translatable(title,description)، fillable(image,type,link,ios_link,android_link,active,categories(JSON),show_in_home,order_id)، `allCategories()` (CategoryOfApplication)، ratings(type='app').

## 19. Database Tables
يعيد استخدام جداول الكيانات (places/stores/zad_elgadels/events/guides/swalefs +translations). **الفريد:** `applications` (id, image, type, link, ios_link, android_link, active, categories(JSON), show_in_home, order_id) + `category_of_applications`(+translations).

## 20. Relationships
كل نوع بعلاقاته المُوثّقة. Application: categories عبر JSON (allCategories→CategoryOfApplication)، ratings(type='app'). لا تفاصيل/مفضّل لـapplications (قوائم فقط).

## 21. Validation Rules
PlaceController يستخدم `IndexRequest`؛ البقية Request عادي (متساهل). لا FormRequest صارم لمعظم الأنواع.

## 22. Authorization & Permissions
عام بالكامل. is_favorite/is_saved per-user (حيث ينطبق: places/stores/zads/events/guides/swalefs). applications بلا مفضّل.

## 23. Error Handling
- **Flutter:** PlacesError؛ Loading/LoadingMore/empty states في القوائم.
- **Laravel:** كل controller يعيد Collection عبر ApiModalController؛ فلاتر متساهلة.

## 24. Edge Cases
- نوع غير معروف في switch → سلوك غير محدّد (يجب معالجته).
- قائمة فارغة → empty indicator.
- applications بلا تفاصيل → النقر يفتح ios/android link.
- events: ترتيب زمني CASE (جارية/قادمة/منتهية).
- guides: top_rated افتراضي + excludedId (استبعاد النفس).
- ترقيم: hasNextPage = currentPage < totalPages.
- بلا موقع → order_id/id.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, get_it. Laravel: كل تبعيات الكيانات (translatable/media). لا جديد عدا Application/CategoryOfApplication.

## 26. Files Involved
**Flutter:** `lib/features/tasneef/**` (views: tasneef_view/places_list/apps_list/tour_guides_list/stories_list؛ cubits/places_cubit/*؛ data/models/{unified_response,unified_place,laravel_pagination,filter_model,...}؛ data/repositories/tasneef_repository_impl؛ domain/repositories/tasneef_repository؛ widgets + filtering)، `end_points.dart`.
**Laravel:** controllers الكيانات + `app/Http/Controllers/Api/ApplicationsController.php`، `app/Models/Application.php` (+CategoryOfApplication)، `app/Http/Resources/Applications/{ApplicationResource,ApplicationCollection}.php`، Collections الكيانات، `routes/api.php`.

## 27. Complete Execution Flow (places بفلاتر)
```
TasneefPlacesListView(type='places',categoryId,regionId) → PlacesCubit.getItemsByType
  → switch type → endpoint='places' → emit(Loading)
  → TasneefRepositoryImpl.getItems → _buildQueryParams → GET places?page&per_page&category_id&region_id
  → PlaceController@index (active=1 + eager + JSON_CONTAINS category + region + distance/listing + paginate)
  → PlaceCollection → {code,data:{items,...}}
  → UnifiedResponseModel → List<UnifiedPlaceModel>
  → emit(PlacesSuccess) → ListView(TasneefPlacesItemCard) → tap → push(route by type, slug/id)
  → scroll → loadMoreItems(page+1) → LoadingMore → append.
(applications: نفس التدفق، endpoint='applications' → ApplicationsController@index → ApplicationCollection؛ النقر يفتح ios/android link بلا تفاصيل.)
```

## 28. Performance Considerations
- 🟠 يرث N+1 من controllers الكيانات (allCategories/ceo).
- ✅ ترقيم على الكل؛ eager loading حيث ينطبق.
- 🟠 UnifiedPlaceModel ضخم (646 سطر) — كلفة parsing لكل عنصر.
- 🟡 فلترة نطاق الإحداثيات (whereBetween) في places index قد تحتاج فهرسة.

## 29. Security Considerations
- عام بالكامل (مقبول لقوائم اكتشاف).
- يرث تسريب email من ratings (لكن القوائم غالبًا لا تُظهر التقييمات الكاملة — التفاصيل تفعل).
- يرث كشف active في `show` (لا يخصّ القوائم — القوائم تفلتر active).
- 🟡 JSON_CONTAINS بمدخل category_id غير مُتحقّق (بناء SQL بالتسلسل — يفضّل binding).

## 30. Technical Debt
- switch type منتشر (cubit + repo + navigation) — إضافة نوع تتطلب تعديل 3 مواضع.
- UnifiedPlaceModel يخدم 7 أنواع (حقول nullable كثيرة، تعقيد).
- عدم اتساق endpoint stories='swalefs' مقابل نوع 'stories'.
- بناء JSON_CONTAINS بالتسلسل (لا binding) عبر عدة controllers.
- فلاتر متساهلة (لا FormRequest موحّد).

## 31. Improvement Opportunities
- توحيد فلاتر عبر FormRequest مشترك.
- binding آمن لـ JSON_CONTAINS.
- تجزئة UnifiedPlaceModel أو استخدام sealed types.
- توحيد switch عبر خريطة type→config واحدة.
- كاش للفئات/المناطق (نادرة التغيّر) لتخفيف الفلاتر.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 04/05/06/09/10/11 | Tasneef يعرض قوائمها (يعيد استخدام index) |
| 02 Home | "View All" → قوائم Tasneef |
| 08 Exploration | يشارك الفلاتر/البحث/الموقع |
| 13 Favorites / 14 Rates | زر مفضّل في بطاقات القوائم |
| 18 Taxonomy | كل الفلاتر (region/city/category/language/food) |
| applications (فريد) | لا feature منفصل — يعيش داخل Tasneef |
> تغيير UnifiedPlaceModel أو `_buildQueryParams` يؤثّر على **كل** القوائم و7 أنواع.

## 33. How to Modify This Feature
- **إضافة نوع جديد:** endpoint Laravel + توسيع UnifiedPlaceModel (حقول + isType) + case في cubit switch + case في navigation + `_buildQueryParams`.
- **فلتر جديد:** FilterModel + widget + `_buildQueryParams` + controller.
- **APIs:** places/stores/zads/events/guides/swalefs/applications.
- **المخاطر:** switch منتشر (3 مواضع)، تعقيد UnifiedPlaceModel، إرث N+1/email من الكيانات، عدم اتساق stories/swalefs.
- **اختبارات:** كل نوع (قائمة+فلاتر+ترقيم)، تبديل endpoint، navigation صحيح بالنوع، applications (روابط).

## 34. Regression Checklist
- [ ] كل نوع يُحمّل قائمته الصحيحة (places/stores/zads/events/guides/stories/apps).
- [ ] فلاتر مشتركة (search/region/city/category) تعمل عبر الأنواع.
- [ ] فلاتر خاصة: food_categories(zads)، is_online(stores)، daterange(events)، language/top_rated(guides).
- [ ] ترقيم لانهائي + LoadingMore + آخر صفحة.
- [ ] refresh يعيد لـ page 1.
- [ ] navigation صحيح لكل نوع (slug/id/{slug,id}).
- [ ] applications: بطاقة تفتح ios/android link.
- [ ] empty/error states.
- [ ] لغة/RTL.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «نوع خاطئ يُعرَض» | switch type/endpoint | `places_cubit.getItemsByType` |
| «فلتر لا يمرَّر» | _buildQueryParams type-aware | `tasneef_repository_impl:70-133` |
| «navigation خاطئ» | switch navigation | `tasneef_places_list_view:401-436` |
| «zad فئات لا تفلتر» | categories vs category_id | `_buildQueryParams` + ZadController join |
| «ترقيم مكرّر/ناقص» | hasNextPage/append | `places_cubit` loadMore |
| «applications بلا تفاصيل» | مقصود (قوائم فقط) | ApplicationsController |
| «stories endpoint» | type 'stories'→'swalefs' | endpoint mapping |

## 36. Debug Guide
1. راقب طلب `GET {endpoint}?params` (تحقّق endpoint الصحيح للنوع).
2. تتبّع PlacesCubit (Loading/LoadingMore/Success) + type الممرّر.
3. افحص `_buildQueryParams` للنوع (zads→categories، غيره→category_id).
4. تحقّق UnifiedResponseModel parsing + type لكل عنصر.
5. للتنقّل: تأكّد switch(item.type) يعطي route+extra صحيح.
6. جانب الخادم: تأكّد الـcontroller المناسب + فلاتره + Collection.

## 37. Search Keywords
`tasneef`, `تصنيف`, `unified listing`, `PlacesCubit`, `getItemsByType`, `TasneefRepository`, `TasneefRepositoryImpl`, `_buildQueryParams`, `UnifiedPlaceModel`, `UnifiedResponseModel`, `LaravelPaginationModel`, `FilterModel`, `TasneefView`, `TasneefPlacesListView`, `TasneefAppsListView`, `TasneefTourGuidesListView`, `TasneefStoriesListView`, `tasneef_places_item_card`, `applications`, `ApplicationsController`, `Application model`, `ios_link`, `android_link`, `CategoryOfApplication`, `top_rated`, `top_featured`, `top_visited`, `food_categories`, `language_id`, `daterange`, `is_online`, `type switch`, `aggregator`, `hub`, `endpoint switching` → **07_Tasneef.md**

## 38. Future Improvements
- خريطة type→config واحدة (endpoint/route/filters) لإزالة switch المنتشر.
- FormRequest موحّد + binding آمن لـJSON_CONTAINS.
- تجزئة UnifiedPlaceModel (sealed/union types).
- توحيد تسمية stories/swalefs.
- كاش الفلاتر الثابتة؛ معالجة النوع غير المعروف.
- تطبيق إصلاحات الكيانات (email/N+1) تنعكس تلقائيًا هنا.

---
**تدفق مرجعي:** UI(type) → PlacesCubit(switch endpoint) → TasneefRepo(_buildQueryParams) → API(entity index) → Controller → Model → DB → Collection → UnifiedPlaceModel → UI → navigate by type. **Tasneef = aggregator يعيد استخدام [[04_Places]]/[[05_Stores]]/[[06_Restaurants]]/events/guides/swalefs + نوع applications الفريد.**
