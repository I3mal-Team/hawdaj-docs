# Feature 11 — Tour Guides (أدلة سياحية)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/tourـguide) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 يتبع قالب [[04_Places]] جزئيًا، لكنه **ميزة ملف شخصي + تسجيل ذاتي** (المستخدم يسجّل نفسه كدليل)، لا محتوى يُقدَّمه أدمن فقط. الفريد: نموذج تسجيل ذاتي، regions/languages متعدّدة الاختيار، ترتيب top_rated بمتوسط، **دليل واحد لكل مستخدم**.

---

## الفريد عن Places (ملخّص)
| المحور | Places | **Tour Guides** |
|---|---|---|
| المصدر | محتوى أدمن | **تسجيل ذاتي من المستخدم** (POST store-guide) |
| علاقة بالمستخدم | اختيارية | **دليل واحد فقط لكل user_id** (enforced) |
| الفئات | Category | **regions[] + languages[] (JSON، متعدّد اختيار)** |
| الترتيب الافتراضي | order_id/distance | **top_rated: AVG(rate) DESC** (withCount) |
| الصورة | عامة | **رفع منفصل** (`update-guide-photo`، مستقل عن الفورم) |
| الاجتماعي | fb/instagram | x/twitter/linkedin/instagram/personal_account (تطبيع https تلقائي) |

## 1. Feature Name
Tour Guides — أدلة سياحيون يسجّلون أنفسهم (اسم/لقب/خبرة/مناطق/لغات/تواصل اجتماعي/صورة)، تُعرَض للمستخدمين بترتيب الأعلى تقييمًا، مع تقييمات ومفضّلة.

## 2. Business Goal
تمكين مستخدمين (أدلّاء) من إنشاء ملف تعريفي عام، واكتشاف المستخدمين لهم بالمنطقة/اللغة، مرتّبين حسب الأعلى تقييمًا.

## 3. Business Rules
- `active=1`. **دليل واحد لكل مستخدم** (`Guide::where(user_id)`، store يحدّث إن وُجد وإلا ينشئ).
- الترتيب top_rated: `withCount(['ratings as average_rate'=>AVG(rate)])->orderByDesc('average_rate')`.
- فلاتر: search(name/nickName)، experience، region_id (JSON_CONTAINS)، language_id (JSON_CONTAINS)، excludedId (استبعاد النفس من "أدلّة مشابهة").
- regions/languages تُخزَّن كـJSON arrays.
- روابط اجتماعية تُطبَّع بإضافة `https://` تلقائيًا (client-side).
- رفع الصورة **منفصل** عن حفظ الفورم (multipart مستقل).
- show_in_home لعرض مميّز.

## 4. User Flow
**عرض:** Home/Tasneef → بطاقة دليل (id) → TourGuideDetailsView (قراءة فقط) → تقييمات/مشاركة/مفضّل.
**تسجيل ذاتي:** AddTourGuideDetailsView (فورم) → ملء الاسم/اللقب/الوصف/الخبرة → اختيار مناطق/لغات (متعدّد) → روابط اجتماعية → رفع صورة (منفصل) → حفظ → StoreGuideCubit → نجاح.
**تعديل:** نفس الفورم بـ`isEdit=true` مع `initForEditFromModel`.

## 5. Flutter Screens
| Screen | File |
|---|---|
| TourGuideView (قائمة) | `presentation/view/tour_guide_view.dart:6-29` |
| TourGuideDetailsView (تفاصيل قراءة فقط) | `presentation/view/tour_guide_details_view.dart:6-29` |
| AddTourGuideDetailsView (فورم تسجيل/تعديل) | `presentation/view/add_tour_guide_view.dart:21-115` |

