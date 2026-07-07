# STATE_MANAGEMENT — إدارة الحالة

> Flutter: **Cubit** (flutter_bloc). Web: خدمات RxJS. لا تخلط.

## Flutter — Cubit
- كل ميزة: Cubit واحد أو أكثر تحت `presentation/manager/<name>_cubit/`.
- **الحالات القياسية:** `Initial` · `Loading` · `Loaded`/`Success` · `Error`. (بعض الميزات تضيف `Ready` لتحقّق النماذج.)
- Cubit **لا يعرف الشبكة** — يستدعي Repository ويحوّل `Either` لحالة:
  ```dart
  emit(Loading());
  final res = await repo.getX();
  res.fold((f) => emit(Error(f.message)), (data) => emit(Loaded(data)));
  ```
- **DI:** الـCubit يُحقن repo عبر get_it. سجّل Cubit كـ`registerFactory` لو له حالة لكل شاشة، repo كـ`registerLazySingleton`.
- **التزويد:** `BlocProvider` عند الـroute (في `routes.dart`).
- **الترقيم:** `PagingController` (infinite_scroll_pagination) — النمط الموحّد للقوائم.
- **لا منطق شبكة/تحويل في الـView.** الـView يستمع (`BlocBuilder`/`BlocListener`) ويعرض.

## قواعد
- حالة واحدة مصدر الحقيقة للشاشة. لا `setState` لمنطق أعمال.
- بعد إجراء (toggle/submit): إمّا تحديث محلي للحالة أو إعادة جلب — كن متّسقًا داخل الميزة. (ملاحظة: favorites/rates حاليًا يعيدون الجلب كاملًا — دَين D15.)
- `hydrated_bloc` مُعلَن لكن **غير مستخدم** — لا تعتمد عليه.

## Web — RxJS
- خدمات singleton + `BehaviorSubject` للحالة المشتركة. **لا NgRx.** Reactive Forms للنماذج.

## مضادّات نمط (تجنّبها)
- منطق أعمال في widgets.
- استدعاء Dio مباشرة من الـView/Cubit (مرّ عبر repo).
- حالة مكرّرة بين cubits (استخرجها لـ`core/managers/`).
