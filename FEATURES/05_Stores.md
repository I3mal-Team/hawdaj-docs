# Feature 05 — Stores (المتاجر)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/stores) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 **يتبع قالب [[04_Places]]** (translatable + media + gallery + rates + favorites/saved + distance + slug detail). هذا الملف يركّز على **الفروق**.

---

## الفروق الجوهرية عن Places (ملخّص سريع)
| المحور | Places | **Stores** |
|---|---|---|
| النموذج (Flutter) | Place/UnifiedPlaceModel | **ZadModel** (موحّد Stores+Zad) |
| Type ثابت | 'Place' | **'Store'** (`Store::TYPE_STORE`) |
| `is_online` | ❌ | ✅ boolean (متجر إلكتروني) |
| الفئات | `Category` مسطّح | **`CategoryOfStore` هرمي** (parent_id) |
| علاقة "قريب" | near_stores | **near_places** |
| علاقة "مرتبط" | related_places | **related_stores** |
| endpoint إضافي | — | **`stores/stats`** (عدد online/offline) |
| الخريطة بالتفاصيل | دائمًا | **تُخفى إن is_online=true** |
| العنوان | دائمًا | **`'store_online'` إن أونلاين** |
| حقول خاصة بالآخر | seasons/price/temperature/prefered (Places فقط) | — |

---

## 1. Feature Name
Stores — قائمة المتاجر (بحث/فلاتر بينها `is_online` + ترتيب جغرافي + ترقيم) + تفاصيل بالـ slug + إحصائية online/offline.

## 2. Business Goal
اكتشاف المتاجر (فيزيائية وإلكترونية)، بتفاصيل غنية وتصنيف هرمي، مع تمييز المتجر الأونلاين (بلا خريطة/موقع).

## 3. Business Rules
- القائمة/التفاصيل: `active=1` فقط.
- `is_online`: يخفي الخريطة ويعرض "store_online"؛ في StoreListResource fallback: `is_online ?? (!lat && !long)`.
- فلاتر: search، category_id (JSON_CONTAINS)، region_id، city_id، address_type، **is_online**، حدود خريطة (lat/lng±x/y).
- الترتيب: بموقع → مسافة؛ top_visited → views_num DESC؛ top_featured؛ وإلا order_id/id.
- `stats`: عبر `DB::table('stores')->selectRaw(SUM CASE is_online)` حيث active=1.
- related_stores/near_places كـ IDs (JSON).

## 4. User Flow
Home/Tasneef/Search → بطاقة متجر → `stores/{slug}` → StoresDetailsItems → تفاصيل (خريطة تُخفى إن أونلاين) → مفضّل/حفظ/مشاركة/اجتماعي/تقييم.

## 5. Flutter Screens
| Screen | File |
|---|---|
| StoresDetailsItems (scaffold + BlocBuilder) | `lib/features/stores/presentation/view/stores_details_items.dart` |
| StoresDetailsItemsBody (UI، CustomScrollView) | `.../view/widgets/stores_details_items_body.dart:38-306` |
> لا list cubit مخصّص — القائمة عبر Tasneef/Home.

## 6. Widgets
يعيد استخدام widgets Places/Home: `CustomAppBarHomeDetails` (معرض+أزرار، `:91`)، `DetailsSection` (`:146`، فرع is_online للعنوان)، `SocialLinkItem` (`:154-210`)، `MapSection` (`:213`، مشروط `!isOnline`)، كاروسيل تقييمات (`:225-287`)، `ShareYourRateButton` (`:291`)، `BookTicketButtomWidget` (`:300`).

## 7. Cubits / Blocs
- **StoresDetailsCubit** (`stores_details_cubit.dart:8-23`): `(storesRepo, slug)`، `getStoresInfo()` → Loading→Success(ZadModel)/Error. States في `stores_details_state.dart`.
- تكامل مفضّل/حفظ/تقييم عبر MultiBlocListener (`:59-87`) → أي نجاح يستدعي `getStoresInfo()`.
- القائمة: عبر cubits مشتركة (Home/Tasneef).

## 8. State Flow
مطابق لـ Places: constructor → getStoresInfo → Loading → repo.fetchStores(slug) → fold → Success/Error. listeners تُحدّث بعد مفضّل/حفظ/تقييم.

