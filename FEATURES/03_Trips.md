# Feature 03 — Trips (v1 Legacy + v2 Enhanced) 🚩 Flagship

> **Status:** ✅ Analyzed (100%) · **Priority:** P0 (Flagship, أكبر ميزة) · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/trip) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-07

---

## 1. Feature Name
Trips — مخطّط الرحلات: يبني للمستخدم برنامج رحلة يومًا بيوم عبر مناطق، ثم يحضّره (prepare) بخوارزمية اختيار أماكن، يعرضه، يعدّله، يحفظه، يشاركه بالبريد، ويستعرض/يحذف رحلاته. **نسختان متعايشتان:** v1 (قديمة، قوائم مسطّحة) و v2 Enhanced (فترات صباح/مساء + أوصاف).

## 2. Business Goal
الميزة المميِّزة للتطبيق: تحويل نية السفر إلى برنامج جاهز (أماكن مرتّبة بالقرب الجغرافي عبر مسار Google Maps، مقسّمة صباح/مساء، مع أوصاف وموسم)، مع مرونة تعديل قبل الحفظ ومشاركة بالبريد.

## 3. Business Rules
- الإدخال: تاريخ بداية/نهاية، منطقتا بداية/نهاية، عدد أماكن/يوم (1–20)، فئات، نطاق أسعار، نوع مركبة.
- v2 `places_per_period = ceil(places_per_day/2)` (نصف صباح، نصف مساء).
- الاختيار: أماكن `active=1` ضمن 50كم من نقاط المسار، مفلترة بالفئات (بحث JSON)، السعر (`price_id IN`)، الموسم (شهر التاريخ).
- نفس المنطقتين → بحث داخل منطقة واحدة (`inRandomOrder`)؛ منطقتان مختلفتان → نقاط مسار Google Maps ثم أماكن قربها.
- الترتيب: كل يوم يُرتَّب بالقرب من آخر مكان في اليوم السابق (Haversine)؛ عند نفاد الأماكن → إعادة استخدام.
- `reprepare` (v2): يعيد التوليد مستبعِدًا الأماكن المستخدمة.
- الحفظ يتحقّق من عدم تكرار اسم الرحلة لكل مستخدم؛ يحذف prepareTrip بعد الحفظ.
- token عشوائي (20 حرف) لكل رحلة محضّرة/محفوظة.
- المستخدم يعدّل الأماكن (حذف/نقل بين الفترتين) محليًا قبل الحفظ؛ لا حذف إن بقي مكان واحد فقط باليوم.

## 4. User Flow
StartTripView → AddTripView (معالج 4 خطوات: المدة → المنطقتان → التفاصيل [أماكن/يوم، أسعار، مركبة] → الفئات) → prepare → FinishTrip → NewTripPlan (عرض يوم بيوم، صباح/مساء) → تعديل/refresh → Save (اسم) أو Share (بريد). لاحقًا: NewMyTripView (قائمة مصفّحة) → ShowMyTrip (عرض) / حذف.

## 5. Flutter Screens
StartTripView (`start_trip_view.dart`)، AddTripView (معالج، `add_trip_view.dart:26-195`)، FinishTrip (`finish_trip.dart`)، NewTripPlan (عرض v2 المحضّر، `new_trip_plan.dart:18-192`)، ShowMyTrip (عرض المحفوظ v2، `show_my_trip.dart`)، NewMyTripView / MyTripView (قوائم)، TripPlan/MyTrip (v1 legacy). ملحقات: SaveTrip/SaveTripToEmail عبر bottom sheets، MapTripView.

## 6. Widgets
`NewTripPlanBody` (CustomScrollView، `new_trip_plan_body.dart:23-159`)، `TripDetailsCard` (تواريخ/مناطق + refresh + calendar)، `TripProgramHeader` (خريطة)، `TripDayCard` (`trip_day_card.dart`) → `DayHeaderCard` + `PeriodSection`×2 (صباح/مساء، `period_section.dart`) → `PlacesList` (بطاقات + حذف/نقل). المعالج: `TripDurationPicker`, `RegionSelectionStep`, `CustomWidgetsHeadTripDetails`, `TripCategoriesSelector`, `StepIndicator`. القوائم: `NewMyTripViewBody` (PagedListView)، `MyTripViewBodyItemsList`، `DeleteTripBottomSheet`. الحفظ: `NewSaveTripV2ViewBaseSheet`, `SaveTripToEmailView`, `BottomSheetTripSaveTripDone`.

