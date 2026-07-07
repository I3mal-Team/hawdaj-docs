# NAMING — التسمية

> اتبع الأسماء الفعلية في الكود.

## Flutter / Dart
- **ملفات:** `snake_case.dart` — `place_model.dart`, `get_landmarks_cubit.dart`, `land_mark_view.dart`.
- **أصناف:** `PascalCase` — `PlacesCubit`, `LandmarkModel`, `LandMarkView`, `TasneefRepositoryImpl`.
- **Cubit + State:** `<Name>Cubit` + `<Name>State` (+ حالات: `<Name>Initial/Loading/Loaded/Success/Error`).
- **Repository:** abstract `<Feature>Repository` / `<Feature>RepositoryRepo` · impl `<Feature>RepositoryImpl` (أو `..._repo_imp.dart` — النمط موجود، حافظ على المتّبع في نفس الميزة).
- **Model:** `<Feature>Model` (استجابة) · `Create<Feature>Request` / `<Feature>Response` عند الفصل.
- **متغيّرات/دوال:** `camelCase`. ثوابت: `camelCase` أو `kPrefixed` للمسارات (`kAddLandmarkView`).
- **مفاتيح التنقّل:** `RoutesKeys.k<Name>View`.
- **Endpoints:** camelCase getters في `end_points.dart` — `landmarkList(page)`, `addFavorites`, `getChatGpt`.
- **مفاتيح i18n:** snake_case وصفية — `filter_places`, `terms_intro_text`. **لا نصوص مكتوبة يدويًا** (راجع دَين D39).

## Backend / Laravel
- **Models:** `PascalCase` مفرد — `Place`, `ZadElgadel`, `LandMark`. (لاحظ عدم الاتساق التاريخي: `LandMark` vs `Landmark`.)
- **Controllers:** `<Name>Controller`. **Resources:** `<Name>Resource`/`<Name>ListResource`/`<Name>Collection`.
- **Migrations:** `YYYY_MM_DD_HHMMSS_verb_noun`. **Tables:** snake_case جمع — `land_marks`, `zad_elgadels`, `{model}_translations`.
- **الحقول polymorphic type:** ⚠️ غير متسق تاريخيًا (`zad`/`Zad_elgadel`) — عند الإضافة الجديدة وحّد ووثّق.

## Web / Angular
- **مكوّنات:** `<name>.component.ts` + `PascalCaseComponent`. **خدمات:** `<name>.service.ts`. **Routes:** `<feature>-routes.ts`.

## قاعدة عامة
عند الغموض، **قلّد أقرب ملف مماثل في نفس الميزة** — لا تُدخل نمط تسمية جديد.
