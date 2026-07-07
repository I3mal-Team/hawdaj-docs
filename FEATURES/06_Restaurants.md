# Feature 06 — Restaurants (Zad / ZadElgadel — مطاعم)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/restaurants) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 **يتبع قالب [[04_Places]]** ويشارك **ZadModel** مع [[05_Stores]]. الفريد هنا: **Menus + Offers + food_categories + menu_file(PDF)**. هذا الملف يركّز على الفريد.

---

## الفريد عن Places/Stores (ملخّص)
| ميزة | التنفيذ | Flutter | Laravel |
|---|---|---|---|
| **Menus** (أصناف بصور/أسعار) | نموذج Menu، ترقيم | MenuCubit + MenuListSection | Menu model, MenuController, menus table |
| **تحميل PDF للمنيو** | توليد client-side | `downloadMenuAsPdf` (pdf package) | — (لا PDF من الخادم) |
| **Offers** (عروض بنافذة زمنية) | خصم + سعر بعد الخصم | FetchOffersCubit + OfferSection | Offer model, OfferController |
| **food_categories** | تصنيف مطبخ | ZadModel.foodCategories | belongsToMany pivot |
| **menu_file** | مرجع PDF منيو | ZadModel.menuFile | zad_elgadels.menu_file (fallback default) |
| الفئات | **belongsToMany** (pivot) لا JSON | — | category_zad_elgadels + food_category_zad_elgadels |

---

## 1. Feature Name
Restaurants (Zad/ZadElgadel) — قائمة/تفاصيل المطاعم مع أصناف المنيو (menus)، العروض (offers)، تصنيفات المطبخ (food_categories)، وملف منيو PDF.

## 2. Business Goal
اكتشاف المطاعم بتفاصيل غنية + استعراض المنيو (صور/أسعار وتحميله PDF) + عروض ترويجية بنافذة زمنية وخصم، والفلترة حسب نوع المطبخ.

## 3. Business Rules
- القائمة: `active=1`. فلاتر: search، region_id، city_id، **food_categories[]** (pivot join)، categories (CSV pivot join)، **rate** (AVG having)، nearest/lat/lng.
- الترتيب: بموقع → مسافة؛ nearest؛ وإلا order_id/id (`applyListingOrder`).
- المنيو: `Menu` belongsTo Zad (zad_id)، مرتّب order_id، مصفّح (10). أصناف بـ price + صورة + عنوان/وصف مترجَم.
- العروض: `Offer` belongsTo Menu، بنافذة `from/to` (DATE) + discount(%). `price_after_discount = menu.price * (discount/100)`. ⚠️ **بلا تحقّق انتهاء server-side** (العميل يفلتر).
- menu_file fallback → `front_assets/imgs/zad1.jpg` إن مفقود.
- الفئات (categories & food_categories) عبر **pivot belongsToMany** (لا JSON — يختلف عن Places/Stores).

## 4. User Flow
Home/Tasneef → بطاقة مطعم (slug+id) → RestaurantsDetailsItems (3 cubits) → تفاصيل + food_categories + قسم منيو (عرض bottom sheet / تحميل PDF) + قسم عروض (كاروسيل مصفّح) + تقييمات.

## 5. Flutter Screens
| Screen | File |
|---|---|
| RestaurantsDetailsItems (scaffold) | `lib/features/restaurants/presentation/view/restaurants_details_items.dart` |
| RestaurantsDetailsItemsBody (UI) | `.../view/widgets/restaurants_details_items_body.dart` |
> يُنشَأ مع 3 cubits من الراوت (`routes.dart:354-387`): RestaurantsDetailsCubit(slug)..getRestaurantsInfo + MenuCubit(id) + FetchOffersCubit(id).

## 6. Widgets
`CustomAppBarHomeDetails` (`:92`)، `DetailsSection` (`:164`)، food categories header (`:174-182`)، **`ListOfSuppliesSection`** (زر تحميل PDF + عرض المنيو bottom sheet)، **`MenuListSection`** (`menu_list_section.dart`، PagedListView صور)، روابط اجتماعية (`:188-215`)، `MapSection` (`:219`، مشروط `!isOnline`)، **`OfferSection`/`OffersAvailableItems`** (`:232-241`، كاروسيل عروض)، كاروسيل تقييمات (`:244-315`)، `ShareYourRateButton`، `BookTicketButtomWidget`.

