# Feature 01 — Authentication & Profile

> **Status:** ✅ Analyzed (100%) · **Priority:** P0 (Critical) · **Category:** Core
> **Repos:** Flutter `/Users/mac/hawdaj/Untitled` · Laravel `/Users/mac/hawdaj-api`
> **Last updated:** 2026-07-05

---

## 1. Feature Name
Authentication & Profile — تسجيل الدخول/الإنشاء، الدخول الاجتماعي، استعادة كلمة المرور، وإدارة الملف الشخصي (تحديث البيانات/الصورة/كلمة المرور، حذف الحساب، FCM token، الموقع).

## 2. Business Goal
تمكين المستخدم من امتلاك حساب موحّد للوصول للميزات المحمية (الرحلات، المفضّلة، المحتوى المُقدَّم من المستخدم، النقاط)، مع دعم دخول سريع عبر Google/Apple، وإدارة كاملة لملفه الشخصي.

## 3. Business Rules
- البريد فريد عند التسجيل (`unique:users,email`).
- كلمة المرور ≥ 6 (ضعيفة — راجع §29).
- الدخول الاجتماعي يُنشئ الحساب تلقائيًا إن لم يكن موجودًا، ويُعيد إحياء الحساب المحذوف soft-deleted تلقائيًا.
- التوكن (Sanctum) لا ينتهي (`sanctum.expiration = null`)، ولا يوجد refresh token.
- تسجيل الخروج/حذف الحساب يمسح التخزين المحلي **دائمًا** حتى لو فشل نداء الـ API.
- حذف الحساب = soft delete + حذف البيانات المرتبطة (UserSocial, Guide, Story, Trip).
- كود استعادة كلمة المرور: `rand(1111,9999)` في الإنتاج، ثابت `1234` في التطوير، ويُخزَّن نصًّا صريحًا.
- تحديث الموقع محكوم بعتبة مسافة 30م أو زمن 5 دقائق (يعمل للضيوف عبر cache باستخدام IP hash).

## 4. User Flow
1. Splash → إن لم يكن مسجّلًا → LoginView.
2. Login (email/password) أو زر Google/Apple → Home عند النجاح.
3. Register (firstName, lastName, email, password) → Home.
4. Forgot Password (email) → إرسال كود → (OtpVerificationView — **غير مكتمل**، راجع §24) → إعادة تعيين.
5. من Profile: تعديل البيانات/الصورة، تغيير كلمة المرور، حذف الحساب، تسجيل الخروج.

## 5. Flutter Screens
| Screen | File | ملاحظات |
|---|---|---|
| LoginView | `lib/features/auth/presentation/views/login_view.dart` | email + password + "Remember Me" + أزرار Google/Apple |
| RegisterView | `lib/features/auth/presentation/views/register_view.dart` | firstName/lastName/email/password + أزرار اجتماعية |
| OtpVerificationView | `lib/features/auth/presentation/views/otp_verification_view.dart` | `flutter_otp_text_field` — **stub-like، بلا منطق تحقق مكتمل** |
| ForgotPasswordView | `lib/features/auth/presentation/views/forgot_password_view.dart` | email + إرسال كود |
| AuthPagesWrapper | `lib/features/auth/presentation/views/auth_pages_wrapper.dart` | غلاف شاشات المصادقة |
| Profile screens | `lib/features/profile/presentation/.../EditProfileView, UpdatePasswordView, ProfileView` | تعديل الملف/كلمة المرور |

## 6. Widgets
- `CustomTextField` (email/password/name)، `CustomCheckBoxCircle` (Remember Me)، `SocialAuthPlatformWidget` (أزرار اجتماعية)، `FlutterOtpTextField` (OTP)، `Form`+`GlobalKey<FormState>` (Register).
- Gender modal sheet، اختيار صورة (CameraHelper — gallery/camera).

