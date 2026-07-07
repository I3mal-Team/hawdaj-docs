# BUG_FIX — تدفّق إصلاح خطأ

> **Trigger:** المستخدم يقول `اعمل Bug` / `Bug` / `فيه مشكلة` (+ وصف قصير للعطل).
> Claude يقرأ `CLAUDE.md` ويمشي هذا التدفّق **لوحده حتى الموافقة**. لا شرح مطلوب من المستخدم.
> المحور: `WORKFLOWS/MASTER_WORKFLOW.md`.

```
BUG
 ↓
Impact Analysis
 ↓
Plan
 ↓
Approval        ⛔ توقّف هنا
 ↓
Implementation
 ↓
Review
 ↓
Regression
 ↓
Update AI Docs
```

---

## 1. BUG — استيعاب العطل
- خُذ وصف المستخدم القصير (شاشة/رسالة خطأ/سلوك خاطئ).
- **اقرأ:** `FEATURE_INDEX.md` → حدّد الميزة → افتح `FEATURES/NN_*.md` (خصوصًا أقسام **Common Bugs** + **Debug Guide** + **Full Execution Flow**).
- استخدم `SEARCH_INDEX.md` لو الميزة غير واضحة، `API_INDEX.md`/`DATABASE_INDEX.md` لو العطل في endpoint/جدول.
- أعد إنتاج العطل ذهنيًا عبر تدفّق التنفيذ (UI→Cubit→Repo→API→Controller→Model→DB).
- **الغموض؟** اسأل سؤالًا واحدًا محدّدًا (أي شاشة؟ رسالة الخطأ الحرفية؟) — لا تخمّن.

## 2. Impact Analysis
- **السبب الجذري** (root cause) لا العَرَض. حدّد الملف:السطر المرشّح.
- **المنصّات:** هل العطل في الموبايل فقط؟ أم في عقد API (يمسّ الويب كمان)؟ راجع قسم "Web Frontend" بمستند الميزة + `WEB_MOBILE_COMPARISON.md`.
- **الطبقات المتأثّرة:** Flutter / Backend / Web / Database.
- **آثار جانبية:** ميزات مرتبطة (من `FEATURE_GRAPH.md`)، تعريب/RTL، حالات حافّة.
- اذكر صراحة: تغيير مباشر واحد أم سلسلة؟

## 3. Plan
- خيار أو اثنين (لا أكثر). لكل خيار:
  - خطوات عالية المستوى.
  - الملفات/الأسطر بالضبط.
  - إيجابيات/سلبيات.
  - تعقيد (Small/Medium/Large) + مخاطر (Low/Medium/High) + تخفيف.
- **fix السبب الجذري** مُقدَّم على patch العَرَض. لو patch سريع مطلوب مؤقتًا، وضّح أنه مؤقت + سجّله في `DECISIONS/TECH_DEBT.md`.

## 4. ⛔ Approval
- **توقّف. لا كود.** اعرض الخطة وانتظر موافقة صريحة على خيار (أو نسخة معدّلة).
- موافقة غير واضحة → اسأل.

## 5. Implementation
- نفّذ المعتمَد فقط. أصغر تغيير يحلّ السبب الجذري.
- اتبع `STANDARDS/` (حالة Cubit، `Either<Failure,T>`، تسمية، بنية المجلدات).
- **القاعدة الذهبية:** عطل في عقد API → أصلح Backend أولًا، ثم عدّل الموبايل والويب على العقد.
- لا تُصلح أشياء خارج نطاق العطل (scope creep) — لو لقيت عطلًا آخر، سجّله لا تُصلحه بلا موافقة.

## 6. Review (self)
- السبب الجذري انحلّ فعلًا (لا العَرَض فقط)؟
- المعايير/التسمية/الحالة/معالجة الأخطاء سليمة؟
- التعريب/RTL/UX ما تأثّرت سلبًا؟
- لا كود ميّت/تكرار/تعقيد زائد؟
- لا كسر لميزة مرتبطة (راجع `FEATURE_GRAPH.md`)؟

## 7. Regression
- من `TESTING/REGRESSION_CHECKLIST.md` + قسم **Regression Checklist** بمستند الميزة.
- اذكر التدفّقات المتأثّرة + الاختبارات (unit/integration/widget/smoke) المطلوب تشغيلها أو إنشاؤها.
- على الأقل: الحالة اللي كانت تعطل + الحالة العكسية + حالة حافّة واحدة.

## 8. Update AI Docs (إلزامي — الجلسة لا تنتهي قبله · بلا استئذان)
- `FEATURES/NN_*.md`: حدّث **Common Bugs**/**Debug Guide** لو ظهر نمط جديد (ألحِق، لا تُعِد إنشاء).
- **`CHANGE_LOG.md`: سطر للإصلاح (دائمًا).** `PROJECT_PROGRESS.md`: قيّد سرديًا.
- **`KNOWN_ISSUES.md`: لو عطل تشغيلي/بيئي — سجّله أو علّمه ✅ عند الحل.**
- `DECISIONS/TECH_DEBT.md`: لو الإصلاح كشف/سدّ دَينًا، أو كان patch مؤقتًا.
- لو تغيّر عقد API: حدّث `API_INDEX.md` + قسم "Web Frontend" بالمستند + `WEB_MOBILE_COMPARISON.md`. DB → `DATABASE_INDEX.md`. تبعيات → `FEATURE_GRAPH.md`.

---

## مثال مصغّر
> المستخدم: `اعمل Bug — التقييم بنجمة صفر بيتقبل`
> Claude: يفتح `FEATURES/14_RatesReviews.md` → السبب: `RateController` بلا `numeric|between:1,5` (§18) → طبقة Backend (يمسّ الموبايل+الويب) → خطر Medium → خطة: أضف validation + رسالة → ⛔ ينتظر الموافقة.