## 9. Models
**ZadModel** (`lib/features/home/data/model/zad_model/zad_model.dart`): موحّد Stores+Zad. حقول مشتركة (id, slug, type, categories, foodCategories, address, image, menuFile, روابط اجتماعية، lat/long, featured/visited/viewsNum/rate/reviewCount, title/description, ratings, galleries, city, region, meta, isFavorite, isSaved, ticketLink, addressType, distance). **خاصة بـ Store:** `isOnline`, `relatedPlaces`, `nearPlaces`, `active` (fromJson `:89-146`).

## 10. Repositories
- **Flutter** `stores_repo.dart` + `stores_repo_imp.dart:9-21`: `fetchStores(slug)` → GET `stores/{slug}` → ZadModel.
- **Laravel:** لا repository — منطق في StoreController.

## 11. Services
لا service مخصّص. يعيد استخدام `getLocationCoords`/`applyDistanceSorting`/`applyListingOrder`/`isFavorite`/`isSaved`. يشارك **PlaceObserver** (boot).

## 12. API Endpoints
| Method | Endpoint | Auth | ملاحظة |
|---|---|---|---|
| GET | `stores` (قائمة+فلاتر) | عام | + is_online filter |
| GET | `stores/{slug}` (تفاصيل) | عام | |
| GET | `stores/stats` | عام | **خاص بالمتاجر** — online/offline |
| GET | `stores/categories` | عام | CategoryOfStore |
| GET | `stores/{store}/related-stores` | عام | |

## 13. Request Models
لا body للقراءة. query params: search, category_id, region_id, city_id, address_type, **is_online**, lat, lng, x, y, per_page, top_visited, top_featured.

## 14. Response Models
- قائمة: `StoreCollection` → `{current_page, items[StoreListResource], total, ...}`.
- تفاصيل: `StoreResource` (يضيف `is_online`؛ related_places/near_places = `[]` غير مُروّاة).
- stats: `{stores:{total_offlines, total_onlines}}`.

## 15. Laravel Routes
`routes/api.php`: `GET stores → index` · `GET stores/categories → StoreCategoryController@index` · `GET stores/stats → stats` · `GET stores/{store}/related-stores → related_stores` · `GET stores/{store} → show`. عامة.

## 16. Controllers
**StoreController** (`app/Http/Controllers/Api/StoreController.php`): `stats` (:16-33)، `index` (:35-117، فلاتر+is_online+ترتيب+ترقيم، eager `ratings,galleries,city,region,ceo`)، `show` (:119-124، firstOrFail، ⚠️ **بلا eager loading → N+1**)، `related_stores` (:132-148). الفرق عن PlaceController: is_online filter + stats + بلا seasons/price.

## 17. Service Classes (Laravel)
لا service. مساعِدات App.php المشتركة. `StoreCategoryController` للفئات الهرمية.

## 18. Models (Laravel)
**Store** (`app/Models/Store.php:18-153`): translatable (`title,description`)، `HasImage`, `HasPlaceTranslationUpdate`, SoftDeletes، يشارك `PlaceObserver`. casts: categories/related_stores/near_places → array، **is_online → boolean**. appends: rate/review/type('store'). relations: region/city/user/**category(CategoryOfStore)**/galleries/ceo/ratings/favorites(morph). media: image + small(250)/medium(500).
**CategoryOfStore** (`app/Models/CategoryOfStore.php`): translatable (name,notes)، **هرمي** (parent()/childes()/allParents()).

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `stores` | id, categories(JSON), related_stores(JSON), near_places(JSON), address, image, روابط اجتماعية، lat/long, active, address_type(enum), **is_online(bool, default 0)**, region_id/city_id(FK), timestamps, deleted_at |
| `store_translations` | id, store_id(FK cascade), locale, title, description، unique(store_id,locale) |
| `category_of_stores` | id, parent_id(هرمي), icon, timestamps |
| `category_of_store_translations` | id, category_of_store_id(FK), locale, name, notes، unique |

