# Feature 04 — Places

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> ⚠️ **نمط مرجعي:** Places هو القالب الأساسي لكيانات المحتوى (Stores/Zad/Events/Swalef/Guides تتبع نفس البنية: translatable + media + gallery + rates + favorites/saved + distance sorting + slug detail). فهمه يختصر تحليل الميزات 05/06/09/10/11.

---

## 1. Feature Name
Places — قائمة الأماكن السياحية (بحث/فلاتر/ترتيب جغرافي/ترقيم) + شاشة تفاصيل بالـ slug (معرض، تقييم، خريطة، روابط، تقييمات، مفضّل/محفوظ/تقييم).

## 2. Business Goal
عرض واكتشاف الأماكن السياحية بتفاصيل غنية، وربطها بالمفضّلة/المحفوظات/التقييمات، وتغذية شاشة Home والرحلات. مصدر عناصر الرحلة الأساسي.

## 3. Business Rules
- القائمة تعرض `active=1` فقط (المُقدَّم من المستخدم يبدأ `active=0` بانتظار اعتماد الأدمن، `show_in_home=0`, `address_type='map'`).
- الترتيب: بموقع → مسافة Haversine ثم order_id/id؛ `top_visited` → views_num DESC؛ وإلا order_id/id.
- الفلاتر: search (whereTranslationLike على title)، city_id، region_id، category_id (JSON_CONTAINS، متعدّد بالـ explode)، price_id، temperature، حدود خريطة (lat/lng ± x/y).
- `rate` = متوسط التقييمات، `review` = عددها (accessors).
- `is_favorite`/`is_saved` per-user (تتطلب توكن).
- seasons مخزّنة نصًّا مفصولًا بفواصل ← تُحوَّل لمصفوفة/تُترجَم.
- related_places / near_stores مخزّنة IDs (JSON)؛ related عبر endpoint منفصل.

## 4. User Flow
Home/Tasneef/Search → بطاقة مكان (tap) → `push(kPlacesDetailsItems, extra: slug)` → PlacesDetailsItems → عرض التفاصيل → تفاعلات (مفضّل/حفظ/مشاركة/خريطة/اتصال/واتساب/تذكرة/تقييم/رؤية كل التقييمات).

## 5. Flutter Screens
| Screen | File |
|---|---|
| PlacesDetailsItems (scaffold + BlocBuilder) | `lib/features/places/presentation/view/places_details_items.dart` |
| PlacesDetailsItemsBody (UI الكامل، 359س) | `.../view/widgets/places_details_items_body.dart` |
| MapSection (خريطة مضمّنة + اتجاهات) | `.../view/widgets/map_section.dart` |
> القائمة تُعرَض عبر شاشات Tasneef (Feature 07)، لا شاشة قائمة مخصّصة داخل places.

## 6. Widgets
`CustomAppBarHomeDetails` (معرض + أزرار مفضّل/حفظ/مشاركة/موقع، `:91-146`)، `DetailsSection` (عنوان/تقييم/عنوان، `:149`)، `PropertiesGrid` (فئة/سعر/مواسم/تذاكر، `:158-197`)، روابط اجتماعية (`:200-243`)، `MapSection` (`:245`)، كاروسيل تقييمات (`:254-317`)، `ShareYourRateButton` (`:319`)، `BookTicketButtomWidget` (`:330`).

## 7. Cubits / Blocs
- **PlaceDetailsCubit** (`place_details_cubit.dart:1-22`): يستقبل `(placesRepo, slug)`، `getPlaceInfo()` → Loading→Success(UnifiedPlaceModel)/Error. States في `place_details_state.dart`.
- **PlacesCubit** (tasneef، `getItemsByType(type='places')`): قائمة مصفّحة بفلاتر، states Loading/LoadingMore/Success(places,page,total,hasNext)/Error، pagination في الذاكرة.
- تكامل: MultiBlocListener على FavoriteCubit/SavedCubit/RateCubit → أي نجاح يستدعي `getPlaceInfo()` لإعادة التحميل (`:61-89`).

