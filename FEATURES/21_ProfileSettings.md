# Feature 21 — Profile Settings (الملف الشخصي والإعدادات)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Supporting (account management)
> **Flutter:** `lib/features/profile/` + auth cubits (delete/logout) · **Laravel:** ProfileController, ContactusController, SettingController
> **One-line:** Account/settings hub: profile view + edit, change-password (current-password verified ✅), delete-account, logout, language switch, and the `get-profile-page` aggregation (PersonalData + guide + social + myStories + myTrips). Static pages (terms/contact) are **hardcoded in Flutter localization/UI, NOT backend-served**; the contact-us form appears **display-only (no API submission wired)** while the backend `contactus/send` endpoint is **public with no throttle**. Account endpoints use **manual per-controller auth (no route middleware)**, and **logout does NOT clear `fcm_token`** (orphaned push, per [[17_Notifications]]).

---

## 1. Feature Name & Identity
- **Name:** Profile Settings — account management + static/info sub-screens + profile aggregation.
- **Scope here:** profile hub, edit profile, change password, contact-us, terms, delete account, logout, language switch, get-profile-page. (Favorites/Saved [[13_FavoritesSaved]], My Properties [[15_MyProperties]], Stories [[16_UserStories]], Notifications [[17_Notifications]] are separate.)

## 2. Business Goal & Rules
- **Goal:** Let users manage identity/credentials and reach app info/support.
- **Rules:**
  - Change password requires current-password match (`Hash::check`) ✅.
  - Delete account: soft-deletes user (SoftDeletes) + manual cascade on some relations; local storage cleared regardless of API result.
  - Contact-us endpoint public (no auth), minimal validation, no rate limit.
  - Static content (terms/contact) from Flutter localization/hardcoded — not CMS.
  - Language switch = local EasyLocalization preference (no backend).

## 3. User Flow
1. Profile hub (`ProfileView`) → menu: edit profile, change password, contact-us, terms, language, logout, delete account.
2. Edit → update name/email/phone/gender/photo → POST update-profile / update-photo.
3. Change password → current + new → POST update-password.
4. Delete account → confirm → DELETE delete-profile → clear local → login.
5. Logout → POST logout (revoke token) → clear local → login.

## 4. Screens / Views
| File | Purpose |
|------|---------|
| `profile/presentation/views/profile_view.dart` | Settings hub menu + language bottom sheet (274–280) |
| `edit_profile_view.dart` | Edit name/email/phone/gender/photo |
| `update_password_view.dart` | Change password form |
| `contact_us_view.dart` (body 10–91) | **Static** phone `+966508707717`, email `hello@hawdaj.net`, Riyadh addr — tap-to-call/email/WhatsApp; no form submit |
| `terms_and_conditions_view.dart` (body 21–28) | `terms_*` strings from .arb (hardcoded i18n) |

## 5. Widgets
- Settings menu tiles, LanguageBottomSheet, edit form fields, password fields, contact info tiles (call/email/WhatsApp actions), terms text blocks.

## 6. Cubits / Blocs
| Cubit | Location | Role |
|-------|----------|------|
| GetProfileCubit | `profile/.../get_profile_cubit` | fetch get-profile-page |
| EditProfileCubit | `profile/.../edit_profile_cubit` (47–89) | form state, updateProfile, refresh UserInfo+GetProfile |
| UpdatePasswordCubit | `profile/.../update_password_cubit` (49–69) | current/new validation, update |
| DeleteProfileCubit | `auth/.../delete_profile_cubit` (12–32) | call API then `AuthManager.logout()` always |
| LogoutCubit | `auth/.../logout_cubit` | logout + clear local |

## 7. State Flow
- Edit: load → validate (`isFormValid`) → updateProfile → update UserInfoCubit (31) + refresh GetProfileCubit (37).
- Password: validate current+new → update → success/error.
- Delete: API → `AuthManager.logout()` unconditionally (20) → navigate login.

