# CHANGE_LOG — سجل التغييرات

> سطور مضغوطة قابلة للمسح السريع، **أحدث أولًا**. لكل تغيير فعلي (كود أو توثيق).
> الصيغة: `التاريخ · [النوع] · الميزة · ملخّص · الملفات`. الأنواع: `feat` `fix` `docs` `chore` `security`.
> `PROJECT_PROGRESS.md` = سرد/حالة تفصيلية · هذا الملف = سطر واحد لكل تغيير. يُحدَّث في الخطوة 11 من أي تدفّق.

---

## 2026-07-08
- `fix` · [[02_Home]] · **إصلاح ترتيب `/home` بالموقع (باك).** نقل `getLocationCoords()` خارج الـclosure المكاش + دالة `homeCacheKey()` جديدة (مفتاح = locale + خلية موقع مبطّنة ~1km + md5(params: page/‏*_per_page/search)) + توقيع `fetchHomeData($request,$location)` + TTL 3600→600. يزيل تجميد الترتيب وتسرّب اللغة عبر المستخدمين. شكل الرد مطابق. مُختبَر منطق المفتاح standalone (نفس الخلية→نفس المفتاح، بعيد/لغة/noloc/page→مختلف). · `hawdaj-api/app/Http/Controllers/Api/HomeController.php`
- `fix` · [[02_Home]] [[24_Networking]] · **إيصال الموقع لـ`/home` (موبايل).** `HomeCubit.fetchHome({lat,lng})` (fallback على `LocationCache`) + `HomeRepo(Imp)` يمرّر `queryParameters` + `BlocListener<LocationCubit>` في `home_view_body` يعيد الجلب عند `LocationLoaded`. (سابقًا get عارٍ بلا موقع، LocationCubit غير مُستهلَك.) `flutter analyze` نظيف. · `hawdaj/Untitled/lib/features/home/{presentation/manager/home_cubit,data/repo,presentation/view/widgets/shared/home_view_body}`
- `docs` · [[02_Home]] [[24_Networking]] · مزامنة: تعليم تلوّث كاش الموقع/تجميد الترتيب محلولًا (§12/13/24/28/29/35/30)، بند [[TECH_DEBT#D47]] ✅. · `FEATURES/02_Home.md`, `FEATURES/24_Networking.md`, `DECISIONS/TECH_DEBT.md`, `PROJECT_PROGRESS.md`

## 2026-07-07
- `docs` · Governance · **سياسة "ابحث قبل أن تسأل" (Search_Before_Ask).** إنشاء `STANDARDS/Search_Before_Ask.md`: قبل أي سؤال توضيحي، ابحث في KB (Common Bugs/Debug Guide/Known Incidents/KNOWN_ISSUES/TECH_DEBT/CHANGE_LOG/SEARCH_INDEX) → حالة مشابهة تُعرَض بنسبة تشابه؛ وإلا سؤال واحد فقط. دُمجت في `CLAUDE.md §2`, `MASTER_WORKFLOW`, `BUG_FIX` (خطوة 1), README (قاعدة 6 + الشجرة). · `STANDARDS/Search_Before_Ask.md`, `CLAUDE.md`, `WORKFLOWS/MASTER_WORKFLOW.md`, `WORKFLOWS/BUG_FIX.md`, `README.md`
- `docs` · Governance · **سياسة مزامنة التوثيق الإلزامية (Documentation_Sync).** إنشاء `STANDARDS/Documentation_Sync.md`: SSOT + تدفّق القرار (Code change → يؤثّر؟ → Update hawdaj-docs → commit → push → complete) + التحذير الإلزامي (نص عربي) + شروط INCOMPLETE (منها commit+push). دمجها في `CLAUDE.md §2` (+ التحذير + خطوة 12)، `Completion_Gate` (بندَا commit+push للتوثيق + التحذير)، `MASTER_WORKFLOW` (قاعدة ثابتة)، وكل التدفّقات السبعة (بند مزامنة في كتلة Governance)، README (قاعدة 5 + الشجرة)، شجرة CLAUDE. · `STANDARDS/Documentation_Sync.md`, `CLAUDE.md`, `STANDARDS/Completion_Gate.md`, `WORKFLOWS/*`, `README.md`
- `docs` · Governance · **سياستان إلزاميتان: Impact Analysis + Completion Gate.** إنشاء `STANDARDS/Impact_Analysis.md` (كل الأبعاد + Risk + Deps + Breaking + Migration + Rollback + Testing) + `STANDARDS/Completion_Gate.md` (قائمة إتمام؛ بند ناقص = INCOMPLETE). دمجهما في `CLAUDE.md §2` (خطوة 3 Impact + خطوة 12 Gate)، `MASTER_WORKFLOW` (pipeline 12 خطوة + قواعد ثابتة)، وكل التدفّقات (BUG_FIX/NEW_FEATURE/REFACTOR/PERFORMANCE/SECURITY/CODE_REVIEW) عبر كتلة "سياسات إلزامية". تحديث README + شجرة CLAUDE. · `STANDARDS/Impact_Analysis.md`, `STANDARDS/Completion_Gate.md`, `CLAUDE.md`, `WORKFLOWS/*`, `README.md`
- `chore` · Governance · **نقل قاعدة المعرفة لريبو مستقل `hawdaj-docs` (I3mal-Team) كمصدر حقيقة وحيد.** إنشاء `CLAUDE.md` خفيف في كل ريبو كود (mobile/back/web) يشير للريبو المشترك (تحميل سياق إلزامي + وعي عابر للريبوهات + سياسة مزامنة). حذف النسخة المتداخلة المكرّرة من ريبو الموبايل. تحديث دستور الريبو المشترك §0. · `hawdaj/Untitled/CLAUDE.md`, `hawdaj-api/CLAUDE.md`, `hawdaj-frontend/CLAUDE.md`, `hawdaj-docs/CLAUDE.md`
- `docs` · Governance · إضافة سياسة مزامنة KB + إنشاء `CHANGE_LOG.md` + `KNOWN_ISSUES.md` + تسجيل حوادث اليوم (K1–K4) + بنود دَين D44–D46 · `AI/CLAUDE.md`, `AI/KNOWN_ISSUES.md`, `AI/CHANGE_LOG.md`, `AI/DECISIONS/TECH_DEBT.md`, `AI/WORKFLOWS/*`, `AI/FEATURE_INDEX.md`, `AI/README.md`
- `docs` · [[03_Trips]] · تسجيل حادثة SMTP 500 (بريد متزامن) في Common Bugs/Security · `AI/FEATURES/03_Trips.md`
- `docs` · [[26_DashboardAdmin]] [[27_RolesPermissions]] [[21_ProfileSettings]] · تسجيل كاش الصلاحيات + بريد الإنشاء queued/catch صامت · مستندات الميزات
- `docs` · Testing · أُضيف سجل الاختبارات الفعلية (bloc_test + mocktail، أول suite: SocialAuth) · `AI/TESTING/FEATURE_TESTS.md` *(تعديل خارجي — مُلتقَط)*

## 2026-07-06
- `docs` · Governance · بناء طبقة الحوكمة: الدستور + WORKFLOWS(7) + STANDARDS(7) + TESTING(3) + DECISIONS(2) + README · `AI/CLAUDE.md`, `/CLAUDE.md`, `AI/WORKFLOWS/*`, `AI/STANDARDS/*`, `AI/TESTING/*`, `AI/DECISIONS/*`, `AI/README.md`
- `docs` · Web · Phase 3: دمج واجهة الويب (Angular) — WEB_FRONTEND + WEB_MOBILE_COMPARISON + الميزة 32 (AI ChatGPT) + أقسام "Web Frontend" في 01/02/03/04/05/08/21/22/24 · `AI/WEB_*.md`, `AI/FEATURES/32_AiAssistant.md`, `AI/FEATURE_INDEX.md`
- `docs` · KB · Phase 2: توثيق 31 ميزة (FEATURES/01–31) + 6 فهارس · `AI/FEATURES/*`, `AI/FEATURE_INDEX.md`, `AI/*_INDEX.md`, `AI/FEATURE_GRAPH.md`

---

## كيفية الاستخدام
- بعد أي تنفيذ (كود/توثيق): أضِف سطرًا هنا في الأعلى.
- اربط الميزة بـ`[[NN_Name]]`. اذكر الملفات الرئيسية فقط (لا تعداد كامل).
- التغييرات الأمنية: نوع `security` + سجّل مقابلها في `KNOWN_ISSUES.md`/`TECH_DEBT.md`.
