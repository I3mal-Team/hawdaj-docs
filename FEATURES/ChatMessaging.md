# Feature — Chat / Messaging (محادثة User ↔ Tour Guide)

> **Status:** 🟡 PLANNED / DESIGN (0% مبني) · **Priority:** P2 · **Category:** Core (تواصل)
> **Repos:** Laravel `/Users/mac/hawdaj-api` (backend-first) · Flutter `/Users/mac/hawdaj/Untitled` (`lib/features/chat`) · Web لاحقًا (parity اختياري)
> **Branch:** `feature/chat-messaging` (docs + api) · **Env:** test فقط — إنتاج read-only، لا prod creds، لا migrations إنتاج.
> **Last updated:** 2026-07-08

> 🔗 يربط [[11_TourGuides]] (المدخل: زر مراسلة في تفاصيل الدليل) + يبني على سقالة [[25_Realtime]] المعطّلة (chatMessageStream / private-conversation / ConversationMessageSent). المسار: **Phase 1 = REST + Polling** ثم **Phase 2 = Realtime (يشيل Polling كله)**.

---

## 1. Feature Name & Identity
- **Name:** Chat / Messaging — محادثة نصّية 1:1 بين المستخدم والمرشد السياحي داخل التطبيق.
- **Approach:** REST + Polling أولًا (Option B)؛ لا WebSocket في Phase 1. Realtime لاحقًا يستبدل Polling.

## 2. Business Goal & Rules
- **الهدف:** تمكين المستخدم من مراسلة المرشد (استفسار/تنسيق رحلة) داخل التطبيق بدل قنوات خارجية.
- **القواعد:**
  - محادثة 1:1 فقط بين `user` و`guide` (عبر `guide.user_id`).
  - **محادثة واحدة فريدة لكل زوج** (`unique(user_id, guide_id)`) — إعادة الفتح تُرجع نفس المحادثة.
  - المشاركان فقط يقرآن/يكتبان (authz صارم).
  - طول الرسالة محدود (validate) + throttle على الإرسال (منع spam).
  - لا مدفوعات، لا مرفقات في Phase 1 (نص فقط) — التوسّع لاحقًا.

## 3. Polling Policy (السياسة المعتمَدة) ⭐
| السياق | السلوك |
|---|---|
| **داخل شاشة المحادثة (Chat Thread)** | Polling كل **2 ثانية** — `GET messages?since=<last_ts>` |
| **الخروج من شاشة المحادثة** | **إيقاف Polling فورًا** (cancel Timer في `close()`/dispose) |
| **قائمة المحادثات (Chat List)** | **لا Polling** — Refresh فقط عند فتح الشاشة أو الرجوع إليها (`didPopNext`/`onResume`) |
| **Phase 2 (Realtime)** | **يُحذف Polling بالكامل** ويُستبدل بـWebSocket streams |

- Timer داخل `ChatThreadCubit` فقط، يعمل عند دخول الثريد، يتوقّف عند `close()`.
- `since` = طابع آخر رسالة مستلمة → تقليل الحمل (delta فقط، لا إعادة تحميل كامل).
- Chat List يعتمد lifecycle: refresh عند `initState` + عند الرجوع من الثريد.

## 4. User Flow
1. تفاصيل الدليل ([[11_TourGuides]] `tour_guide_details_view`) → زر **مراسلة**.
2. `POST conversations {guide_id}` → إنشاء/إرجاع المحادثة → فتح `ChatThreadView`.
3. داخل الثريد: تحميل أولي `GET messages` → بدء Polling 2s → إرسال `POST messages` → تعليم مقروء `POST read`.
4. الخروج → إيقاف Polling.
5. `ConversationsView` (القائمة) → refresh عند الفتح/الرجوع → دخول أي محادثة.

## 5. Data Model (DB — test فقط)
**`conversations`:** `id`, `user_id` (FK users), `guide_id` (FK guides), `last_message_at` (nullable), `timestamps`, `softDeletes` · **unique(user_id, guide_id)**.
**`messages`:** `id`, `conversation_id` (FK), `sender_id` (FK users), `is_from_guide` (bool), `body` (text), `read_at` (nullable), `timestamps`, `softDeletes` · **index(conversation_id, created_at)**.

## 6. Models (Laravel)
- `Conversation` — belongsTo User, belongsTo Guide, hasMany Message, `scopeForParticipant`.
- `Message` — belongsTo Conversation, belongsTo sender(User).