## 6. Widgets
`add_tour_guide_details_view_body.dart:28-427` (حقول: اسم/لقب/وصف/خبرة، مناطق متعدّدة، لغات متعدّدة، مدير روابط اجتماعية إضافة/تعديل/حذف، منتقي صورة+تقدّم رفع)، `tour_guide_body.dart:23-355` (عرض دليلي الخاص: صورة/اسم/خبرة/وصف/اجتماعي/لغات chips/مناطق كاروسيل/تقييمات/زر تعديل)، `tour_guide_details_items_body.dart:27-369` (عرض عام قراءة فقط + مشاركة + تقييم)، `social_links_section.dart:7-73` (أزرار x/instagram/linkedin/youtube/موقع شخصي).

## 7. Cubits / Blocs
| Cubit | File | دور |
|---|---|---|
| FetchAllTourGuideCubit | `:9-21` | يجلب دليل المستخدم الحالي (GET get-guide) |
| FetchTourGuideByTopRatedCubit | `:9-130` | قائمة مصفّحة top_rated + cache + loadMore/refresh |
| TourGuideDetailsCubit | `:8-24` | تفاصيل دليل بالـid |
| StoreGuideCubit | `:10-50` | حفظ (إنشاء/تحديث) + تطبيع روابط |
| TourGuideFormCubit | `:10-140` | حالة الفورم (name, nickName, description, experience, selectedRegionIds, selectedLanguageIds, socialLinks{}, imageFile/Url)؛ `initForEditFromModel` |
| UpdateGuidePhotoCubit | `:9-21` | رفع صورة multipart منفصل |
| FetchLanguagesCubit / FetchRegionCubit | `:8-20` كل | تعبئة قوائم الفورم |

## 8. State Flow
list: fetch(page) → repo → UnifiedResponseModel → Success(items,page,hasNext) بذاكرة مؤقّتة. store: validate(name/regions/languages) → normalize social → repo.storeGuide → POST → GuideModel → Success → toast+bottom sheet+clear form. photo: منفصل → Success(imageUrl) → يحدّث form state عبر listener.

## 9. Models
**GuideModel** (`guide_model.dart:9-77`): id, type, name, nickName, description, image, showInHome, experience, gender، **regions[RegionModel]**, **languages[LanguageModel]**, social(SocialModel), rate, ratings[], galleries[].
**TourGuideFormState** (`:3-113`): name, nickName, description, experience, selectedRegionIds[], selectedLanguageIds[], socialLinks{}, imageUrl, imageFile?.

## 10. Repositories
**Flutter** `tour_guide_repo.dart:10-42` + `tour_guide_repo_imp.dart:15-153`: fetchTourGuideByTopRated(page,perPage,excludedId,topRated) → GET `guides` · fetchTourGuide(id) → GET `guides/{id}` · getMyGuide → GET `get-guide` · storeGuide(body) → POST `store-guide` (بناء body شرطي للروابط الاجتماعية) · updateGuidePhoto(file) → POST `update-guide-photo` (FormData) · fetchLanguages → GET `languages` · fetchRegions → GET `region`.
**Laravel:** لا repository — GuideController.

## 11. Services
لا service. Spatie Media للصورة (small250/medium500). Astrotomic Translatable للاسم/اللقب/الوصف.

## 12. API Endpoints
| Method | Endpoint | Auth |
|---|---|---|
| GET | `guides` (قائمة+top_rated+فلاتر) | عام |
| GET | `guides/{id}` (تفاصيل) | عام |
| POST | `store-guide` (إنشاء/تحديث) | ✅ مطلوب |
| GET | `get-guide` (دليلي) | ✅ مطلوب |
| POST | `update-guide` | ✅ مطلوب |
| POST | `update-guide-social` | ✅ مطلوب |
| POST | `update-guide-photo` | ✅ مطلوب |

## 13. Request Models
list query: search, experience, region_id, language_id, top_rated, excludedId, per_page. store/update body: name*, nickName*, description, experience(numeric), regions[](numeric), languages[](numeric), facebook/x/twitter/linkedin/instagram/personal_account(url). photo: FormData{image}.

