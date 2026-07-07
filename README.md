# AI/ — قاعدة المعرفة التقنية لمشروع Hawdaj

توثيق شامل لنظام Hawdaj عبر **3 طبقات**: تطبيق Flutter (موبايل) · Laravel (باك/أدمن) · Angular (ويب).
الهدف: أي مطوّر أو أداة AI يفهم أي ميزة ويعدّل عليها **دون إعادة قراءة الكود من الصفر**.

> كل ما هنا **توثيق فقط** — لا يمسّ كود الإنتاج. الروابط بين الملفات بصيغة `[[NN_Feature]]`.

---

## 🚀 من أين تبدأ (3 خطوات)
1. **`CLAUDE.md`** — الدستور: نظرة عامة + سير العمل + المعايير + أهم الدَّين. (5 دقائق)
2. **`DECISIONS/TECH_DEBT.md`** — 43 بند مخاطر مرتّبة بالأولوية (أخطرها أمني حرج). 
3. **`FEATURE_INDEX.md`** — فهرس الـ32 ميزة، ثم افتح ما يهمّك في `FEATURES/`.

---

## 🗂️ الهيكل

```
AI/
├── README.md                 هذا الملف
├── CLAUDE.md                 ⭐ الدستور — يُقرأ أول أي عمل
├── FEATURE_INDEX.md          فهرس الـ32 ميزة + عمود تغطية الويب — نقطة البداية
├── PROJECT_PROGRESS.md       الحالة + سجل سردي زمني
├── CHANGE_LOG.md             سطر لكل تغيير (أحدث أولًا)
├── KNOWN_ISSUES.md           أعطال/حوادث حيّة (K1..)
│
├── FEATURE_GRAPH.md          تبعيات الميزات + أثر التغيير
├── SEARCH_INDEX.md           كلمة مفتاحية → ميزة
├── API_INDEX.md              endpoint → ميزة + Controller
├── DATABASE_INDEX.md         جدول → Model + الميزات
│
├── WEB_FRONTEND.md           معمارية واجهة الويب (Angular 16 SSR)
├── WEB_MOBILE_COMPARISON.md  مقارنة ويب↔موبايل + الناقص + التوصيات
│
├── FEATURES/                 مستند لكل ميزة (01..32) — 38 قسمًا + قسم "Web Frontend"
│                             (تدفّق التنفيذ · ملفات:أسطر · مخاطر · أخطاء شائعة · debug · regression)
│
├── WORKFLOWS/                سير العمل (كلمة تشغيل قصيرة → pipeline بموافقة إلزامية)
│   ├── MASTER_WORKFLOW.md    المحور + كلمات التشغيل
│   ├── BUG_FIX.md · NEW_FEATURE.md · REFACTOR.md
│   └── CODE_REVIEW.md · PERFORMANCE.md · SECURITY.md
│
├── STANDARDS/                أنماط الكود + سياسات إلزامية
│   ├── ARCHITECTURE.md · FOLDER_STRUCTURE.md · NAMING.md
│   ├── STATE_MANAGEMENT.md · API_RULES.md · ERROR_HANDLING.md · CODING_STANDARDS.md
│   ├── Impact_Analysis.md      ⚠️ إلزامي قبل أي تنفيذ (تحليل أثر كامل)
│   ├── Completion_Gate.md      ✅ إلزامي قبل الإتمام (بند ناقص = INCOMPLETE)
│   └── Documentation_Sync.md   🔄 إلزامي: حدّث hawdaj-docs + commit + push (SSOT)
│
├── TESTING/                  REGRESSION_CHECKLIST · FEATURE_TESTS · SMOKE_TESTS
│
└── DECISIONS/
    ├── TECH_DEBT.md          43 بند دَين مصنّف بالخطورة + مربوط بالميزات
    └── ARCHITECTURE_DECISIONS.md   12 قرارًا معماريًا + القاعدة الذهبية
```

---

## 🧭 كيف تستخدمها عمليًا

- **تبغى تفهم ميزة؟** `FEATURE_INDEX.md` → مستند الميزة في `FEATURES/`.
- **تبغى تصلح bug؟** اقرأ قسمَي **Common Bugs** + **Debug Guide** بمستند الميزة، واتبع `WORKFLOWS/BUG_FIX.md`.
- **endpoint غامض؟** `API_INDEX.md`. **جدول غامض؟** `DATABASE_INDEX.md`. **كلمة؟** `SEARCH_INDEX.md`.
- **تعدّل شيء؟** شوف من يتأثّر في `FEATURE_GRAPH.md`.
- **فرق الويب عن الموبايل؟** `WEB_MOBILE_COMPARISON.md`.

## ⚖️ قواعد ذهبية
1. **عقد API يتغيّر → Backend أولًا**، ثم الموبايل والويب على العقد الجديد، ثم التوثيق.
2. **لا كود قبل الموافقة** — كل تدفّق عمل يمرّ ببوابة موافقة (الخطوة 7).
3. **لا تنفيذ بلا تحليل أثر** — `STANDARDS/Impact_Analysis.md` إلزامي قبل التنفيذ (كل الأبعاد + Risk + Deps + Breaking + Migration + Rollback + Testing).
4. **لا إتمام بلا بوابة الإنجاز** — `STANDARDS/Completion_Gate.md`: أي بند ناقص (ومنه مزامنة التوثيق) = ❌ INCOMPLETE.
5. **مزامنة التوثيق إلزامية** — `STANDARDS/Documentation_Sync.md`: أي تغيير كود يؤثّر على المعرفة → حدّث `hawdaj-docs` ثم **commit + push** قبل الإتمام. **`hawdaj-docs` = مصدر الحقيقة الوحيد؛ يجب ألّا يصبح أقدم من الكود.**

## 📌 أخطر ما يحتاج إجراء عاجل
راجع `DECISIONS/TECH_DEBT.md` القسم 🔴 — أبرزها: أثر RCE (`monarx-analyzer.php`)، `rates` بلا auth + تسريب email، الـAPI بلا صلاحيات دقيقة، backdoor بيئة بالإنتاج.
