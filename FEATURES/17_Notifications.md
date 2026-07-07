# Feature 17 — Notifications (FCM Push + Marketing Broadcast)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Supporting (engagement/messaging)
> **Flutter:** `lib/core/services/notification_manager.dart` (+ main.dart) · **Laravel:** `ProfileController::updateFcmToken` + `MarketingPush` service + FCM notifications
> **One-line:** **Asymmetric & half-wired.** Laravel side is real: stores `users.fcm_token`, sends FCM push via `laravel-notification-channels/fcm` + `kreait/firebase-php` (HTTP v1, service-account credentials from env), and broadcasts "new content" marketing pushes to all token-holding users via a queued chunked job triggered by model create/activate. Flutter side is **effectively DISABLED**: `firebase_messaging` is NOT in pubspec, `notification_manager.dart` / `local_notification_service.dart` / `reverb_websocket_service.dart` are fully commented out, there is NO active token-registration code, NO push handlers, and NO in-app notifications list UI. Backend `notifications` table exists but has **no API list/mark-read endpoint** (dashboard-only).

---

## 1. Feature Name & Identity
- **Name:** Notifications — device push (FCM) + marketing broadcast + (planned) in-app list.
- **Two working backend paths:** (1) token registration endpoint, (2) marketing broadcast pipeline. In-app list + Flutter client are **not implemented/active**.
- **Planned-but-disabled alt transport:** Laravel Reverb WebSocket (private channels) — commented out in Flutter.

