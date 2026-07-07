# CLAUDE.md — دستور مشروع Hawdaj

> هذا الملف هو **الدستور** وقاعدة المعرفة المشتركة تعيش في ريبو مستقل: **`/Users/mac/hawdaj-docs`** (GitHub: `I3mal-Team/hawdaj-docs`) — **مصدر الحقيقة الوحيد** لكامل نظام Hawdaj.
> ريبوهات الكود الثلاثة (موبايل/باك/ويب) تحوي فقط `CLAUDE.md` خفيفًا يشير هنا. أي جلسة في أي ريبو تبدأ من هذا الدستور.
> Language: رُدّ على المستخدم بالعربية. اكتب الكود/الـcommits/التوثيق بشكل احترافي عادي.

---

## 0. نظرة عامة على المشروع

**Hawdaj** = نظام سياحة سعودي، **ثلاث طبقات** تتشارك نفس الـBackend/API:

| الطبقة | المستودع | التقنية |
|--------|----------|---------|
| **Mobile** | `/Users/mac/hawdaj/Untitled` | Flutter (v5.10.23+40) · Cubit/flutter_bloc · get_it · go_router · Dio · dartz Either · easy_localization |
| **Backend + Admin** | `/Users/mac/hawdaj-api` | Laravel 8.12 · Sanctum (Bearer) · MySQL · Astrotomic Translatable · Spatie Permission/MediaLibrary · nwidart modules |
| **Web** | `/Users/mac/hawdaj-frontend` | Angular 16 SSR (NgUniversal) · RxJS services · HttpClient · ngx-translate · PrimeNG/Material/Bootstrap |

| **Shared AI KB** | `/Users/mac/hawdaj-docs` | هذا الريبو — مصدر الحقيقة الوحيد (docs فقط، لا كود) |

- **Base URLs:** prod `https://dashboard.hawdaj.net/api/` · test `https://test.dashboard.hawdaj.net/api/`.
- **Response envelope:** `{code/status, message, data}` (كل الطبقات).
- **Locales:** ar/en/ru/zh + RTL.
- **Payments:** لا توجد.
- **Cross-Repo:** ريبو الكود ليس معزولًا — الـAPI مشترك بين الموبايل والويب. قبل أي تعديل حدّد الأثر عبر Mobile/Backend/Web/DB/APIs. القاعدة الذهبية: تغيير عقد API يبدأ Backend أولًا. بعد أي تنفيذ: زامن هذا الريبو (docs) — المهمة لا تكتمل قبلها.

---

## 1. هيكل قاعدة المعرفة (AI/)

**ابدأ دائمًا من `FEATURE_INDEX.md` → مستند الميزة → الفهارس المتقاطعة. لا تُعِد تحليل الكود من الصفر.**

```
AI/
├── CLAUDE.md                 ⭐ الدستور (هذا الملف)          ✅
├── README.md                 فهرس عام للمطوّر                 ✅
├── FEATURE_INDEX.md          فهرس الـ32 ميزة + عمود الويب     ✅ نقطة البداية
├── PROJECT_PROGRESS.md       الحالة + سجل سردي                ✅
├── CHANGE_LOG.md             سطر لكل تغيير (أحدث أولًا)        ✅
├── KNOWN_ISSUES.md           أعطال/حوادث حيّة (K1..)          ✅
├── FEATURE_GRAPH.md          تبعيات الميزات + أثر التغيير      ✅
├── SEARCH_INDEX.md           كلمة → ميزة                     ✅
├── API_INDEX.md              endpoint → ميزة + controller     ✅
├── DATABASE_INDEX.md         جدول → model + ميزات            ✅
├── WEB_FRONTEND.md           معمارية الويب (Angular)          ✅
├── WEB_MOBILE_COMPARISON.md  مصفوفة ويب↔موبايل + توصيات       ✅
├── FEATURES/                 مستند لكل ميزة 01..32 (38 قسمًا) ✅
├── WORKFLOWS/                MASTER_WORKFLOW · BUG_FIX · NEW_FEATURE · REFACTOR · CODE_REVIEW · PERFORMANCE · SECURITY   ✅
├── DECISIONS/                TECH_DEBT · ARCHITECTURE_DECISIONS   ✅
├── STANDARDS/                ARCHITECTURE · CODING_STANDARDS · NAMING · FOLDER_STRUCTURE · STATE_MANAGEMENT · API_RULES · ERROR_HANDLING   ✅
└── TESTING/                  REGRESSION_CHECKLIST · FEATURE_TESTS · SMOKE_TESTS   ✅
```

كل مستند ميزة فيه 38 قسمًا + قسم **"Web Frontend"** عند اختلاف الويب. الروابط `[[NN_Name]]` تربط الميزات ببعضها.