## 20. Relationships
Store BelongsTo Region/City/User/**CategoryOfStore** · HasMany Gallery(type=stores)/Rate(type=store) · HasOne Ceo · MorphMany Favorite · translations. CategoryOfStore self-referential (parent/children). related_stores/near_places كـ IDs.

## 21. Validation Rules
فلاتر index متساهلة (Request عادي، لا FormRequest صارم مثل Places IndexRequest). is_online يُفلتر كـ bool. الإنشاء (لو موجود) يبدأ active=0.

## 22. Authorization & Permissions
عام بالكامل. is_favorite/is_saved per-user. ⚠️ `show` لا يفلتر active (كشف غير المنشور بالـ slug — مطابق لعيب Places).

## 23. Error Handling
- **Flutter:** `Either<Failure,T>`؛ StoresDetailsError.
- **Laravel:** firstOrFail → 404؛ الغلاف ApiModalController.

## 24. Edge Cases
- متجر أونلاين → لا خريطة، عنوان "store_online".
- is_online غير مضبوط → استنتاج من غياب الإحداثيات (StoreListResource).
- بلا موقع → order_id/id.
- related_stores/near_places فارغة → قوائم فارغة (بل تُعاد `[]` دائمًا حتى لو موجودة IDs — غير مُروّاة).
- ⚠️ `show` بلا active filter + بلا eager loading (N+1).
- ratings **خام** → تسريب email (مطابق لـ Places).
- ترجمة ناقصة → fallback.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, google_maps (مشروط)، url_launcher، share، cached_network_image. Laravel: astrotomic/translatable، spatie/medialibrary، DB raw (stats)، JSON_CONTAINS.

## 26. Files Involved
**Flutter:** `lib/features/stores/**`، `lib/features/home/data/model/zad_model/zad_model.dart`، widgets مشتركة من home/places/events، `end_points.dart` (stores, detailsStores).
**Laravel:** `app/Http/Controllers/Api/StoreController.php` (+StoreCategoryController)، `app/Models/{Store,CategoryOfStore,CategoryOfStoreTranslation}.php`، `app/Http/Resources/Stores/{StoreResource,StoreListResource,StoreCollection}.php`، migrations stores/store_translations/category_of_stores(+trans)، `routes/api.php`.

## 27. Complete Execution Flow
**القائمة:** (Home/Tasneef)Cubit → GET `stores?filters(+is_online)` → StoreController@index (active=1 + eager + فلاتر + distance/listing + paginate) → StoreCollection → ZadModel list → UI.
**التفاصيل:** getStoresInfo → GET `stores/{slug}` → StoreController@show (firstOrFail) → StoreResource (is_online + rate/review + is_favorite/is_saved + galleries/ratings/meta) → ZadModel.fromJson → StoresDetailsSuccess → StoresDetailsItemsBody (خريطة مشروطة بـ !isOnline).
**stats:** GET `stores/stats` → selectRaw → `{total_onlines,total_offlines}`.

## 28. Performance Considerations
- 🔴 **`show` بلا eager loading** → N+1 (ratings/galleries/city/region/ceo تُحمَّل عند الوصول). الإصلاح: `->with(...)` مثل index.
- 🟠 `allCategories()` N+1 محتمل في القوائم.
- 🟠 `ceo` eager في index رغم قلة استخدامه.
- 🟠 ratings كاملة بلا حد في التفاصيل.
- ✅ index يعمل eager loading؛ stats استعلام واحد.

## 29. Security Considerations
1. 🟠 **ratings خام في StoreResource** → تسريب email المقيّمين.
2. 🟠 **`show` لا يفلتر active** → كشف متجر غير منشور بالـ slug.
3. 🟡 القراءة عامة (مقبول).
4. 🟡 لا تحقّق تنافُر is_online مع الإحداثيات (متجر أونلاين بإحداثيات ممكن).

## 30. Technical Debt
- `show` ينقص eager loading (يوجد في index) — عدم اتساق.
- related_places/near_places تُعاد `[]` دائمًا (غير مُروّاة) رغم وجود IDs.
- ZadModel موحّد يخدم Stores+Zad (تعقيد).
- عمود `Instagram_link` بحرف كبير (تسمية غير متسقة).
- ratings خام (كما Places) — يحتاج RatingResource.
- image legacy + MediaLibrary.

## 31. Improvement Opportunities
- إضافة eager loading لـ `show`.
- ترواية related_stores/near_places فعليًا أو توثيق أنها IDs.
- RatingResource يخفي email (مشترك مع Places).
- فلترة active في show.
- تحقّق منطق is_online (تنافُر مع الإحداثيات).
- FormRequest للفلاتر (توحيد مع Places IndexRequest).

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 04 Places | نفس القالب؛ near_places يربط Store→Places |
| 06 Restaurants (Zad) | يشارك ZadModel نفسه |
| 02 Home | topVisitedStores في home |
| 07 Tasneef | قائمة stores |
| 08 Exploration | global-map-data يشمل المتاجر |
| 13 Favorites / 14 Rates | مفضّل/حفظ/تقييم المتجر (type='store'/'Store') |
| 18 Taxonomy | CategoryOfStore الهرمي، region/city |
> ملاحظة عدم اتساق type: النموذج يُعيد `'store'` (accessor) بينما favorites تستخدم `Store::TYPE_STORE='Store'` — انتبه عند مقارنة النوع.

## 33. How to Modify This Feature
- **حقل جديد:** migration stores + Store casts + StoreResource + ZadModel fromJson + widget.
- **فلتر جديد:** StoreController@index + بناء query params.
- **فئة هرمية:** CategoryOfStore + StoreCategoryController.
- **APIs:** stores, stores/{slug}, stores/stats, related-stores, stores/categories.
- **المخاطر:** N+1 في show، تسريب email، كشف active، ازدواج ZadModel مع Zad، عدم اتساق type ('store' vs 'Store').
- **اختبارات:** قائمة (+is_online)، تفاصيل (أونلاين/فيزيائي)، stats، مفضّل/حفظ/تقييم، related/categories.

## 34. Regression Checklist
- [ ] قائمة stores: فلاتر + is_online + ترتيب + ترقيم.
- [ ] متجر أونلاين: لا خريطة + عنوان "store_online".
- [ ] متجر فيزيائي: خريطة + اتجاهات.
- [ ] تفاصيل بالـ slug: معرض/تقييم/اجتماعي/تقييمات.
- [ ] stats: أعداد online/offline صحيحة.
- [ ] الفئات الهرمية (parent/children).
- [ ] مفضّل/حفظ/تقييم → إعادة تحميل.
- [ ] 404 لمتجر غير موجود.
- [ ] لغة/RTL + ترجمة.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «خريطة تظهر لمتجر أونلاين» | فرع is_online | `stores_details_items_body.dart:213` |
| «بطء التفاصيل» | N+1 (show بلا eager) | `StoreController@show:119` |
| «حالة مفضّل خاطئة» | type 'store' vs 'Store' | isFavorite + Store::TYPE_STORE |
| «is_online خاطئ» | fallback الإحداثيات | `StoreListResource:56` |
| «email المقيّم ظاهر» | ratings خام | `StoreResource:61` |
| «related فارغة دائمًا» | غير مُروّاة | StoreResource related_places/near_places |
| «stats خاطئ» | active filter | `StoreController@stats` |

## 36. Debug Guide
1. راقب `GET stores?...`, `stores/{slug}`, `stores/stats`.
2. تحقّق is_online في الرد وفرع الخريطة.
3. تتبّع StoresDetailsCubit (Loading→Success/Error).
4. للأداء: DB::listen لرصد N+1 في show.
5. للفئات: تحقّق CategoryOfStore parent/children.
6. للنوع: قارن accessor type ('store') بـ TYPE_STORE ('Store') في favorites.

## 37. Search Keywords
`store`, `stores`, `StoreController`, `StoreResource`, `StoreListResource`, `StoreCollection`, `StoresDetailsCubit`, `stores/{slug}`, `detailsStores`, `stores/stats`, `is_online`, `store_online`, `CategoryOfStore`, `related_stores`, `near_places`, `StoreCategoryController`, `ZadModel`, `Store::TYPE_STORE`, `store_translations`, `category_of_stores`, `stores_details_items_body`, `PlaceObserver`, `getRateAttribute`, `isFavorite Store` → **05_Stores.md**

## Web Frontend (Angular)
- **Same core API:** `/stores`, `/stores/{id}`, `/stores/{id}/related-stores`, `/stores/categories`, `/stores/stats`, `/stores/favorite`, `/rates` (`stores.service.ts`). Components `StoresList`/`StoreDetails`.
- **WEB-ONLY AI ChatGPT:** `getChatGpt`/`addChatGpt` on Store detail (`stores.service.ts:78–82`, `store-details.component.ts:589–602`) — see [[32_AiAssistant]].
- **Filters:** region/city/category/is_online/top-visited/top-featured (same as mobile). See [[WEB_MOBILE_COMPARISON]].

## 38. Future Improvements
- eager loading في show؛ ترواية related/near فعليًا.
- RatingResource (مشترك) يخفي email.
- توحيد type ('store' vs 'Store').
- فلترة active في show؛ FormRequest للفلاتر.
- تحقّق منطق is_online؛ تصحيح تسمية `Instagram_link`.
- توحيد الوسائط على MediaLibrary.

---
**تدفق مرجعي:** UI → Cubit → Repo → API → Route → StoreController → Store/CategoryOfStore → DB(stores/store_translations) → Response(is_online) → Flutter UI ✅ موثّق. **الفرق الأساسي عن [[04_Places]]: is_online + CategoryOfStore الهرمي + stats + near_places.**