## 7. Cubits / Blocs
| Cubit | نسخة | دور |
|---|---|---|
| PrepareTripWizardCubit (`:18-438`) | v1 primary | معالج 4 خطوات، draft في SharedPreferences، يبني daterange (end-start) |
| PrepareTripCubit (`:11-39`) | both | يطلق prepare (v1 أو v2 عبر useNew) → TripModel/EnhancedTripResponse |
| PrepareTripShowCubit (`:8-163`) | v2 | جلب المحضّر بالـ token + تعديلات محلية (removePlaceFromDay/movePlaceToOtherPeriod) |
| ReprepareTripCubit | v2 | إعادة التوليد |
| SaveTripV2Cubit | v2 | حفظ (v2/trips/save) |
| SaveTripCubit | v1 | حفظ (trips/store) |
| SaveTripToEmailCubit (`:11-153`) | both | مشاركة بالبريد |
| MyTripCubit / NewMyTripCubit | v1/v2 | قوائم بترقيم لانهائي (PagingController) |
| NewViewTripCubit | v2 | عرض المحفوظ |
| DeleteTripCubit | v2 | حذف |
| FetchPricesCubit / FetchCategoryCubit | both | جلب الأسعار/الفئات للمعالج |

## 8. State Flow
كل cubit: `Loading → Success/Failure` عبر `Either`. المعالج يخزّن draft ويتحقّق كل خطوة. prepare → success → FinishTrip → NewTripPlan (يجلب بالـ token). التعديلات محلية على `PrepareTripShowState` (بلا API) حتى الحفظ حيث تُستخرج `items: List<List<int>>` عبر `extractItemIdsFromEnhancedTrip`.

## 9. Models
**Flutter v2:** `NewTripPrepareParams` (startDate,endDate,startRegionId,endRegionId,placesPerDay,categories[],priceRange[],vehicleType)، `EnhancedTripResponse→EnhancedTripData→EnhancedDay→EnhancedPeriod(morning/evening)→Place` (json_serializable + محوّلات مخصّصة لـ lat/long/rate/distance)، `NewMyTripsResponse→TripItem`. **v1:** `PrepareTripModel` (daterange end-start)، `TripModel` (List\<List\<UnifiedPlaceModel>>)، `MyTripModel`، `TripDetails`. مشترك: `SaveTripToEmailParams`, `PricesModel`. Mapper: TripItem→MyTripModel.
**Laravel:** `Trip` (casts: enhanced_data/morning_descriptions/evening_descriptions/items = array، is_enhanced=bool؛ scopes enhanced/regular/forUser؛ accessors getMorning/EveningDescription، getTotalPlaces)، `prepareTrip` (enhanced_generated_data/categories_array/price_range_array/items/places = array).

## 10. Repositories
- **Flutter** `trip_repo.dart:15-52` + `trip_repo_imp.dart:20-170`: fetchPrices/Category، prepareTrip(v1)، newPrepareTrip(v2)، showPrepareTrip، saveTrip(v1)/saveTripV2/saveTripToEmail، myTrip/newMyTrip (مصفّح)، viewTrip(v1)/newViewTrip، deleteTrip، reprepareTrip.
- **Laravel:** لا repository pattern؛ منطق في Controllers + `EnhancedTripService`.

## 11. Services
**Laravel `EnhancedTripService`** (المحرّك): `planEnhancedTrip` (تنسيق)، `generateEnhancedTripPlan` (نفس المنطقة/مسار)، `getRouteWaypoints` (Google Maps sync)، `findPlacesAlongRoute`/`findPlacesInRegion` (Haversine + فلاتر)، `organizePlacesByPeriodsAndDays` (ترتيب بالقرب + تقسيم صباح/مساء + إعادة استخدام)، `calculateDistance`، `generatePeriodDescription` (⚠️ `determineCategoryType` مبتور → دائمًا default)، `determineSeason`، `saveEnhancedTrip` (transaction)، `generateEnhancedDataFromCustomItems`، `getDayMainCityInfo`، backward-compat extractors. Flutter: لا service مخصّص.