## 7. Cubits / Blocs
| Cubit | File | States |
|---|---|---|
| LoginCubit | `.../manager/cubits/login_cubit/login_cubit.dart` | Initial → Loading → Success(user) / Error(msg) |
| RegisterCubit | `.../manager/cubit/register/register_cubit.dart` | Initial → Loading → Success(authResponse) / Error |
| SocialAuthCubit | `.../manager/cubits/social_auth_cubit/social_auth_cubit.dart` | Initial → Loading → Success(user,msg) / Error — signInWithGoogle/Apple/Twitter |
| LogoutCubit | `.../manager/cubits/logout_cubit/logout_cubit.dart` | Initial → Loading → Success — يمسح التخزين دائمًا |
| ForgotPasswordCubit | `.../manager/cubits/forgot_password_cubit/forgot_password_cubit.dart` | Initial → Loading → Success(msg) / Error |
| VerifyEmailCodeCubit | `.../manager/cubit/verify_email_code_cubit.dart` | Initial فقط — **STUB، بلا منطق** |
| DeleteProfileCubit | `.../manager/cubits/delete_profile_cubit/delete_profile_cubit.dart` | Initial → Loading → Success / Error |
| EditProfileCubit | `lib/features/profile/.../edit_profile_cubit.dart` | حالة واحدة غنية (copyWith) — بيانات/صورة/روابط اجتماعية |
| UpdatePasswordCubit | `lib/features/profile/.../update_password_cubit.dart` | Initial → Loading → Success(user) / Error — تحقق محلي min:8 |

## 8. State Flow
- Cubit-based (flutter_bloc) + Equatable (لا freezed). النمط الموحّد: `emit(Loading)` → استدعاء repo → `emit(Success/Error)` مع `try/catch`.
- **UserInfoCubit** (core) هو مصدر الحقيقة لحالة المستخدم؛ عند النجاح `userInfoCubit.setUser(user, token)` → يحفظ عبر `AuthManager` ثم يُصدر الحالة.
- `AuthManager.authStream` (Broadcast<bool>) متاح للاستماع لتغيّر حالة الدخول (معرّف لكنه غير مستهلك فعليًا في التوجيه).

## 9. Models
**Flutter:** `UserModel` (18 حقل)، `AuthResponseModel`+`AuthDataModel`، `SocialAuthRequestModel`، `GoogleAuthModel`، `AppleAuthModel`، `TwitterAuthModel` (غير مُنفّذ)، `LogoutResponseModel`، `UpdateProfileResponseModel`+`UpdateProfileDataModel`+`SocialDataModel`.
**Laravel:** `User` (guarded=['id'], SoftDeletes, HasApiTokens, HasRoles, InteractsWithMedia)، `UserSocial`، `UserLocation`.

## 10. Repositories
- **Contract:** `lib/features/auth/domain/repositories/auth_repository.dart` — register, login, logout, deleteProfile, forgotPassword, updateProfile, updatePhoto, googleLogin, appleLogin, twitterLogin.
- **Impl:** `lib/features/auth/data/repositories/auth_repository_impl.dart` — كل دالة تبني `FormData`/JSON وتنادي `apiConsumer` وتحوّل الرد لموديل. (⚠️ يحتوي `print()` كثيرة تسرّب التوكن/الرد — راجع §29).
- **لا يوجد Repository pattern في Laravel** — استعلامات Eloquent مباشرة.

## 11. Services
- **Flutter:** `SocialAuthService` (Singleton) — Google (`GoogleSignIn`)، Apple (`SignInWithApple` + `FirebaseAuth`)، Twitter (`UnimplementedError`). يستخرج `providerId` من JWT (`sub`) مع بدائل (googleUserId → hash email).
- **Laravel:** لا خدمة auth مخصّصة؛ المنطق داخل الـ Controllers مباشرة. `laravel/socialite` مُثبّت لكن الدخول الاجتماعي يعتمد على بيانات العميل لا على Socialite server-side.

## 12. API Endpoints
| Op | Method | Endpoint | Auth |
|---|---|---|---|
| Register | POST | `register` | ❌ |
| Login | POST | `login` | ❌ |
| Logout | POST | `logout` | ✅ |
| Social | POST | `social/callback` | ❌ |
| Forgot | POST | `forgot-password` | ❌ |
| Verify code | POST | `verify-email-code` | ❌ |
| Reset | POST | `reset-password` | ❌ |
| Get profile | GET | `get-profile` | ✅ |
| Get profile page | GET | `get-profile-page` | ✅ |
| Update profile | POST | `update-profile` | ✅ |
| Update password | POST | `update-password` | ✅ |
| Update social | POST | `update-social` | ✅ |
| Update photo | POST | `update-photo` | ✅ |
| Update location | POST | `update-location` | ✅ (optional) |
| Update FCM | POST | `update-fcm-token` | ✅ |
| Delete profile | DELETE | `delete-profile` | ✅ |

## 13. Request Models
- `LoginRequestModel` → `{email, password}`.
- `RegisterRequestModel` → `{email, first_name, last_name, password}`.
- `SocialAuthRequestModel` → `{provider_type, provider_id, email, name}`.
- Update profile → JSON `{first_name, last_name, email, phone, gender, x, twitter, instagram, linkedin, youtube, tiktok}` (روابط تُطبَّع بإضافة `https://`).
- Update photo → `FormData{ photo: MultipartFile }`.
- Forgot → `{emailAddress}` · Verify → `{emailAddress, code}` · Update password → `{currentPassword, password, password_confirmation}`.