## 8. State Flow
تفاصيل: constructor → `getPlaceInfo()` فورًا → emit(Loading) → repo.fetchPlace(slug) → fold → Success/Error. القائمة: getItemsByType → Loading/LoadingMore → repo.getItems → Success بإلحاق الصفحات. المفضّل/الحفظ/التقييم يُحدّثان التفاصيل عبر listeners.

## 9. Models
- **PlaceModel** (`lib/features/home/data/model/places_model/place_model.dart`, 249س): id, slug, type, address, image, coverImage, rate, review, ratings[], روابط اجتماعية، temperature, seasons[], categories[], galleries[], price, city, region, addressType, viewsNum, featured, lat, long, visited, distance, title, description, meta, isFavorite, isSaved.
- **UnifiedPlaceModel** (647س): يوسّع PlaceModel لكل الكيانات (places/zads/events/stories/apps/guides) — relatedPlaces, nearStores, keyWords, prefered, showInHome, social, إلخ.
- **GalleryModel**, **PriceModel**, **MetaModel** (SEO), **PlaceResponse** (currentPage, items[], total).

## 10. Repositories
- **Flutter** `places_repo.dart` + `places_repo_imp.dart:9-44`: `fetchPlace(slug)` → GET `places/{slug}` → UnifiedPlaceModel. القائمة عبر `TasneefRepositoryImpl.getItems(endpoint='places', queryParams)`.
- **Laravel:** لا repository pattern — منطق في `PlaceController`.

## 11. Services
لا service مخصّص. يعيد استخدام مساعِدات `app/Helper/App.php`: `getLocationCoords` (:877-946)، `applyDistanceSorting` (:960-970)، `applyListingOrder`، `isFavorite`/`isSaved` (:677-718).

## 12. API Endpoints
| Method | Endpoint | Auth |
|---|---|---|
| GET | `places` (قائمة + فلاتر + ترقيم) | عام (توكن اختياري لـ is_favorite/is_saved) |
| GET | `places/{slug}` (تفاصيل) | عام |
| GET | `places/places-data-for-map` | عام |
| GET | `places/{place}/related-places` | عام |
| POST | `properties` (إنشاء مكان مستخدم) | مصادقة (Feature 15) |

## 13. Request Models
لا body للقراءة. query params (IndexRequest): search, city_id(exists), region_id(exists), category_id, sub_category_id, price_id(exists), temperature, lat, lng, x, y, page, per_page, top_visited, top_featured.

## 14. Response Models
- قائمة: `PlaceCollection` → `{current_page, items[PlaceListResource], total, last_page, next_page_url, ...}`.
- تفاصيل: `PlaceResource` → كل الحقول + categories(allCategories) + galleries + city/region/price + ratings(خام) + meta(Seo) + is_favorite/is_saved + distance. الغلاف `{code,message,data}`.

## 15. Laravel Routes
`routes/api.php`: `GET places → index` · `GET places/places-data-for-map → placesDataForMap` · `GET places/{place}/related-places → related_places` · `GET places/{place} → show`. عامة (بلا middleware مصادقة).

## 16. Controllers
**PlaceController** (`app/Http/Controllers/Api/PlaceController.php`): `index` (:26-115، فلاتر+ترتيب+ترقيم، eager `ratings,galleries,city,region,ceo`)، `show` (:147-151، `where(slug)->firstOrFail`)، `related_places` (:123-139)، `placesDataForMap` (:153-176، أماكن بإحداثيات)، `store` (:178-247، إنشاء بـ active=0).

## 17. Service Classes (Laravel)
لا service class. المنطق في الـ controller + مساعِدات App.php (§11).

## 18. Models (Laravel)
**Place** (`app/Models/Place.php`): translatable (`title,description`، `$with=[translations]`)، relations: region/city/price/user(withDefault)، galleries (hasMany parent_id, type='places')، ratings (hasMany parent_id, type='place')، ceo (hasOne)، favorites (morphMany). accessors: rate/review/type('place')/seasons/seasonsTra/lat/long/categories، `allCategories()`. SoftDeletes. casts: categories/related_places/near_stores/key_words → array.

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `places` | id, categories(JSON), address, image, city_id/region_id/price_id(FK), lat/long, address_type(enum link/map/latlong), active, temperature, seasons(نص), views_num, visited, related_places(JSON), near_stores(JSON), featured, ticket_link/website_link/instagram/whatsapp/facebook/x_link, key_words(JSON), prefered, place_icon, user_id, show_in_home, ownership_proof_file, timestamps, deleted_at |
| `place_translations` | id, place_id(FK cascade), locale, title, description، unique(place_id,locale) |
فهارس: active, featured, views_num, order_id, city_id, region_id.

