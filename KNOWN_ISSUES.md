# KNOWN_ISSUES — الأعطال والحوادث الحيّة

> سجل الأعطال **التشغيلية/البيئية** المكتشَفة (production/config/infra). مختلف عن `DECISIONS/TECH_DEBT.md` (دَين معماري/كود مقصود).
> الحالة: ⬜ مفتوح · 🔧 قيد المعالجة · ✅ محلول. حدّث البند عند التغيّر (التاريخ + من عالجه). يُحدَّث في الخطوة 11 من أي تدفّق.
> الصيغة: العنوان · الأعراض · السبب الجذري (بدليل) · الأثر · حلّ مؤقّت · حلّ جذري · الحالة · الميزة.

---

## K1 — فشل SMTP بالإنتاج (بورت 587 timeout) 🔴 ⬜
- **الأعراض:** `POST trips/save-trip-to-email` يرجّع **500** بعد ~31 ثانية.
- **السبب الجذري (دليل حرفي من رد السيرفر):** `Unable to connect to 173.194.203.27:587 (Connection timed out)` · `Swift_TransportException`. `173.194.203.27:587` = Gmail SMTP. سيرفر الإنتاج لا يقدر يفتح اتصالًا صادرًا على بورت 587. الإرسال **متزامن** داخل الطلب (`app/Http/Controllers/Api/TripController.php:85` — `Mail::send`).
- **الأثر:** كل بريد يعتمد SMTP يفشل — حفظ الرحلة بالبريد + بريد إنشاء الأدمن (K4). الطلب يعلّق 31s قبل ما يفشل.
- **حلّ مؤقّت:** لا يوجد للمستخدم النهائي (ميزة البريد معطّلة فعليًا).
- **حلّ جذري:** فتح البورت الصادر 587/465 على firewall السيرفر، **أو** التحوّل لمزوّد بريد عبر API على 443 (SendGrid/Mailgun/SES). + جعل الإرسال **queued** بدل متزامن + `try/catch`. تحقّق `MAIL_*` في `.env`.
- **الميزة:** [[03_Trips]] · يطال [[21_ProfileSettings]] [[26_DashboardAdmin]]. مرتبط بدَين [[TECH_DEBT#D45]].

## K2 — `APP_DEBUG=true` في الإنتاج 🔴 ⬜
- **الأعراض:** استجابات الأخطاء (500) تُرجع **stack trace كامل + مسارات السيرفر المطلقة + Debugbar** (ظهر في trace رد K1).
- **السبب الجذري:** `APP_DEBUG=true` في `.env` بالإنتاج.
- **الأثر:** تسريب معلومات (بنية المشروع، مسارات vendor، إعدادات) — سطح استطلاع للمهاجم.
- **حلّ جذري:** `APP_DEBUG=false` في الإنتاج + `php artisan config:cache`.
- **الميزة:** أمني عام. مرتبط [[TECH_DEBT#D44]].

## K3 — كاش صلاحيات Spatie لا يُفرَّغ 🟡 ⬜
- **الأعراض:** إعطاء مستخدم دورًا/صلاحية من اللوحة **لا يسري** — يدخل ولا يشوف الأقسام.
- **السبب الجذري (دليل):** `config/permission.php` كاش مدّته **24 ساعة** (`expiration_time => 24 hours`)، و**لا يوجد أي `forgetCachedPermissions()` / `permission:cache-reset` في الكود** (grep فارغ).
- **الأثر:** أي تعديل أدوار/صلاحيات لا يظهر إلا بعد 24 ساعة أو مسح يدوي → التباس تشغيلي.
- **حلّ مؤقّت:** `php artisan permission:cache-reset` (أو `cache:clear`) بعد أي تعديل.
- **حلّ جذري:** استدعاء `app(PermissionRegistrar::class)->forgetCachedPermissions()` بعد التعديل في `RolesController`/`PermissionsController`/`UsersController`.
- **الميزة:** [[27_RolesPermissions]] [[26_DashboardAdmin]]. مرتبط [[TECH_DEBT#D46]].

## K4 — بريد إنشاء المستخدم queued + catch صامت 🟡 ⬜
- **الأعراض:** إنشاء مستخدم (مثل مشرف) من اللوحة → **لا تصله كلمة المرور بالبريد**، ولا يظهر خطأ للأدمن.
- **السبب الجذري (دليل):** `App\Notifications\dashboard\userMailNotification` **`implements ShouldQueue`** → يحتاج `queue:work` شغّال. والإرسال ملفوف بـ`try { ... } catch (\Exception $e) {}` **صامت** في `app/Http/Controllers/Dashboard/Users/UsersController.php@store` (~سطر 95–98). + احتمال SMTP (K1).
- **الأثر:** المشرف لا يدخل (لم تصله كلمة المرور)؛ الفشل غير مرئي.
- **حلّ مؤقّت:** الأدمن يحدّد كلمة المرور وقت الإنشاء ويعطيها المشرف مباشرة (لا انتظار بريد).
- **حلّ جذري:** تشغيل/إدارة `queue:work` دائمًا (Supervisor/systemd) + إصلاح SMTP (K1) + تسجيل الفشل بدل ابتلاعه + عرض تنبيه للأدمن. (تذكرة: `hawdaj-api/TICKET_supervisor_login_email.md`)
- **الميزة:** [[26_DashboardAdmin]] [[21_ProfileSettings]] [[27_RolesPermissions]].

---

## كيفية الاستخدام
- عند تشخيص عطل تشغيلي: سجّله هنا فورًا (K رقم تالٍ) بنفس الصيغة، واربطه بـ TECH_DEBT/الميزة.
- عند الحل: علّم ✅ + التاريخ + الحلّ المطبَّق، وسجّل في `CHANGE_LOG.md`.
- الفرق عن TECH_DEBT: هنا **حوادث حيّة/بيئية** (SMTP، APP_DEBUG، كاش، queue)؛ هناك **دَين كود/معماري** (N+1، mass-assignment، auth ناقص…).