## 14. Response Models
- `AuthResponseModel{code, message, data:{user:UserModel, token:String}}`.
- `UpdateProfileResponseModel{code, message, data:{personalData:UserModel, social:SocialDataModel}}`.
- `LogoutResponseModel{code, message}`.
- Envelope موحّد من `ApiModalController`: `{code/status, message, data}`.

## 15. Laravel Routes
`routes/api.php` (front group): register(46), login(47), logout(48), social/callback(49), forgot-password(52), verify-email-code(53), reset-password(57), get-profile(181), get-profile-page(182), update-profile(183), update-password(184), update-social(185), update-photo(186), update-location(187), update-fcm-token(188), delete-profile(189).
⚠️ المسارات المحمية (181–189) **لا تحمل middleware `auth:api`** على مستوى الراوت — الاعتماد على فحص `auth('api')->check()` داخل الـ controller (هشّ — راجع §22/§29).

## 16. Controllers
- `Api/Auth/AuthenticationController` — register, login, logout, userData().
- `Api/Auth/SocialController` — callback().
- `Api/Auth/ForgotPasswordController` — forgot(), verify().
- `Api/Auth/ResetPasswordController` — reset().
- `Api/ProfileController` — getProfile, getProfilePageData, updateProfile, updatePassword, updateSocial, updatePhoto, updateFcmToken, deleteProfile, updateLocation.

## 17. Service Classes (Laravel)
لا توجد خدمة مخصّصة للمصادقة؛ المنطق inline. توليد التوكن عبر `HasApiTokens::createToken('auth-token')`. الوسائط عبر Spatie MediaLibrary. الترجمة/الأدوار غير مؤثرة في هذا الـ feature مباشرة.

## 18. Models (Laravel)
- **User** (`app/Models/User.php`): `$guarded=['id']`, `$hidden=['password','remember_token']`, `$appends=['full_name']`, media collection `photo` (singleFile, disk `media`). Relations: location (HasOne), socials، وغيرها. أعمدة إضافية: `forget_password_code`, `provider_id`, `provider_type`, `api_token` (legacy), `fcm_token`, `total_points`.
- **UserSocial**: facebook, x, twitter, instagram, linkedin, youtube, tiktok — `updateOrCreate` بمفتاح `user_id`.
- **UserLocation**: lat/lng، فريد على `user_id`، cascade delete.

## 19. Database Tables
| Table | أعمدة رئيسية | ملاحظات |
|---|---|---|
| `users` | id, company_id?, first_name, last_name?, email(unique), phone?, photo?, password, gender enum, is_vendor, remember_token, +(forget_password_code, provider_id, provider_type, api_token, fcm_token(512), total_points), timestamps, deleted_at | SoftDeletes |
| `user_socials` | id, user_id(FK set null), facebook, x, twitter, instagram, linkedin, youtube, tiktok | صف واحد/مستخدم |
| `user_locations` | id, user_id(unique, FK cascade), lat DECIMAL(10,8), lng DECIMAL(11,8) | index user_id |
| `personal_access_tokens` | id, tokenable_type, tokenable_id, name, token(unique 64), abilities, last_used_at | Sanctum morph |

## 20. Relationships
`User 1—1 UserLocation` · `User 1—1 UserSocial` (updateOrCreate) · `User 1—* personal_access_tokens` (morph) · `User 1—* Guide/Story/Trip/Favorite/Save` · `User HasRoles/Permissions` (Spatie) · Media morph (photo).

## 21. Validation Rules
- **Register:** first_name/last_name required|string|max:50 · email required|email|unique:users · password required|string|min:6.
- **Login:** email required|email|exists:users · password required.
- **Social:** name required|max:100 · email required|email · provider_id required · provider_type required|string. **(بلا تحقق توقيع/JWT)**.
- **Update profile:** first_name/last_name/email required · phone/gender nullable · روابط اجتماعية nullable|string|min:3|url. **(بلا فحص تكرار email عند التحديث)**.
- **Update password:** currentPassword required · password required|confirmed|min:6 (بينما العميل يفرض min:8).
- **Update photo:** photo required|image|mimes:jpeg,png,jpg,gif,svg,webp|max:2048.
- **FCM:** fcm_token required|max:512. **Location:** lat -90..90، lng -180..180.

