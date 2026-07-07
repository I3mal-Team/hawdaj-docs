# ARCHITECTURE — المعمارية

> الأنماط الفعلية المعتمَدة (من تحليل 32 ميزة). اتبع الموجود، لا تخترع.

## Flutter (Mobile)
- **Feature-First + Clean رفيعة.** كل ميزة: `lib/features/<feature>/{data,presentation}`.
  - `data/`: `models/` · `repositories/` (abstract + impl) · أحيانًا `datasources/`.
  - `presentation/`: `manager/` (Cubits + states) · `view/` (شاشات) · `view/widgets/`.
- **طبقة domain كاملة غير مستخدمة** (مبسّطة عمدًا) — لا تضِف UseCases إلا لو الفريق قرّر.
- **التدفّق القياسي:** `View → Cubit → Repository → DioConsumer → API → (Laravel) → Response → Either → Cubit → View`.
- **DI:** `get_it` (`core/services/service_locator.dart`) — `registerLazySingleton` للـrepos، `registerFactory` للـcubits عند الحاجة.
- **التنقّل:** `go_router` (مسارات مركزية + `RoutesKeys`).
- **الأخطاء:** `Either<Failure,T>` (dartz) في كل repo. راجع `ERROR_HANDLING.md`.
- **الشبكة:** `DioConsumer` (implements `ApiConsumer`) + interceptors. Endpoints في `core/databases/api/end_points.dart`. راجع `API_RULES.md`.
- **التعريب:** `easy_localization` + `.tr()` + `EdgeInsetsDirectional`. 4 لغات (ar/en/ru/zh) + RTL.

## Backend (Laravel)
- **envelope موحّد** عبر `ApiModalController`: `{code/status, message, data}`.
- **حارسان:** `auth('api')` (Sanctum bearer) للعملاء · `auth:web` + Spatie للوحة.
- **المحتوى:** Translatable (Astrotomic `{model}_translations`) · Media (Spatie small250/medium500) · soft deletes.
- **Resources** لتشكيل الاستجابة (List/Detail/Collection). Observers للترجمة التلقائية.
- **Modules** (nwidart) للوحدات المعزولة (CarModel/Report — خارج النطاق).

## Web (Angular 16 SSR)
- خدمات RxJS singleton (BehaviorSubjects) — **لا NgRx**. Reactive Forms. HttpClient + interceptors. SSR + SEO. `/:lang/` prefix. domains/ للميزات الأحدث (trip v2, profile-settings v2).

## القاعدة الذهبية
تغيير يمسّ **عقد API** → Backend أولًا → ثم الموبايل والويب على العقد → ثم التوثيق. (سبب: عميلان يتشاركان API واحد.)

## قرارات مرجعية
راجع `DECISIONS/ARCHITECTURE_DECISIONS.md` (A1–A12).