## 8. Models
- Profile response model (`profile_reponce.dart`): PersonalData(UserResource), userGuideData, social[], myStories(paginated), myTrips(paginated), totalStories, totalTrips.
- UserResource fields: id, first_name, last_name, email, phone, photo(+small/medium), gender, full_name, total_points.

## 9. Repositories
- Auth/profile repo methods: updateProfile, updatePassword, deleteProfile, getProfilePage, logout. Contact-us has no wired repo call on client.

## 10. Services / Helpers
- `AuthManager.logout()` (auth_manager.dart:61–64): clears `_userKey` + secure `_tokenKey` (local only).
- `StorageService`, `Hash` (backend), EasyLocalization (language).

## 11. API Endpoints
| Method | Endpoint | Auth | Controller |
|--------|----------|------|-----------|
| GET | `get-profile` | manual auth('api') | ProfileController (181) |
| GET | `get-profile-page` | manual | ProfileController@…(36–78) |
| POST | `update-profile` | manual | ProfileController (80–140) |
| POST | `update-password` | manual | ProfileController (166–197) |
| POST | `update-photo` | manual | ProfileController (186) |
| POST | `update-fcm-token` | manual | (see [[17_Notifications]]) |
| DELETE | `delete-profile` | manual | ProfileController (269–289) |
| POST | `contactus/send` | **NONE** | ContactusController (14–30) |
| POST | `logout` | auth | AuthenticationController |
| GET | social media | — | SettingController@getSocialMedia |

`routes/api.php:118, 181–189`.

## 12. Request / Response Models
- **update-profile req:** `{first_name,last_name,email,phone,gender}` (+ social). Explicit assignment.
- **update-password req:** `{currentPassword, password, password_confirmation}`.
- **get-profile-page resp:** `{PersonalData, userGuideData, social[], myStories, myTrips, totalStories, totalTrips}` — all PII exposed.
- **contactus/send req:** `{name, email, phone, message}` → `Opinion::create`.

## 13. Laravel Routes
`routes/api.php:118` (contactus/send public), `181–189` (profile/account, manual auth in controller), logout in auth group.

## 14. Laravel Controllers
- ProfileController: `getProfilePage` (36–78 aggregation), `updateProfile` (80–140, explicit fields), `updatePassword` (166–197: `Hash::check(currentPassword)` at 188 ✅ then `Hash::make` 193), `deleteProfile` (269–289: manual cascade UserSocial/Guide/Story/Trip then `$user->delete()`; fcm_token NOT nulled).
- ContactusController (14–30): validate (name regex max25, email, phone max14, message) → `Opinion::create($request->all())` → success. No auth/throttle.
- SettingController: only `getSocialMedia()` (no terms/about/faq endpoint).

## 15. Laravel Services
- None dedicated. `Setting` model has `$translatedAttributes=['value']` (key-value translatable) but **no API endpoint** exposes it for static pages.

## 16. Laravel Models
- User (`User.php`): `SoftDeletes` (22), `$guarded=['id']` (26). Notifiable (fcm_token).
- Opinion (contact): `$guarded=[]` (fully fillable).
- Setting: translatable value; UserSocial (social links).

## 17. Database Tables & Relationships
- users (soft deletes, fcm_token, points), user_socials, opinions (contact messages), settings (key-value translatable), + guides/stories/trips (cascaded on delete).
- Delete cascade explicit only for UserSocial/Guide/Story/Trip; **Properties/Favorites/Saved/Rates NOT cleared**.

## 18. Validation
- update-password: current required + match; new + confirmation.
- update-profile: field validation (email/phone).
- contactus: name regex/max25, email, phone max14, message (no min length, no CAPTCHA, no rate limit).

## 19. Auth & Permissions
- Account endpoints: **manual `auth('api')->check()` per controller, no route middleware** → fragile; easy to omit on new methods.
- contactus/send: **public, no throttle** → spam vector.
- logout: revokes current access token only.

## 20. Error Handling
- Flutter Either/cubit states; delete clears local even on API failure.
- Laravel: 422 on password mismatch/validation; success envelopes.

