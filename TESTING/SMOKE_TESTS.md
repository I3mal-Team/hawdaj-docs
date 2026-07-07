# SMOKE_TESTS — اختبارات الدخان

> تحقّق يدوي سريع أن النظام "يشتغل" بعد أي تغيير. دقائق، لا دقائق طويلة.

## موبايل — smoke أساسي (بعد أي بناء)
1. [ ] التطبيق يقلع (ForceUpdate → Splash → Home) بلا crash.
2. [ ] الرئيسية تحمّل المحتوى (`/home`).
3. [ ] فتح Place detail + Store detail بلا crash.
4. [ ] login بحساب اختبار → get-profile-page يحمّل.
5. [ ] favorite toggle على عنصر → ينعكس.
6. [ ] تبديل اللغة ar↔en → النصوص + الاتجاه يتغيّران.
7. [ ] بحث/استكشاف يرجّع نتائج.
8. [ ] logout → يرجع لحالة زائر.

## ويب — smoke أساسي
1. [ ] `/home` يرندر (SSR) + الميتا/العنوان صحيح.
2. [ ] `/places` + detail يفتح · ChatGPT widget يحمّل.
3. [ ] login (بريد + social popup) → profile.
4. [ ] تبديل `/ar/` ↔ `/en/` → نصوص + RTL.
5. [ ] 401 (توكن منتهٍ) → logout تلقائي + redirect `/home`.
6. [ ] بحث `/global-search` + خريطة.

## Backend — smoke (بعد تغيير API)
1. [ ] `GET /home` يرجّع envelope صحيح.
2. [ ] endpoint المعدّل: طلب صحيح ينجح · طلب بلا auth (لو خاص) يُرفض · قيمة خاطئة → 422.
3. [ ] الاستجابة لا تحتوي PII غير مقصود (email في ratings؟).

## متى تشغّلها
- بعد أي bug fix / feature / refactor (الخطوة 7 في التدفّق).
- قبل أي دمج/إصدار.
- ركّز على المسارات المتأثّرة بالتغيير + الأساسيات أعلاه.

## قاعدة
smoke ≠ regression. smoke = "ما انكسر شيء واضح". لتغطية أعمق راجع `REGRESSION_CHECKLIST.md` + `FEATURE_TESTS.md`.
