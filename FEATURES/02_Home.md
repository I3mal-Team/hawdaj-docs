# Feature 02 — Home & Sliders (Discovery)

> **Status:** ✅ Analyzed (100%) · **Priority:** P0 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

---

## 1. Feature Name
Home & Sliders (Discovery) — الشاشة الرئيسية بعد الدخول: بانرات إعلانية (Sliders) + أقسام اكتشاف مجمّعة (أماكن، مطاعم، متاجر، فعاليات، سوالف، أدلّة، تطبيقات) في نداء واحد `home`.

## 2. Business Goal
واجهة الهبوط الأساسية التي تعرض أفضل المحتوى (المميّز/الأكثر زيارة) وتوجّه المستخدم بسرعة نحو التفاصيل أو القوائم المصنّفة (Tasneef) أو تخطيط رحلة، مع دعم ترتيب حسب القرب الجغرافي.

## 3. Business Rules
- المحتوى عام (بلا مصادقة)؛ نفس البيانات لكل المستخدمين (cache عام 60 دقيقة).
- الترتيب: إن توفّر موقع (lat/lng) → ترتيب بالمسافة (Haversine)، وإلا → `order_id` ثم `id` (وبعض الأقسام `views_num`).
- الفلاتر: `active=1` لكل المحتوى، `prefered=1` للأماكن المفضّلة، `show_in_home=1` للأدلّة/السوالف، الفعاليات `date_to > today` فقط.
- Sliders: `active=1` مرتّبة `order_id,id` (بلا cache على مستوى الـ controller).
- حدود لكل قسم عبر query params (places=14، topVisited=16، events=7، zad=6، stores=10…).
- الترجمة تتبع locale الطلب (Astrotomic fallback ar↔en).

## 4. User Flow
Splash → Home. عند الدخول: HomeCubit + SlidersCubit يجلبان بالتوازي، وLocationCubit يطلب إذن الموقع. المستخدم يمرّر عموديًا عبر الأقسام، يضغط بطاقة → تفاصيل، أو "View All" → قائمة Tasneef، أو بانر → فتح رابط خارجي، أو بطاقة الرحلة → StartTrip. Pull-to-refresh يعيد الجلب.

## 5. Flutter Screens
| Screen | File |
|---|---|
| HomeView (entry، يطلب الموقع في initState) | `lib/features/home/presentation/view/home_view.dart:1-25` |
| HomeViewBody (CustomScrollView + slivers) | `.../view/widgets/shared/home_view_body.dart` |
| HomeDetailsView (تفاصيل مكان) | `.../view/home_details_view.dart` |

## 6. Widgets
> **تنظيم (2026-07-07):** widgets Home اتقسّمت من مجلد واحد مسطّح لمجلدات فرعية بالمسؤولية تحت `.../view/widgets/`: `app_bar/` · `applications/` · `explore_category/` · `guides/` · `offers/` · `properties/` · `sliders/` · `suggestions/` · `trip_card/` · `shared/`. الأسماء ثابتة، بس المسارات اتغيّرت (import paths اتحدّثت بكل المستهلكين). المسارات أدناه نسبية للمجلد الفرعي المناسب.

CustomScrollView مع أقسام Sliver بالترتيب (`shared/home_view_body.dart`):
1. `CustomAppBarHome` (SliverAppBar pinned 240h، بحث + خلفية متدرّجة) — `app_bar/custom_app_bar_home.dart` + `app_bar/app_bar_*.dart`
2. `HomeSlidersSection` (CarouselSlider 180h، autoPlay 4s، `AnimatedSmoothIndicator`) — `home_sliders_section.dart:1-152`
3. `ExploreByCategorySection` (فئات — **dummy hardcoded**)
4. `DailySuggestionsSection` (ListView أفقي، بطاقات topVisitedPlaces) — `daily_suggestions_section.dart:11-56`
5. `ApplicationsSection` (بطاقات تطبيقات، LayoutBuilder)
6. `TripCardWidget` (بطاقة ترويجية ثابتة → StartTrip)
7. `DistinctivePlacesSection` (topVisitedPlaces) — `distinctive_places_section.dart:11-60`
8. `HowdahGuidesSection` (أدلّة، يخفي نفسه إن فارغ) — `howdah_guides_section.dart:11-74`
مساعِدة: `HeadTitleSection`, `ViewAllWidget`, `CategoryDailySuggestionsItems` (بطاقة+تقييم+زر مفضّل)، `HomeSkeletonLoader` (shimmer، `home_view_body.dart:113-289`)، `SlidersShimmer`.

