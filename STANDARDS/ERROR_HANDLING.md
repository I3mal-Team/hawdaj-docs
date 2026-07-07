# ERROR_HANDLING — معالجة الأخطاء

> Flutter: `Either<Failure,T>` (dartz). Backend: envelope + HTTP codes. Web: ErrorInterceptor + toasts.

## Flutter
- **النمط:** repo يرجّع `Either<Failure,T>` — **لا يرمي exceptions للأعلى**.
  ```dart
  try {
    final res = await apiConsumer.get(EndPoints.x);
    return Right(Model.fromJson(res['data']));
  } on DioException catch (e) {
    return Left(ServerFailure.fromDioException(e));
  } catch (e) {
    return Left(ServerFailure(e.toString()));
  }
  ```
- **Failure types:** `Failure` (عام) · `ServerFailure` (`core/errors/failure.dart` + `exceptions.dart`). `ServerFailure.fromDioException` يطابق DioExceptionType + status (400/401/403/404/422/500) لرسائل.
- **Cubit:** `res.fold((f)=>emit(Error(f.message)), (d)=>emit(Loaded(d)))`.
- **View:** يعرض حالة Error (رسالة/إعادة محاولة). القوائم: `pagingController.error`.
- **⚠️ نقاط ضعف حالية:** envelope يفترض `message` (خطر crash عند غيابه) · 401 مُتجاهَل (دَين D12). عند لمس الشبكة، عالِجها.

## Backend
- validation → 422 (تلقائي). غير موجود → 404 (`findOrFail`). نجاح/فشل عبر envelope `message`.
- **ملكية:** تحقّق `user_id == auth()->id` قبل تعديل/حذف (تعلّم من نمط Story delete ✅ vs ثغرات أخرى).

## Web (Angular)
- `ErrorInterceptor`: **401 → مسح localStorage + logout + redirect `/home`**. باقي الأكواد عبر PrimeNG `MessageService`/`AlertsService.openToast`.

## قواعد
- لا تبتلع الأخطاء صامتًا — سجّل أو اعرض.
- رسائل مستخدم مترجمة (`.tr()`)، لا نصوص خام.
- لا تسرّب تفاصيل داخلية (stack/PII) للمستخدم أو السجلات (⚠️ logger يسرّب توكنات — دَين D11).
- الأخطاء المتوقّعة (شبكة/validation) تُعالَج بلطف؛ غير المتوقّعة تُسجّل.
