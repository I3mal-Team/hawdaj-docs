# Feature 10 — Swalef & Stories (سوالف — المحتوى التراثي)

> **Status:** ✅ Analyzed (100%) · **Priority:** P1 · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` (lib/features/stories) · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

> 🔗 **يتبع قالب [[04_Places]]** لكن بـ**سكيما محتوى غنية** (سرد تراثي متعدّد الحقول + صوت). ملاحظة: هذا Swalef (تراث)، **ليس** [[16_UserStories]] (ستوري آخر يوم). هذا الملف يركّز على السرد الغني والصوت.

---

## الفريد عن Places (ملخّص)
| المحور | Places | **Swalef** |
|---|---|---|
| السرد | description واحد | **content + timePeriod + location + storyImportance + relevanceToPresent + source** (مترجَمة) |
| صوت | ❌ | **audioStoryLink + audioStoryTitle** (بالـDB، **بلا UI بعد**) |
| أقسام | عنوان/عنوان | سياق تراثي + قائمة شخصيات (mainCharacters) |
| فئات | Category | JSON + **CategoryOfSwalef هرمي** |
| النموذج (Flutter) | UnifiedPlaceModel | **SafwestModel** |

## 1. Feature Name
Swalef (سوالف) — محتوى تراثي/ثقافي سعودي (شعر/مثل عربي/قصة/ملف) بسرد غني، صور، صوت (مخطّط)، تقييمات، مفضّل.

## 2. Business Goal
حفظ وعرض التراث القصصي بحقول سردية غنية (أهمية القصة، الحقبة الزمنية، صلتها بالحاضر، المصدر، الشخصيات) لإثراء تجربة الاكتشاف.

## 3. Business Rules
- `active=1`. `show_in_home` لقسم home (safwests). `featured` لـtop_featured.
- الترتيب: `applyListingOrder` (order_id, id).
- الأنواع: شعر/مثل عربي/قصة/ملف.
- السرد الغني مترجَم عبر SwalefTranslation (9 حقول). Observer يترجم تلقائيًا (⚠️ mainCharacters **معلّق** — لا يُترجَم).
- الصوت (audioStoryLink/Title) مخزّن **لكن بلا مشغّل UI**.
- slug يُولَّد من ترجمة en.

## 4. User Flow
Home(safwests)/Tasneef → بطاقة سالفة (slug) → StoriesDetailsView → معرض + عنوان/تقييم + حقول سردية (أهمية/صلة/حقبة/شخصيات) + مفضّل/مشاركة/موقع.

## 5. Flutter Screens
| Screen | File |
|---|---|
| StoriesDetailsView (scaffold/loader/error) | `lib/features/stories/presentation/view/stories_details_view.dart` |
| StoriesDetailsItemsBody (سرد غني) | `.../view/widgets/stories_details_items_body.dart:1-167` |

## 6. Widgets
`CustomAppBarHomeDetails` (معرض، `:48`)، `DetailsSection` (عنوان/تقييم/وصف، `:83-88`)، أقسام سردية بعناوين (storyImportance `:91-105`، relevanceToPresent `:107-121`، timePeriod `:122-136`، mainCharacters قائمة نقطية `:137-160`)، مشاركة (`:56`)، موقع (`:70-78`). ⚠️ **لا مشغّل صوت**.

## 7. Cubits / Blocs
**StoriesDetailsCubit** (`stories_details_cubit.dart:1-24`): `(slug, storiesRepo)`، `getStoriesInfo()` → Loading→Success(SafwestModel)/Error. تكامل مفضّل/حفظ عبر listener (`:38-46`).

## 8. State Flow
getStoriesInfo → Loading → repo.fetchStories(slug) → fold → Success/Error. مفضّل/حفظ نجاح → refresh.

## 9. Models
**SafwestModel** (`safwest_model.dart:4-33`): id, slug, image, type, active, featured, showInHome، rate/review، title/description/**content**، **timePeriod/location/storyImportance/relevanceToPresent/source** (سردية)، **audioStoryLink/audioStoryTitle** (بلا UI)، **mainCharacters(List)**، categories, galleries, translations, isFavorite, isSaved.

## 10. Repositories
- **Flutter** `stories_repo.dart` + `stories_repo_imp.dart:1-21`: `fetchStories(slug)` → GET `swalefs/{slug}` (`end_points.dart:74`) → SafwestModel.
- **Laravel:** لا repository — SwalefController.

## 11. Services
لا service. Observer `SwalefObserver` (ترجمة Google تلقائية لـ9 حقول عند الحفظ). مساعِدات App.php (listing/isFavorite).

## 12. API Endpoints
| Method | Endpoint | Auth |
|---|---|---|
| GET | `swalefs` (قائمة+فلاتر) | عام |
| GET | `swalefs/{slug}` (تفاصيل) | عام |

## 13. Request Models
لا body. query: search، category_id(CSV)، top_featured، per_page.

## 14. Response Models
- قائمة: `SwalefCollection` → items[SwalefListResource].
- تفاصيل: `SwalefResource` (حقول سردية مترجَمة + mainCharacters(JSON) + categories + galleries + is_favorite/is_saved). الغلاف `{code,message,data}`.

## 15. Laravel Routes
`routes/api.php:114-115`: `GET swalefs → index` · `GET swalefs/{id} → show` (⚠️ param اسمه `id` لكنه slug). عامة.

## 16. Controllers
**SwalefController** (`app/Http/Controllers/Api/SwalefController.php:1-56`): `index` (:19-49، active + search + category(JSON_CONTAINS) + top_featured + listing order + paginate)، `show` (:51-55، firstOrFail).

## 17. Service Classes (Laravel)
لا service. `SwalefObserver:1-68` (ترجمة تلقائية لكل locale لـ9 حقول؛ mainCharacters معلّق؛ توليد slug من en).

## 18. Models (Laravel)
**Swalef** (`app/Models/Swalef.php:1-124`): translatable **9 حقول** (title, description, content, timePeriod, location, storyImportance, relevanceToPresent, source, audioStoryTitle، `:28-31`)، `$with=[translations]`. casts: categories→array، show_in_home→boolean. appends: rate/review/type('swalef'). relations: ratings(type='swalef')/favorites(morph)/galleries(type='swalefs')/user. media image + small(250)/medium(500). SoftDeletes.
**CategoryOfSwalef** هرمي (parent_id).

## 19. Database Tables
| Table | أعمدة مفتاحية |
|---|---|
| `swalefs` | id, active, featured, image, slug, show_in_home, user_id(FK), order_id, address, **audioStoryLink**, **mainCharacters(JSON)**, categories(JSON), timestamps, deleted_at |
| `swalef_translations` | swalef_id(FK), locale, **title, description, content, timePeriod, location, storyImportance, relevanceToPresent, source, audioStoryTitle** (كلها text)، unique(swalef_id,locale) |
| `category_of_swalefs` | id, icon, parent_id(هرمي), order_id (+translations) |

## 20. Relationships
Swalef BelongsTo User · HasMany Gallery(type=swalefs)/Rate(type=swalef) · MorphMany Favorite · HasOne Ceo · translations HasMany (9 حقول). categories كـJSON (allCategories→CategoryOfSwalef).

## 21. Validation Rules
index متساهل (Request عادي). لا FormRequest صارم.

## 22. Authorization & Permissions
عام. is_favorite/is_saved per-user. ⚠️ `show` لا يفلتر active (كشف بالـslug).

## 23. Error Handling
- **Flutter:** StoriesDetailsError.
- **Laravel:** firstOrFail → 404؛ الغلاف ApiModalController.

## 24. Edge Cases
- بلا صوت → لا مشكلة (UI لا يعرضه أصلًا).
- حقول سردية ناقصة → أقسام تُخفى/fallback ("no_characters").
- mainCharacters غير مترجَم (Observer معلّق).
- payload ترجمة كبير (9 حقول × locales، بلا scoping للـlocale المطلوب).
- 404؛ ترجمة ناقصة → سلوك Astrotomic الافتراضي.

## 25. Dependencies
Flutter: dio, dartz, flutter_bloc, cached_network_image, url_launcher. Laravel: astrotomic/translatable، spatie/medialibrary، stichoza/google-translate (Observer).

## 26. Files Involved
**Flutter:** `lib/features/stories/**` (repo، stories_details cubit، view + widgets)، `lib/features/home/data/model/safwests_model/safwest_model.dart`، `end_points.dart:74`.
**Laravel:** `app/Http/Controllers/Api/SwalefController.php`، `app/Models/{Swalef,SwalefTranslation,CategoryOfSwalef}.php`، `app/Observers/SwalefObserver.php`، `app/Http/Resources/Swalefs/{SwalefResource,SwalefListResource,SwalefCollection}.php`، migrations swalefs/swalef_translations/category_of_swalefs، `routes/api.php:114-115`.

## 27. Complete Execution Flow
StoriesDetailsView → StoriesDetailsCubit.getStoriesInfo → GET `swalefs/{slug}` → SwalefController@show (firstOrFail + eager translations) → حساب rate/review + type('swalef') → SwalefResource (9 حقول مترجَمة + mainCharacters JSON + categories + galleries + is_favorite/is_saved) → SafwestModel.fromJson → Success → StoriesDetailsItemsBody (سرد غني).

## 28. Performance Considerations
- 🟠 **N+1 على ratings** (`ratings()->count()` + `sum()` في accessor = استعلامان/سالفة).
- 🟠 **payload ترجمة كبير** (9 حقول × كل locale، بلا scoping).
- 🟠 `allCategories` N+1 محتمل.
- ✅ media conversions non-queued.

## 29. Security Considerations
1. 🟠 **ratings خام** → تسريب email/user_id المقيّمين.
2. 🟠 **`show` لا يفلتر active** → كشف بالـslug.
3. 🟡 القراءة عامة.

## 30. Technical Debt
- **الصوت مخزّن بلا UI** (audioStoryLink/Title غير مستخدمة).
- **mainCharacters غير مترجَم** (Observer معلّق، TODO).
- route param `{id}` مضلّل (slug فعليًا).
- N+1 ratings (accessor).
- payload ترجمة بلا scoping للـlocale.
- SafwestModel مشترك (يخدم swalef).

## 31. Improvement Opportunities
- إضافة **مشغّل صوت** (audio player) لعرض audioStory.
- ترجمة mainCharacters (تفعيل Observer).
- RatingResource يخفي email؛ فلترة active في show.
- scoping ترجمة للـlocale المطلوب فقط (تقليل payload).
- إصلاح N+1 (eager ratings أو withCount).
- تسمية route param slug.

---

## 32. Related Features
| Feature | العلاقة |
|---|---|
| 04 Places | نفس القالب |
| 02 Home | safwests section (show_in_home) |
| 07 Tasneef | قائمة swalefs (type 'stories'→endpoint 'swalefs') |
| 08 Exploration | ضمن البحث/الخريطة (type story) |
| 13 Favorites / 14 Rates | مفضّل/حفظ/تقييم (type 'swalef') |
| 16 UserStories | منفصل (ستوري آخر يوم — لا تخلط) |
| 18 Taxonomy | CategoryOfSwalef الهرمي |

## 33. How to Modify This Feature
- **حقل سردي جديد:** migration swalef_translations + Swalef translatedAttributes + SwalefResource + SafwestModel + widget + Observer.
- **إضافة صوت UI:** widget audio player في stories_details_items_body + SafwestModel.audioStoryLink.
- **APIs:** swalefs, swalefs/{slug}.
- **المخاطر:** N+1 ratings، payload ترجمة، تسريب email، كشف active، mainCharacters غير مترجَم.
- **اختبارات:** تفاصيل (حقول سردية)، قائمة (فلاتر)، صوت (بعد UI)، مفضّل/تقييم.

## 34. Regression Checklist
- [ ] تفاصيل: معرض + عنوان + حقول سردية (أهمية/صلة/حقبة/شخصيات).
- [ ] حقل سردي ناقص → قسم يُخفى/fallback.
- [ ] قائمة swalefs: فلاتر (search/category/top_featured).
- [ ] home safwests (show_in_home).
- [ ] مفضّل/حفظ → refresh.
- [ ] 404؛ لغة/RTL + ترجمة 9 حقول.
- [ ] (بعد UI) مشغّل صوت.

## 35. Common Bugs
| العطل | السبب | ابدأ من |
|---|---|---|
| «الصوت لا يظهر» | لا UI player | `stories_details_items_body` |
| «الشخصيات بلغة واحدة» | mainCharacters غير مترجَم | `SwalefObserver:22` |
| «بطء التفاصيل» | N+1 ratings + payload | `Swalef` accessors |
| «email المقيّم ظاهر» | ratings خام | `SwalefResource` |
| «سالفة غير منشورة ظاهرة» | show بلا active | `SwalefController@show` |
| «حقل سردي فارغ» | ترجمة ناقصة | swalef_translations |

## 36. Debug Guide
1. راقب `GET swalefs/{slug}`.
2. تحقّق الحقول السردية الـ9 في الرد.
3. تتبّع StoriesDetailsCubit (Loading→Success/Error).
4. للأداء: DB::listen (N+1 ratings).
5. للترجمة: بدّل locale + swalef_translations.
6. للصوت: audioStoryLink موجود بالرد لكن بلا widget.

## 37. Search Keywords
`swalef`, `swalefs`, `سوالف`, `story`, `stories`, `heritage`, `SwalefController`, `SwalefResource`, `SwalefListResource`, `SwalefCollection`, `Swalef model`, `SafwestModel`, `StoriesDetailsCubit`, `swalefs/{slug}`, `content`, `timePeriod`, `location`, `storyImportance`, `relevanceToPresent`, `source`, `audioStoryLink`, `audioStoryTitle`, `mainCharacters`, `SwalefObserver`, `swalef_translations`, `CategoryOfSwalef`, `show_in_home`, `safwests`, `شعر مثل قصه`, `TYPE swalef` → **10_Swalef.md**

## 38. Future Improvements
- مشغّل صوت + ترجمة mainCharacters.
- RatingResource (مشترك)؛ فلترة active في show.
- scoping ترجمة للـlocale (تقليل payload).
- إصلاح N+1 ratings.
- عرض source/relevance بتنسيق أغنى.

---
**تدفق مرجعي:** UI → StoriesDetailsCubit → Repo → API → SwalefController → Swalef(9 حقول مترجَمة) → DB(swalefs/swalef_translations) → SwalefResource → SafwestModel → UI ✅ موثّق. **الفرق عن [[04_Places]]: سرد تراثي غني + صوت (مخطّط).**