## 22. Authorization & Permissions
- Guard `api` (Sanctum Bearer). المسارات المحمية تتحقق داخليًا عبر `auth('api')->check()/user()/id()`.
- Spatie Permission موجود لكنه **غير مُطبَّق على مسارات المصادقة/الملف** — الأدوار مخصّصة أساسًا للـ dashboard (web guard).
- ⚠️ غياب middleware `auth:api` على الراوت يجعل بعض الدوال عرضة لـ null user (استعلام فارغ أو NullPointer حسب الدالة).

## 23. Error Handling
- **Flutter:** `Either<Failure,T>` عبر dartz في طبقة الشبكة؛ لكن دوال auth تعيد الموديل مباشرة وتلتقط `DioException` وتُصدر `*Error`. `forgotPassword/verifyEmailCode` **ترمي Exception** بدل الإرجاع.
- **Laravel:** `Validator::make` → 422 بأول رسالة · 404 مستخدم غير موجود · 422 بيانات خاطئة · 500 خطأ إرسال بريد. Envelope `error(code,msg)`/`success(data,code,msg)`.
- Logout/Delete: نجاح محلي يتجاوز فشل الـ API (يمسح التخزين دائمًا).

## 24. Edge Cases
- بريد مكرّر عند التسجيل → 422.
- كلمة مرور خاطئة → 422 `invalid_credentials`.
- مستخدم محذوف soft-deleted يدخل اجتماعيًا → **إعادة إحياء تلقائي** (خطر انتحال — §29).
- إلغاء المستخدم للدخول الاجتماعي → `SocialAuthInitial` (لا خطأ).
- Twitter → `UnimplementedError` (يُلتقط ويُظهر خطأ لطيف).
- OTP/Verify email → **التدفق غير مكتمل**: `VerifyEmailCodeCubit` stub، لا ربط فعلي بين ForgotPassword و OtpVerificationView و reset-password.
- فشل إرسال بريد الاستعادة → الكود يُحفظ مع ذلك ويُعاد خطأ 500.
- تحديث الصورة في بيئة test → إصلاح دومين URL يدويًا داخل الـ repo.

## 25. Dependencies
`dio`, `get_it`, `flutter_bloc`, `equatable`, `dartz`, `flutter_secure_storage`, `shared_preferences`, `google_sign_in`, `sign_in_with_apple`, `firebase_auth`, `firebase_core`, `cached_network_image`, `flutter_otp_text_field`, `image_picker`, `easy_localization`.
Laravel: `laravel/sanctum`, `spatie/laravel-permission`, `spatie/laravel-medialibrary`, `laravel/socialite` (مُثبّت وغير مُستغَل server-side).

## 26. Files Involved
**Flutter:** `lib/features/auth/**` (models, domain/repositories, data/repositories, presentation/views+manager, services/social_auth_service.dart)، `lib/features/profile/**` (edit_profile_cubit, update_password_cubit)، `lib/core/utils/{auth_manager,storage_service}.dart`، `lib/core/managers/user_info_cubit/**`، `lib/core/databases/api/{dio_consumer,end_points,token_interceptor}.dart`.
**Laravel:** `app/Http/Controllers/Api/Auth/{AuthenticationController,SocialController,ForgotPasswordController,ResetPasswordController}.php`، `app/Http/Controllers/Api/ProfileController.php`، `app/Http/Resources/Users/UserResource.php`، `app/Models/{User,UserSocial,UserLocation}.php`، `routes/api.php`، migrations (users, user_socials, user_locations, personal_access_tokens).

## 27. Complete Execution Flow (Login مثالًا)
```
LoginView  →  LoginCubit.login()  →  emit(Loading)
   →  AuthRepositoryImpl.login()  →  FormData  →  ApiConsumer.post(login)
   →  [TokenInterceptor/LanguageInterceptor]  →  POST /api/login
   →  AuthenticationController@login  →  Validator  →  User::where(email)
   →  Hash::check(password)  →  userData() → createToken('auth-token')
   →  DB: SELECT users · INSERT personal_access_tokens
   →  Response { code:200, data:{ user:UserResource, token } }
   →  AuthResponseModel.fromJson
   →  UserInfoCubit.setUser → AuthManager.saveUser (prefs user_data + secure auth_token)
   →  emit(LoginSuccess) → BlocListener → context.go(kHomeView)
```
(نفس النمط لـ Register، Social login عبر `social/callback`، Update profile/photo عبر `update-profile`/`update-photo`.)