## 12. API Endpoints
**v1:** POST `trips/prepare` (بلا مصادقة) · GET `trips/prepare/{token}` · POST `trips/store` · POST `trips/save-trip-to-email` (بلا مصادقة) · GET `my-trips` · GET `view_trip/{token}` (⚠️ بلا مصادقة) · DELETE `delete_trip/{id}`.
**v2:** POST `v2/trips/prepare` · POST `v2/trips/reprepare` · GET `v2/trips/prepare/{token}` · DELETE `v2/trips/prepare/{token}` · POST `v2/trips/save` · GET `v2/trips/my-trips` · GET `v2/trips/my-prepared-trips` · GET `v2/trips/view/{token}` (+ownership) · DELETE `v2/trips/{token}` (+ownership) · GET `v2/trips/statistics`. مساعِد: GET `prices`, `categories`.

## 13. Request Models
- v2 prepare: `{start_date, end_date, start_region_id, end_region_id, places_per_day(1-20), categories[], price_range[], vehicleType}`.
- v2 save: `{prepare_token, name, items:[[placeId,...],...]}` (items.*.* موجودة في places).
- v2 reprepare: `{prepare_token}`.
- v1 prepare (inline): `{daterange, type, funny_place_per_day, region1, region2, categories}`.
- email: `{email, user_name, name, items(jsonEncoded), date, start_date, end_date, item_per_day, region1, region2}`.

## 14. Response Models
- prepare/view (v2): `{token, dates, total_days, places_per_day, places_per_period, start_region, end_region, enhanced_data:[{day_number,date,city,morning:{places[],description},evening:{...}}]}`.
- my-trips (v2): قائمة TripItem مصفّحة (id,name,token,dates,totalPlaces,regions,createdAt).
- statistics: total_enhanced_trips, total_prepared, total_places_visited.
- الغلاف `{code,message,data}`.

## 15. Laravel Routes
`routes/api.php` — v1: أسطر 120–130؛ v2: 133–145. مصادقة `auth('api')` على كل v2 وأغلب v1، **عدا** `trips/prepare`, `save-trip-to-email`, `view_trip/{token}` (عامة).

## 16. Controllers
- **TripController** (v1): prepare_trip (`:93-308`)، save_trip (`:26-60`)، save_trip_to_email (`:62-90`)، view_trip (`:358-397`)، delete_trip (`:399-403`)، my_trips (`:405-417`)، get/delete/get_my prepare. `getRoutePoints` (`:311-355`، ⚠️ مفتاح+توكن مثبّتان).
- **EnhancedTripV2Controller** (v2): prepare (`:30-64`)، reprepare (`:69-138`)، getPrepared (`:143-231`)، save (`:236-283`)، getMyTrips (`:288-335`)، getMyPrepared (`:340-387`)، view (`:392-490`، +ownership `:404`)، delete (`:495-509`)، deletePrepared، statistics (`:533-547`).

## 17. Service Classes (Laravel)
`EnhancedTripService` (المحرّك الكامل، راجع §11). `config/enhanced_trip.php` (radius 50كم، max 20/يوم، أوصاف الفترات، المواسم، مفتاح Google، حدود التحقّق). ملاحظة: خيارات cache/logging/mock معرّفة **وغير مستخدمة**.

## 18. Models (Laravel)
`Trip` (`app/Models/Trip.php`، `$guarded=[]`، casts JSON، scopes، accessors) · `prepareTrip` (`app/Models/prepareTrip.php`، `$guarded=[]`، casts JSON، scope active). كلاهما BelongsTo User + region1/2 → Region.

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `trips` | id, name, item_per_day, days, items(JSON), date, user_id→users, start_date, end_date, region1/region2→regions, **enhanced_data(JSON), morning_descriptions(JSON), evening_descriptions(JSON), is_enhanced(bool, index), places_per_period(int)** |
| `prepare_trips` | id, items/places/selected_categories(JSON), start/end_date, funny, funny_place_per_day, days, lat1/long1/lat2/long2, token, user_id→users, region1/2→regions, **enhanced_generated_data(JSON), is_enhanced(bool,index), places_per_period, categories_array(JSON), price_range_array(JSON)** |
⚠️ تسمية إرثية مربكة: `funny`, `funny_place_per_day`. فهارس ناقصة: `(user_id,is_enhanced)`, `(token)`.

