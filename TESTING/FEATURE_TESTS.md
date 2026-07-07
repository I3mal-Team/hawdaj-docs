# FEATURE_TESTS — اختبارات الميزات

> إرشاد كتابة اختبارات لكل ميزة عند التعديل. الهدف: كل bug fix/feature يترك اختبارًا يمنع العودة.

## أنواع (Flutter)
- **Unit:** منطق Cubit + Repository (mock الـApiConsumer). الأهم — يغطّي الحالة/التحويل/`Either`.
- **Widget:** الشاشات الحرجة (تعرض Loading/Loaded/Error صحيح).
- **Integration:** تدفّق كامل عبر شاشات (اختياري للمسارات الحرجة).

## نمط Cubit test (الأساس)
```dart
// mock repo يرجّع Right/Left
blocTest<XCubit, XState>(
  'emits [Loading, Loaded] on success',
  build: () => XCubit(repo: mockRepo..stubRight(data)),
  act: (c) => c.getX(),
  expect: () => [isA<XLoading>(), isA<XLoaded>()],
);
// وحالة الفشل → [Loading, Error]
```

## نمط Repository test
- mock `ApiConsumer` → تأكّد أن النجاح يرجّع `Right(Model)` والخطأ `Left(Failure)`.
- تأكّد أن الـendpoint الصحيح يُستدعى بالـparams الصحيحة.

## ماذا تختبر لكل ميزة (من مستندها)
- **Full Execution Flow:** الحالة السعيدة.
- **Edge Cases** (قسم 21): كل حالة حافّة موثّقة → اختبار.
- **Common Bugs** (قسم 35): كل bug معروف → اختبار يمنع عودته.
- **Regression Checklist** (قسم 34): حوّلها لـassertions.

## Backend (لو مُتاح)
- Feature test لكل endpoint: auth مطلوب · validation (قيم خارج المدى تُرفض) · الاستجابة لا تسرّب PII · الملكية على التعديل/الحذف.

## قاعدة
- **كل bug fix يرافقه اختبار** يعيد إنتاج العطل ثم يثبت الإصلاح.
- **كل feature جديدة:** unit للـCubit + repo كحدّ أدنى.
- سمِّ الاختبار بالسلوك: `emits Error when rate out of range`.

## الاختبارات الموجودة فعليًا (سجل)
- **البنية التحتية:** `dev_dependencies` أُضيف `bloc_test: ^10.0.0` + `mocktail: ^1.0.4` (2026-07-07) — أول تجهيز اختبار في المشروع. مجلد `test/` يعكس بنية `lib/features/`.
- **[[01_Authentication]] — الدخول الاجتماعي (أول suite):**
  - `test/features/auth/social_auth_cubit_test.dart` (9 blocTest) — `SocialAuthCubit` بـ`MockAuthRepository`/`MockSocialAuthService`، حالات نجاح/فشل Google/Apple + تخزين التوكن.
  - `test/features/auth/social_auth_service_test.dart` (7 test) — `SocialAuthService`: استخراج claim `sub` من JWT، بناء طلبات Google/Apple، حالات التوكن المشوّه.

> الوضع الحالي: تغطية اختبارات لسه ضعيفة (بدأت بالدخول الاجتماعي فقط). ابنِ التغطية تدريجيًا مع كل تعديل — لا تنتظر حملة شاملة. أولوية تالية مقترحة: `PrepareTripWizardCubit` (منطق daterange + المسودّة) و`ReprepareTripCubit` بعد تغييرات [[03_Trips]].
