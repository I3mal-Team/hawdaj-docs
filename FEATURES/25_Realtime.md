# Feature 25 — Real-time (Reverb / Pusher / WebSockets / Agora)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Shared (real-time infra)
> **Flutter:** `lib/core/services/reverb_websocket_service.dart` (**100% commented out**) · **Laravel:** broadcasting configured but `BROADCAST_DRIVER=null`
> **One-line:** Real-time is **PLANNED/SCAFFOLDED, NOT WIRED.** Flutter has a full WebSocket service skeleton (Reverb, 3 streams: notification/chat/agoraEndSession) but it's entirely commented out, its `WebSocketConfig` import points to a **non-existent file**, and no `laravel_echo`/`pusher_client`/`web_socket_channel`/`agora_rtc_engine` deps exist. Laravel has broadcasting registered + one live event (`RealTimeMessage` on car check-in, public channel) but defaults driver to **null (no-op)**, and its `routes/channels.php` channels **do NOT match** the channels Flutter expects. Chat + Agora video/voice are placeholders on the client with zero backend support.

---

## 1. Feature Name & Identity
- **Name:** Real-time — WebSocket push (Reverb/Pusher), broadcast events, private/presence channels, live notifications/chat, Agora session signaling.
- **State:** Client fully disabled; backend configured-but-off. A channel-name mismatch means even if enabled, client subscriptions wouldn't authorize.

## 2. Business Goal & Rules
- **Intended goal:** Live notifications + chat + Agora video/voice end-session signaling over WebSockets.
- **Actual rules today:**
  - Backend broadcast driver = `null` (env not set) → nothing broadcasts to a real socket.
  - Only `RealTimeMessage` event is dispatched (car check-in), on a **public** `events-{user_id}` channel.
  - Client subscribes to nothing (all commented).

## 3. User Flow
- **Intended:** connect (wss + token) → auth private channels → receive notification/chat/agora events → update UI / show local notification.
- **Actual:** none — no live connection on client; backend emits to a null driver.

## 4. Screens / Views
- None. No chat screen, no live-call screen wired. (Notifications list also absent per [[17_Notifications]].)

## 5. Widgets
- None active.

## 6. Cubits / Blocs
- None for real-time. (`NotificationsCubit` referenced by the commented `notification_manager` but not fed by sockets.)

## 7. State Flow
- N/A (disabled). Intended: stream controllers (notificationStream/chatMessageStream/agoraEndSessionStream) → listeners → cubits.

## 8. Models
- Intended `NotificationModel` on `notificationStream`. Chat/Agora payloads as `Map<String,dynamic>` / `void`. No active model wiring.

## 9. Repositories
- None. Would use `/api/broadcasting/auth` for private-channel auth (Bearer + channel_name + socket_id).

## 10. Services / Helpers
| Component | File:Lines | State |
|-----------|-----------|-------|
| ReverbWebSocketService | `core/services/reverb_websocket_service.dart:1–478` | **100% commented**; wss+token, 3 streams, channels `notification.patient.$userId`/`notification.parentinfo.$userId`/`private-conversation.$channel`, `/api/broadcasting/auth`, events `notification.event`/`ConversationMessageSent`/`AgoraEndSession`, reconnect backoff 2–30s, `_authorizeChannel()` 196–237 |
| notification_manager | `core/services/notification_manager.dart:1–197` | **100% commented**; subscribes notificationStream+agoraEndSessionStream |
| websocket_config.dart | imported line 7 | **FILE MISSING** (reverbKey/reverbHost undefined) |

## 11. API Endpoints
- `/broadcasting/auth` (Laravel `Broadcast::routes()`, built-in) — for private-channel auth. Client can't use it as-is (Sanctum-web default vs mobile token; no matching channels).

## 12. Request / Response Models
- Intended auth request: `{channel_name, socket_id}` + Bearer. Event payloads: `RealTimeMessage` `{invitation_id, host_id, driver, title, message}`.

