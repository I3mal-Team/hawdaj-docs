# MASTER_WORKFLOW — المحور

> المرجع الكامل لسير العمل الذي يشير إليه `AI/CLAUDE.md §2`.
> **الفكرة:** المستخدم يقول كلمة تشغيل قصيرة → Claude يقرأ الدستور → يمشي الـpipeline المناسب **لوحده حتى نقطة الموافقة** → لا ينفّذ كود قبلها.

---

## كلمات التشغيل (Triggers)

| يقول المستخدم | Claude يفتح | الملف |
|---------------|------------|-------|
| `اعمل Bug` / `Bug` / `فيه مشكلة` / `bug fix` | تدفّق إصلاح خطأ | `WORKFLOWS/BUG_FIX.md` |
| `feature جديدة` / `اعمل feature` / `new feature` | تدفّق ميزة جديدة | `WORKFLOWS/NEW_FEATURE.md` |
| `refactor` / `اعد الهيكلة` | تدفّق إعادة هيكلة | `WORKFLOWS/REFACTOR.md` |
| `review` / `راجع` / `code review` | تدفّق مراجعة | `WORKFLOWS/CODE_REVIEW.md` |
| `بطيء` / `performance` / `أداء` | تدفّق أداء | `WORKFLOWS/PERFORMANCE.md` |
| `security` / `أمان` / `ثغرة` | تدفّق أمان | `WORKFLOWS/SECURITY.md` |

المستخدم لا يشرح التفاصيل. Claude يستنتج الميزة المتأثّرة من `FEATURE_INDEX.md` ويسأل فقط عند الغموض.

---

## الـPipeline العام (12 خطوة — من الدستور §2)

```
TRIGGER (كلمة قصيرة)
        ↓
[1] Read AI Docs        قائمة التحميل الإلزامية (CLAUDE + الفهارس + مستند الميزة + KNOWN_ISSUES)
        ↓
[2] Affected Features   طابق الطلب على FEATURE_INDEX (أو اقترح ميزة جديدة)
        ↓
[3] ⚠️ IMPACT ANALYSIS  إلزامي — القالب الكامل في STANDARDS/Impact_Analysis.md
                        (كل الأبعاد + Risk Level + Deps + Breaking + Migration + Rollback + Testing)
        ↓
[4] Affected Layers     Flutter · Backend · Web · Database  (ضمن تحليل الأثر)
        ↓
[5] Risks               Low/Medium/High/Critical + تخفيف
        ↓
[6] Plan                خيار أو اثنين: خطوات · ملفات · إيجابيات/سلبيات · تعقيد (S/M/L)
        ↓
[7] ⛔ APPROVAL          توقّف. لا كود قبل موافقة صريحة. غموض؟ اسأل.
        ↓
[8] Implementation      نفّذ المعتمَد · اتبع STANDARDS/ · صغير ومركّز
        ↓
[9] Self Review         معايير · تسمية · حالة · أخطاء · تعريب · لا كود ميّت
        ↓
[10] Regression         checklist من TESTING/ (unit/integration/widget/smoke)
        ↓
[11] Update AI Docs     مستند الميزة (ألحِق) · INDEX · PROGRESS · CHANGE_LOG (دائمًا)
                        · KNOWN_ISSUES (حادثة) · TECH_DEBT/DECISIONS (دَين/قرار)
        ↓
[12] ✅ COMPLETION GATE   القائمة الإلزامية في STANDARDS/Completion_Gate.md
                        أي بند ناقص → ❌ INCOMPLETE (المهمة لا تُعتبر منتهية)
```

> الخطوتان الملزِمتان: **[3] Impact Analysis** ([[Impact_Analysis]]) قبل أي تنفيذ · **[12] Completion Gate** ([[Completion_Gate]]) قبل إعلان الإتمام. لا تدفّق يبدأ تنفيذًا بلا [3]، ولا تدفّق ينتهي بلا [12] (ومنه مزامنة التوثيق).

---

## قواعد ثابتة عبر كل التدفّقات

- **تحليل الأثر (خطوة 3) إلزامي قبل أي تنفيذ** — [[Impact_Analysis]]. لا كود بلا تحليل أثر كامل.
- **مزامنة التوثيق إلزامية** — [[Documentation_Sync]]: أي تغيير كود يؤثّر على المعرفة → حدّث `hawdaj-docs` ثم **commit + push**. الكود والتوثيق ريبوهان منفصلان؛ التحديث المحلي لا يكفي.
- **بوابة الإنجاز (خطوة 12) إلزامية قبل إعلان الإتمام** — [[Completion_Gate]] (تشمل commit+push للتوثيق). بند ناقص = ❌ INCOMPLETE.
- **بوابة الموافقة (خطوة 7) إلزامية.** لا استثناء إلا لو المستخدم قال صراحة "نفّذ بدون خطة".
- **القاعدة الذهبية:** تغيير يمسّ عقد API → **Backend أولًا**، ثم الموبايل والويب، ثم التوثيق.
- **قراءة قبل الكتابة:** لا تعدّل/تحذف ملفًا لم تفحصه.
- **الأمان:** اكتب عادي (بلا اختصار) في المهام/التأكيدات الأمنية.
- **أخرِج دائمًا:** ماذا تنوي · لماذا آمن · كيف يتوافق مع المعمارية.
- **حدّث التوثيق دائمًا (خطوة 11)** — الجلسة لا تنتهي قبلها. **بلا استئذان** (تحديث التوثيق جزء من المهمة). كل تغيير → سطر في `CHANGE_LOG`؛ كل حادثة تشغيلية → بند في `KNOWN_ISSUES`.

---

## الحالة
🏁 **قاعدة المعرفة مكتملة بالكامل.**
- ✅ WORKFLOWS (7) · DECISIONS (2) · STANDARDS (7) · TESTING (3).
- ✅ FEATURES (01–32) · 6 فهارس · WEB docs · CLAUDE.md + جذر مؤشّر.