## 7. Cubits / Blocs
| Cubit | File | States |
|---|---|---|
| RestaurantsDetailsCubit | `.../manager/restaurants_cubit/restaurants_details_cubit.dart:8-24` | Initial→Loading→Success(ZadModel)/Error |
| **MenuCubit** | `.../manager/menu_cubit/menu_cubit.dart:10-83` | Initial→Loading(PagingController)→Error + **Download InProgress/Success/Failure**؛ auto `_fetchPage(1)` |
| **FetchOffersCubit** | `.../manager/fetch_offers_cubit/fetch_offers_cubit.dart:9-60` | Initial→Loading(PagingController)→Error |

## 8. State Flow
التفاصيل: getRestaurantsInfo → Loading → repo.fetchRestaurants(slug) → fold → Success/Error؛ listeners مفضّل/حفظ/تقييم تُحدّث. المنيو/العروض: PagingController auto → `_fetchPage` → appendPage/LastPage. تحميل PDF: `downloadMenu()` → يجمع كل الأصناف → `downloadMenuAsPdf` → Success/Failure.

## 9. Models
- **ZadModel** (مشترك): + `foodCategories[]`، `menuFile`، (كل حقول Store). type='zad'.
- **MenuItemModel** (`menu_item_model.dart`): title, price, description, image.
- **OfferItemModel** (`offer_item_model.dart`): image, from, to, title, description?, discount(int), price, priceAfterDiscount(num).
- `PaginatedResponse<MenuItemModel/OfferItemModel>`.

## 10. Repositories
- **Flutter** `restaurants_repo.dart` + `restaurants_repo_imp.dart:12-50`: `fetchRestaurants(slug)` → GET `zads/{slug}` → ZadModel · `fetchMenu(id,page)` → GET `zads/{id}/menus` → Paginated<MenuItemModel> · `fetchOffers(id,page)` → GET `zads/{id}/offers` → Paginated<OfferItemModel>.
- **Laravel:** لا repository — controllers.

## 11. Services
لا service مخصّص. مساعِدات App.php (distance/listing/isFavorite). client PDF عبر `download_menu.dart` (http.get صور + pdf + path_provider temp).

## 12. API Endpoints
| Method | Endpoint | Auth |
|---|---|---|
| GET | `zads` (قائمة+فلاتر) | عام |
| GET | `zads/{slug}` (تفاصيل) | عام |
| GET | `zads/{id}/menus?page=&perPage=` | عام |
| GET | `zads/{id}/offers` | عام |
| GET | `menus/{id}` (صنف) | عام |
| GET | `zads/categories` · `zads/food-categories` | عام |

## 13. Request Models
لا body. query: search, region_id, city_id, **food_categories[]**, categories(CSV), rate, nearest, lat, lng, per_page (list)؛ title, per_page (menus)؛ per_page (offers).

## 14. Response Models
- قائمة: `ZadCollection` → items[ZadListResource].
- تفاصيل: `ZadResource` (+food_categories، menu_file، is_favorite/is_saved).
- منيو: `MenuCollection` → [MenuResource{id,order_id,title,price,description,image}].
- عروض: `OfferCollection` → [OfferResource{id,image,from,to,title,description,discount,price(menu.price),price_after_discount}].

## 15. Laravel Routes
`routes/api.php:105-111`: `GET zads → ZadController@index` · `zads/categories → ZadCategoryController` · `zads/food-categories → ZadFoodCategoryController` · `zads/{zadElgadel} → show` · `zads/{zad}/offers → OfferController@offers` · `zads/{zad}/menus → MenuController@getMenusByZad` · `menus/{menu} → MenuController@show`. عامة.

## 16. Controllers
- **ZadController** (`app/Http/Controllers/Api/ZadController.php`): index (`:14-75`، فلاتر + **food_categories/categories عبر join pivot** + rate AVG having + distance + paginate، eager `ratings,galleries,city,region,ceo`)، show (`:77-82`، firstOrFail).
- **MenuController** (`:9-32`): getMenusByZad($id) (where zad_id + title? + listing order + paginate(10) onEachSide(2))، show(Menu).
- **OfferController** (`:18-29`): offers($zad) (join offers→menus→zad_elgadels + with('menu') + listing order + paginate).

## 17. Service Classes (Laravel)
لا service. مساعِدات App.php. ZadCategoryController/ZadFoodCategoryController للفئات.

