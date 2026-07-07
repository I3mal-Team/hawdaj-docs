# FOLDER_STRUCTURE — بنية المجلدات

> النمط الفعلي في `lib/`. أي feature جديدة تتبعه حرفيًا.

## Flutter — feature
```
lib/features/<feature>/
├── data/
│   ├── models/                 <feature>_model.dart · request/response models
│   └── repositories/
│       ├── <feature>_repository_repo.dart   (abstract)
│       └── <feature>_repository_impl.dart   (impl — يرجّع Either<Failure,T>)
└── presentation/
    ├── manager/
    │   └── <name>_cubit/
    │       ├── <name>_cubit.dart
    │       └── <name>_state.dart
    └── view/
        ├── <feature>_view.dart
        └── widgets/
            └── <widget>.dart
```

## Flutter — core (مشترك)
```
lib/core/
├── databases/api/    end_points.dart · dio_consumer.dart · api_consumer(+extension) · interceptors
├── services/         service_locator.dart (get_it) · storage_service · auth_manager
├── routing/          routes.dart (go_router) · RoutesKeys
├── managers/         cubits مشتركة (regions/cities/connectivity…)
├── errors/           failure.dart · exceptions.dart
├── utils/ · components/ · views/   (widgets/أدوات مشتركة)
└── locale/           locale_cubit.dart
assets/translations/  ar.json · en.json · ru.json · zh.json
```

## قواعد
- ميزة = مجلد واحد تحت `features/`. لا تنثر ملفاتها في core.
- المشترك بين ميزتين+ → `core/`.
- Cubit + state في نفس مجلد `manager/<name>_cubit/`.
- Model لكل شكل بيانات (request/response منفصلان عند الاختلاف).
- الأسماء snake_case للملفات. راجع `NAMING.md`.

## Backend (مرجع سريع)
`app/Http/Controllers/Api/*` · `Controllers/Dashboard/*` · `Models/*` · `Http/Resources/<Feature>/*` · `Observers/*` · `database/migrations/*` · `Modules/<Name>/*`.

## Web (مرجع سريع)
`src/app/components/<feature>/` (v1) · `src/app/domains/<feature>/` (v2) · `services/` · `services/interceptors/` · `modules/shared/`.