## 28. Performance Considerations
- `get-profile-page` يجمع user + guide + socials + stories(paginated) + trips(paginated) — حِمل ثقيل؛ راقب N+1 على العلاقات.
- `update-location` يستخدم cache (TTL 5د) + عتبة مسافة لتقليل كتابات DB — جيد.
- الصور تُعالَج عبر MediaLibrary (أحجام small/medium) — رفع متزامن قد يبطئ الطلب.
- `PrettyDioLogger` مُفعّل دائمًا (تكلفة I/O في الإنتاج).

## 29. Security Considerations
1. 🔴 **Social login بلا تحقق ID token** (`SocialController@callback`) — يقبل `provider_id/email` من العميل = انتحال أي مستخدم.
2. 🔴 **إعادة إحياء الحساب المحذوف** تلقائيًا عبر الدخول الاجتماعي = استيلاء محتمل عند إعادة استخدام البريد.
3. 🟠 **كلمة مرور min:6** (العميل min:8، الخادم min:6) — ضعيفة وغير متسقة.
4. 🟠 **لا rate limiting** على login/register/forgot/verify — brute-force + تعداد بريد.
5. 🟠 **كود الاستعادة نصّي** في `users.forget_password_code`، وكود تطوير ثابت `1234`.
6. 🟠 **بيانات المستخدم في SharedPreferences** بنص صريح (التوكن فقط في SecureStorage).
7. 🟠 **`print()` تسرّب** الطلب/الرد/التوكن في `auth_repository_impl.dart` (+PrettyDioLogger بالإنتاج).
8. 🟡 **Mass-assignment** `$guarded=['id']` — أي عمود آخر قابل للإسناد (يعتمد على تحديد الحقول يدويًا).
9. 🟡 **غياب `auth:api` على الراوت** — الاعتماد على فحص داخلي هشّ.
10. 🟡 **MustVerifyEmail معرّف بلا تدفق تحقق فعلي**.

## 30. Technical Debt
- `VerifyEmailCodeCubit` و OTP flow **غير مكتملين**؛ `TwitterAuth` **غير مُنفّذ**.
- تكرار منطق (Logout/Delete متطابقان تقريبًا).
- الوصول العام لـ `userInfoCubit` عبر `parentKey.currentContext` (هشّ).
- `firebase_auth`/Firestore مضافة واستخدام محدود (Apple فقط)؛ `hydrated_bloc` غير مستخدم.
- `api_token` (legacy) مازال في الجدول رغم اعتماد Sanctum.
- طبقة domain موجودة لكن بلا use-cases (repo يُستدعى مباشرة).

## 31. Improvement Opportunities
- التحقق server-side من idToken (Google/Apple) عبر Socialite أو Firebase Admin.
- إضافة `middleware('auth:api')` صراحةً للمسارات المحمية.
- `throttle` على مسارات المصادقة، ورفع min password إلى 8–12 مع تعقيد.
- تخزين كود الاستعادة مُجزّأً (hash) بصلاحية زمنية، وإكمال تدفق OTP/reset.
- نقل كامل بيانات المستخدم إلى SecureStorage وإزالة `print()` من الإنتاج.
- استبدال `$guarded` بـ `$fillable` صريح.
- توحيد معالجة الأخطاء على `Either<Failure,T>` لكل دوال auth.

---
**تدفق مرجعي:** UI → Cubit → Repository → API → Laravel Route → Controller → (Service) → Model → Database → Response → Flutter UI ✅ موثّق أعلاه لـ Login/Register/Social/Update-Profile.

---

## 32. Related Features (الميزات المعتمدة على المصادقة)

المصادقة هي البنية التحتية لهوية المستخدم والتوكن؛ تعتمد عليها كل ميزة تكتب/تقرأ بيانات خاصة بالمستخدم.