## 18. Models (Laravel)
- **ZadElgadel** (`app/Models/ZadElgadel.php:17-177`): translatable(title,description)، SoftDeletes، `TYPE_ZAD_ELGADEL='Zad_elgadel'`. casts: categories/food_categories/related_stores/near_places → array (لكن الفلترة تستخدم pivot). appends rate/review/type('zad'). relations: region/city/user/ceo/ratings(type='zad')/galleries(type='zad_elgadels')/**foodCategories(belongsToMany)**/categories(belongsToMany)/favorites(morph). accessor `getMenuFileAttribute` (fallback default).
- **Menu** (`app/Models/Menu.php`): translatable(title,description)، fillable(image,zad_id,price,order_id)، belongsTo Zad. `translationForeignKey=menu_id`.
- **Offer** (`app/Models/Offer.php`): translatable(title,description)، fillable(image,to,from,discount,menu_id,order_id)، belongsTo Menu.

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `zad_elgadels` | id, title/description(→trans), categories/related_stores/near_places(JSON legacy), address, image, روابط اجتماعية، x_link, lat/long, active, address_type, featured/visited, views_num, city_id/region_id, **menu_file**, ownership_proof_file, user_id, ticket_link, order_id, softDeletes |
| `zad_elgadels_translations` | zad_elgadel_id, locale, title, description |
| `menus` | id, image, zad_id(FK), **price(DECIMAL)**, order_id, softDeletes |
| `menu_translations` | menu_id, locale, title, description، unique |
| `offers` | id, image, **from(DATE), to(DATE), discount(DOUBLE)**, menu_id(FK), order_id, softDeletes |
| `offer_translations` | offer_id, locale, title, description، unique |
| `category_of_zads` / `food_category_of_zads` | فئات هرمية (translatable) |
| `category_zad_elgadels` / `food_category_zad_elgadels` | **pivot** (zad_elgadel_id + category_id/food_category_id) |

## 20. Relationships
ZadElgadel BelongsToMany CategoryOfZad + FoodCategoryOfZad (pivots) · HasMany Menu(zad_id) — Menu HasMany Offer(menu_id) — Offer belongsTo Menu · HasMany Gallery/Rate · HasOne Ceo · MorphMany Favorite · BelongsTo Region/City/User. سلسلة العروض: Offer→Menu→Zad.

## 21. Validation Rules
فلاتر index متساهلة (Request عادي). لا FormRequest صارم. المنيو title اختياري. العروض بلا فلترة زمنية server-side.

## 22. Authorization & Permissions
عام بالكامل. is_favorite/is_saved per-user. ⚠️ `show` لا يفلتر active (كشف بالـ slug).

## 23. Error Handling
- **Flutter:** `Either<Failure,T>`؛ RestaurantsDetailsError، MenuError، OffersError، MenuDownloadFailure ("لا يوجد منتجات"/"فشل في التحميل").
- **Laravel:** firstOrFail → 404؛ الغلاف ApiModalController.

## 24. Edge Cases
- لا منيو → "no_zads" indicator؛ تحميل PDF فارغ → "لا يوجد منتجات".
- عروض منتهية → **تُعرَض كما هي** (لا فلترة server؛ العميل يجب أن يفلتر `today∈[from,to]`).
- لا food_categories → قائمة فارغة.
- menu_file مفقود → صورة افتراضية.
- 404 لمطعم غير موجود.
- مطعم أونلاين → لا خريطة.
- ترجمة ناقصة → fallback.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, infinite_scroll_pagination، **pdf + http + path_provider** (تحميل المنيو)، cached_network_image، google_maps، url_launcher. Laravel: astrotomic/translatable، spatie/medialibrary، DB joins/raw.

## 26. Files Involved
**Flutter:** `lib/features/restaurants/**` (models: menu_item/offer_item؛ repo؛ cubits: restaurants_details/menu/fetch_offers؛ views + widgets: menu_list_section, download_menu)، `lib/features/home/data/model/zad_model/zad_model.dart`، widgets home (list_of_supplies_section, offers_available_items, offer_section)، `end_points.dart`، `routes.dart:354-387`.
**Laravel:** `app/Http/Controllers/Api/{ZadController,MenuController,OfferController,ZadCategoryController,ZadFoodCategoryController}.php`، `app/Models/{ZadElgadel,Menu,Offer,CategoryOfZad,FoodCategoryOfZad}.php`، `app/Http/Resources/{Zads/ZadResource,ZadListResource,ZadCollection, Menu/MenuResource,MenuCollection, Offers/OfferResource,OfferCollection}.php`، migrations (zad_elgadels/menus/offers +trans، pivots)، `routes/api.php:105-111`.

