# Hawdaj — Feature Graph

> خريطة العلاقات والتبعيات بين الميزات. يُحدَّث كلما اكتمل تحليل feature جديدة.
> Status: ✅ = محلّلة 100% · ⬜ = لم تُحلّل بعد. حاليًا المحلّلة: 01 Authentication.

---

## 1. Dependency Layers (طبقات الاعتماد)

```
┌─────────────────────────────────────────────────────────────┐
│ L0 — SHARED INFRASTRUCTURE (يعتمد عليها الجميع)               │
│   Networking/Interceptors · Env switching · Localization/RTL │
│   Media/Gallery/Uploads · Taxonomy (Regions/Cities/Lang/Cat) │
│   Notifications(FCM) · Real-time(Pusher) · Auto-Translation   │
└─────────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────────┐
│ L1 — IDENTITY                                                │
│   01 Authentication & Profile ✅  (توكن + user_id + FCM)      │
└─────────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────────┐
│ L2 — CONTENT (قراءة عامة، كتابة تحتاج L1)                     │
│   04 Places · 05 Stores · 06 Restaurants(Zad) · 09 Events    │
│   10 Swalef/Stories · 11 Tour Guides · 12 Landmarks          │
│   02 Home/Sliders · 07 Tasneef · 08 Exploration/Search       │
└─────────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────────┐
│ L3 — USER ENGAGEMENT (تحتاج L1 + L2)                          │
│   03 Trips · 13 Favorites/Saved · 14 Rates · 15 My Properties│
│   16 User Stories                                            │
└─────────────────────────────────────────────────────────────┘

ADMIN (منفصل، web guard): 26 Dashboard · 27 Roles/Perms · 28 CarModel · 29 Report
```

---

## 2. Feature → Depends On (تبعيات كل ميزة)

| Feature | يعتمد على | نوع |
|---|---|---|
| 01 Authentication ✅ | Networking, Storage, Localization, Media (photo), FCM | إلزامي |
| 02 Home/Sliders | Networking, Taxonomy, Media, (Auth اختياري لحالة المفضّل) | جزئي |
| 03 Trips | **01 Auth**, Taxonomy (Regions/Prices/Categories), Places, Maps API | إلزامي |
| 04 Places | Taxonomy, Media, Rates, Favorites | جزئي |
| 05 Stores | Taxonomy, Media, Rates, Favorites | جزئي |
| 06 Restaurants(Zad) | Taxonomy, Media, Menus/Offers, Rates, Favorites | جزئي |
| 07 Tasneef | 04/05/06/09/10/11 (واجهة قوائم موحّدة), Taxonomy | تجميعي |
| 08 Exploration/Search | 04/05/06/09/10/11, Location, Maps | تجميعي |
| 09 Events | Taxonomy, Media, Rates, Favorites | جزئي |
| 10 Swalef/Stories | Media, Gallery, Rates, Favorites | جزئي |
| 11 Tour Guides | **01 Auth** (add/edit), Taxonomy (Regions/Languages), Media, Rates | جزئي |
| 12 Landmarks | **01 Auth** (add/my), Media | جزئي |
| 13 Favorites/Saved | **01 Auth**, محتوى L2 (polymorphic) | إلزامي |
| 14 Rates | **01 Auth**, محتوى L2 (pseudo-morph type+parent_id) | إلزامي |
| 15 My Properties | **01 Auth**, Taxonomy, Media | إلزامي |
| 16 User Stories | **01 Auth**, Media/File | إلزامي |
| 17 Notifications(FCM) | **01 Auth** (fcm_token), Real-time | إلزامي |
| 18 Taxonomy | — (أساس) | أساس |
| 22 Localization | — (أساس) | أساس |
| 23 Media/Uploads | — (أساس) | أساس |

---

## 3. Shared Business Logic (منطق مشترك)

| المنطق المشترك | أين | يخدم |
|---|---|---|
| Sanctum token + `Authorization: Bearer` | `TokenInterceptor` · `HasApiTokens` | كل الميزات المحمية |
| `UserModel` / `UserResource` (شكل المستخدم) | auth + core | أي ميزة تعرض/تربط مستخدمًا |
| Envelope الرد `{code,message,data}` | `ApiModalController` · Cubits (`code==200`) | **كل** endpoint |
| Favorites polymorphic (`favoritable_*`) | `Favorite` model | Place/Store/Event/Swalef/Zad/Guide |
| Rates pseudo-morph (`type`+`parent_id`) | `Rate` model | نفس الكيانات أعلاه |
| Gallery عام (`parent_id`+`type`) | `Gallery` model | Places/Stores/Events/Swalefs/Guides/Zad |
| Astrotomic Translatable | 18+ نموذج + Observers | كل محتوى مترجَم |
| Spatie MediaLibrary (أحجام small/medium) | `UploadService` · InteractsWithMedia | User/Place/Store/Event/Zad/Guide/Landmark/Slider |
| Distance sorting (Haversine) | Place/Store controllers · `applyDistanceSorting` | Places/Stores/Exploration/Home |
| Taxonomy filters (region/city/category/lang) | IndexRequests | Tasneef/Places/Stores/Zad/Guides |
| Pagination (`paginate(per_page)`) | Controllers + `infinite_scroll_pagination` | كل القوائم |

---

## 4. Impact Matrix (ماذا يتأثر عند تغيير X)

| إذا غيّرت… | يتأثر مباشرة |
|---|---|
| **01 Authentication** (توكن/UserModel/envelope) | **كل الميزات المحمية**: 03,11,12,13,14,15,16,17 + حالة مفضّل في 02,04,05,06,09,10 |
| Envelope الرد `{code,message,data}` | **كل** الـ Cubits في كل الميزات |
| `UserModel`/`UserResource` | auth, profile, trips(my), guides, stories, favorites, أي شاشة تعرض المستخدم |
| `TokenInterceptor`/شكل الترويسة | كل الطلبات المحمية |
| Taxonomy (Regions/Cities/Categories) | 02,03,04,05,06,07,08,09,11,15 |
| Media/Upload logic | 01,04,05,06,09,10,11,12 (+Sliders) |
| Favorites model/polymorphic | 13 + أزرار المفضّل في كل محتوى L2 |
| Rates model | 14 + عرض التقييم في كل محتوى L2 |
| Localization/interceptor لغة | كل الميزات (ترويسة `accept-language`) |
| FCM token/notifications | 17 + أي حملة استهداف مستخدم |

---

## 5. Cross-Repo Contract Points (نقاط العقد بين الطرفين)
تغيير أيٍّ منها يتطلب تعديلًا متزامنًا في **الطرفين**:
- مسارات `end_points.dart` (Flutter) ↔ `routes/api.php` (Laravel).
- مفاتيح JSON: `*RequestModel`/`*ResponseModel` (Flutter) ↔ `*Resource`/Validator (Laravel).
- envelope `{code,message,data}`.
- شكل الترويسات: `Authorization`, `accept-language`, `accept-tokenapi`.

---
_آخر تحديث: 2026-07-05 — بعد اكتمال Feature 01. سيُوسَّع الجدول كلما اكتملت ميزات أخرى._