## 7. API Endpoints (العقد — `auth('api')` · envelope `{code,message,data}`)
| Method | Endpoint | دور |
|---|---|---|
| GET | `conversations` | قائمة محادثاتي + آخر رسالة + عدّاد غير مقروء |
| POST | `conversations` | ابدأ/أرجِع محادثة مع `{guide_id}` |
| GET | `conversations/{id}/messages?since=<ts>` | رسائل بعد الطابع (Polling delta) |
| POST | `conversations/{id}/messages` | إرسال `{body}` (validate طول) |
| POST | `conversations/{id}/read` | تعليم مقروء |

- كل مسار: تحقّق أن الطالب مشارك في المحادثة (user_id أو guide.user_id) → غير ذلك 403.
- throttle middleware على `POST messages`.

## 8. Flutter Architecture
- **مجلد:** `lib/features/chat/{data,presentation}` (feature-first).
- **Repo:** `chat_repo` + `chat_repo_imp` (`DioConsumer`, `Either<Failure,T>`, endpoints في `end_points.dart`).
- **Cubits:**
  - `ConversationsCubit` — list + refresh (لا Timer).
  - `ChatThreadCubit` — load + `Timer.periodic(2s)` poll + send + read + **cancel في `close()`**.
- **Views:** `ConversationsView` · `ChatThreadView` (+ widgets: message bubble، input bar).
- **مدخل:** زر مراسلة في `tour_guide_details_view`.
- **تعريب:** مفاتيح ar/en/ru/zh + `EdgeInsetsDirectional` + RTL.

## 9. Realtime Migration Path (Phase 2)
- إحياء [[25_Realtime]]: driver=pusher/reverb، إضافة `websocket_config.dart` (مفقود)، deps (`laravel_echo`/`pusher`/`web_socket_channel`).
- Backend: `ConversationMessageSent` كـ`ShouldBroadcast` على `private-conversation.{id}` + auth بـtoken في `channels.php`.
- Client: subscribe `chatMessageStream` → **حذف كل Timer/Polling**.
- العقد REST يبقى (history/send) — WebSocket للدفع اللحظي فقط.

## 10. Security Considerations
1. 🟠 **401 مُتجاهَل حاليًا على الموبايل** ([[24_Networking]]) — Polling يحتاج token سليم؛ سأعالج 401 محليًا في flow المحادثة (إيقاف + إعادة توجيه).
2. authz صارم: المشاركان فقط (403 لغيرهما) — منع قراءة محادثات الآخرين.
3. throttle على الإرسال (منع spam/flood).
4. validate طول body + sanitize (منع payload ضخم).
5. لا تسريب PII في قائمة المحادثات (اسم/صورة فقط، لا email).

## 11. Testing (test env فقط)
- حسابان على test API + test DB (مستخدم + مالك دليل).
- تدفّق: إنشاء محادثة → إرسال → استلام عبر Polling 2s → مقروء → إيقاف Polling عند الخروج → refresh القائمة عند الرجوع.
- authz: طرف ثالث يحاول الوصول → 403.
- throttle: إرسال متتابع → 429.

## 12. Rollback
- حذف الفرع `feature/chat-messaging`، أو `migrate:rollback` على **test DB فقط**.
- لا أثر على إنتاج (read-only).

## 13. Impact / Affected Features
[[11_TourGuides]] (مدخل + guide.user_id) · [[25_Realtime]] (سقالة Phase 2) · [[17_Notifications]] (إشعار رسالة جديدة — لاحقًا، FCM معطّل موبايل) · [[24_Networking]] (401) · [[27_RolesPermissions]] (authz المشاركين).

## 14. Open Items (قبل التنفيذ)
- [ ] فرع الموبايل: من `main` أو `add_ai`؟ (لم يُحسم — الحالي `add_ai`)
- [ ] إشعار رسالة جديدة خارج الشاشة: Phase 1 (poll list عند فتح) أم انتظار Realtime؟
- [ ] parity الويب: الآن أم بعد استقرار الموبايل؟
- [ ] هل "التصميم على الويب" منفّذ فعلًا أم mockup؟ (لمطابقة العقد)

---

## Changelog (هذا المستند)
- **2026-07-08** — إنشاء المستند (DESIGN/PLAN). Option B معتمَد: REST + Polling 2s داخل الثريد، إيقاف عند الخروج، لا Polling في القائمة (refresh عند الفتح/الرجوع)، Realtime لاحقًا يشيل Polling. فرع `feature/chat-messaging`. لا كود بعد.