## 7. Cubits / Blocs
| Cubit | File | States |
|---|---|---|
| HomeCubit | `.../manager/home_cubit/home_cubit.dart:1-22` | Initial → Loading → Loaded(HomeResponse) / Error(msg) |
| SlidersCubit | `.../manager/sliders_cubit/sliders_cubit.dart:1-22` | Initial → Loading → Loaded(List\<SliderModel>) / Error(msg) |
| LocationCubit (core) | `lib/core/managers/location_cubit/location_cubit.dart:1-60` | يجلب الموقع، يخزّنه cache، يحدّث كل 30ث |
| FavoriteCubit (core) | — | تغيّره يُطلق `fetchHome()` لتحديث حالة المفضّل |

## 8. State Flow
`fetchHome()` → `emit(Loading)` → `homeRepo.fetchHome()` يعيد `Either<Failure,HomeResponse>` → `fold` → `emit(Loaded/Error)`. نفس النمط لـ Sliders. التحديث يُطلق من: main.dart عند الإنشاء، Pull-to-refresh (`Future.wait` للاثنين)، تغيّر اللغة (LocaleCubit listener في main.dart)، تغيّر المفضّل (BlocListener).

## 9. Models
- **HomeResponse** (`home_response.dart:1-71`): places, topVisitedPlaces, topEvents, zads, topVisitedStores, applications[], safwests, guides, services[], globalMapData[], mapData[].
- **SliderModel** (`slider_model.dart:1-43`): id, title?, orderId?, image, imageSmall?, imageMedium?, link?.
- **PlaceModel** (`place_model.dart`): id, slug, type, صور، title/description، rate/review/ratings، lat/long/distance، featured/visited/isFavorite/isSaved، categories/galleries/price/city/region، روابط اجتماعية، viewsNum، keyWords.
- **PlaceResponse** (paginated): currentPage, items[], total. (+ZadResponse, TopEventsResponse, GuideResponse, SafwestResponse, ApplicationModel, ServiceModel, MapDataModel).

## 10. Repositories
- **Contract** `home_repo.dart:1-10`: `fetchHome() → Either<Failure,HomeResponse>` · `fetchSliders() → Either<Failure,List<SliderModel>>`.
- **Impl** `home_repo_imp.dart:1-32`: عبر `apiConsumer.handleRequest` + `get(EndPoints.home/sliders)`؛ يستخرج `data['data']` ثم `fromJson`.

## 11. Services
لا خدمة Flutter مخصّصة. Laravel: منطق التجميع داخل `HomeController::fetchHomeData`؛ مساعِدات جغرافية في `app/Helper/App.php` (`getLocationCoords:877-946`, `applyDistanceSorting:960-970`, `applyListingOrder`). تسجيل الزيارة عبر `visit()`.

## 12. API Endpoints
| Method | Endpoint | Auth | Cache |
|---|---|---|---|
| GET | `home` | ❌ عام | 60 دقيقة (مفتاح عام `home_data`) |
| GET | `sliders` | ❌ عام | بلا cache على الـ controller |
Query params لـ home: `lat`, `lng`, `search`, و`*_per_page` لكل قسم.

## 13. Request Models
لا request body (GET). المدخلات query params اختيارية: `lat`, `lng` (من LocationCubit)، `search`، أحجام الصفحات لكل قسم. الموقع يُمرَّر عبر params/headers.

## 14. Response Models
- `home`: envelope `{code,message,data:{11 sections}}`؛ كل قسم قوائم مُصفّحة (PlaceCollection…) أو مصفوفات (applications/services/mapData).
- `sliders`: `{code,message,data:[SliderResource…]}` (id, title, link, order_id, image, image_small, image_medium).

## 15. Laravel Routes
`routes/api.php` (front group، عام): `GET home → HomeController@getHome` (سطر 63) · `GET sliders → SliderController@index` (سطر 65). بلا middleware مصادقة.

