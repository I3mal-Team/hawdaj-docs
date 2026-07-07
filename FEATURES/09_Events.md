# Feature 09 — Events (فعاليات)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/events) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 **يتبع قالب [[04_Places]]**. الفريد: **نافذة زمنية (date_from/date_to)** + **ترتيب زمني CASE (جارية→قادمة→منتهية)** + display_type (banner/top_bar) + status/rejected_reason (UGC) + book ticket. هذا الملف يركّز على المنطق الزمني.

---

## الفريد عن Places/Stores (ملخّص)
| المحور | Places/Stores | **Events** |
|---|---|---|
| نافذة زمنية | ❌ | **date_from/date_to** + widget OfferDateRange |
| فلترة تاريخ | ❌ | **daterange** (تداخل) |
| ترتيب | مسافة/views | **CASE: جارية(1)→قادمة(2)→منتهية(3) ثم date_from** |
| نوع العرض | type='place' | **display_type='banner'/'top_bar'** (للـhome) |
| home | latest/views | **date_to>today فقط** (يستبعد المنتهية) |
| UGC | محدود | **status + rejected_reason + ownership_proof + user_id** |
| closed flag | ❌ | **`!now.between(from,to)`** |

## 1. Feature Name
Events — قائمة/تفاصيل الفعاليات ذات نافذة زمنية، بترتيب زمني ذكي، مع حجز تذاكر خارجي.

## 2. Business Goal
عرض الفعاليات (جارية/قادمة) بأولوية زمنية، ربطها بالمفضّلة/التقييم، وتوجيه المستخدم لحجز التذاكر.

## 3. Business Rules
- `active=1`. القائمة تشمل الكل مع ترتيب زمني؛ home يستبعد المنتهية (`date_to>today`) ويرتّب بـviews_num.
- الحالات: جارية (`from<=now<=to`)، قادمة (`from>now`)، منتهية (`to<now`).
- ترتيب index: `CASE WHEN جارية THEN 1 WHEN قادمة THEN 2 ELSE 3 END, date_from ASC, order_id, id`.
- فلتر daterange: تداخل (event يلفّ النافذة OR داخلها).
- `closed = !now.between(from,to)` (يُحسب في الطرفين).
- display_type/type = banner|top_bar (فلتر address_type).
- UGC: مقدَّم من مستخدم يبدأ بـstatus + rejected_reason (⚠️ لا يُفلتر بـstatus='approved' في index — راجع §29).

## 4. User Flow
Home(topEvents)/Tasneef/Search → بطاقة فعالية (slug) → EventsDetailsView → معرض + نافذة تاريخ + روابط اجتماعية + خريطة + تقييمات + حجز تذكرة.

## 5. Flutter Screens
| Screen | File |
|---|---|
| EventsDetailsView (غلاف) | `lib/features/events/presentation/view/events_details_view.dart:1-29` |
| EventsDetailsItemsBody (UI) | `.../view/widgets/events_details_items_body.dart:1-328` |
> لا شاشة قائمة مخصّصة — القوائم عبر Tasneef/Home.

## 6. Widgets
`CustomAppBarHomeDetails` (`:94-142`)، `DetailsSection` (`:145`، `button:event.closed`)، **`OfferDateRange`** (`:154-180`، تاريخ بداية أخضر/نهاية أحمر — فريد)، **`SocialLinkItem`** ×5 (fb/linkedin/instagram/whatsapp/website، `:182-235`)، `MapSection` (`:244`)، كاروسيل تقييمات (`:247-309`)، `ShareYourRateButton`، **`BookTicketButtomWidget`** (`:322`، إن ticketLink).

## 7. Cubits / Blocs
**EventsDetailsCubit** (`events_details_cubit.dart:1-23`): `(slug, eventsRepo)`، `getEventsInfo()` → Loading→Success(EventModel)/Error. تكامل مفضّل/حفظ/تقييم عبر listeners → refresh.

