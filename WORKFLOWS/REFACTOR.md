# REFACTOR — تدفّق إعادة الهيكلة

> **Trigger:** `refactor` / `اعد الهيكلة` / `نظّف`. المحور: `MASTER_WORKFLOW.md`.
> **مبدأ:** إعادة الهيكلة **تحافظ على السلوك بالضبط** — لا تغيير وظيفي. لو تبغى تغيّر سلوك، هذا BUG_FIX أو NEW_FEATURE مش refactor.

```
TARGET
 ↓
Behavior Baseline (وثّق السلوك الحالي + الاختبارات القائمة)
 ↓
Impact + Blast Radius
 ↓
Plan (خطوات صغيرة عكوسة)
 ↓
Approval        ⛔
 ↓
Implementation (خطوة خطوة، لا تغيير سلوك)
 ↓
Review (سلوك متطابق؟)
 ↓
Regression (نفس النتائج قبل/بعد)
 ↓
Update AI Docs
```

## 1. Target + Baseline
- افتح مستند الميزة → افهم **Full Execution Flow** الحالي.
- وثّق السلوك المرجعي (inputs→outputs) + الاختبارات الموجودة. لو مافي اختبارات، اكتب smoke قبل البدء.

## 2. Impact / Blast Radius
- من `FEATURE_GRAPH.md`: من يعتمد على الكود المُعاد هيكلته؟
- عقد API لا يتغيّر (وإلا يكسر الموبايل+الويب). التوقيعات العامة تبقى.
- الطبقات المتأثّرة + المستهلكون.

## 3. Plan
- خطوات صغيرة **عكوسة** (كل خطوة تُختبر وحدها). لا refactor عملاق.
- ملفات · تعقيد · مخاطر. حدّد ما **لن** يتغيّر (العقود/السلوك).

## 4. ⛔ Approval — توقّف.

## 5. Implementation
- خطوة خطوة. بعد كل خطوة: السلوك ثابت.
- اتبع `STANDARDS/`. لا scope creep (لا إصلاح bugs أثناء الـrefactor — سجّلها).
- استخدم worktree/فرع منفصل لو التغيير واسع.

## 6. Review — السلوك متطابق 100%؟ لا تغيير عقد؟ الكود أنظف فعلًا (أقل تكرار/تعقيد)؟

## 7. Regression — نفس المخرجات قبل/بعد. شغّل كل الاختبارات المتأثّرة + smoke.

## 8. Update AI Docs
- حدّث مستند الميزة لو تغيّرت الملفات/البنية (Files Involved، How to Modify).
- `PROJECT_PROGRESS.md` سجل. `DECISIONS/TECH_DEBT.md`: علّم الدَّين المُسدَّد أو المتبقّي.
- `STANDARDS/` لو الـrefactor أرسى نمطًا جديدًا يُعتمد.

---

## سياسات إلزامية (Governance)
- **مزامنة التوثيق (إلزامي):** أي أثر على قاعدة المعرفة → حدّث `hawdaj-docs` ثم **commit + push** قبل الإتمام — [[Documentation_Sync]].
- **قبل التنفيذ (خطوة 2/Impact + Blast Radius):** تحليل أثر كامل — [[Impact_Analysis]]. ركّز على Breaking Changes = **لا شيء** (العقود/السلوك ثابتة) + Rollback (خطوات عكوسة).
- **قبل الإتمام:** استوفِ [[Completion_Gate]] — أي بند ناقص (ومنه مزامنة التوثيق) = ❌ INCOMPLETE.