| Feature | نوع الاعتماد | لماذا |
|---|---|---|
| **Trips (v1+v2)** | إلزامي للحفظ | `trips/store`, `my-trips`, `v2/trips/save`, `DELETE v2/trips/{id}` تربط الرحلة بـ `user_id` من التوكن. حذف الحساب يحذف رحلاته. |
| **Favorites & Saved** | إلزامي | `favorites`, `get-my-favorites`, `get-my-saved` تعتمد `auth('api')->user()`؛ بلا توكن → فارغ/خطأ. |
| **Rates & Reviews** | إلزامي للكتابة | `POST rates` يربط التقييم بالمستخدم. |
| **My Properties** | إلزامي | `my-properties`, `POST properties` تُنشئ محتوى باسم المستخدم (بانتظار اعتماد). |
| **Tour Guides (add/edit)** | إلزامي | `store-guide`, `update-guide-photo` تربط الدليل بـ `user_id`؛ حذف الحساب يحذف Guide. |
| **User Stories** | إلزامي | `stories/store`, `stories/{id}/delete`, `myLastDayStories` خاصة بالمستخدم؛ حذف الحساب يحذف Story. |
| **Landmarks (add/my)** | إلزامي | `landmark/store`, `landmark/myLandMarks` تربط المعلم بالمستخدم. |
| **Profile page** | إلزامي | `get-profile-page` يجمع بيانات المستخدم + guide + stories + trips. |
| **Notifications (FCM)** | إلزامي للاستهداف | `update-fcm-token` يخزّن التوكن على `users.fcm_token`؛ حملات FCM تعتمده. |
| **Home / Places / Stores / Events / Swalef (قراءة)** | اختياري | عامة تعمل بلا توكن، لكن حالة «مفضّل» و«محفوظ» تظهر فقط عند وجود مستخدم. |
| **Exploration / update-location** | اختياري | يعمل للضيوف عبر IP hash، ويربط بالمستخدم عند وجود توكن. |

> ملاحظة: أي تغيير في شكل التوكن، أو ترويسة `Authorization`، أو `UserModel`، أو envelope الرد → يؤثر على **كل** ما سبق.

---

## 33. How to Modify This Feature (كيفية التعديل)

### 33.1 ملفات Flutter للتعديل حسب نوع التغيير
- **حقل جديد في المستخدم:** `lib/features/auth/data/models/user_model.dart` (fromJson/toJson) + `UserResource.php` (الخادم) + migration. حدّث `EditProfileCubit` إن كان قابلًا للتعديل.
- **منطق دخول/تسجيل:** Cubit المعني (`login_cubit`/`register_cubit`) + `auth_repository_impl.dart` + العقد `auth_repository.dart`.
- **شاشة/تحقّق UI:** `views/*_view.dart` + `ValidationHelpers` (للتسجيل).
- **مزوّد اجتماعي جديد/إكمال Twitter:** `services/social_auth_service.dart` + `social_auth_cubit.dart` + `*_auth_model.dart`.
- **تخزين/جلسة:** `lib/core/utils/auth_manager.dart`, `storage_service.dart`, `lib/core/managers/user_info_cubit/**`.
- **ترويسة/توكن/اعتراض:** `lib/core/databases/api/{token_interceptor,dio_consumer,end_points}.dart`.

### 33.2 APIs المتأثرة
`login`, `register`, `logout`, `social/callback`, `forgot-password`, `verify-email-code`, `reset-password`, `get-profile`, `get-profile-page`, `update-profile`, `update-password`, `update-social`, `update-photo`, `update-location`, `update-fcm-token`, `delete-profile`.

### 33.3 ملفات Laravel للتعديل
- Controllers: `Api/Auth/{AuthenticationController,SocialController,ForgotPasswordController,ResetPasswordController}.php`, `Api/ProfileController.php`.
- Model/Resource: `app/Models/{User,UserSocial,UserLocation}.php`, `app/Http/Resources/Users/UserResource.php`.
- Routes: `routes/api.php` (أسطر 46–57 و181–189).
- Migrations: جداول `users`/`user_socials`/`user_locations`. (⚠️ لا تعدّل production DB مباشرة — migration فقط.)

### 33.4 المخاطر
- كسر تطابق مفاتيح JSON بين `UserModel` و`UserResource` → أعطال صامتة في كل الشاشات.
- تغيير شكل envelope الرد → يكسر كل الـ Cubits (تعتمد `code == 200`).
- تعديل تخزين التوكن → تسجيل خروج جماعي غير مقصود.
- إضافة `middleware('auth:api')` قد يكشف مسارات كانت «تنجح» بمستخدم null → تغيّر سلوك ظاهري.
- Mass-assignment: أي حقل جديد في `users` يصبح قابلًا للإسناد فورًا (`$guarded=['id']`).

### 33.5 الاختبارات المطلوبة
- Unit: كل Cubit (نجاح/فشل/شبكة)، `auth_repository_impl` (mapping + DioException).
- Widget: تحقّق نماذج Login/Register، BlocListener للتنقّل.
- Integration/manual: دورة كاملة login→محمي→logout؛ social login (إنشاء/إحياء)؛ update-photo (رفع + evict cache).
- Backend (`phpunit`): validation، status codes، إنشاء التوكن، soft-delete/restore، updateOrCreate لـ UserSocial.

---

## 34. Regression Checklist (بعد أي تعديل على المصادقة)

