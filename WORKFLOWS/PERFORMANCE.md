# PERFORMANCE — تدفّق الأداء

> **Trigger:** `بطيء` / `performance` / `أداء` / `laggy`. المحور: `MASTER_WORKFLOW.md`.
> **مبدأ:** قِس أولًا، لا تُخمّن. أصلح العنق الحقيقي لا المفترض.

```
SYMPTOM (بطء/lag/استهلاك)
 ↓
Measure / Locate (أين فعلًا؟)
 ↓
Impact + Root Cause
 ↓
Plan (1–2 خيار)
 ↓
Approval        ⛔
 ↓
Implementation
 ↓
Review + Re-measure (تحسّن مُثبت؟)
 ↓
Regression (لا كسر سلوك)
 ↓
Update AI Docs
```

## 1. Symptom + Locate
- شاشة بطيئة؟ استجابة API؟ إقلاع؟ تمرير؟
- اقرأ قسم **Performance** بمستند الميزة (كل مستند فيه واحد).
- عنق شائع موثّق:
  - **N+1** التقييمات/المفضلة/الترجمات (Backend).
  - **الرئيسية** موبايل نداء واحد مجمّع؛ ويب 8+ نداءات.
  - **الوسائط** تحويلات nonQueued تحجب الرفع ([[23_MediaGallery]]).
  - **الترجمة** كشط Google متزامن ([[31_TranslationObservers]]).
  - **الإقلاع** مكالمات شبكة عند السبلاش ([[19_SplashOnboarding]]).
  - Flutter: إعادة بناء زائدة، ListView غير مُحسّن، صور غير مُخزَّنة.

## 2. Impact + Root Cause
- حدّد المصدر بدقّة (ملف:سطر / استعلام / widget). قِس (وقت استجابة/عدد استعلامات/frames).
- الطبقة: Backend (استعلام/فهرس/queue) · Flutter (بناء/render) · Web (SSR/bundle) · DB (index).

## 3. Plan
- خيار أو اثنين: التحسين + الأثر المتوقّع + المخاطرة (لا تكسر السلوك). تعقيد S/M/L.
- أمثلة: `withCount`/`withAvg` بدل accessor، فهرس مركّب، `->queued()` للتحويلات، `const`/`select`/pagination، caching.

## 4. ⛔ Approval — توقّف.

## 5. Implementation — أصغر تغيير يزيل العنق. القاعدة الذهبية لو عقد API يتغيّر.

## 6. Review + Re-measure — **أثبت التحسّن بقياس** (قبل/بعد). لا "أحسّ أسرع".

## 7. Regression — السلوك/المخرجات ثابتة. اختبر الحالات المتأثّرة.

## 8. Update AI Docs
- قسم **Performance** بمستند الميزة (حدّث ما انحلّ). `PROJECT_PROGRESS.md` سجل بالأرقام.
- `DECISIONS/TECH_DEBT.md`: علّم عنق الأداء المُسدَّد. `DATABASE_INDEX.md` لو أُضيف فهرس.

---

## سياسات إلزامية (Governance)
- **مزامنة التوثيق (إلزامي):** أي أثر على قاعدة المعرفة → حدّث `hawdaj-docs` ثم **commit + push** قبل الإتمام — [[Documentation_Sync]].
- **قبل التنفيذ (خطوة 2/Impact + Root Cause):** تحليل أثر كامل — [[Impact_Analysis]]. تأكّد أن التحسين **لا يكسر السلوك** (Breaking = لا) + Testing scope يشمل إعادة القياس.
- **قبل الإتمام:** استوفِ [[Completion_Gate]] — أي بند ناقص (ومنه مزامنة التوثيق + إثبات التحسّن بقياس) = ❌ INCOMPLETE.