## 20. Relationships
Place BelongsTo Region/City/Price/User · HasMany Gallery(type=places)/Rate(type=place) · HasOne Ceo · MorphMany Favorite · translations HasMany. related_places/near_stores كـ IDs (بلا FK، تُحمَّل بـ whereIn).

## 21. Validation Rules
`IndexRequest` (Places): كلها `sometimes`؛ city_id/region_id integer|exists، price_id exists؛ category_id/lat/lng/x/y/temperature بلا قيود صارمة. إنشاء (`store`): title unique، description، image، categories، region/city/price، seasons، lat/long.

## 22. Authorization & Permissions
القراءة عامة بالكامل. is_favorite/is_saved تعتمد المستخدم المصادق (وإلا false). الإنشاء يتطلب توكن ويبدأ `active=0` (اعتماد أدمن). ⚠️ `show($slug)` لا يفلتر active → مكان غير منشور قابل للعرض إن عُرف الـ slug.

## 23. Error Handling
- **Flutter:** `Either<Failure,T>`؛ PlaceDetailsError يعرض رسالة + إعادة.
- **Laravel:** `firstOrFail` → 404؛ فلاتر متساهلة. الغلاف عبر ApiModalController.

## 24. Edge Cases
- مكان غير موجود → 404.
- بلا موقع → ترتيب order_id/id.
- related_places/near_stores فارغة → قوائم فارغة.
- ⚠️ `show` لا يفلتر active → كشف غير المنشور بالـ slug.
- JSON categories تالف → قد يفشل JSON_CONTAINS بصمت.
- ترجمة ناقصة → fallback.
- near_stores تُعاد كـ IDs فقط (لا كائنات) في التفاصيل.
- ratings تُعاد **خام** (غير مُحوّلة عبر Resource) — تسرّب email المقيّم.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, google_maps (MapSection)، url_launcher، share، cached_network_image، go_router. Laravel: astrotomic/translatable، spatie/medialibrary، JSON_CONTAINS (MySQL 5.7+)، Cache (الموقع).

## 26. Files Involved
**Flutter:** `lib/features/places/**`، `lib/features/home/data/model/places_model/*` (PlaceModel/UnifiedPlaceModel/Gallery/Price/Meta)، `lib/features/tasneef/**` (القائمة)، `end_points.dart` (places, detailsPlaces).
**Laravel:** `app/Http/Controllers/Api/PlaceController.php`، `app/Models/Place.php` + PlaceTranslation، `app/Http/Resources/Places/{PlaceResource,PlaceListResource,PlaceCollection}.php`، `app/Http/Requests/Places/IndexRequest.php`، `app/Helper/App.php`، migrations places/place_translations، `routes/api.php`.

## 27. Complete Execution Flow
**القائمة:** PlacesCubit.getPlaces → repo.getItems → GET `places?filters` → PlaceController@index (active=1 + eager + فلاتر + distance/listing + paginate) → PlaceCollection → UnifiedResponseModel → PlacesSuccess → ListView → tap → push(slug).
**التفاصيل:** push(kPlacesDetailsItems, slug) → PlaceDetailsCubit(slug)..getPlaceInfo → GET `places/{slug}` → PlaceController@show (firstOrFail) → PlaceResource (accessors rate/review + is_favorite/is_saved + galleries/ratings/meta) → UnifiedPlaceModel → PlaceDetailsSuccess → PlacesDetailsItemsBody (معرض/تفاصيل/شبكة/اجتماعي/خريطة/تقييمات/CTA).

