# API_RULES — قواعد الـAPI والشبكة

> عقد موحّد بين Backend والعميلين (موبايل + ويب).

## العقد (Envelope)
- كل استجابة: `{code/status, message, data}` (`ApiModalController`).
- النجاح: اقرأ `data`. الخطأ: اقرأ `message`. (⚠️ العميل يفترض وجود `message` — تحقّق من عدم كسره.)

## Base URLs
- prod `https://dashboard.hawdaj.net/api/` · test `https://test.dashboard.hawdaj.net/api/`.
- **موبايل:** يُبدّل عبر `AppEnvironmentManager` (pref `environment`). ⚠️ backdoor `dev123` بالإنتاج (دَين D10).
- **ويب:** environment files وقت البناء.

## الهيدرز (كلا العميلين)
`Authorization: Bearer <token>` · `locale` · `accept-language` · static `accept-tokenapi` (⚠️ مكتوب بالمصدر، دَين D9) · geolocation (lat/lng).

## Flutter
- **كل endpoint** في `core/databases/api/end_points.dart` (getter camelCase). لا تكتب مسارًا نصيًا في repo.
- **كل نداء** عبر `DioConsumer` (get/post/put/delete). Multipart عبر `FormDataHelper`.
- repo يرجّع `Either<Failure,T>` (لا يرمي). راجع `ERROR_HANDLING.md`.
- **401:** حاليًا **مُتجاهَل** (دَين D12) — الويب يعمل logout. عند الإصلاح: refresh أو logout.

## Backend — عند إضافة/تغيير endpoint
1. **auth:** أضِف `auth('api')` للمسارات الخاصّة (تعلّم من دَين D2/D6 — لا تترك عامًّا).
2. **validation:** FormRequest أو inline، بمدى/أنواع صريحة (تعلّم من دَين D4).
3. **Resource:** لا تُرجّع الموديل خامًا (تسريب PII — دَين D3). استخدم Resource نحيف.
4. **الترقيم:** `paginate()` — لا تُرجّع مجموعات غير محدودة.
5. **الفهرسة/الأداء:** تجنّب N+1 (`withCount`/`with`) — راجع `WORKFLOWS/PERFORMANCE.md`.
6. **الفلترة:** `active=1` للمحتوى العام (تعلّم من دَين D26).

## القاعدة الذهبية
تغيير العقد يبدأ Backend → ثم `end_points.dart` (موبايل) + service (ويب) → ثم `API_INDEX.md` + قسم "Web Frontend" + `WEB_MOBILE_COMPARISON.md`.

## اختلاف موبايل/ويب موثّق
- الرئيسية: موبايل `/home` واحد · ويب 8+ نداءات.
- البحث: موبايل SearchController · ويب `/global-search?viewAs=`.
- (المرجع الكامل: `WEB_MOBILE_COMPARISON.md` §API Differences.)