## 20. Relationships
Trip/prepareTrip BelongsTo User · region1Object/region2Object BelongsTo Region. الأماكن مخزّنة كـ JSON IDs (لا FK)؛ تُحمَّل بـ `Place::whereIn(...)->with(city,region,price,galleries,ceo,ratings)`.

## 21. Validation Rules
- `PrepareTripRequest`: start_date required|date|after_or_equal:today · end_date after_or_equal:start_date · start/end_region_id exists:regions · places_per_day integer|1..20 · categories/price_range nullable array (عناصر integer). `prepareForValidation` يحقن user_id.
- `SaveEnhancedTripRequest`: name required|max:255 · prepare_token required · items required array · items.*.* integer|exists:places.
- `ReprepareEnhancedTripRequest`: prepare_token required.
- v1 save (inline): name/item_per_day/days/items/date/user_id مطلوبة.

## 22. Authorization & Permissions
v2 محكم: كل المسارات `auth('api')` + فحص ملكية على view/delete/prepared (`user_id===auth id`، 403 خلاف ذلك). v1: `my-trips`/`delete_trip`/prepared محمية بالملكية، لكن **`view_trip/{token}` و`prepare` و`save-to-email` عامة** (ثغرات، §29).

## 23. Error Handling
- **Flutter:** `Either<Failure,T>` لكل عملية؛ حالات Failure تُعرَض مع زر إعادة. تعديلات محلية آمنة (تمنع حذف آخر مكان).
- **Laravel:** Form Requests → 422 مترجمة؛ ملكية → 403؛ غير موجود → 404؛ `saveEnhancedTrip` داخل transaction. fallback عند غياب نقاط المسار → بحث منطقة1.

## 24. Edge Cases
- منطقتان متطابقتان → بحث منطقة واحدة (random).
- نقاط مسار فارغة من Google → fallback منطقة1.
- `places_per_day` > المتاح → إعادة استخدام الأماكن (غير موثّق للمستخدم).
- لا أماكن بالمنطقة → أيام فارغة (بلا خطأ).
- ترجمة المنطقة `translate('en')` بلا fallback → احتمال null (باك، `EnhancedTripService` — منفصل تمامًا عن نقطة تالية).
- ✅ **مُصلَح (موبايل):** أسماء مناطق `RegionSelectionStep`/`RegionField` كانت دايمًا إنجليزي (مصدرها `Country` بخريطة SVG ثابتة `assets/images/map.svg`، مش من API الـtaxonomy المترجم). بقى فيه `name-ar`/`name-ru`/`name-zh` على كل منطقة بالـSVG + `Country.localizedName(languageCode)` بـ`map_two.dart`، ومُستخدَمة بعرض الخريطة وبتخزين الاسم بـ`PrepareTripWizardCubit` عند الاختيار.
- ⚠️ token المحضّر بلا انتهاء → نمو DB بلا حدود؛ رحلات الضيوف (user_id=null) دائمة.
- ✅ **مُصلَح (موبايل):** `daterange` بـ`PrepareTripWizardCubit` بقى بصيغة عادية start-end (كان end-start مربك). ملاحظة: v1 `trips/prepare` بالباك ما زال (حسب التوثيق) يتوقّع الصيغة القديمة end-start — لكن هذا المسار غير قابل للوصول فعليًا بالواجهة الحالية (`useNew: true` مثبّت بـ`add_trip_view.dart`)، فما فيه أثر فعلي. لو استُخدم v1 مجددًا لازم تنسيق مع الباك أولًا (قاعدة عقد الـAPI).
- المستخدم يعدّل items قبل الحفظ (يقبلها الخادم دون التحقق من مطابقتها للمحضّر).

## 25. Dependencies
Flutter: `dio`, `dartz`, `flutter_bloc`, `infinite_scroll_pagination`, `json_serializable`, `shared_preferences`(draft), `go_router`, خرائط/تواريخ. Laravel: Google Maps Directions API (CURL sync)، `Str::random`، `Mail`/`ShouldQueue`، `astrotomic/translatable`, `spatie/medialibrary`, `DB transaction`.