## 14. Response Models
`GuideResource` (id, order_id, type, name, nickName, description, image+small/medium, show_in_home, experience, gender(من user), regions(allRegions), languages(allLanguages), social, rate(مقرَّب), ratings, galleries). `GuideListResource` (مختصر). `GuideCollection` (ترقيم قياسي). الغلاف `{code,message,data}`.

## 15. Laravel Routes
`routes/api.php:191-199`: `GET guides → index` (عام) · `GET guides/{id} → show` (عام) · `POST store-guide → storeGuide` (auth) · `GET get-guide → getGuide` (auth) · `POST update-guide → updateGuide` (auth) · `POST update-guide-social → updateSocial` (auth) · `POST update-guide-photo → updatePhoto` (auth).

## 16. Controllers
**GuideController** (`app/Http/Controllers/GuideController.php:15-362`): `index` (:17-69، فلاتر + **top_rated withCount AVG** + excludedId + paginate)، `show` (:71-74)، `storeGuide` (:76-181، **دليل واحد/مستخدم** — تحديث إن وُجد وإلا إنشاء، :123-167)، `getGuide` (:183-203، دليل المستخدم الحالي أو null)، `updateGuide` (:205-282، يتطلّب وجود دليل)، `updateSocial` (:284-323)، `updatePhoto` (:325-361، استبدال media collection).

## 17. Service Classes (Laravel)
لا service. منطق inline. Spatie MediaLibrary للصورة.

## 18. Models (Laravel)
**Guide** (`app/Models/Guide.php:14-115`): fillable(regions,languages,experience,facebook,x,twitter,linkedin,instagram,personal_account,show_in_home,active,user_id,order_id)، casts regions/languages→array، translatable(name,nickName,description via guide_translation). relations: user()/ratings(type='guide')/galleries(type='guides'). accessors: type('guide')، **rate=AVG(ratings.rate)**، allLanguages()/allRegions(). media: image + small(250)/medium(500).
**GuideTranslation** (`GuideTranslation.php:8-17`): جدول `guide_translation` (بلا timestamps).

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `guides` | id, image, **regions(JSON), languages(JSON)**, experience(int), facebook/x/twitter/linkedin/instagram/personal_account(text), show_in_home, **user_id(FK)**, timestamps |
| `guide_translation` | guide_id(FK), locale, name, nickName, description، unique(guide_id,locale) |

## 20. Relationships
Guide BelongsTo User (**واحد لكل مستخدم — enforced في الكود لا FK unique**) · HasMany Gallery(type=guides)/Rate(type=guide) · translations HasMany. regions/languages كـJSON IDs (allRegions/allLanguages تحلّها).

## 21. Validation Rules
storeGuide/updateGuide: name required، nickName required، experience/regions/languages numeric، روابط اجتماعية url. لا FormRequest منفصل — تحقّق inline.

## 22. Authorization & Permissions
list/show عامان. store/update/photo/social **تتطلّب auth**. ⚠️ لا يوجد تحقّق صريح على أن الدليل المُعدَّل يخص المستخدم الحالي بخلاف البحث بـ`user_id=auth()->id()` (آمن ضمنيًا طالما الاستعلام يقيّد بـuser_id).

## 23. Error Handling
- **Flutter:** التحقّق المحلي (name/regions/languages غير فارغة) قبل الإرسال؛ Success/Error toasts.
- **Laravel:** 401 "sign up first" إن بلا auth (updatePhoto)؛ 422 "need to join first" إن لا دليل عند رفع صورة قبل الإنشاء؛ الغلاف ApiModalController.

## 24. Edge Cases
- مستخدم بلا دليل → getGuide يعيد null → الفورم بوضع "إنشاء".
- محاولة رفع صورة قبل إنشاء الدليل → 422.
- لا لغات/مناطق مختارة → رفض محلي قبل الإرسال.
- تقييمات فارغة → rate=0 (AVG على صفر صفوف).
- 404 لدليل غير موجود.
- ترجمة ناقصة → fallback.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, image_picker, infinite_scroll_pagination (list top_rated). Laravel: astrotomic/translatable، spatie/medialibrary.

