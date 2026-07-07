# NEW_FEATURE — تدفّق ميزة جديدة

> **Trigger:** `feature جديدة` / `اعمل feature` / `new feature` (+ وصف قصير).
> يقرأ `CLAUDE.md` ويمشي لوحده حتى الموافقة. المحور: `MASTER_WORKFLOW.md`.

```
FEATURE REQUEST
 ↓
Feature Entry (اقتراح)
 ↓
Impact + Cross-Platform
 ↓
Plan (1–2 خيار)
 ↓
Approval        ⛔
 ↓
Implementation (Backend أولًا لو فيه عقد API)
 ↓
Review
 ↓
Regression
 ↓
Update AI Docs (+ مستند ميزة جديد)
```

## 1. Feature Entry
- اقرأ `FEATURE_INDEX.md` — الميزة موجودة (توسعة) أم جديدة كليًا؟
- اقترح إدخالًا: **الاسم · وصف مختصر · المنصّات (Flutter/Backend/Web/DB) · التبعيات (من `FEATURE_GRAPH.md`) · الأولوية**.
- لو تشبه ميزة قائمة، أعد استخدام نمطها (مثال: أي محتوى جديد يتبع قالب Places [[04_Places]]).

## 2. Impact + Cross-Platform
- **العقد:** هل تحتاج endpoint جديد؟ عمود/جدول DB؟ → Backend أولًا.
- **الاتساق:** هل تُبنى في الموبايل فقط أم الويب كمان؟ حدّد الفجوة صراحة (راجع `WEB_MOBILE_COMPARISON.md`).
- **الطبقات:** Flutter (feature folder + Cubit + repo + go_router) · Backend (controller/model/migration/resource) · Web · DB (schema/index/translatable?).
- تعريب (4 لغات + observers [[31_TranslationObservers]]) · وسائط ([[23_MediaGallery]]) · صلاحيات.

## 3. Plan
- خيار أو اثنين: خطوات · ملفات لكل طبقة · تعقيد (S/M/L) · مخاطر + تخفيف.
- اتبع البنية القائمة (`STANDARDS/FOLDER_STRUCTURE.md` + `ARCHITECTURE.md`). لا تخترع نمطًا جديدًا.

## 4. ⛔ Approval — توقّف. لا كود قبل موافقة صريحة.

## 5. Implementation
- **Backend أولًا** (migration → model → controller → resource → route) لو فيه عقد.
- ثم Flutter: `lib/features/<f>/{data,presentation}` · Cubit · repo (`Either<Failure,T>`) · endpoint في `end_points.dart` · شاشة + route.
- ثم Web لو مطلوب التماثل.
- كل النصوص عبر `.tr()` + مفاتيح في ملفات i18n الأربعة.

## 6. Review — معايير · تسمية · حالة · أخطاء · تعريب/RTL · لا تكرار · لا كسر تبعيات.

## 7. Regression — من `TESTING/` + قسم Regression بمستند الميزة الجديد.

## 8. Update AI Docs (إلزامي)
- **أنشئ `FEATURES/NN_NewFeature.md`** (38 قسمًا، اتبع قالب المستندات القائمة).
- حدّث `FEATURE_INDEX.md` (صف + عمود Web) · `PROJECT_PROGRESS.md` · `FEATURE_GRAPH.md` (تبعيات) · `API_INDEX.md`/`DATABASE_INDEX.md`/`SEARCH_INDEX.md` حسب المضاف.
- قرار معماري مهم → `DECISIONS/ARCHITECTURE_DECISIONS.md`.