## 27. Complete Execution Flow
**التفاصيل:** getRestaurantsInfo → GET `zads/{slug}` → ZadController@show (firstOrFail) → ZadResource (food_categories + menu_file + rate/review + is_favorite/is_saved) → ZadModel → Success → Body.
**المنيو:** MenuCubit auto `_fetchPage(1)` → GET `zads/{id}/menus?page=1` → MenuController@getMenusByZad (where zad_id + listing + paginate) → MenuCollection → appendPage → MenuListSection. تحميل: downloadMenu → downloadMenuAsPdf(http.get صور + pdf) → temp/menu.pdf.
**العروض:** FetchOffersCubit `_fetchPage(1)` → GET `zads/{id}/offers` → OfferController@offers (join offers→menus→zad + with menu + paginate) → OfferResource (price_after_discount = menu.price*discount/100) → OfferSection.

## 28. Performance Considerations
- 🟠 `getMenusByZad` بـ `with('zad')` لكن **بلا `with('offers')`** — N+1 إن وُصلت العروض من المنيو.
- 🟠 `allCategories/allFoodCategories` تستعلم لكل عنصر (N+1 في القوائم).
- 🟠 `rate` filter عبر AVG having + leftJoin rates — قد يبطئ.
- ✅ index eager loading؛ ترقيم على الكل.
- 🟡 تحميل PDF يجلب كل صور المنيو تسلسليًا (بطيء لمنيو كبير).

## 29. Security Considerations
1. 🟠 **ratings خام في ZadResource** → تسريب email (مطابق Places/Stores).
2. 🟠 **`show` لا يفلتر active** → كشف مطعم غير منشور بالـ slug.
3. 🟠 **عروض منتهية تُعرَض** (لا فلترة زمنية server) — قد تُظهر خصمًا غير صالح.
4. 🟡 القراءة عامة.

## 30. Technical Debt
- عمودا categories/food_categories (JSON legacy) موجودان مع pivot tables (ازدواج — الفلترة تستخدم pivot).
- تسمية `Instagram_link` بحرف كبير (كما Stores).
- ZadModel موحّد ضخم (يخدم store+zad).
- منطق العروض لا يتحقّق من النافذة الزمنية (يعتمد على العميل).
- `page` في endpoints menus/offers ثابت أحيانًا (يُدار عبر PagingController).
- ratings خام (يحتاج RatingResource).

## 31. Improvement Opportunities
- فلترة عروض صالحة server-side (`whereDate to >= today`).
- `with('offers')` في getMenusByZad عند الحاجة؛ إزالة N+1 (allCategories).
- RatingResource يخفي email (مشترك).
- فلترة active في show.
- توليد PDF منيو server-side (أسرع/موحّد) بدل client.
- توحيد type ('zad' accessor مقابل 'Zad_elgadel' في favorites، و rates type='zad').
- إزالة أعمدة JSON legacy لصالح pivot.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 04 Places / 05 Stores | نفس القالب؛ 05 يشارك ZadModel |
| 02 Home | zads/topVisitedStores في home + offers |
| 07 Tasneef | قائمة zads |
| 08 Exploration | ضمن الخريطة/البحث |
| 13 Favorites / 14 Rates | مفضّل/حفظ/تقييم (type 'zad'/'Zad_elgadel') |
| 18 Taxonomy | CategoryOfZad + FoodCategoryOfZad الهرمية |
> ⚠️ عدم اتساق type: accessor 'zad'، favorites 'Zad_elgadel'، rates 'zad'، galleries/ceo 'zad_elgadels'.

## 33. How to Modify This Feature
- **صنف منيو/عرض جديد:** Menu/Offer model + migration + Resource + MenuItemModel/OfferItemModel + widget.
- **فلتر مطبخ:** ZadController@index (join pivot) + food-categories endpoint.
- **حقل مطعم:** migration zad_elgadels + ZadElgadel + ZadResource + ZadModel.
- **APIs:** zads, zads/{slug}, zads/{id}/menus, zads/{id}/offers، food-categories.
- **المخاطر:** N+1 (menus→offers, allCategories)، عروض منتهية، تسريب email، عدم اتساق type، ازدواج JSON/pivot.
- **اختبارات:** تفاصيل، منيو مصفّح + تحميل PDF، عروض (صالحة/منتهية)، فلتر food_categories، مفضّل/تقييم.