## 21. Edge Cases
- Delete leaves orphaned Properties/Favorites/Saved/Rates (partial cascade); FK failure risk.
- Logout leaves `fcm_token` → ex-user device keeps receiving push.
- Contact form: backend exists but client shows static info only → user "messages" go nowhere via app.
- Terms/about not translatable from backend → content change = app release.
- get-profile-page always ships full PII.
- Soft-deleted user unrecoverable via API (no restore).

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dartz, easy_localization, url_launcher (call/email/WhatsApp), image_picker (photo), StorageService.
- **Laravel:** Hash, SoftDeletes, Eloquent, ApiModalController.
- **Cross-feature:** [[01_Authentication]] (identity/token/logout), [[17_Notifications]] (fcm_token cleanup gap), [[13_FavoritesSaved]]/[[15_MyProperties]]/[[16_UserStories]] (profile tabs), [[22_Localization]] (language switch), [[11_TourGuides]] (guide data in profile).

## 23. Files Involved
**Flutter:** profile views (profile_view, edit_profile_view, update_password_view, contact_us_view, terms_and_conditions_view), profile cubits (get/edit/update_password), auth cubits (delete_profile, logout), AuthManager, profile_reponce model, repos, routes, service_locator.
**Laravel:** ProfileController, ContactusController, SettingController, AuthenticationController(logout), User/Opinion/Setting/UserSocial models, UserResource, routes/api.php:118+181–189.

## 24. Full Execution Flow (UI → DB → UI)
**Change password:** `update_password_view` → UpdatePasswordCubit (validate current+new) → repo POST `update-password` → ProfileController@updatePassword → `Hash::check(currentPassword)` (188) → `Hash::make` + update (193) → success → UI.
**Delete:** confirm → DeleteProfileCubit → DELETE `delete-profile` → manual cascade (UserSocial/Guide/Story/Trip) → `$user->delete()` (soft) → `AuthManager.logout()` (clear local) → login.
**Profile page:** GetProfileCubit → GET `get-profile-page` → aggregate user+guide+social+stories+trips → PII response → render tabs.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| get-profile-page aggregates many relations in one call | 🟢 Low | ProfileController |
| No pagination control on embedded stories/trips | 🟢 Low | aggregation |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **contactus/send public + no throttle** | 🔴 CRITICAL | routes:118, ContactusController |
| **Account endpoints no route middleware (manual auth)** | 🟡 HIGH | routes:181–189 |
| **Logout doesn't clear fcm_token** (orphaned push) | 🟡 HIGH | AuthenticationController/ProfileController |
| Delete-account incomplete cascade (Properties/Favorites/Saved/Rates orphaned) | 🟡 Med | ProfileController:280–283 |
| PII always in profile response, no privacy mode | 🟡 Med | UserResource:19–34 |
| Opinion `$guarded=[]` (mass-assign) | 🟢 Low | Opinion model |
| Change-password verifies current ✅ | OK | :188 |

## 27. Technical Debt
- Static pages (about/terms/privacy/faq) hardcoded in Flutter — no CMS despite translatable `Setting` model existing.
- Contact form UI shows static info; backend endpoint unused by client (dead wiring).
- Manual auth pattern (no middleware) across account routes.
- Partial delete cascade.
- Logout fcm_token gap.
- No soft-delete restore path.

## 28. Improvement Opportunities
- Wrap account routes in `auth:api` middleware group; add `throttle:5,1` + auth (or CAPTCHA) to contactus.
- Clear `fcm_token` on logout + delete.
- Complete delete cascade (or FK onDelete) for all user content; add data-export/restore per privacy law.
- Expose translatable `Setting`-backed static pages (`GET /settings/{key}`) and wire terms/about/faq/contact to backend.
- Wire the contact form to `contactus/send`.
- Privacy-scoped profile resource (hide PII where not needed).

---