## 8. State Flow
getEventsInfo → Loading → repo.fetchEvents(slug) → fold → Success/Error. أي نجاح مفضّل/حفظ/تقييم → getEventsInfo (refresh).

## 9. Models
**EventModel** (`lib/features/home/data/model/top_events_model/event_model.dart:1-115`): fromDate(date_from), toDate(date_to)، displayType، ticketLink، closed، facebook/linkedin/instagram/whatsapp/website، + مشترك (id, rate, featured, viewsNum, isFavorite, isSaved, lat, long, city, region, galleries, ratings, meta). ⚠️ لا يلتقط `x` من الخادم (عيب #1).

## 10. Repositories
- **Flutter** `events_repo.dart` + `events_repo_imp.dart:1-21`: `fetchEvents(slug)` → GET `events/{slug}` → EventModel.
- **Laravel:** لا repository — EventController.

## 11. Services
لا service. مساعِدات App.php (distance/isFavorite). المنطق الزمني inline (CASE + date filter).

## 12. API Endpoints
| Method | Endpoint | Auth |
|---|---|---|
| GET | `events` (قائمة+فلاتر زمنية) | عام |
| GET | `events/{slug}` (تفاصيل) | عام |

## 13. Request Models
لا body. query: search، **daterange** (from-to)، address_type (banner/top_bar)، category_id، lat/lng/x/y، per_page.

## 14. Response Models
- قائمة: `EventCollection` → items[EventListResource] (+status, rejected_reason, ownership_proof_file).
- تفاصيل: `EventResource` (date_from/date_to, closed, display_type, ticket_link, روابط، is_favorite/is_saved). الغلاف `{code,message,data}`.

## 15. Laravel Routes
`routes/api.php:67-68`: `GET events → EventController@index` · `GET events/{event} → show`. عامة (Front).

## 16. Controllers
**EventController** (`app/Http/Controllers/Api/EventController.php:1-97`): `index` (:12-91، active + search + **daterange تداخل :29-39** + address_type + category + geo + **CASE زمني :71-85** + paginate، eager `ratings,galleries,city,region,ceo`)، `show` (:93-96، firstOrFail). `HomeController` topEvents (`:85-92`، date_to>today + views DESC).

## 17. Service Classes (Laravel)
لا service. المنطق الزمني (CASE/تداخل) inline في index.

## 18. Models (Laravel)
**Event** (`app/Models/Event.php:1-146`): translatable(title,description)، SoftDeletes، `TYPE_EVENT='Event'`. fillable: date_from/date_to/display_type/type/ticket_link/status/rejected_reason/added_by_user/ownership_proof_file/user_id + مشترك. accessors: addressType(=type)، type('event')، lat/long/rate. relations: region/city/user/galleries(type=events)/ceo/ratings(type='event')/favorites(morph).

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `events` | id, slug, **date_from(DATE), date_to(DATE), display_type, type**, description, address, categories(JSON), image, ticket_link, روابط اجتماعية، lat/long, city_id/region_id, active, featured, visited, views_num, link, related, order_id, **status, added_by_user(index), ownership_proof_file, rejected_reason, user_id(FK)**, softDeletes |
| `event_translations` | event_id(FK cascade), locale, title, description، unique |

## 20. Relationships
Event BelongsTo Region/City/User · HasMany Gallery(type=events)/Rate(type=event) · HasOne Ceo · MorphMany Favorite · translations. related كـ IDs.

## 21. Validation Rules
index متساهل (Request عادي). daterange يُفكَّك بـexplode('-'). لا FormRequest صارم.

## 22. Authorization & Permissions
عام. is_favorite/is_saved per-user. ⚠️ `show` لا يفلتر active؛ ⚠️ index لا يفلتر status='approved' (فعاليات UGC غير معتمدة قد تظهر).

## 23. Error Handling
- **Flutter:** EventsDetailsError.
- **Laravel:** firstOrFail → 404؛ الغلاف ApiModalController.

## 24. Edge Cases
- فعالية منتهية → ترتيب 3 (تُعرَض بالقائمة، مستبعدة من home)؛ التفاصيل بلا قيد.
- جارية/قادمة → أولوية 1/2.
- daterange تداخل (لا مطابقة تامة).
- closed = خارج النافذة.
- بلا موقع → CASE زمني.
- ⚠️ عيوب اجتماعية: `x` غير ملتقَط، `facebook` **معلَّق في EventResource**، `linkedin` غير مُرسَل → روابط فارغة (§30).
- 404 لفعالية غير موجودة.
- ترجمة ناقصة → fallback.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, google_maps، url_launcher، cached_network_image. Laravel: astrotomic/translatable، spatie/medialibrary، Carbon (NOW/today)، orderByRaw.

## 26. Files Involved
**Flutter:** `lib/features/events/**` (repo، events_details_cubit، view + widgets: events_details_items_body/social_link_item/open_link)، `lib/features/home/data/model/top_events_model/{event_model,top_events_response}.dart`، `offer_date_range.dart`، `end_points.dart:80`، `routes.dart:389-407`.
**Laravel:** `app/Http/Controllers/Api/EventController.php` (+HomeController topEvents)، `app/Models/{Event,EventTranslation}.php`، `app/Http/Resources/Events/{EventResource,EventListResource,EventCollection}.php`، migrations events/event_translations (+UGC/user_id)، `routes/api.php:67-68`.

## 27. Complete Execution Flow
**القائمة:** (Home/Tasneef) → GET `events?daterange&address_type` → EventController@index (active + daterange تداخل + CASE زمني + paginate) → EventCollection → EventModel list → UI.
**التفاصيل:** getEventsInfo → GET `events/{slug}` → EventController@show (firstOrFail) → EventResource (date_from/to + closed + is_favorite/is_saved) → EventModel → Success → EventsDetailsItemsBody (نافذة تاريخ + اجتماعي + خريطة + تقييمات + تذكرة).

## 28. Performance Considerations
- ✅ eager loading في index.
- 🟠 **CASE/orderByRaw** بطيء بلا فهرس على (date_from, date_to).
- 🟠 `allCategories`/`ceo` N+1 محتمل.
- 🟠 ratings كاملة بلا حد في التفاصيل.
- 🟡 daterange بـexplode('-') هشّ (تنسيق).

## 29. Security Considerations
1. 🟠 **ratings خام في EventResource** → تسريب email المقيّمين.
2. 🟠 **`show` لا يفلتر active** → كشف بالـ slug.
3. 🟠 **index لا يفلتر status='approved'** → فعاليات UGC غير معتمدة قد تظهر.
4. 🟠 **ownership_proof_file عام في EventListResource** → ملف إثبات ملكية مكشوف (يجب للمالك/الأدمن فقط).
5. 🟡 القراءة عامة.

## 30. Technical Debt
- عيوب روابط اجتماعية: `x` غير ملتقَط في EventModel، `facebook` **معلَّق** في EventResource، `linkedin` غير مُرسَل → روابط لا تعمل.
- تسمية `type`/`display_type` مربكة (type accessor='event'، العمود type=display_type).
- daterange بـexplode('-') (يكسر مع تواريخ فيها '-').
- ratings خام (يحتاج RatingResource).
- UGC workflow (status/rejected) غير ظاهر في واجهة Flutter.

## 31. Improvement Opportunities
- إصلاح روابط الاجتماعي (إرسال facebook/linkedin/x + التقاطها).
- فلترة status='approved' في index؛ إخفاء ownership_proof.
- RatingResource يخفي email؛ فلترة active في show.
- فهرسة (date_from, date_to) + (active, date_to).
- توضيح تسمية type/display_type.
- عرض عدّاد تنازلي للفعالية القادمة.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 04 Places | نفس القالب |
| 02 Home | topEvents (date_to>today, banner/top_bar) |
| 07 Tasneef | قائمة events (فلتر daterange) |
| 08 Exploration | ضمن البحث/الخريطة |
| 13 Favorites / 14 Rates | مفضّل/حفظ/تقييم (type 'event'/'Event') |
| 15 My Properties/UGC | فعاليات مقدَّمة من مستخدم (status/proof) |
| 18 Taxonomy | region/city/category |

## 33. How to Modify This Feature
- **حقل زمني/حالة:** migration events + Event fillable + EventResource + EventModel + widget.
- **منطق ترتيب زمني:** EventController@index (CASE).
- **إصلاح اجتماعي:** EventResource (uncomment facebook + add linkedin) + EventModel.fromJson.
- **APIs:** events, events/{slug}.
- **المخاطر:** CASE بلا فهرس، تسريب email/proof، status غير مُفلتر، تسمية type مربكة.
- **اختبارات:** قائمة (daterange/address_type/ترتيب زمني)، تفاصيل، جارية/قادمة/منتهية، home (استبعاد المنتهية).

## 34. Regression Checklist
- [ ] قائمة events: ترتيب زمني (جارية→قادمة→منتهية).
- [ ] فلتر daterange (تداخل صحيح).
- [ ] فلتر address_type (banner/top_bar).
- [ ] home: يستبعد المنتهية (date_to>today) + views DESC.
- [ ] تفاصيل: نافذة تاريخ + closed flag صحيح.
- [ ] روابط اجتماعية (بعد الإصلاح).
- [ ] حجز تذكرة (إن ticketLink).
- [ ] مفضّل/حفظ/تقييم → refresh.
- [ ] 404؛ لغة/RTL + ترجمة.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «فعالية منتهية بالأعلى» | CASE/ترتيب | `EventController@index:71-85` |
| «رابط فيسبوك/x فارغ» | معلَّق/غير ملتقَط | `EventResource:45` + `EventModel:81` |
| «daterange لا يفلتر» | explode('-') | `EventController@index:29-39` |
| «منتهية تظهر في home» | date_to>today | `HomeController:85-92` |
| «فعالية غير معتمدة ظاهرة» | لا status filter | `EventController@index` |
| «email المقيّم/proof ظاهر» | خام/عام | `EventResource`/`EventListResource` |
| «closed خاطئ» | now.between | `EventResource:40` |

## 36. Debug Guide
1. راقب `GET events?daterange` و`events/{slug}`.
2. تحقّق ترتيب CASE + closed في الرد.
3. تتبّع EventsDetailsCubit (Loading→Success/Error).
4. للتاريخ: افحص date_from/date_to + NOW/today.
5. للاجتماعي: قارن ما يُرسله EventResource بما يلتقطه EventModel.
6. للأداء: DB::listen + خطة CASE/orderByRaw.

## 37. Search Keywords
`event`, `events`, `فعاليات`, `EventController`, `EventResource`, `EventListResource`, `EventCollection`, `Event model`, `EventModel`, `EventsDetailsCubit`, `events/{slug}`, `detailsEvents`, `date_from`, `date_to`, `daterange`, `closed`, `display_type`, `banner top_bar`, `temporal sorting`, `CASE WHEN ongoing`, `upcoming past`, `OfferDateRange`, `topEvents`, `ticket_link`, `BookTicketButtomWidget`, `status rejected_reason`, `added_by_user`, `ownership_proof_file`, `event_translations`, `TYPE_EVENT`, `SocialLinkItem` → **09_Events.md**

## 38. Future Improvements
- إصلاح روابط الاجتماعي (fb/linkedin/x).
- status='approved' filter + إخفاء proof.
- RatingResource (مشترك)؛ فلترة active في show.
- فهرسة زمنية + عدّاد تنازلي.
- توضيح type/display_type.
- واجهة UGC للفعاليات (تقديم/حالة).

---
**تدفق مرجعي:** UI → EventsDetailsCubit → Repo → API → Route → EventController(CASE زمني) → Event → DB(events/event_translations) → EventResource → EventModel → UI ✅ موثّق. **الفرق عن [[04_Places]]: نافذة زمنية + ترتيب جارية/قادمة/منتهية + UGC + تذكرة.**