- [ ] تسجيل جديد ببريد جديد → نجاح + توكن + انتقال Home.
- [ ] تسجيل ببريد مكرّر → 422 برسالة واضحة.
- [ ] Login صحيح → نجاح؛ Login بكلمة خاطئة → 422 `invalid_credentials`.
- [ ] Login ببريد غير موجود → 422 (قاعدة `exists`).
- [ ] Google sign-in لمستخدم جديد → إنشاء حساب؛ لمستخدم قائم → دخول.
- [ ] Google sign-in لحساب محذوف soft-deleted → إعادة إحياء (تأكيد السلوك المقصود).
- [ ] إلغاء المستخدم للنافذة الاجتماعية → عودة لـ Initial بلا خطأ.
- [ ] Apple sign-in على iOS فقط؛ إخفاء الزر على Android.
- [ ] Logout → مسح `user_data` + `auth_token` حتى لو فشل الـ API.
- [ ] Delete profile → soft delete + حذف UserSocial/Guide/Story/Trip + مسح محلي.
- [ ] الطلبات المحمية بعد Login تحمل `Authorization: Bearer`.
- [ ] الطلبات المحمية بلا توكن → 401/سلوك متوقع (بلا انهيار).
- [ ] Update profile → تحديث users + updateOrCreate user_socials (صف واحد).
- [ ] Update photo → صورة جديدة + إزالة القديمة + تحديث URL في الواجهة (cache evicted).
- [ ] Update password → رفض كلمة حالية خاطئة، فرض min/confirmed.
- [ ] Update FCM token → حفظ على `users.fcm_token`.
- [ ] Update location: مستخدم مسجّل vs ضيف (IP)، احترام عتبة 30م/5د.
- [ ] Forgot password → إرسال كود + حفظه؛ فشل البريد → كود محفوظ + خطأ 500.
- [ ] تبديل اللغة (ar/en/ru/zh) → ترويسة `accept-language` صحيحة + RTL.
- [ ] تبديل بيئة test/prod → baseUrl صحيح + إصلاح دومين صورة الملف.

---

## 35. Common Bugs (الأعطال الأكثر احتمالًا)

| العطل | السبب المحتمل | ابدأ الفحص من |
|---|---|---|
| «Login ينجح لكن يعود لشاشة الدخول» | التوكن لم يُحفظ/لا يُقرأ | `AuthManager.saveUser/getToken`, `TokenInterceptor` |
| «الطلبات المحمية 401 رغم الدخول» | الترويسة غير مُحقَنة أو التوكن null | `token_interceptor.dart`, `UserInfoCubit.token` |
| «حقل بيانات يظهر null» | عدم تطابق مفتاح JSON بين `UserModel` و`UserResource` | `user_model.dart` fromJson ↔ `UserResource@toArray` |
| «الصورة القديمة تبقى بعد التحديث» | عدم إبطال الكاش | `auth_repository_impl.updatePhoto` (`evictFromCache`) + إصلاح الدومين |
| «FormData يفشل بـ Content-Type» | تعارض الترويسة مع multipart | منطق FormData في `dio_consumer.dart` + `TokenInterceptor` |
| «Apple sign-in يفشل على Android» | غير مدعوم بالمنصّة | `SocialAuthService.isAppleSignInAvailable` |
| «Twitter لا يعمل» | `UnimplementedError` مقصود | `social_auth_service.signInWithTwitter` |
| «كود الاستعادة دائمًا 1234» | بيئة تطوير | `ForgotPasswordController@forgot` (فرع rand vs '1234') |
| «إنشاء حسابين لنفس المستخدم اجتماعيًا» | اختلاف `provider_id` بين محاولات (fallback hash) | `_getGoogleProviderId` + بحث الخادم بالبريد |
| «تعديل الملف لا يحفظ الروابط الاجتماعية» | تطبيع/مفاتيح backend خاطئة | `EditProfileCubit._platformToBackendKey` + `ProfileController@updateProfile` |
| «logout لا يسجّل خروجًا حقيقيًا من الخادم» | التوكن لا ينتهي + مسح محلي فقط | `AuthenticationController@logout` (currentAccessToken) |

---

## 36. Debug Guide (خطوة بخطوة)