## 32. Related Features
- **[[01_Authentication]]** — login/token/logout, identity.
- **[[17_Notifications]]** — fcm_token registration + logout-clear gap.
- **[[13_FavoritesSaved]] [[15_MyProperties]] [[16_UserStories]] [[11_TourGuides]]** — surfaced in profile page.
- **[[22_Localization]]** — language switch.
- **[[18_Taxonomy]]/[[26_DashboardAdmin]]** — Settings key-value source.

## 33. How to Modify This Feature
- **Add a settings menu item:** add tile in `profile_view.dart` + route/screen.
- **Add backend static page:** `GET /settings/{key}` returning translatable `Setting.value`; Flutter fetch + render.
- **Fix logout token:** add `$user->update(['fcm_token'=>null])` in logout + delete.
- **Secure contact:** add auth/throttle to route; wire Flutter form to POST contactus/send.
- **Complete delete cascade:** add Properties/Favorites/Saved/Rates deletes or FK onDelete.

## 34. Regression Checklist
- [ ] Edit profile persists + reflects on profile page.
- [ ] Change password rejects wrong current, accepts correct.
- [ ] Delete account clears session + soft-deletes user.
- [ ] Logout revokes token + clears local.
- [ ] Language switch persists.
- [ ] (After fix) logout clears fcm_token.
- [ ] (After fix) contact form submits + throttled.

## 35. Common Bugs
- Contact form "sends" nothing (client static only).
- Ex-user device still gets push (fcm_token not cleared).
- Orphaned content after account delete.
- Terms/about outdated (hardcoded, needs release).
- New account route forgets manual auth check → open endpoint.

## 36. Debug Guide
- **Password change fails:** confirm currentPassword matches hash; check 422 message.
- **Profile not updating:** verify explicit fields sent; check UserInfoCubit + GetProfile refresh.
- **Push after logout:** check users.fcm_token still set.
- **Contact not received:** client doesn't call API — expected; backend Opinion table only via direct POST.
- **Delete leaves data:** audit cascade list vs actual user relations.

## 37. Search Keywords
`ProfileController`, `get-profile-page`, `update-profile`, `update-password`, `delete-profile`, `contactus/send`, `ContactusController`, `Opinion`, `Setting translatedAttributes`, `Hash::check currentPassword`, `AuthManager.logout`, `EditProfileCubit`, `DeleteProfileCubit`, `fcm_token logout`, `terms_and_conditions`, الملف الشخصي, الإعدادات, تغيير كلمة المرور, حذف الحساب, اتصل بنا.

## Web Frontend (Angular)
- **v2 settings hub:** `/settings/*` (`domains/profile-settings/`): personal-info, posts, tour-guide-info, landmarks, my-favourites, my-properties. v1 `/Profile` also present. Reactive Forms + custom validators (`property-form.validators.ts`: url/phone-or-url/date-range/store-rules/required-file-array).
- **Contact/Newsletter WORK on web (DIFFERS):** `POST contactus/send` (leaveInquiry) + `POST subscribe` are wired to real forms (`home.service.ts:128–136`). Mobile's contact screen is **display-only** — inconsistent product behavior; recommend wiring mobile to the same endpoint.
- **Change password:** same `/update-password`. **Logout:** POST `/logout` (+ web also auto-logout on 401 via interceptor).
- **Static pages (terms/about):** web i18n strings (like mobile) — no CMS. See [[WEB_MOBILE_COMPARISON]].

## Known Incidents (live)
- **[[KNOWN_ISSUES#K4]] / [[KNOWN_ISSUES#K1]] — بريد الحساب لا يصل:** بريد إنشاء المستخدم queued (يحتاج worker) + فشل SMTP بالإنتاج. يطال أي إشعار بريدي متعلّق بالحساب. راجع [[26_DashboardAdmin]].

## 38. Future Improvements
- Middleware-grouped, rate-limited account API; CAPTCHA/throttle on contact.
- CMS-backed translatable static pages + working contact form.
- Full delete cascade + GDPR export/restore.
- Privacy-scoped profile; token cleanup on logout/delete.
- Consolidated account-settings service across auth + profile.