## 13. Laravel Routes / Channels
[`routes/channels.php`](../../../hawdaj-api/routes/channels.php) `:16–22`:
- `App.Models.User.{id}` (private, `user->id === $id`)
- `events-{cat}` (public, true)
- `cars.{cat}` (public, true)
- **Missing:** `notification.patient.*`, `notification.parentinfo.*`, `private-conversation.*` (Flutter's expected channels) → **mismatch**.

## 14. Laravel Controllers / Events
- [`RealTimeMessage`](../../../hawdaj-api/app/Events/RealTimeMessage.php) `:9–30`: ShouldBroadcast on `Channel("events-".$user_id)` (PUBLIC), name `RealTimeMessage`. Dispatched in `CarRequestDetailsController.php:99` (car check-in).
- [`CarEvent`](../../../hawdaj-api/app/Events/CarEvent.php) `:9–31`: ShouldBroadcast on `cars.{site_id}` — **NEVER dispatched** (dead).
- `Notification` model: `scopePrimary()` (auth-scoped); **no ShouldBroadcast** (DB-only).

## 15. Laravel Services / Config
- [`config/broadcasting.php`](../../../hawdaj-api/config/broadcasting.php): `default => env('BROADCAST_DRIVER','null')` (18) → **null**. Pusher config present (32–44, local 127.0.0.1:6001) but no `.env` sets it.
- `BroadcastServiceProvider` registered (config/app.php); `Broadcast::routes()`.
- No `config/reverb.php` (despite Flutter naming "Reverb").

## 16. Laravel Models
- Notification (DB), User (private channel by id). No broadcast models.

## 17. Database Tables & Relationships
- `notifications` table (DB-only, see [[17_Notifications]]). No socket persistence. Broadcasting is stateless.

## 18. Validation
- N/A. Channel auth would be closure-based in channels.php.

## 19. Auth & Permissions
- Private `App.Models.User.{id}`: owner-only ✅. Public `events-*`/`cars.*`: **anyone** (broadcast payload not user-scoped despite embedding user_id in channel name).
- Mobile token auth for `/broadcasting/auth` not wired for the client's expected channels.

## 20. Error Handling
- Client (commented): reconnect backoff 2–30s intended. Backend: null driver silently no-ops.

## 21. Edge Cases
- Even if client enabled: subscribes to `notification.patient.*` which backend doesn't define → auth fails.
- `RealTimeMessage` on public channel → any client could read another user's car-checkin payload (if driver enabled).
- Missing `websocket_config.dart` → client wouldn't compile if uncommented as-is.
- Agora streams with no Agora SDK/backend → dead.

## 22. Dependencies
- **Flutter (declared):** none for realtime (no laravel_echo/pusher/web_socket_channel/agora). **Missing.**
- **Laravel:** pusher-php-server (config present), Broadcasting.
- **Cross-feature:** [[17_Notifications]] (intended live notifications; currently FCM path also disabled), [[19_SplashOnboarding]]/[[20_ForceUpdate]] (Firebase), CarModel module ([[28_CarModule]] — RealTimeMessage source).

## 23. Files Involved
**Flutter:** reverb_websocket_service.dart (commented), notification_manager.dart (commented), missing websocket_config.dart, pubspec.yaml (no realtime deps), service_locator (no registration).
**Laravel:** config/broadcasting.php, routes/channels.php, BroadcastServiceProvider, Events/RealTimeMessage.php, Events/CarEvent.php (dead), CarRequestDetailsController.php:99, Notification model.

## 24. Full Execution Flow (intended vs actual)
- **Intended:** Laravel event `ShouldBroadcast` → driver (pusher/reverb) → client Echo/WS subscribed to private channel (auth via /broadcasting/auth) → stream → UI.
- **Actual:** `RealTimeMessage` dispatched on car check-in → **null driver → discarded**. Client not connected. No end-to-end path.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| N/A (inactive). If enabled: reconnect backoff already designed | 🟢 | reverb service (commented) |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| `RealTimeMessage` on PUBLIC channel with per-user payload (if enabled) | 🟡 Med | RealTimeMessage.php |
| Channel mismatch → client auth would fail (fail-safe by accident) | 🟢 Low | channels.php vs Flutter |
| No presence-channel security concerns (none defined) | 🟢 OK | — |
| Currently inert → no live attack surface | 🟢 OK | driver=null |

## 27. Technical Debt
- Large commented WebSocket service + manager (dead weight, drift risk).
- Missing `websocket_config.dart` (broken import if uncommented).
- Channel-name mismatch client↔server.
- `CarEvent` defined but never dispatched.
- Agora referenced with no SDK/backend.
- Notifications not broadcast (DB-only) despite intent.
- "Reverb" naming but Pusher config (no reverb.php).

## 28. Improvement Opportunities
- Decide: implement (add deps, config, matching channels, private notification channels, mobile token auth) OR delete the dead scaffolding.
- If implementing: set driver (pusher/reverb), define `notification.{user}` private channels with token auth, make Notification `ShouldBroadcast`, scope car payloads to private channels, add Agora SDK + token endpoint if video/voice truly needed.
- Remove dead `CarEvent`; add `websocket_config.dart` or remove references.

---

## 32. Related Features
- **[[17_Notifications]]** — intended live-notification transport (FCM path also disabled).
- **[[28_CarModule]]** — only live event source (`RealTimeMessage` on car check-in).
- **[[24_Networking]]** — token/auth reused for /broadcasting/auth.
- **[[01_Authentication]]** — private channel identity.

## 33. How to Modify This Feature
- **Activate backend:** set `BROADCAST_DRIVER=pusher` (+ keys) or install Reverb (+config/reverb.php); define matching private channels in channels.php with mobile token auth.
- **Activate client:** add laravel_echo + pusher/web_socket_channel deps; create `websocket_config.dart`; uncomment services; register in get_it; align channel names.
- **Broadcast notifications:** add `ShouldBroadcast` to notification event on a private per-user channel.
- **Remove if unwanted:** delete reverb_websocket_service, notification_manager, CarEvent.

## 34. Regression Checklist (if activated)
- [ ] Client connects (wss + token).
- [ ] Private channel auth succeeds via /broadcasting/auth.
- [ ] Channel names match server definitions.
- [ ] Event received → stream → UI update.
- [ ] Reconnect on drop.
- [ ] Per-user payloads on private (not public) channels.

## 35. Common Bugs (current)
- Nothing real-time works (by design: disabled both sides).
- If half-enabled: compile error (missing websocket_config) or auth failure (channel mismatch).
- Car-checkin broadcasts vanish (null driver).

## 36. Debug Guide
- **No live updates:** expected — driver=null + client commented.
- **Enabling fails:** check BROADCAST_DRIVER, deps present, websocket_config exists, channel names match, /broadcasting/auth reachable with token.
- **Car event not received:** driver null; and channel is public `events-{id}`.

## 37. Search Keywords
`reverb`, `Reverb`, `pusher`, `Pusher`, `websocket`, `WebSocket`, `reverb_websocket_service`, `notificationStream`, `chatMessageStream`, `agoraEndSessionStream`, `broadcasting.php`, `BROADCAST_DRIVER null`, `channels.php`, `ShouldBroadcast`, `RealTimeMessage`, `CarEvent`, `/broadcasting/auth`, `Agora`, `websocket_config`, الوقت الحقيقي, بث مباشر.

## 38. Future Improvements
- Full real-time stack (Reverb + Echo) with private per-user notification/chat channels + token auth; make Notifications broadcastable.
- Agora video/voice with server token generation (if the telehealth-style `patient/parentinfo` channels indicate a planned care/chat product).
- Or remove all dead scaffolding to reduce confusion.