## 28. Performance Considerations
- ✅ eager loading في index يمنع N+1 للعلاقات الأساسية.
- 🟠 `allCategories()` يستعلم categories لكل مكان (N+1 محتمل في القوائم) — يفضّل استخدام JSON مباشرة.
- 🟠 `ceo` eager-load في القوائم رغم عدم استخدامه غالبًا.
- 🟠 ratings تُحمَّل كاملة في التفاصيل (قد تكون كبيرة) — لا حد/ترقيم.
- ✅ الموقع مخزّن cache (5د). فهارس على أعمدة الترتيب/الفلترة.

## 29. Security Considerations
1. 🟠 **`show` لا يفلتر active** → كشف أماكن غير منشورة بالـ slug.
2. 🟠 **ratings خام في PlaceResource** → تسريب `email` المقيّمين (خصوصية).
3. 🟡 القراءة عامة (مقبول لمحتوى تسويقي).
4. 🟡 JSON_CONTAINS مع مدخل غير مُتحقّق (category_id) — منخفض لكنه غير محكم.
5. ✅ is_favorite/is_saved per-user، الإنشاء يتطلب اعتمادًا.

## 30. Technical Debt
- منطق category/sub_category مكرّر في index.
- `UnifiedPlaceModel` ضخم (647س) يخدم 6 أنواع — تعقيد.
- seasons نصّ مفصول بفواصل بدل جدول/JSON.
- ratings خام (غير Resource) — يجب RatingResource يخفي email.
- `image` (legacy) موجود مع MediaLibrary (ازدواج).
- لا شاشة قائمة places مخصّصة (تعتمد Tasneef).

## 31. Improvement Opportunities
- فلترة active في `show` (أو 404 لغير المنشور).
- `RatingResource` يخفي email ويحدّد/يرقّم التقييمات.
- إزالة `ceo`/`allCategories` N+1 من القوائم؛ استخدام JSON categories.
- توحيد الوسائط على MediaLibrary وإزالة عمود image.
- فهرسة JSON categories (generated column) للأداء.
- توثيق address_type والمواسم كثوابت.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 02 Home | يعرض places (topVisited/distinctive) |
| 03 Trips | Place = عنصر الرحلة الأساسي (Place model/Resource) |
| 05/06/09/10/11 | تتبع نفس نمط Place (translatable+media+gallery+rate+favorite+distance) |
| 07 Tasneef | قائمة places عبر getItemsByType |
| 08 Exploration | places-data-for-map + بحث + موقع |
| 13 Favorites / 14 Rates | مفضّل/حفظ/تقييم المكان (morph/pseudo-morph) |
| 15 My Properties | إنشاء مكان مستخدم (store، active=0) |
| 18 Taxonomy | region/city/price/category |
> تغيير Place/PlaceResource يؤثّر على Home والرحلات والمفضّلة وTasneef.

## 33. How to Modify This Feature
- **حقل جديد:** migration places + Place casts/accessor + PlaceResource + PlaceModel/UnifiedPlaceModel (fromJson) + widget عرض.
- **فلتر قائمة جديد:** `PlaceController@index` + `IndexRequest` + بناء query params في `TasneefRepositoryImpl`.
- **قسم تفاصيل جديد:** `places_details_items_body.dart` + PlaceResource.
- **APIs:** places, places/{slug}, related-places, places-data-for-map.
- **المخاطر:** N+1 (allCategories/ceo)، عدم تطابق مفاتيح JSON، تسريب email، كشف active، تأثير على Home/Trips (نموذج مشترك).
- **اختبارات:** قائمة بفلاتر/موقع/ترقيم، تفاصيل بالـ slug، 404، مفضّل/حفظ/تقييم refresh، related/near فارغة.