## 34. Regression Checklist
- [ ] تفاصيل المطعم: معرض/تقييم/food_categories/وصف/خريطة(إن فيزيائي).
- [ ] المنيو: ترقيم لانهائي + عرض bottom sheet + تحميل PDF.
- [ ] تحميل PDF فارغ → رسالة "لا يوجد منتجات".
- [ ] العروض: كاروسيل مصفّح + خصم + سعر بعد الخصم + نافذة زمنية.
- [ ] فلتر food_categories/categories يعمل.
- [ ] فلتر rate (متوسط).
- [ ] مفضّل/حفظ/تقييم → إعادة تحميل.
- [ ] 404 لمطعم غير موجود؛ menu_file مفقود → افتراضي.
- [ ] لغة/RTL + ترجمة المنيو/العروض.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «عرض منتهي ظاهر» | لا فلترة زمنية server | `OfferController@offers` + العميل |
| «سعر بعد الخصم خاطئ» | menu.price*discount/100 | `OfferResource:price_after_discount` |
| «المنيو فارغ» | zad_id/ترقيم | `MenuController@getMenusByZad` |
| «تحميل PDF يفشل» | صور غير متاحة/بطيئة | `download_menu.dart` http.get |
| «فلتر المطبخ لا يعمل» | join pivot | `ZadController@index food_categories` |
| «حالة مفضّل خاطئة» | type 'zad' vs 'Zad_elgadel' | isFavorite + TYPE_ZAD_ELGADEL |
| «email المقيّم ظاهر» | ratings خام | `ZadResource` |
| «menu_file مكسور» | مسار/fallback | `getMenuFileAttribute` |

## 36. Debug Guide
1. راقب `zads/{slug}`, `zads/{id}/menus`, `zads/{id}/offers`.
2. تتبّع 3 cubits (Details/Menu/Offers) + PagingController.
3. للعروض: تحقّق from/to/discount والحساب.
4. للمنيو PDF: تتبّع downloadMenu → downloadMenuAsPdf (صور http).
5. للفلاتر: افحص join pivot food_category_zad_elgadels + SQL.
6. للنوع/المفضّل: قارن accessor 'zad' بـ TYPE_ZAD_ELGADEL و rates type='zad'.
7. DB::listen لرصد N+1 (menus→offers, allCategories).

## 37. Search Keywords
`zad`, `zads`, `zad_elgadel`, `ZadElgadel`, `restaurant`, `مطعم`, `ZadController`, `MenuController`, `OfferController`, `ZadResource`, `MenuResource`, `OfferResource`, `MenuCubit`, `FetchOffersCubit`, `RestaurantsDetailsCubit`, `MenuItemModel`, `OfferItemModel`, `menu`, `menus`, `offer`, `offers`, `food_categories`, `FoodCategoryOfZad`, `CategoryOfZad`, `menu_file`, `downloadMenuAsPdf`, `price_after_discount`, `discount`, `from to offer`, `food_category_zad_elgadels`, `category_zad_elgadels`, `zads/{id}/menus`, `zads/{id}/offers`, `TYPE_ZAD_ELGADEL`, `getMenuFileAttribute` → **06_Restaurants.md**

## 38. Future Improvements
- فلترة عروض صالحة server-side + توثيق النافذة الزمنية.
- توليد PDF منيو server-side؛ إزالة N+1 (menus→offers/allCategories).
- RatingResource (مشترك) يخفي email؛ فلترة active في show.
- توحيد type عبر الكيان.
- إزالة أعمدة JSON legacy لصالح pivot.
- كاش للفئات/الفئات الغذائية (نادرة التغيّر).

---
**تدفق مرجعي:** UI(3 cubits) → Repo → API → Route → {Zad,Menu,Offer}Controller → Models(ZadElgadel/Menu/Offer) → DB(+pivots) → Response → Flutter UI ✅ موثّق لـ detail/menus/offers. **الفريد عن [[04_Places]]/[[05_Stores]]: menus + offers + food_categories + menu_file + pivot categories.**
