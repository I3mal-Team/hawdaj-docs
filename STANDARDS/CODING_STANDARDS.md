# CODING_STANDARDS — معايير الكود

> قواعد كتابة عامّة عبر الطبقات. المرجع البنيوي: `ARCHITECTURE.md` · `FOLDER_STRUCTURE.md` · `NAMING.md`.

## عام
- تغييرات **صغيرة ومركّزة**. لا refactor مع bug fix. لا scope creep.
- الكود يقرأ مثل جيرانه — طابق كثافة التعليقات والأسلوب والـidioms المحيطة.
- تعليقات فقط عند منطق **غير بديهي** — لا تعليقات تشرح الواضح.
- لا كود ميّت / معلّق. لو مؤقّت، سجّله في `DECISIONS/TECH_DEBT.md`.
- لا أسرار في الكود (مفاتيح/توكنات/كلمات مرور) — راجع دَين D8/D9.

## Flutter / Dart
- `const` حيثما أمكن (أداء البناء). `final` بدل `var` عند الثبات.
- widgets صغيرة مركّزة — استخرج الـwidget عند التضخّم (نمط `view/widgets/`).
- كل النصوص عبر `.tr()`. تباعد اتجاهي `EdgeInsetsDirectional` (لا `EdgeInsets` للـhorizontal). لا نصوص عربية مكتوبة (دَين D39).
- Cubit للأعمال، لا `setState`. راجع `STATE_MANAGEMENT.md`.
- repo يرجّع `Either`. راجع `ERROR_HANDLING.md`.
- endpoint في `end_points.dart` فقط. راجع `API_RULES.md`.
- تجنّب إعادة البناء الزائدة (BlocBuilder مُحدّد النطاق، `const` children).

## Backend / Laravel
- Resource لتشكيل الاستجابة (لا موديل خام — تسريب PII، دَين D3).
- validation صريحة (FormRequest مُفضّل). `active=1` للمحتوى العام. auth على المسارات الخاصّة.
- تجنّب N+1 (`with`/`withCount`). فهارس على أعمدة الفلترة.
- `$fillable` صريح لا `$guarded=[]` (mass-assignment، دَين D23).

## Web / Angular
- Reactive Forms + validators. خدمات RxJS. Standalone components + lazy `loadComponent`. `isPlatformBrowser` للأمان مع SSR.

## قبل الإنهاء (self-review)
معايير ✓ · تسمية ✓ · حالة ✓ · أخطاء ✓ · تعريب/RTL ✓ · لا تكرار/ميّت ✓ · لا كسر تبعيات (`FEATURE_GRAPH.md`) ✓.