## 16. Controllers
- **HomeController** (`app/Http/Controllers/Api/HomeController.php:1-223`): `getHome` (تسجيل زيارة + `Cache::remember('home_data',3600)`) → `fetchHomeData:43-222` (11 قسمًا، eager loading، ترتيب بالمسافة/القائمة).
- **SliderController** (`SliderController.php:1-23`): `index` → `Slider::active()->ordered()->get()` → `SliderResource::collection`.

## 17. Service Classes (Laravel)
لا service class مخصّص؛ منطق inline في الـ controller. مساعِدات جغرافية global في `App.php`. `ApiModalController::success` للغلاف.

## 18. Models (Laravel)
- **Slider** (`app/Models/Slider.php:1-78`): TranslatableContract + HasMedia؛ `$fillable=[order_id,active,link]`؛ `translatedAttributes=[title]`؛ `$with=[translations]`؛ media collection `image` + conversions small(400×250)/medium(800×500)؛ scopes `active()`/`ordered()`.
- يقرأ أيضًا: Place, Store, ZadElgadel, Event, Guide, Swalef, Application, Setting, MostVisit (راجع ملفات ميزاتها).

## 19. Database Tables
| Table | Model | ملاحظات |
|---|---|---|
| `sliders` | Slider | id, order_id, active(bool), link, timestamps |
| `slider_translations` | — | slider_id, locale, title |
| `media` (Spatie) | — | صور السلايدر + التحويلات |
| (قراءة) `places, stores, zad_elgadels, events, guides, swalefs, applications, most_visits, settings` | — | مصادر أقسام home — راجع `DATABASE_INDEX.md` |

## 20. Relationships
Slider 1—* slider_translations · Slider —media (morph). أقسام home تقرأ علاقات كل كيان: `with('ratings','galleries','city','region','ceo')` + translations. `isFavorite/isSaved` تُفحص مقابل جدول favorites/saves للمستخدم.

## 21. Validation Rules
لا تحقّق مفروض على `home`/`sliders` — كل الـ params اختيارية. `lat/lng` تُستخدم إن وُجدت وإلا fallback للـ cache/DB. `search` يُطبَّق عبر `whereTranslationLike` إن وُجد.

## 22. Authorization & Permissions
عام بالكامل (public). بلا guard، بلا أدوار. المصادقة تؤثر فقط على `getLocationCoords` (بحث cache/DB للموقع) وعلى قيم `isFavorite/isSaved` عند وجود مستخدم.

## 23. Error Handling
- **Flutter:** `Either<Failure,T>`؛ `HomeError/SlidersError(message)` تُعرَض كنص. الاتصال يُدار عبر ConnectivityCubit (main.dart) → دفع `kNoInternetView`.
- **Laravel:** الأخطاء تُغلَّف عبر `ApiModalController`. تسجيل الزيارة يستخدم IP عشوائي fallback.

## 24. Edge Cases
- بلا موقع → ترتيب بالقائمة (`order_id,id`) موحّد للجميع.
- أقسام فارغة → Sliders/Guides تُظهر `SizedBox.shrink()`.
- فعالية منتهية (`date_to < today`) → مستبعدة.
- تغيير اللغة → نفس الـ cache (لغة-محايد للبيانات) لكن الترجمات تتبع locale الطلب.
- ⚠️ **cache عام 60د**: سلايدر/محتوى جديد قد لا يظهر حتى ساعة؛ والمحذوف يبقى ظاهرًا حتى ساعة؛ وتغيّر المفضّل لا ينعكس في بيانات home المخزّنة (فقط فحص isFavorite وقت الرسم).
- ⚠️ **تلوّث cache بالموقع:** أول طلب يملأ الـ cache العام؛ إن كان بموقع، يرى مستخدمون بلا موقع ترتيبًا بالمسافة والعكس.

## 25. Dependencies
Flutter: `carousel_slider`, `smooth_page_indicator`, `cached_network_image`, `shimmer`, `flutter_bloc`, `dartz`, `geolocator`(عبر LocationCubit), `url_launcher`(فتح رابط البانر), `go_router`.
Laravel: `spatie/laravel-medialibrary`, `astrotomic/laravel-translatable`, `Cache`.

