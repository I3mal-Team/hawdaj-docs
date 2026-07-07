# SECURITY — تدفّق الأمان

> **Trigger:** `security` / `أمان` / `ثغرة` / `vulnerability`. المحور: `MASTER_WORKFLOW.md`.
> **مبدأ:** اكتب **عادي** (بلا اختصار) في كل خطوات الأمان. أكّد قبل أي إجراء صعب التراجع. لا تكشف تفاصيل استغلال قابلة للأذى.

```
CONCERN / REPORT
 ↓
Confirm (ثغرة حقيقية؟ أين؟)
 ↓
Severity + Blast Radius
 ↓
Plan (إصلاح + تحقّق)
 ↓
Approval        ⛔
 ↓
Implementation (Backend أولًا غالبًا)
 ↓
Verify (الثغرة أُغلقت فعلًا؟)
 ↓
Regression
 ↓
Update AI Docs (TECH_DEBT ✓ مُغلق)
```

## 1. Confirm
- اقرأ قسم **Security** بمستند الميزة + `DECISIONS/TECH_DEBT.md` (كثير موثّق مسبقًا).
- أكّد أنها ثغرة حقيقية لا إنذار كاذب. حدّد الملف:السطر + ناقل الهجوم.

## 2. Severity + Blast Radius
- صنّف: 🔴 Critical / High · 🟡 Medium · 🟢 Low.
- من يتأثّر؟ عقد API مشترك → الموبايل + الويب معًا.
- **قائمة الدَّين الأمني الحرج المعروف (المرجع الكامل `DECISIONS/TECH_DEBT.md`):**
  - ⚠️ `monarx-analyzer.php` RCE — حذف فوري ([[26_DashboardAdmin]]).
  - `rates` بلا auth + تسريب email + بلا مدى 1–5 ([[14_RatesReviews]]).
  - API بلا authz دقيق · كل الأدوار=كل الصلاحيات · creds `123456` ([[27_RolesPermissions]]).
  - CORS `*` + مفتاح `accept-tokenapi` بالمصدر ([[24_Networking]]).
  - Mobile: backdoor `dev123` · logger يسرّب توكنات · 401 مُتجاهَل · fcm_token لا يُمسح ([[24_Networking]], [[21_ProfileSettings]]).
  - SVG upload (XSS) ([[23_MediaGallery]], [[16_UserStories]]).

## 3. Plan
- إصلاح جذري + **طريقة تحقّق** (كيف نثبت الإغلاق). تعقيد + مخاطر التوافق الرجعي.
- الأمان أولوية على الراحة — لكن حافظ على التوافق الرجعي حيث أمكن (auth مضاف قد يكسر عملاء قدامى — نسّق).

## 4. ⛔ Approval — توقّف. الإجراءات صعبة التراجع (حذف ملف، تدوير مفتاح، تغيير auth) تحتاج تأكيدًا صريحًا.

## 5. Implementation — Backend أولًا غالبًا. أضف auth/validation/throttle/سياسة. دوّر الأسرار المكشوفة.

## 6. Verify — أثبت أن الثغرة أُغلقت (طلب بلا توكن يُرفض، مدى خارج 1–5 يُرفض، إلخ). لا تفترض.

## 7. Regression — المسار الشرعي ما انكسر. اختبر الإيجابي + السلبي.

## 8. Update AI Docs
- قسم **Security** بمستند الميزة (حدّث الحالة). `DECISIONS/TECH_DEBT.md`: علّم البند **✓ مُغلق** + التاريخ.
- `PROJECT_PROGRESS.md` سجل. لو تغيّر عقد → `API_INDEX.md` + قسم Web + `WEB_MOBILE_COMPARISON.md`.

> بديل جاهز: `/security-review` (أداة الهارنس) لمراجعة أمنية للفرع.

---

## سياسات إلزامية (Governance)
- **قبل التنفيذ (خطوة 2/Severity):** تحليل أثر كامل — [[Impact_Analysis]]. الأمان غالبًا Risk = 🟠/🔴 → اكتب عادي (بلا اختصار)، حدّد Breaking (auth مضاف قد يكسر عملاء) + Rollback + Migration.
- **قبل الإتمام:** استوفِ [[Completion_Gate]] — أي بند ناقص (ومنه مزامنة التوثيق + إثبات إغلاق الثغرة + KNOWN_ISSUES/TECH_DEBT ✓) = ❌ INCOMPLETE.