## 34. Regression Checklist
- [ ] قائمة places: فلاتر (region/city/category/price/search) + ترقيم + LoadingMore.
- [ ] ترتيب: بموقع (مسافة) وبلا موقع (order_id) و top_visited.
- [ ] تفاصيل بالـ slug: معرض/عنوان/تقييم/شبكة/اجتماعي/خريطة/تقييمات.
- [ ] 404 لمكان غير موجود.
- [ ] مفضّل/حفظ/تقييم → إعادة تحميل التفاصيل.
- [ ] أزرار: خريطة/اتجاهات/واتساب/انستغرام/تذكرة تفتح صحيحًا.
- [ ] related-places / near_stores فارغة تُدار بلا خطأ.
- [ ] لغة/RTL + ترجمة العنوان/الوصف.
- [ ] is_favorite/is_saved صحيحة للمستخدم (وfalse للضيف).

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «تفاصيل فارغة/خطأ» | slug خاطئ/غير موجود | `PlaceController@show` firstOrFail |
| «حالة مفضّل خاطئة» | is_favorite per-user + توكن | `PlaceResource` isFavorite + FavoriteCubit listener |
| «ترتيب غير جغرافي» | لا موقع/params | getLocationCoords + applyDistanceSorting |
| «بطء القائمة» | allCategories/ceo N+1 | `Place::allCategories` + eager ceo |
| «email المقيّم ظاهر» | ratings خام | `PlaceResource` ratings |
| «الصورة لا تظهر» | media vs image legacy | getImageWithFallback + AppEnvironmentManager |
| «فلتر الفئة لا يعمل» | JSON_CONTAINS/explode | `PlaceController@index:51-58` |
| «مكان غير منشور ظاهر» | show بلا active | `PlaceController@show` |

## 36. Debug Guide
1. راقب `GET places?...` و`GET places/{slug}` (PrettyDioLogger/Chucker).
2. تحقّق envelope `{code,data}` وحقول is_favorite/distance.
3. تتبّع PlaceDetailsCubit (Loading→Success/Error) وslug الممرّر.
4. للفلاتر: افحص query params المبنية في TasneefRepositoryImpl + SQL على الخادم.
5. للترتيب: تأكّد lat/lng مُرسَلة + Haversine.
6. للتقييمات/المفضّل: تتبّع Rate/Favorite/SavedCubit → listener → getPlaceInfo.
7. جانب الخادم: DB::listen لرصد N+1 (allCategories/ceo).

## 37. Search Keywords
`place`, `places`, `PlaceController`, `PlaceResource`, `PlaceListResource`, `PlaceCollection`, `PlaceModel`, `UnifiedPlaceModel`, `PlaceDetailsCubit`, `places/{slug}`, `detailsPlaces`, `places-data-for-map`, `related_places`, `near_stores`, `whereTranslationLike`, `JSON_CONTAINS categories`, `applyDistanceSorting`, `getLocationCoords`, `isFavorite`, `isSaved`, `seasons`, `address_type`, `views_num`, `featured`, `prefered`, `place_translations`, `MapSection`, `places_details_items_body`, `IndexRequest`, `firstOrFail`, `GalleryModel`, `MetaModel SEO`, `ceo` → **04_Places.md**

## Web Frontend (Angular)
- **Same core API:** `/places`, `/places/{id}`, `/places/{id}/related-places`, taxonomy (regions/cities/categories/sub-categories/prices/languages), `/places/favorite`, `/rates` (`places.service.ts`). Components `PlacesListV2`/`PlaceDetailsV2`.
- **WEB-ONLY AI ChatGPT:** `getChatGpt`/`addChatGpt` on Place detail (`places.service.ts:111–115`, `place-details.component.ts:448–460`) — see [[32_AiAssistant]]. Mobile has none.
- **WEB map data:** `/places/places-data-for-map` (Google Maps) — mobile uses in-app markers from search.
- **Filters:** region/city/category/subcategory/price/top-visited/top-featured (same intent as mobile). See [[WEB_MOBILE_COMPARISON]].

## 38. Future Improvements
- فلترة active في show + RatingResource يخفي email.
- إزالة N+1 (allCategories/ceo)، فهرسة JSON.
- توحيد الوسائط على MediaLibrary.
- شاشة قائمة places مخصّصة أو تحسين Tasneef.
- تحويل seasons لجدول/JSON مُهيكل.
- ترقيم التقييمات في التفاصيل، وتخزين distance مُحسوب.
- تطبيق النمط الموحّد لتقليل ازدواج UnifiedPlaceModel عبر الأنواع.

---
**تدفق مرجعي:** UI → Cubit → Repo → API → Route → Controller → Model(Place) → DB(places/place_translations) → Response → Flutter UI ✅ موثّق لـ list + detail. **قالب لكيانات المحتوى 05/06/09/10/11.**