## 26. Files Involved
**Flutter:** `lib/features/trip/**` (data/model/{trip_model/*, enhanced_trip_response(.g).dart, new_trip_prepare_request, new/*, save_trip_to_email_params, prices_model, my_trip_model}, data/repo/{trip_repo,trip_repo_imp}, presentation/manager/* (13 cubit)، presentation/view/* + widgets + new_widgets)، `end_points.dart`.
**Laravel:** `app/Http/Controllers/Api/{TripController,EnhancedTripV2Controller}.php`، `app/Services/EnhancedTripService.php`، `app/Models/{Trip,prepareTrip}.php`، `app/Http/Requests/{EnhancedPrepareTrip/PrepareTripRequest, EnhancedTrip/SaveEnhancedTripRequest, EnhancedTrip/ReprepareEnhancedTripRequest}.php`، `app/Http/Resources/{Trips/TripResource, PrepareTrips/PrepareTripResource}.php`، `app/Mail/SaveNewTrip.php`، `resources/views/front/emails/trip.blade.php`، `config/enhanced_trip.php`، `routes/api.php:120-145`، migrations trips/prepare_trips.

## 27. Complete Execution Flow (Prepare v2)
```
AddTripView(4 خطوات) → PrepareTripCubit.prepareTrip(useNew=true) → emit(Loading)
  → TripRepoImp.newPrepareTrip → POST /api/v2/trips/prepare
  → PrepareTripRequest(validate + inject user_id) → EnhancedTripV2Controller@prepareEnhancedTrip
  → EnhancedTripService.planEnhancedTrip:
       prepareTrip::create → generateEnhancedTripPlan
       (نفس منطقة؟ findPlacesInRegion : getRouteWaypoints[Google sync] + findPlacesAlongRoute[Haversine+فلاتر])
       → organizePlacesByPeriodsAndDays (ترتيب بالقرب، تقسيم صباح/مساء، أوصاف)
       → update(enhanced_generated_data, items, places)
  → DB: create + (waypoints استعلامات) + update · Response{token, enhanced_data[...]}
  → EnhancedTripResponse.fromJson → emit(Success) → FinishTrip → NewTripPlan
  → PrepareTripShowCubit.showTrip → GET v2/trips/prepare/{token} → NewTripPlanBody (TripDayCard×days)
Save: extractItemIds → SaveTripV2Cubit → POST v2/trips/save → EnhancedTripService.saveEnhancedTrip (transaction: Trip::create + prepareTrip.delete) → NewMyTripView.
Reprepare (موبايل، جديد): زر "refresh" بـNewTripPlanBody → BottomSheetTripDealt تأكيد (reprepare_trip_confirm/warning) → ReprepareTripCubit.reprepareTrip(token) → TripRepoImp.reprepareTrip → POST /v2/trips/reprepare (ReprepareEnhancedTripRequest بالباك) → EnhancedTripResponse → إعادة بناء الخطة (يفقد التعديلات اليدوية).
```

## 28. Performance Considerations
- 🔴 **Google Maps CURL متزامن** (30–60ث) يحجب الطلب في كل prepare (`TripController:311`, `EnhancedTripService:123`) — يجب queue + cache للمسارات.
- 🔴 **N+1 في `PrepareTripResource:25-30`** (استعلام Place لكل يوم) — استخدم `whereIn(allIds)->keyBy('id')`.
- 🟠 استعلام أماكن لكل يوم في getPrepared/view (يمكن دمجه بنداء واحد).
- 🟠 لا cache لنقاط المسار رغم تكرارها.
- 🟠 فهارس DB ناقصة `(user_id,is_enhanced)`, `(token)`.
- 🟡 `enhanced_data` JSON ضخم (50–100KB/رحلة).

## 29. Security Considerations
1. 🔴 **مفتاح Google Maps مثبّت** (`TripController:321,335`، fallback في `EnhancedTripService:131` و`config/enhanced_trip.php:91`) = إساءة استخدام/فوترة.
2. 🔴 **توكن Bearer مثبّت** (`TripController:338`: `22|sw1Gf...`) = انتحال مستخدم.
3. 🔴 **`view_trip/{token}` (v1) بلا مصادقة** = تعداد token وكشف رحلات الآخرين. ⚠️ الثغرة نفسها بالباك لسه موجودة، لكن **الموبايل ما يستدعيها بعد الآن** (شاشات v1 المستهلكة لها حُذفت بالكامل — انظر §30).
4. 🟠 `trips/prepare` و`save-trip-to-email` عامة بلا rate-limit = إغراق DB/سبام بريد.
5. 🟠 رحلات الضيوف (user_id=null) وtokens المحضّرة بلا انتهاء = نمو غير محدود.
6. 🟡 المستخدم يعدّل items قبل الحفظ دون تحقّق من مطابقتها للمحضّر (تُتحقّق فقط أنها places موجودة).
7. ✅ v2 محكم (auth + ownership على الكل).

## 30. Technical Debt
- ✅ **جديد (موبايل):** إعادة التحضير — `ReprepareTripCubit` + `trip_repo.reprepareTrip` + `EndPoints.reprepareTrip` (`/v2/trips/reprepare`) موصولة بزر refresh في `NewTripPlanBody` عبر `BottomSheetTripDealt` تأكيدي (مفاتيح `reprepare_trip_confirm/warning` بالـ4 لغات). تحذير المستخدم صريح: إعادة البناء تفقد التعديلات اليدوية (حذف/نقل).
- ✅ **مُنظَّف (موبايل):** `FinishTrip` شال باراميتر `tripModel` (بقايا v1) — بقى `tripResponse` فقط. و`PrepareTripWizardCubit` رفع مفتاح المسودّة `prepare_trip_draft_v1→_v2` بعد تبديل ترميز daterange (يبطل المسودّات القديمة بالصيغة المعكوسة).
- ✅ **مُصلَح (موبايل):** كود v1 المستقل بشاشات الموبايل اتحذف بالكامل بعد تأكيد إنه dead code فعليًا (تتبّع تدفّق `AddTripView→useNew:true→FinishTrip→دايمًا kNewTripPlan`، فرع v1 ما ينفّذ أبدًا). اتحذف: `TripPlan`, `MyTripView`, `MyTripDetailsView`, `TripPlanBody`, `TripDayRow`, `SaveTripView` + cubits `TripPlanCubit`, `FinishTripDetailsCubit`, `SaveTripCubit`, `ViewTripCubit`, `MyTripCubit` + route keys `kTripPlan/kMyTripView/kMyTripDetailsView`. الـmodels/repo methods (`TripModel`, `PrepareTripModel`, `trip_repo.prepareTrip/saveTrip/myTrip/viewTrip`) خُلّيت كما هي (متشابكة مع wizard/prepare cubit المشترك مع v2، حذفها أخطر ويحتاج مهمة منفصلة).
- تعايش v1/v2 بالباك (controllers/models مزدوجة) — لسه قائم، خارج نطاق الموبايل.
- تسمية `funny`/`funny_place_per_day` إرثية مربكة.
- `determineCategoryType` مبتور → الأوصاف دائمًا "default".
- خيارات config (cache/logging/mock) معرّفة وغير مستخدمة.
- ~~daterange معكوس (end-start)~~ → ✅ مُصلَح بالموبايل start-end (v1 بالباك فقط لا يزال يتوقّع القديم، غير قابل للوصول).
- fallback مفتاح مثبّت داخل config.
- backward-compat يحتفظ بعمودَي items/places مع enhanced_data (ازدواج).

## 31. Improvement Opportunities
- نقل نداء Google Maps إلى queue + cache بالمسار (origin,destination) لإزالة الحجب.
- إصلاح N+1 في PrepareTripResource ودمج استعلامات view/prepared.
- إضافة مصادقة/ملكية لـ `view_trip` v1، وrate-limit لـ prepare/email.
- نقل المفتاح/التوكن لـ env وإبطالهما، وإزالة fallback المثبّت.
- انتهاء صلاحية للـ prepared tokens + تنظيف دوري + سياسة رحلات الضيوف.
- إكمال `determineCategoryType` لأوصاف سياقية، وترجمة نص البريد.
- فهارس DB، وإهلاك v1 تدريجيًا لصالح v2.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 01 Authentication | إلزامي لـ v2 كله وحفظ v1 (user_id من التوكن) |
| 02 Home | بطاقة الرحلة في home → StartTrip |
| 04 Places | مصدر عناصر الرحلة (Place model + PlaceResource) |
| 18 Taxonomy | Regions (بداية/نهاية)، Prices، Categories في المعالج |
| 08 Exploration/Location | الموقع يؤثّر على ترتيب `organizePlacesByPeriodsAndDays` |
| 17 Notifications/Mail | مشاركة الرحلة بالبريد (SaveNewTrip) |
> تغيير شكل Place/PlaceResource أو enhanced_data يؤثّر على عرض الرحلة بالكامل.

## 33. How to Modify This Feature
- **تعديل الخوارزمية:** `EnhancedTripService` (findPlaces*/organizePlaces*/generatePeriodDescription).
- **حقل جديد في الرحلة:** migration trips/prepare_trips + `Trip`/`prepareTrip` casts + Resource + نماذج Flutter (EnhancedTripResponse) + widget.
- **خطوة معالج جديدة:** `prepare_trip_wizard_cubit.dart` + widget خطوة + `add_trip_view.dart`.
- **APIs:** v2/trips/* (المعتمدة)؛ تجنّب توسيع v1.
- **المخاطر:** حجب Google Maps، N+1، عدم تطابق مفاتيح JSON بين النسختين، تسمية إرثية، كسر backward-compat.
- **اختبارات:** prepare (نفس/مختلف منطقة، بلا أماكن)، reprepare، save/view/delete (+ownership)، email، pagination، تعديلات محلية.

## 34. Regression Checklist
- [ ] معالج 4 خطوات: تحقّق كل خطوة + الانتقال.
- [ ] prepare v2: منطقتان مختلفتان (مسار) ونفس المنطقة (random) وبلا أماكن (أيام فارغة).
- [ ] enhanced_data يقسّم صباح/مساء صحيحًا + أوصاف + مدينة اليوم.
- [ ] عرض NewTripPlan: أيام/فترات/بطاقات أماكن صحيحة.
- [ ] تعديل محلي: حذف/نقل مكان، منع حذف آخر مكان باليوم.
- [ ] reprepare: أماكن جديدة مستبعِدة القديمة.
- [ ] Save v2: اسم فريد، items صحيحة، حذف prepareTrip، ظهور بالقائمة.
- [ ] رفض اسم مكرّر لنفس المستخدم.
- [ ] my-trips: ترقيم لانهائي + آخر صفحة.
- [ ] view محفوظ (+ownership 403 لغير المالك).
- [ ] delete (+ownership).
- [ ] email: تحقّق البريد + إرسال.
- [ ] v1 القديم لا يزال يعمل حيث يُستخدم.
- [ ] لغة/RTL على العرض والتواريخ.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «prepare بطيء جدًا/timeout» | Google Maps متزامن | `getRouteWaypoints` / `getRoutePoints` |
| «أماكن مكرّرة عبر الأيام» | نفاد الأماكن → reuse | `organizePlacesByPeriodsAndDays:271` |
| «الأوصاف دائمًا عامة» | `determineCategoryType` مبتور | `EnhancedTripService:363` |
| «بطء قائمة/عرض محضّر» | N+1 | `PrepareTripResource:25` / view per-day |
| «رحلة الآخر ظاهرة (v1)» | view_trip بلا auth | `TripController:358` |
| «التواريخ معكوسة» | daterange end-start | `prepare_trip_wizard_cubit` |
| «حفظ يفشل باسم موجود» | فحص التكرار | `saveEnhancedTrip:251` |
| «مكان مفقود بعد الحفظ» | items المعدّلة محليًا | `extractItemIdsFromEnhancedTrip` |
| «null بترجمة المنطقة» | لا fallback | `EnhancedTripService:93` |

## 36. Debug Guide
1. راقب `POST v2/trips/prepare` وزمنه (اشتبه بـ Google Maps إن بطيء).
2. فعّل `ENHANCED_TRIP_DEBUG`/`ENHANCED_TRIP_DEBUG_SQL` لرؤية الاستعلامات.
3. تتبّع cubits: PrepareTripCubit→Show→Reprepare→SaveV2 (Loading→Success/Failure).
4. افحص `enhanced_data` في الرد (أيام/فترات/أماكن).
5. للترتيب: تحقّق lat/long الأماكن + الموقع (getLocationCoords) + Haversine.
6. للحفظ: تتبّع transaction (Trip::create + prepareTrip.delete) و`items` المرسلة.
7. للملكية: قارن `trip.user_id` بـ `auth id` (403).
8. Google Maps: تحقّق المفتاح/الحصّة والرد (fallback منطقة1 عند الفشل).

## 37. Search Keywords
`trip`, `trips`, `prepare`, `reprepare`, `enhanced trip`, `v2/trips`, `itinerary`, `morning evening`, `period`, `places_per_day`, `places_per_period`, `enhanced_data`, `EnhancedTripService`, `EnhancedTripV2Controller`, `TripController`, `prepare_trip`, `getRouteWaypoints`, `getRoutePoints`, `google maps`, `haversine`, `organizePlacesByPeriodsAndDays`, `findPlacesAlongRoute`, `findPlacesInRegion`, `Trip model`, `prepareTrip`, `is_enhanced`, `PrepareTripRequest`, `SaveEnhancedTripRequest`, `PrepareTripResource N+1`, `PrepareTripWizardCubit`, `PrepareTripShowCubit`, `SaveTripV2Cubit`, `ReprepareTripCubit`, `NewMyTripCubit`, `EnhancedTripResponse`, `NewTripPrepareParams`, `save-trip-to-email`, `SaveNewTrip`, `view_trip`, `my-trips`, `funny_place_per_day`, `daterange`, `token enumeration`, `hardcoded api key` → **03_Trips.md**

## Web Frontend (Angular)
- **Same v2 API:** `POST /v2/trips/prepare`, `save`, `reprepare`, `GET my-trips`, `GET view/{token}`, `DELETE {token}` (`domains/trip/services/trips.service.ts`). v1 legacy present but commented.
- **Enhanced planner inputs:** dates, start/end region, category filters, price range, places-per-day, **vehicleType** (FormData `:51`) — same enhanced-trip concept as mobile.
- **WEB-ONLY: PDF/print export.** `TripPdfService.openTripPlanPdf()` (`trip-pdf.service.ts:16–36`) builds raw HTML → Blob → `window.open` → `window.print()` (not jspdf/html2canvas). Triggered by `TripHeaderComponent` `downloadPdf` event. Mobile uses its own share/email path.
- **Components:** TripsList / TripDetails / SaveTripDetails. See [[WEB_MOBILE_COMPARISON]].

## Known Incidents (live)
- **[[KNOWN_ISSUES#K1]] — SMTP 500 على `trips/save-trip-to-email`:** الإرسال متزامن (`TripController.php:85`)؛ فشل SMTP بالإنتاج (بورت 587 timeout) يعلّق الطلب 31s ويرجّع 500. راجع دَين [[TECH_DEBT#D45]]. الحل: queue + try/catch + إصلاح SMTP/ops.
- تذكير: أرسل body **JSON صحيح** (مفاتيح/قيم باقتباس)؛ الحقول المطلوبة: name, user_name, email, date, days, item_per_day, items.

## 38. Future Improvements
- Queue + cache لنقاط المسار؛ خيار `mock_google_api` للاختبار.
- إصلاح N+1 ودمج استعلامات الأماكن؛ إضافة الفهارس.
- تأمين/إهلاك v1؛ توحيد على v2.
- إكمال تصنيف الفئات لأوصاف ذكية؛ ترجمة البريد + معاينة الرحلة داخله.
- انتهاء صلاحية prepared + تنظيف دوري + سياسة رحلات الضيوف.
- نقل الأسرار لـ env؛ rate-limit؛ حماية view_trip.
- تحسين خوارزمية الترتيب (أوقات فتح، تجميع جغرافي أذكى، تفضيلات المستخدم).

---
**تدفق مرجعي:** UI(معالج) → Cubit → Repo → API → Route → Controller → EnhancedTripService → Model(Trip/prepareTrip) → DB → Response(enhanced_data) → Flutter UI ✅ موثّق لـ prepare/save/view/my-trips/delete/email (v1+v2).