---

## 2. سير العمل الإلزامي (قبل أي مهمة)

> **سياسة حاكمة:** قاعدة المعرفة `AI/` جزء من الكود ومصدر الحقيقة. تبقى **مزامنة دائمًا** مع التنفيذ.
> **تحديث التوثيق جزء من المهمة — لا يُستأذَن عليه.** لا تُنهِ مهمة والتوثيق قديم. لو لاحظت اختلافًا بين الكود والتوثيق (تغيير مطوّر آخر)، حدّث التوثيق تلقائيًا.

**قائمة التحميل الإلزامية (اقرأها قبل أي مهمة):**
`CLAUDE.md` · `FEATURE_INDEX.md` · `FEATURE_GRAPH.md` · `API_INDEX.md` · `DATABASE_INDEX.md` · `PROJECT_PROGRESS.md` · `SEARCH_INDEX.md` · `KNOWN_ISSUES.md` · مستند الميزة المتأثّرة · الـworkflow المناسب · المعايير (`STANDARDS/`). لا تتخطَّ هذه الخطوة.

نفّذ هذا التسلسل بالترتيب. **لا تنفّذ كود قبل الخطوة 7 (الموافقة).**

1. **اقرأ توثيق AI/** — قائمة التحميل أعلاه (FEATURE_INDEX أولًا، ثم مستند الميزة، ثم الفهارس، ثم KNOWN_ISSUES).
2. **حدّد الميزات المتأثّرة** — طابق الطلب على `FEATURE_INDEX.md`. لو ميزة جديدة، اقترح إدخالًا: الاسم، وصف مختصر، المنصّات المتأثّرة (Flutter/Backend/Web/DB)، التبعيات.
3. **تحليل الأثر (Impact Analysis)** — سلوك الميزة الحالي، الاتساق عبر المنصّات (موبايل/ويب/باك)، الأداء/الأمان/UX. اذكر صراحة: التغييرات المباشرة، الآثار الجانبية، تحديثات التوثيق/الاختبارات المطلوبة.
4. **اكتشف المكوّنات المتأثّرة** — صنّفها:
   - **Flutter:** شاشات/widgets/تنقّل (go_router)/حالة (Cubit)/أنيميشن.
   - **Backend:** APIs/منطق عمل/تكاملات/jobs/observers.
   - **Web:** مكوّنات Angular/خدمات/تصميم متجاوب/SSR/SEO.
   - **Database:** schemas/migrations/فهارس/queries/models.
5. **اكتشف المخاطر** — عدّدها وصنّفها (Low/Medium/High) مع تخفيف: كسر تدفّق قائم، تدهور أداء، أمان/خصوصية، تعريب/إتاحة، اتساق عبر المنصّات.
6. **اقترح أفضل تنفيذ** — خيار أو اثنين (لا أكثر). لكل خيار: خطوات عالية المستوى، الملفات/الوحدات، إيجابيات/سلبيات، التعقيد (Small/Medium/Large).
7. **انتظر الموافقة الصريحة** — لا تبدأ التنفيذ قبل موافقة المستخدم على خيار أو نسخة معدّلة. لو الموافقة غير واضحة، اسأل.
8. **نفّذ الخطة المعتمَدة** — اتبع الأنماط القائمة (§3). تغييرات مركّزة وصغيرة. تعليقات فقط عند منطق غير بديهي.
9. **مراجعة ذاتية** — المعايير/التسمية/نمط الحالة/معالجة الأخطاء/التعريب/UX سليمة؟ لا كود ميّت ولا تكرار ولا تعقيد زائد؟
10. **checklist انحدار (Regression)** — التدفّقات المتأثّرة + الاختبارات المطلوبة (unit/integration/widget/smoke). قابلة لإعادة الاستخدام.
11. **حدّث توثيق AI/ (إلزامي — المهمة لا تكتمل بدونه):**
   - مستند الميزة المتأثّر (ألحِق/حدّث، لا تُعِد إنشاءه) + قسم "Web Frontend" لو تغيّر الويب.
   - `FEATURE_INDEX` · `PROJECT_PROGRESS` · **`CHANGE_LOG` (دائمًا، سطر لكل تغيير)**.
   - `API_INDEX` لو تغيّر API · `DATABASE_INDEX` لو تغيّر DB · `FEATURE_GRAPH` لو تغيّرت التبعيات.
   - **`KNOWN_ISSUES`** لو حادثة تشغيلية/بيئية · `DECISIONS/TECH_DEBT` لو دَين جديد/مُسدَّد · `DECISIONS/ARCHITECTURE_DECISIONS` لو قرار معماري.

---

## 3. المعايير المعمارية (اتبع الموجود، لا تخترع)

**Flutter (Mobile):**
- Feature-First + Clean Architecture رفيعة: `lib/features/<feature>/{data,presentation}`.
- الحالة: **Cubit** (flutter_bloc). DI: **get_it**. التنقّل: **go_router**. الأخطاء: **`Either<Failure,T>`** (dartz).
- الشبكة: `DioConsumer` + interceptors (Bearer/locale/geo). Endpoints في `core/databases/api/end_points.dart`.
- التعريب: `easy_localization` + `.tr()` + `EdgeInsetsDirectional` (لا نصوص عربية مكتوبة يدويًا).

**Backend (Laravel):**
- `ApiModalController` للـenvelope. Translatable (Astrotomic) `{model}_translations`. Media (Spatie small 250 / medium 500). Soft deletes.
- API vs Dashboard منفصلان: `auth('api')` (Sanctum) للموبايل/الويب · `auth:web` + Spatie permissions للوحة.

**Web (Angular):**
- خدمات RxJS (لا NgRx). Reactive Forms + validators. HttpClient + interceptors (نفس الهيدرز). SSR + SEO (meta/hreflang/canonical). `/:lang/` prefix.

**قاعدة ذهبية:** أي تغيير يمسّ **عقد API** → عدّل **Backend أولًا**، ثم حدّث الموبايل والويب على العقد الجديد، ثم وثّق في AI/.

---

## 4. حواجز الأمان (Guardrails)

- لا تعديلات كود إنتاج بلا موافقة (§2.7). صغيرة ومتوافقة رجعيًا.
- لا افتراض متطلبات ناقصة — اسأل أو اقترح خيارات.
- لا حذف/استبدال ملف لم تُنشئه بلا فحصه أولًا.
- إجراءات خارجية أو صعبة التراجع → أكّد أولًا.
- الأمان: اكتب الكود بشكل عادي (بلا caveman) في المهام الأمنية والتأكيدات.

---

## 5. دَين تقني حرج معروف (سياق قبل أي عمل قريب منه)

**Backend (يخدم العميلين — إصلاحه ينفع الاثنين):**
- ⚠️ **`monarx-analyzer.php`** — أثر RCE (يجلب وينفّذ كودًا عن بُعد). للحذف الفوري. [[26_DashboardAdmin]]
- **التقييمات (14):** POST/DELETE `rates` بلا auth + تسريب email(PII) في مصفوفة ratings + بلا مدى 1–5 + بلا منع تكرار + N+1.
- **التفويض (27):** الـAPI بلا authz دقيق (auth فقط). كل الأدوار = كل الصلاحيات. بيانات دخول مبذورة `123456`.
- **CORS `*`** + مفتاح `accept-tokenapi` مكتوب في المصدر.
- **الوسائط (23):** حد حجم خادم 2MB مقابل عميل 50MB. تحويلات nonQueued تحجب. SVG مسموح (XSS).
- **الترجمة (31):** كشط Google متزامن بلا معالجة أخطاء → فشل الحفظ عند انقطاع الخدمة.

**Mobile:**
- **401 مُتجاهَل** في الشبكة (الويب يعمل logout تلقائي — اعتمد سلوك الويب).
- **backdoor** تبديل البيئة بالإنتاج (`dev123`) + logger يسرّب توكنات + `fcm_token` لا يُمسح عند logout.

**معطّل/سقالة:** الإشعارات(17) + Real-time(25) معطّلان بالعميل. القصص(16) نصف مبنية. views_num(30) عمود ميّت.

**خارج النطاق (backend/admin فقط):** CarModel(28)، Report(29)، الحرّاس/الزوار الداخلي(30-B).

---

## 6. الميزات الحصرية لكل منصّة

- **ويب فقط:** #32 مساعد AI ChatGPT · SSR/SEO · تصدير PDF/طباعة للرحلة · بحث موحّد `/global-search` · نماذج اتصل بنا/النشرة تعمل فعليًا · دخول اجتماعي عبر `/social/callback`.
- **موبايل فقط:** Splash & Onboarding(19) · Force Update(20) · (FCM/Realtime مُخطَّطان لكن معطّلان).
- **الرئيسية:** الموبايل نداء واحد `/home`؛ الويب 8+ نداءات دقيقة.

---

## 7. التواصل

- لغة تقنية موجزة لمطوّر موبايل خبير.
- عند عدم اليقين: اقترح خيارات واطرح أسئلة محدّدة بدل التخمين.
- اطرح دائمًا: **ماذا تنوي · لماذا آمن · كيف يتوافق مع المعمارية القائمة**.
