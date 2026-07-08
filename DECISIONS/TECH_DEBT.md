# TECH_DEBT — سجل الدَّين التقني

> مرجع موحّد لكل الدَّين المكتشَف عبر الطبقات الثلاث (من تحليل الميزات 01–32).
> **الحالة:** ⬜ مفتوح · 🔧 قيد العمل · ✅ مُغلق. حدّث البند عند الإصلاح (التاريخ + من أغلقه).
> تُحدَّث هذه القائمة في الخطوة 8/11 من أي تدفّق يلمس بندًا هنا.

---

## 🔴 حرج (Critical / High) — أولوية أولى

| # | البند | الطبقة | الميزة | الحالة |
|---|-------|--------|--------|--------|
| D1 | **`monarx-analyzer.php`** أثر RCE — يجلب وينفّذ كودًا عن بُعد، ذاتي الحذف. **حذف فوري.** | Backend | [[26_DashboardAdmin]] | ⬜ |
| D2 | `rates` POST/DELETE بلا `auth` (عام تمامًا) | Backend | [[14_RatesReviews]] | ⬜ |
| D3 | تسريب **email (PII)** في مصفوفة `ratings` الخام بكل الموارد | Backend | [[14_RatesReviews]] | ⬜ |
| D4 | `rate` بلا تحقّق مدى `numeric\|between:1,5` (يقبل 0/سالب/>5/نص) | Backend | [[14_RatesReviews]] | ⬜ |
| D5 | بلا منع تكرار التقييم (unique user+parent+type) → تلاعب بالمتوسط | Backend | [[14_RatesReviews]] | ⬜ |
| D6 | **API بلا authz دقيق** — auth فقط، أي مستخدم يصل أي endpoint | Backend | [[27_RolesPermissions]] | ⬜ |
| D7 | كل الأدوار السبعة تملك **كل الـ145 صلاحية** → RBAC صوري | Backend | [[27_RolesPermissions]] | ⬜ |
| D8 | بيانات دخول مبذورة **`123456`** (root/admin/…) | Backend | [[27_RolesPermissions]] | ⬜ |
| D9 | مفتاح **`accept-tokenapi` مكتوب في المصدر** (Dio + Angular) | Mobile+Web | [[24_Networking]] | ⬜ |
| D10 | **backdoor** تبديل البيئة بالإنتاج (5 نقرات + `dev123`) + بلا build flavors | Mobile | [[24_Networking]] | ⬜ |
| D11 | **PrettyDioLogger دائم التفعيل** → توكنات/PII في السجلات | Mobile | [[24_Networking]] | ⬜ |
| D44 | **`APP_DEBUG=true` في الإنتاج** → تسريب stack trace + مسارات + Debugbar (ظهر في رد 500) | Backend/Ops | عام — [[KNOWN_ISSUES#K2]] | ⬜ |

## 🟡 متوسط (Medium)

| # | البند | الطبقة | الميزة | الحالة |
|---|-------|--------|--------|--------|
| D12 | **401 مُتجاهَل** بالموبايل (لا refresh/logout) — الويب يعمل logout تلقائي | Mobile | [[24_Networking]], [[01_Authentication]] | ⬜ |
| D13 | `logout` لا يمسح `fcm_token` → إشعارات يتيمة لمستخدم سابق | Backend+Mobile | [[21_ProfileSettings]], [[17_Notifications]] | ⬜ |
| D14 | **CORS `allowed_origins: ['*']`** (credentials=false يخفّف) | Backend | [[24_Networking]] | ⬜ |
| D15 | **N+1** على `is_favorite`/`is_saved` لكل عنصر (100 عنصر ⇒ 201 استعلام) | Backend | [[13_FavoritesSaved]] | ⬜ |
| D16 | **N+1** حساب متوسط التقييم (accessors تعيد الاستعلام) | Backend | [[14_RatesReviews]] | ⬜ |
| D17 | **N+1** على الترجمات (`Region::all()` يحمّل كل 4 لغات/سجل) | Backend | [[18_Taxonomy]] | ⬜ |
| D18 | **الترجمة المتزامنة** (كشط Google بلا معالجة أخطاء) → فشل الحفظ عند الانقطاع | Backend | [[31_TranslationObservers]] | ⬜ |
| D19 | حد رفع الوسائط: خادم 2MB مقابل عميل 50MB (تناقض) | Backend+Mobile | [[23_MediaGallery]] | ⬜ |
| D20 | تحويلات الوسائط **nonQueued** تحجب استجابة الرفع | Backend | [[23_MediaGallery]] | ⬜ |
| D21 | **SVG upload مسموح** → خطر stored-XSS | Backend | [[23_MediaGallery]], [[16_UserStories]] | ⬜ |
| D22 | لوحة التحكم بلا `auth` middleware على مستوى المجموعة (per-controller فقط) | Backend | [[26_DashboardAdmin]] | ⬜ |
| D23 | `Place` model `guarded=[]` (mass-assignment) + `Store` مشوّه | Backend | [[26_DashboardAdmin]] | ⬜ |
| D24 | تصعيد صلاحيات: `syncPermissions` بلا فحص من يحقّ له المنح | Backend | [[26_DashboardAdmin]], [[27_RolesPermissions]] | ⬜ |
| D25 | مراجعة المحتوى بلا audit trail + خلط status/active | Backend | [[26_DashboardAdmin]], [[15_MyProperties]] | ⬜ |
| D26 | `show()` بلا فلتر `active=1` (Landmarks + تسريب محتوى معلّق) | Backend | [[12_Landmarks]] | ⬜ |
| D27 | حالة الطلب غير موحّدة (accepted/pending/active) عبر الأنواع | Backend+Mobile | [[15_MyProperties]] | ⬜ |
| D28 | نظام Media ثلاثي متعايش (Spatie + galleries + legacy column) | Backend | [[23_MediaGallery]] | ⬜ |
| D29 | تصنيفات مجزّأة 6 موديلات متطابقة (بلا trait مشترك) | Backend | [[18_Taxonomy]] | ⬜ |
| D30 | عقدان مختلفان للبحث (SearchController موبايل vs `/global-search` ويب) | Backend+Web | [[08_Exploration]] | ⬜ |
| D31 | `contactus/send` عام بلا throttle (spam) — والموبايل نموذجه عرض فقط | Backend+Mobile | [[21_ProfileSettings]] | ⬜ |
| D32 | Force Update **fail-open** عند انقطاع Firebase (تجاوز البوابة) | Mobile | [[20_ForceUpdate]] | ⬜ |
| D33 | `views_num` عمود ميّت (لا يُزاد) → "الأكثر زيارة" وهمي | Backend | [[30_VisitorTracking]] | ⬜ |
| D47 | **تجميد ترتيب `/home` بالموقع + تسرّب اللغة** — كاش مفتاح عام `home_data` يتجاهل الموقع/اللغة، و`getLocationCoords()` داخل الـclosure، والموبايل لا يبعت lat/lng | Backend+Mobile | [[02_Home]] | ✅ (2026-07-08) |
| D45 | بريد `save-trip-to-email` **متزامن بلا try/catch** (`TripController.php:85`) → فشل SMTP يعلّق 31s ويرجّع 500 | Backend | [[03_Trips]] — [[KNOWN_ISSUES#K1]] | ⬜ |
| D46 | كاش صلاحيات Spatie **لا يُفرَّغ برمجيًا** (24h، لا `forgetCachedPermissions()`) → تعديلات لا تسري | Backend | [[27_RolesPermissions]] — [[KNOWN_ISSUES#K3]] | ⬜ |

## 🟢 منخفض / معطّل / سقالة

| # | البند | الطبقة | الميزة | الحالة |
|---|-------|--------|--------|--------|
| D34 | الإشعارات FCM معطّلة بالعميل (لا firebase_messaging؛ كود معلّق) | Mobile | [[17_Notifications]] | ⬜ |
| D35 | Real-time (Reverb/WebSocket) سقالة معطّلة + عدم تطابق قنوات | Mobile+Backend | [[25_Realtime]] | ⬜ |
| D36 | القصص نصف مبنية: endpoint mis-map، بلا expires/seen، `show()` مورد خاطئ | Mobile+Backend | [[16_UserStories]] | ⬜ |
| D37 | `LandmarkShowCubit` + شاشة تفاصيل معالم غير موصولة (dead) | Mobile | [[12_Landmarks]] | ⬜ |
| D38 | Home dummyCategories مكتوبة يدويًا (لا API) → بيانات قديمة | Mobile | [[18_Taxonomy]], [[02_Home]] | ⬜ |
| D39 | نصوص عربية مكتوبة يدويًا تتجاوز i18n (15+، incl. onboarding) | Mobile | [[22_Localization]] | ⬜ |
| D40 | middleware يتجاهل `Accept-Language` القياسي (يعتمد header `locale` مخصّص) | Backend | [[22_Localization]] | ⬜ |
| D41 | `car_requests` migration مفقود (نشر جديد قد يفشل) | Backend | [[28_CarModule]] | ⬜ |
| D42 | تصديرات Report على قرص عام + بلا تنظيف + OOM على الضخم | Backend | [[29_ReportModule]] | ⬜ |
| D43 | `getChatGpt`/`addChatGpt` — تحقّق auth/throttle/كلفة LLM + مكان المفتاح | Backend | [[32_AiAssistant]] | ⬜ |

---

## خارج النطاق (قرار: لا عمل ما لم يتغيّر النطاق)
- CarModel ([[28_CarModule]]) · Report ([[29_ReportModule]]) · الحرّاس/الزوار الداخلي ([[30_VisitorTracking]]-B) — عمليات داخلية غير سياحية، لا تخصّ الموبايل/الويب.

## كيفية الاستخدام
- قبل أي عمل قرب بند: افتحه هنا + مستند الميزة. عند الإصلاح: علّم ✅ + التاريخ في الخطوة 8/11.
- بند جديد يُكتشَف: أضِفه فورًا بنفس الصيغة (#, البند, الطبقة, الميزة, الحالة).