## 26. Files Involved
**Flutter:** `lib/features/home/**` (data/model/*, data/repo/*, presentation/manager/{home_cubit,sliders_cubit}/*, presentation/view/* + widgets/~40 مقسّمة لمجلدات فرعية: app_bar/applications/explore_category/guides/offers/properties/sliders/suggestions/trip_card/shared)، `lib/core/managers/location_cubit/**`، `lib/core/databases/api/end_points.dart` (home,sliders)، `lib/main.dart:124-215`.
**Laravel:** `app/Http/Controllers/Api/{HomeController,SliderController}.php`، `app/Models/Slider.php`، `app/Http/Resources/{Sliders/SliderResource, Places/PlaceCollection, Places/PlaceListResource}.php`، `app/Helper/App.php:877-984`، `routes/api.php:62-65`.

## 27. Complete Execution Flow (Home)
```
HomeView.initState → LocationCubit.fetchCurrentLocation()
HomeCubit.fetchHome() (main.dart)  →  emit(Loading)
  → HomeRepoImp.fetchHome → ApiConsumer.get(EndPoints.home) [+lat/lng]
  → GET /api/home  →  HomeController@getHome
  → visit() log · Cache::remember('home_data',3600, fetchHomeData)
  → fetchHomeData: 11 أقسام · with(ratings,galleries,city,region,ceo)
  → getLocationCoords → applyDistanceSorting / applyListingOrder
  → PlaceCollection/EventCollection… · success(data,200)
  → DB: ~40–50 استعلام (بارد) · Response {code,message,data:{...}}
  → HomeResponse.fromJson(data['data'])  →  Either.right
  → emit(HomeLoaded)  →  BlocBuilder  →  CustomScrollView (8 أقسام)
Sliders بالتوازي: SlidersCubit.fetchSliders → GET /api/sliders → SliderController@index → SliderResource → SlidersLoaded → CarouselSlider.
Tap بطاقة → push(kPlacesDetailsItems, extra: slug) / kTourGuideDetailsView(id) / Tasneef / StartTrip.
```

## 28. Performance Considerations
- ⚠️ **cache عام** بلا موقع/لغة في المفتاح → نتائج مضلّلة (راجع §24). الإصلاح: تضمين hash الموقع/اللغة.
- ⚠️ **`isFavorite/isSaved`** قد تسبّب N+1 لكل عنصر (`PlaceListResource:76-77`) — يفضّل eager-load favorites للمستخدم.
- eager-load `ceo` لكل عنصر رغم عدم استخدامه غالبًا في القوائم → حِمل زائد.
- payload ضخم (11 قسمًا، 110–550 عنصرًا، 50KB–2MB) — فكّر في تقسيم/lazy لبعض الأقسام.
- Sliders بلا cache رغم ندرة تغيّرها — يفضّل cache 24س.
- LocationCubit يحدّث كل 30ث (استهلاك بطارية).

## 29. Security Considerations
- endpoint عام — تسريب هيكل المحتوى بالكامل (مقبول لمحتوى تسويقي).
- 🟡 **cache poisoning بالموقع/الخصوصية:** موقع أول مستخدم يؤثر على ترتيب الآخرين (مفتاح cache عام).
- 🟡 روابط Sliders تُفتح عبر `url_launcher` — يجب التحقق من نطاق/سلامة الرابط قبل الفتح.
- تسجيل الزيارة بـ IP خام (اعتبارات خصوصية).

## 30. Technical Debt
- فئات "ExploreByCategory" و"TripCard" **ثابتة (dummy/hardcoded)** لا من الخادم.
- ~~~40 widget في مجلد واحد~~ → ✅ مُنظَّم (2026-07-07): مقسّم لمجلدات فرعية بالمسؤولية (app_bar/applications/offers/properties/sliders/suggestions/guides/explore_category/trip_card/shared).
- عدم اتساق تعليق الـ cache (60د بينما التعليق يقول ساعتين).
- تكرار DailySuggestions/DistinctivePlaces يعرضان نفس `topVisitedPlaces`.
- المفتاح العام يجعل التحديث بطيء الظهور (حتى ساعة).

## 31. Improvement Opportunities
- تضمين locale + hash الموقع في مفتاح الـ cache، أو فصل الترتيب الجغرافي خارج الـ cache.
- eager-load favorites/saves للمستخدم لإزالة N+1، وإزالة `ceo` من قوائم home.
- cache للـ sliders (طويل) + إبطاله عبر Observer عند التعديل.
- جعل الفئات من الخادم بدل hardcoded.
- ضغط الرد (gzip) وتقسيم الأقسام الثقيلة إلى نداءات كسولة.
- التحقق من روابط Sliders قبل الفتح.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 01 Authentication | اختياري — يحدّد `isFavorite/isSaved` وربط الموقع بالمستخدم؛ home يعمل بلا دخول |
| 03 Trips | بطاقة الرحلة في home → StartTrip |
| 04/05/06/09/10/11 (Places/Stores/Zad/Events/Swalef/Guides) | home يعرض ملخّصاتها ويوجّه لتفاصيلها |
| 07 Tasneef | أزرار "View All" → قوائم Tasneef |
| 08 Exploration | يشارك الموقع/الترتيب الجغرافي |
| 13 Favorites | تغيّر المفضّل يُحدّث home (BlocListener) |
| 18 Taxonomy | الفئات/الأقسام تعتمد التصنيفات |
| 23 Media | صور السلايدر/البطاقات عبر MediaLibrary |
> تغيير شكل `HomeResponse` أو أقسامها يؤثّر على شاشة الهبوط بالكامل والتنقّل منها.

## 33. How to Modify This Feature
- **قسم home جديد:** `HomeController::fetchHomeData` (الخادم) + Resource + `HomeResponse` (Flutter fromJson) + widget قسم جديد في `home_view_body.dart`.
- **تعديل السلايدر:** `SliderController`/`Slider` model/`SliderResource` + `SliderModel`/`home_sliders_section.dart`.
- **ترتيب/فلاتر:** `App.php` (getLocationCoords/applyDistanceSorting) + scopes الكيانات.
- **الفئات الثابتة → ديناميكية:** `explore_by_category_section.dart` + endpoint categories.
- **APIs:** `home`, `sliders` (+ endpoints التفاصيل عند التنقّل).
- **المخاطر:** كسر cache، N+1، عدم تطابق مفاتيح JSON، حجم payload، تلوّث cache بالموقع.
- **اختبارات:** جلب home/sliders، pull-to-refresh، تغيير لغة، بلا موقع/بموقع، أقسام فارغة، تنقّل بالـ slug/id.

## 34. Regression Checklist
- [ ] فتح التطبيق → home + sliders يُحمّلان بالتوازي بلا خطأ.
- [ ] shimmer يظهر أثناء التحميل ثم المحتوى.
- [ ] السلايدر يدور (autoPlay) والمؤشّر يتزامن؛ الضغط يفتح الرابط.
- [ ] كل قسم يعرض بياناته الصحيحة (places/topVisited/events/zad/stores/guides/apps).
- [ ] قسم فارغ (Sliders/Guides) يختفي بلا فراغ.
- [ ] Pull-to-refresh يُعيد جلب الاثنين.
- [ ] تغيير اللغة (ar/en/ru/zh) → ترجمات محدّثة + RTL.
- [ ] بموقع → ترتيب بالمسافة؛ بلا موقع → ترتيب بالقائمة.
- [ ] الضغط على بطاقة → تفاصيل صحيحة (slug/id)؛ "View All" → Tasneef.
- [ ] تغيّر المفضّل → انعكاس على home.
- [ ] قطع الإنترنت → NoInternetView؛ العودة → استئناف.
- [ ] فعالية منتهية غير ظاهرة.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «محتوى قديم رغم التحديث» | cache عام 60د | `HomeController@getHome` (Cache::remember) |
| «ترتيب غريب للأماكن» | تلوّث cache بموقع مستخدم آخر | مفتاح `home_data` + getLocationCoords |
| «حالة المفضّل خاطئة» | فحص isFavorite وقت الرسم + cache | `PlaceListResource:76` + FavoriteCubit listener |
| «السلايدر لا يظهر» | active=0 أو صور غير مولّدة | `Slider::active()` + media conversions |
| «بطء/حجم كبير» | payload ضخم + N+1 isFavorite | eager loading + §28 |
| «الفئات ثابتة لا تتغيّر» | hardcoded dummy | `explore_by_category_section.dart` |
| «الصورة لا تُحمّل» | URL/دومين test vs prod | `SliderModel.image` + AppEnvironmentManager |
| «رابط بانر لا يفتح» | link فارغ/غير صالح | `home_sliders_section.dart` url_launcher |

## 36. Debug Guide
1. فعّل الشبكة (PrettyDioLogger/Chucker) وراقب `GET /home` و`GET /sliders`.
2. تحقّق envelope `{code:200,data:{...}}` وأن `data` يحوي 11 قسمًا.
3. راقب حالات HomeCubit/SlidersCubit (Loading→Loaded/Error).
4. لمشاكل الترتيب: افحص هل تُرسَل lat/lng (LocationCubit) وقيمة cache على الخادم (`php artisan cache:clear` للاختبار).
5. لمشاكل الترجمة: بدّل locale وتحقّق ترويسة `accept-language` + جدول `slider_translations`.
6. جانب الخادم: `storage/logs/laravel.log` + عدّاد الاستعلامات (Telescope/DB::listen) لرصد N+1.
7. للمفضّل: تتبّع FavoriteCubit → BlocListener → fetchHome.

## 37. Search Keywords
`home`, `sliders`, `slider`, `banner`, `carousel`, `HomeCubit`, `SlidersCubit`, `HomeResponse`, `SliderModel`, `HomeController`, `getHome`, `fetchHomeData`, `SliderController`, `home_data cache`, `getLocationCoords`, `applyDistanceSorting`, `applyListingOrder`, `topVisitedPlaces`, `show_in_home`, `featured`, `prefered`, `PlaceCollection`, `PlaceListResource`, `SliderResource`, `slider_translations`, `HomeViewBody`, `HomeSlidersSection`, `DailySuggestionsSection`, `DistinctivePlacesSection`, `HowdahGuidesSection`, `HomeSkeletonLoader`, `smooth_page_indicator`, `carousel_slider`, `LocationCubit`, `pull to refresh`, `view all`, `discovery`, `landing` → **02_Home.md**

## Web Frontend (Angular)
- **Granularity DIFFERS (key):** mobile calls a single aggregated `GET /home` (2h cache); web splits home into **8+ endpoints** (`home.service.ts`): `home` (18), `get_top_destinations` (43), `get_travelers` (63), `get_saudi_opening` (83), `getSaudiPlaces` (103), `services` (123), `get_sliderPlaces` (94), `get_events` (23). More requests, more per-section control.
- **Extra home widgets:** travelers, Saudi-opening, Saudi-places, services blocks — assembled client-side.
- **Newsletter/contact on home:** `POST subscribe` + `POST contactus/send` (`home.service.ts:128–136`) — working forms (mobile contact is display-only, [[21_ProfileSettings]]).
- **Component:** `HomePageComponent`. See [[WEB_MOBILE_COMPARISON]] · [[08_Exploration]] (global-search/map).

## 38. Future Improvements
- مفتاح cache يتضمّن locale + موقع (أو ترتيب خارج cache) لإزالة التلوّث.
- eager-load favorites/saves للمستخدم لإزالة N+1، وإزالة علاقة `ceo` غير المستخدمة من القوائم.
- cache طويل للسلايدر + إبطال بالـ Observer.
- الفئات وبطاقة الرحلة من الخادم بدل hardcoded.
- ضغط gzip + تحميل كسول للأقسام الثقيلة + تقليل حقول القوائم.
- تخصيص المحتوى per-user (توصيات) مستقبلًا.
- ~~تنظيم مجلد widgets (تقسيم فرعي)~~ → ✅ تم (2026-07-07). يبقى: التحقق من روابط البانرات.

---
**تدفق مرجعي:** UI → Cubit → Repository → API → Route → Controller → (Cache/Helpers) → Models → DB → Response → Flutter UI ✅ موثّق لـ home + sliders.