1. **فعّل الشبكة:** استخدم `PrettyDioLogger` (مُفعّل) أو `Chucker` عبر `AppEnvironmentManager.isChuckerEnabled=true` لرؤية الطلب/الرد/الترويسات.
2. **تحقّق البيئة:** أكّد `baseUrl` (prod vs test) عبر `AppEnvironmentManager`.
3. **تتبّع الحالة:** راقب انتقالات الـ Cubit (Loading→Success/Error) عبر `BlocObserver`/logger؛ الخطأ يظهر في `*Error(message)`.
4. **افحص التوكن:** اطبع/راقب `AuthManager.getToken()` و`UserInfoCubit.token`؛ تأكّد `TokenInterceptor` يحقن `Authorization`.
5. **افحص التخزين:** `user_data` في SharedPreferences، `auth_token` في SecureStorage.
6. **جانب الخادم:** فعّل `APP_DEBUG`؛ تابع `storage/logs/laravel.log`؛ أضف `\Log::info()` في الـ controller المعني.
7. **قاعدة البيانات:** تحقّق `users`, `personal_access_tokens` (توكن جديد لكل دخول)، `user_socials`, `user_locations`.
8. **إعادة الإنتاج:** كرّر السيناريو من §34 المطابق، وقارن الرد الفعلي بالمتوقّع (`code`, `message`, `data`).
9. **الاجتماعي:** إن فشل، تتبّع `SocialAuthService` (idToken/accessToken)، ثم استخراج `providerId`، ثم رد `social/callback`.

---

## 37. Search Keywords
`auth`, `login`, `register`, `logout`, `sanctum`, `bearer token`, `createToken`, `personal_access_tokens`, `social login`, `google sign in`, `apple sign in`, `provider_id`, `social/callback`, `forgot-password`, `verify-email-code`, `reset password`, `forget_password_code`, `AuthManager`, `UserInfoCubit`, `TokenInterceptor`, `StorageService`, `auth_token`, `user_data`, `FlutterSecureStorage`, `UserModel`, `UserResource`, `AuthResponseModel`, `SocialAuthService`, `LoginCubit`, `RegisterCubit`, `SocialAuthCubit`, `EditProfileCubit`, `UpdatePasswordCubit`, `ProfileController`, `AuthenticationController`, `SocialController`, `update-profile`, `update-photo`, `update-fcm-token`, `fcm_token`, `update-location`, `user_socials`, `user_locations`, `delete-profile`, `soft delete`, `$guarded`, `HasApiTokens`.

---

## Web Frontend (Angular)
- **Email/password:** same endpoints as mobile (`/login`,`/register`,`/logout`,`/forgot-password`,`/verify-email-code`,`/reset-password`,`/update-password`,`/update-profile`,`/get-profile-page`) — `auth.service.ts`.
- **Social login DIFFERS:** web = **Firebase browser popup** (Google/Twitter) → FormData `{name,email,provider_id,provider_type}` → **POST `/social/callback`** → store `data` in localStorage → `getProfileData()` (`auth-firebase.service.ts:57–152`). Mobile uses native SDKs. Verify both yield same user/token shape.
- **Token:** JWT Bearer in `localStorage`, attached by HTTP interceptor.
- **401 handling DIFFERS (web stricter):** `error-interceptor.service.ts:52–58` → 401 clears all localStorage + POST `/logout` + redirect `/home`. Mobile **ignores 401** ([[24_Networking]]).
- **Gated actions:** web opens **JoinUs modal** (`AuthService.checkIsLogin`) or `AuthGuard` redirect `/home` (permissive). See [[WEB_MOBILE_COMPARISON]].

## 38. Future Improvements
- تحقّق server-side من idToken (Firebase Admin / Socialite) لإغلاق ثغرة انتحال الدخول الاجتماعي.
- إضافة `middleware('auth:api')` صراحةً + سياسات (Policies) للملكية.
- refresh tokens + انتهاء صلاحية Sanctum + إبطال التوكن عند تغيير كلمة المرور.
- `throttle` على مسارات المصادقة + قفل مؤقت بعد محاولات فاشلة.
- سياسة كلمة مرور موحّدة (≥12، تعقيد) بين العميل والخادم.
- إكمال تدفق OTP/verify/reset وربط `VerifyEmailCodeCubit` + `OtpVerificationView`.
- إكمال Twitter عبر web auth، أو إزالته من الواجهة إن لم يُعتمد.
- تخزين كامل بيانات المستخدم في SecureStorage + إزالة `print()`/تعطيل logger بالإنتاج.
- تدفق تحقّق بريد فعلي (تفعيل `MustVerifyEmail`).
- 2FA اختياري، وتسجيل نشاط الدخول (activity log الموجود عبر Spatie).
- استبدال `$guarded=['id']` بـ `$fillable` صريح، وإزالة عمود `api_token` القديم.
- استخراج AuthService في Laravel لتقليل منطق الـ controllers، وتوحيد أخطاء Flutter على `Either<Failure,T>`.