## 2. Business Goal & Rules
- **Goal:** Notify users of new/updated content (marketing) and (future) per-user events; maintain a notifications history.
- **Rules (as implemented, backend):**
  - `update-fcm-token` requires auth:api; validates `fcm_token` string max 512.
  - Marketing broadcast fires when a supported model is created or activated (`active=true`), gated by `marketing_push.enabled` + credentials present.
  - Broadcast targets: users where `deleted_at IS NULL AND fcm_token NOT NULL/''`, chunked by 150.
  - Per-user send wrapped in try/catch (one failure doesn't abort the batch).
- **Not implemented:** opt-in/unsubscribe, token cleanup on logout, in-app list API, mark-read API, deep-linking (client disabled).

## 3. User Flow
- **Working (backend):** admin/user creates or activates content → `MarketingPushDispatcher` → queued `BroadcastMarketingNewContentJob` → chunks users → `$user->notify(MarketingNewContentNotification)` → FCM channel → device.
- **Broken/disabled (client):** app launch would getToken → POST `update-fcm-token` → receive push (onMessage/background/terminated) → show local notification → tap → deep-link. **None of this runs** (commented out, no firebase_messaging).
- **In-app list:** no screen; no API to fetch history.

## 4. Screens / Views
- **NONE active.** No notifications list screen, no notification settings screen wired. Infrastructure files exist but are commented out.

## 5. Widgets
- None active. (Local notification channel setup exists in `local_notification_service.dart` but fully commented.)

## 6. Cubits / Blocs
- **None active** for notifications. No NotificationCubit wired.

## 7. State Flow
- N/A on client (disabled). Backend flow is event→dispatcher→job→FCM (no client state).

## 8. Models
- **Flutter:** a notification model was designed but the feature is inactive (no live parse/fetch).
- **Laravel:** `Notification` model over standard `notifications` table (`read_at` nullable for mark-read via Laravel's `markAsRead()`).

## 9. Repositories
- **Flutter:** no active notifications repository.
- **Laravel:** notification dispatch via `Notifiable` trait + FCM channel, not a repo.

## 10. Services / Helpers
| Component | File | Role |
|-----------|------|------|
| MarketingPushDispatcher | `app/Services/MarketingPush/MarketingPushDispatcher.php` | Listens to model Created/Updated; on activate → dispatch broadcast; checks credentials + config |
| BroadcastMarketingNewContentJob | `app/Jobs/BroadcastMarketingNewContentJob.php` | Queued; selects token-holding users; chunks by 150; per-user notify w/ try-catch |
| MarketingNewContentNotification | `app/Notifications/MarketingNewContentNotification.php` | `via()` = FcmChannel; builds FcmMessage (title/body + data payload); uses `first_name` in copy |
| User::routeNotificationForFcm | `app/Models/User.php:116–121` | returns `fcm_token` for FCM channel |
| config/firebase.php | reads `FIREBASE_CREDENTIALS` / `GOOGLE_APPLICATION_CREDENTIALS` | service-account path |
| config/marketing_push.php | enabled, allow_console, queue, chunk (150) | tuning |
| (Flutter) notification_manager.dart / local_notification_service.dart / reverb_websocket_service.dart | **fully commented out** | disabled client infra |

## 11. API Endpoints
| Method | Endpoint | Auth | Status |
|--------|----------|------|--------|
| POST | `update-fcm-token` | auth:api | ✅ active (ProfileController::updateFcmToken, routes/api.php:188) |
| GET | `dashboard/notifications` | auth:web | dashboard-only (not mobile) |
| DELETE | `dashboard/notifications/{id}` | auth:web | dashboard-only |
| GET | `notifications` (list) | — | ❌ **NOT IMPLEMENTED** |
| PATCH | `notifications/{id}/read` | — | ❌ **NOT IMPLEMENTED** |

## 12. Request / Response Models
- **update-fcm-token req:** `{fcm_token: "<=512 chars>"}` → stores `users.fcm_token`. Resp: message envelope.
- **FCM message (outbound):** `FcmMessage::setNotification(title,body)->setData({...})`.
- **notifications table row:** `{id(uuid), type, notifiable_type, notifiable_id, data(JSON), read_at, timestamps}`.

## 13. Laravel Routes
```
POST   update-fcm-token            → ProfileController@updateFcmToken   [auth:api]   (routes/api.php:188)
GET    dashboard/notifications     → Dashboard\NotificationController@index   [auth:web]
DELETE dashboard/notifications/{id}→ Dashboard\NotificationController@destroy [auth:web]
```
- **No mobile API for listing/marking notifications.**

## 14. Laravel Controllers
- [`ProfileController::updateFcmToken()`](../../../hawdaj-api/app/Http/Controllers/Api/ProfileController.php) `:142–164`: auth:api; validate `fcm_token` required|string|max:512; save to authed user; return message.
- `Dashboard\NotificationController@index`: `scopePrimary()` list for web dashboard; not exposed to API.

## 15. Laravel Services
- `MarketingPushDispatcher` (onCreated/onUpdated, credentials + enabled gate, activation check ~L129).
- `BroadcastMarketingNewContentJob` (async queue, chunk 150, user query, per-user notify, error logging).
- FCM send via `NotificationChannels\Fcm\FcmChannel` + `kreait/firebase-php` v6.9.6 (HTTP v1 through Firebase Admin SDK).

## 16. Laravel Models
- `User`: `Notifiable`, `fcm_token` fillable, `routeNotificationForFcm()` (116–121).
- `Notification`: standard Laravel notifications model; `read_at` nullable.
- **Marketing source models** (trigger broadcast on activate): Place, Store, Event, ZadElgadel, Application, Menu, Offer, Swalef, Guide.

## 17. Database Tables & Relationships
- [`users.fcm_token`](../../../hawdaj-api/database/migrations/2026_05_11_000001_add_fcm_token_to_users_table.php): string(512) nullable. **No unique constraint** (same token can map to multiple users).
- [`notifications`](../../../hawdaj-api/database/migrations/2021_09_22_212212_create_notifications_table.php): uuid id, type, notifiable morph, data(text), read_at nullable, timestamps.
- No `device_tokens` table (single token per user column, not multi-device).

## 18. Validation
- `update-fcm-token`: `fcm_token` required|string|max:512. No rate limit, no uniqueness. Marketing payload built server-side (no client input).

## 19. Auth & Permissions
- `update-fcm-token` auth:api ✅.
- Dashboard notification routes auth:web (admin).
- Marketing broadcast is server-triggered (event/observer), not a public endpoint — cannot be invoked by arbitrary clients. ✅
- **Token not cleared on logout** → orphaned token keeps receiving pushes.

## 20. Error Handling
- Job: per-user try/catch, failures logged, batch continues.
- Dispatcher: skips if credentials/config missing.
- Client: N/A (disabled).

## 21. Edge Cases
- Token reused across users (no unique) → wrong-user delivery after device handoff.
- Logout leaves stale token → ex-user's device still notified.
- Broadcast has no opt-out → every content activation pushes all users.
- Client disabled → tokens are only ever registered if/when Flutter code is re-enabled; currently likely no tokens flowing → broadcasts may reach few/no devices.
- `notifications` rows accumulate but are unreachable from mobile (no list API).

## 22. Dependencies
- **Laravel:** `laravel-notification-channels/fcm` v2.7.0, `kreait/firebase-php` v6.9.6, queue worker, service-account JSON.
- **Flutter (declared):** firebase_core, firebase_auth (**firebase_messaging NOT present**), flutter_local_notifications (infra, disabled), web_socket (Reverb, disabled).
- **Cross-feature:** marketing triggered by [[04_Places]] [[05_Stores]] [[09_Events]] [[06_Restaurants]] [[10_Swalef]] [[11_TourGuides]] + Application/Menu/Offer activation. Token registration under [[21_ProfileSettings]]. Reverb ties to [[25_Realtime]].

## 23. Files Involved
**Flutter (all disabled):** main.dart (firebase core init ~L54), core/services/notification_manager.dart, local_notification_service.dart, reverb_websocket_service.dart, pubspec.yaml (no firebase_messaging).
**Laravel:** ProfileController::updateFcmToken, User model (routeNotificationForFcm), Notification model, MarketingPushDispatcher, BroadcastMarketingNewContentJob, MarketingNewContentNotification, config/firebase.php, config/marketing_push.php, migrations (fcm_token, notifications), Dashboard\NotificationController, routes/api.php:188.

## 24. Full Execution Flow (backend, working path)
Content model `save()` with `active=true` → model observer → `MarketingPushDispatcher::onCreated/onUpdated` → check `marketing_push.enabled` + `firebaseCredentialsConfigured()` → dispatch `BroadcastMarketingNewContentJob` (queued) → `User::where(deleted_at null, fcm_token not null)->chunk(150)` → per user `notify(new MarketingNewContentNotification($model))` → `via()=FcmChannel` → `FcmMessage(title,body,data)` → kreait Admin SDK HTTP v1 → FCM → device.
**Token registration (would-be client path, currently inert):** app launch → getToken → POST `update-fcm-token` → `users.fcm_token` set.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Broadcast to ALL users per activation (no targeting/opt-in) | 🟡 Med | MarketingPushDispatcher/Job |
| Chunk 150 mitigates memory, but full-table user scan each broadcast | 🟢 Low | Job query |
| Dashboard list no eager-load of notifiable (N+1) | 🟢 Low | Dashboard\NotificationController |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **Firebase service-account credentials** — must be env/secret, NOT committed | 🔴 HIGH (verify) | config/firebase.php (`FIREBASE_CREDENTIALS`) |
| Token not cleared on logout → orphaned delivery | 🟡 Med | no logout hook |
| No token uniqueness → cross-user delivery risk | 🟡 Med | users.fcm_token |
| update-fcm-token: no rate limit | 🟢 Low | ProfileController |
| Broadcast lacks opt-out / preference filter | 🟢 Low | dispatcher |
| PII (first_name) in notification copy/data | 🟢 Low | MarketingNewContentNotification |

## 27. Technical Debt
- Entire Flutter notifications stack commented out; no firebase_messaging → push non-functional end-to-end on device.
- In-app notifications table exists but no mobile list/mark-read API.
- Single `fcm_token` column (no multi-device `device_tokens` table).
- No logout token cleanup.
- Reverb WebSocket path half-planned, disabled.
- Marketing broadcast untargeted, no unsubscribe.

## 28. Improvement Opportunities
- Add `firebase_messaging`; implement getToken + register on launch + onMessage/background/terminated + local notification display + deep-link by `data.type`.
- Add `GET /notifications` + `PATCH /notifications/{id}/read` (+ unread count) API over the existing table.
- Move to `device_tokens` table (user_id, token, platform, unique) for multi-device; clear on logout.
- Add opt-in/preferences + unsubscribe; target broadcasts by region/interest.
- Verify service-account JSON is gitignored; rotate if leaked.

---

## 32. Related Features
- **[[21_ProfileSettings]]** — token registration lives on profile/user update.
- **[[25_Realtime]]** — Reverb WebSocket (disabled) alt transport.
- **[[04_Places]] [[05_Stores]] [[09_Events]] [[06_Restaurants]] [[10_Swalef]] [[11_TourGuides]]** — activation triggers marketing push.
- **[[01_Authentication]]** — logout should clear token (currently doesn't).
- **[[26_DashboardAdmin]]** — dashboard notifications list/delete.

## 33. How to Modify This Feature
- **Enable push (client):** add firebase_messaging to pubspec; uncomment/rewrite notification_manager; on launch getToken → POST update-fcm-token; register handlers; deep-link by payload type.
- **Add in-app list:** new API `GET /notifications` (paginate authed user's notifications) + `PATCH .../read`; Flutter cubit + screen.
- **Fix logout:** clear `users.fcm_token` (or delete device_tokens row) in logout controller.
- **Change broadcast targeting:** edit BroadcastMarketingNewContentJob user query / add preference filter.

## 34. Regression Checklist
- [ ] update-fcm-token stores token (auth required).
- [ ] Activating content dispatches broadcast job (with credentials + enabled).
- [ ] Job chunks users, skips null tokens, survives per-user failure.
- [ ] (After client work) device receives push foreground/background/terminated.
- [ ] (After API work) list + mark-read work per user.
- [ ] Logout clears token.
- [ ] Service-account JSON not in VCS.

## 35. Common Bugs
- No pushes received on device (client stack disabled / no firebase_messaging).
- Broadcasts reach nobody (no tokens registered because client inert).
- Ex-user's device keeps getting pushes (no logout cleanup).
- Notifications history invisible in app (no API).
- Duplicate/misrouted push if token reused across accounts.

## 36. Debug Guide
- **No push:** confirm firebase_messaging present + token actually POSTed to update-fcm-token; check users.fcm_token populated.
- **Broadcast not firing:** check marketing_push.enabled, credentials configured, model `active` flip, queue worker running.
- **Send errors:** inspect job logs (per-user try/catch), verify service-account validity.
- **Wrong recipient:** check token uniqueness / stale token after logout.
- **List empty:** no mobile API exists — expected.

## 37. Search Keywords
`fcm_token`, `update-fcm-token`, `routeNotificationForFcm`, `FcmChannel`, `FcmMessage`, `MarketingNewContentNotification`, `BroadcastMarketingNewContentJob`, `MarketingPushDispatcher`, `kreait firebase-php`, `FIREBASE_CREDENTIALS`, `notifications table read_at`, `firebase_messaging`, `notification_manager`, `reverb`, الإشعارات, دفع, تسويق.

## 38. Future Improvements
- Complete Flutter FCM integration + in-app notification center (list, unread badge, deep-link, settings).
- Multi-device token table + logout cleanup + token TTL/refresh.
- Targeted, opt-in marketing with unsubscribe + analytics.
- Consolidate push (FCM) and realtime (Reverb) notification streams into one client handler.