## 26. Files Involved
**Flutter:** `lib/features/tourـguide/**` (data/repo، data/model، manager/* [8 cubits]، view/* + widgets)، `lib/features/home/data/model/guide_model/guide_model.dart`، `end_points.dart`.
**Laravel:** `app/Http/Controllers/GuideController.php`، `app/Models/{Guide,GuideTranslation}.php`، `app/Http/Resources/{GuideResource,GuideListResource,GuideCollection}.php`، migrations guides/guide_translation، `routes/api.php:191-199`.

## 27. Complete Execution Flow
**قائمة top_rated:** FetchTourGuideByTopRatedCubit.fetch → GET `guides?top_rated=true&excludedId=` → GuideController@index (active + فلاتر + withCount AVG + orderByDesc + paginate) → GuideCollection → UnifiedResponseModel → Success(cache+loadMore).
**تفاصيل:** TourGuideDetailsCubit.getTourGuideInfo → GET `guides/{id}` → show → GuideResource → GuideModel → عرض قراءة فقط.
**تسجيل ذاتي:** فورم → TourGuideFormCubit state → رفع صورة منفصل (UpdateGuidePhotoCubit → POST update-guide-photo) → حفظ (StoreGuideCubit.storeFromForm → تطبيع اجتماعي → repo.storeGuide → POST store-guide) → GuideController@storeGuide (دليل موجود؟ update:create) → GuideResource → GuideModel → Success → toast+bottom sheet+clear.

## 28. Performance Considerations
- ✅ `withCount` لـAVG (آمن من N+1 الأساسي).
- 🟠 `allRegions()`/`allLanguages()` قد تستعلم لكل دليل في القوائم (N+1 محتمل).
- ✅ media conversions محدّدة الحجم.

## 29. Security Considerations
1. 🟠 **ratings خام محتملة** (نمط مشترك — تحقّق GuideResource لتسريب email).
2. ✅ store/update/photo محمية auth، ومقيَّدة بـuser_id ضمنيًا.
3. 🟡 لا rate-limit على store-guide (سبام تسجيل محتمل).

## 30. Technical Debt
- منطق create/update مكرّر جزئيًا بين storeGuide وupdateGuide (endpoint منفصل رغم توحّد المنطق فعليًا في storeGuide).
- رفع الصورة منفصل عن الفورم (تعقيد UX/تزامن حالة).
- لا قيد DB unique صريح على user_id (الاعتماد على منطق التطبيق فقط لـ"دليل واحد").
- تسمية مجلد الميزة بحرف Arabic tatweel (`tourـguide`) — غير قياسي.

## 31. Improvement Opportunities
- توحيد store/update في مسار واحد (المنطق موحّد فعليًا).
- قيد DB (unique user_id) لضمان دليل واحد فعليًا.
- دمج رفع الصورة ضمن حفظ الفورم الواحد (تجربة أسلس).
- rate-limit على store-guide.
- RatingResource مشترك لإخفاء email إن لزم.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 01 Authentication | إلزامي لكل عمليات الكتابة (دليل مرتبط بـuser_id) |
| 02 Home | قسم أدلّة مميّزة (show_in_home) |
| 03 Trips | الأدلّة قد تُقترَح ضمن الرحلة (سياقيًا) |
| 07 Tasneef | قائمة guides (type='guides') |
| 08 Exploration | ضمن البحث/الخريطة |
| 13 Favorites / 14 Rates | مفضّل/تقييم الدليل (type='guide') |
| 18 Taxonomy | Regions + Languages (متعدّد اختيار) |

## 33. How to Modify This Feature
- **حقل ملف شخصي جديد:** migration guides/guide_translation + Guide fillable/translatedAttributes + GuideResource + GuideModel + form field.
- **قيد "دليل واحد":** إضافة unique constraint على user_id (تحسين).
- **APIs:** guides, guides/{id}, store-guide, get-guide, update-guide(-social/-photo).
- **المخاطر:** كسر منطق create/update الموحّد، فقدان قيد الدليل الواحد، N+1 allRegions/allLanguages.
- **اختبارات:** تسجيل (إنشاء أول مرة)، تعديل (دليل موجود)، رفع صورة قبل/بعد الإنشاء، قائمة top_rated، فلاتر region/language، مفضّل/تقييم.

## 34. Regression Checklist
- [ ] تسجيل دليل جديد (بلا دليل سابق) → إنشاء.
- [ ] تعديل دليل موجود → تحديث لا تكرار.
- [ ] رفع صورة قبل إنشاء الدليل → 422 متوقّع.
- [ ] رفع صورة بعد الإنشاء → تحديث ناجح.
- [ ] قائمة top_rated مرتّبة صحيحًا (AVG DESC).
- [ ] فلاتر region_id/language_id/experience/search.
- [ ] excludedId يستبعد النفس من "أدلّة مشابهة".
- [ ] روابط اجتماعية تُطبَّع بـhttps تلقائيًا.
- [ ] مفضّل/تقييم الدليل.
- [ ] لغة/RTL + ترجمة الاسم/اللقب/الوصف.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «رفع صورة يفشل 422» | لا دليل بعد | `GuideController@updatePhoto` |
| «إنشاء دليل ثانٍ بدل تحديث» | منطق find-or-create | `storeGuide:123-167` |
| «ترتيب top_rated خاطئ» | withCount/AVG | `GuideController@index:50-55` |
| «فلتر region/language لا يعمل» | JSON_CONTAINS | `GuideController@index` |
| «رابط اجتماعي بلا https» | تطبيع محلي فقط | `StoreGuideCubit:16-21` |
| «صورة الدليل لا تتحدّث» | استبدال media collection | `updatePhoto:349-350` |

## 36. Debug Guide
1. راقب `GET get-guide` (هل للمستخدم دليل؟) قبل رفع الصورة.
2. تتبّع TourGuideFormCubit state (regions/languages IDs).
3. للترتيب: تحقّق SQL (`withCount` + `orderByDesc average_rate`).
4. لمنطق create/update: راقب `storeGuide` هل يجد دليلًا موجودًا بـuser_id.
5. للصورة: تتبّع UpdateGuidePhotoCubit → multipart → media collection.

## 37. Search Keywords
`guide`, `guides`, `tour guide`, `أدلة`, `GuideController`, `GuideResource`, `GuideListResource`, `GuideCollection`, `Guide model`, `GuideTranslation`, `guide_translation`, `TourGuideFormCubit`, `StoreGuideCubit`, `FetchTourGuideByTopRatedCubit`, `TourGuideDetailsCubit`, `UpdateGuidePhotoCubit`, `store-guide`, `get-guide`, `update-guide`, `update-guide-photo`, `update-guide-social`, `top_rated`, `average_rate`, `withCount`, `allRegions`, `allLanguages`, `regions JSON`, `languages JSON`, `excludedId`, `one guide per user` → **11_TourGuides.md**

## 38. Future Improvements
- قيد DB unique(user_id) لضمان دليل واحد فعليًا.
- توحيد create/update في مسار واحد.
- دمج رفع الصورة بالفورم الواحد.
- rate-limit على store-guide.
- إصلاح N+1 محتمل في allRegions/allLanguages.

---
**تدفق مرجعي:** UI(فورم/قائمة) → Cubit → Repo → API → Route → GuideController → Guide(دليل واحد/مستخدم) → DB(guides/guide_translation) → GuideResource → GuideModel → UI ✅ موثّق. **الفرق عن [[04_Places]]: تسجيل ذاتي + regions/languages متعدّد + top_rated AVG.**
