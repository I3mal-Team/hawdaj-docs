# Hawdaj — Search Index

> فهرس بحث: كلمة مفتاحية → الميزة/الملف. ✅ = موثّقة بالكامل. يُوسَّع مع كل feature جديدة.
> استخدمه للانتقال السريع: ابحث عن الكلمة ثم افتح ملف الميزة.

---

## By Feature quick-map
| # | Feature | Doc | Status |
|---|---|---|---|
| 01 | Authentication & Profile | `FEATURES/01_Authentication.md` | ✅ |
| 02 | Home & Sliders | — | ⬜ |
| 03 | Trips (v1+v2) | — | ⬜ |
| 04 | Places | — | ⬜ |
| 05 | Stores | — | ⬜ |
| 06 | Restaurants (Zad) | — | ⬜ |
| 07 | Tasneef | — | ⬜ |
| 08 | Exploration/Search | — | ⬜ |
| 09 | Events | — | ⬜ |
| 10 | Swalef/Stories | — | ⬜ |
| 11 | Tour Guides | — | ⬜ |
| 12 | Landmarks | — | ⬜ |
| 13 | Favorites/Saved | — | ⬜ |
| 14 | Rates | — | ⬜ |
| 15 | My Properties | — | ⬜ |
| 16 | User Stories | — | ⬜ |
| 17 | Notifications | — | ⬜ |
| 18 | Taxonomy | — | ⬜ |

---

## Keyword → Feature

### Authentication (01 ✅)
`login`, `register`, `logout`, `sign in`, `sign up`, `auth`, `sanctum`, `bearer`, `token`, `createToken`, `personal_access_tokens`, `social login`, `google sign in`, `apple sign in`, `twitter`, `provider_id`, `social/callback`, `forgot password`, `verify-email-code`, `reset password`, `forget_password_code`, `otp`, `AuthManager`, `UserInfoCubit`, `TokenInterceptor`, `StorageService`, `auth_token`, `user_data`, `FlutterSecureStorage`, `LoginCubit`, `RegisterCubit`, `SocialAuthCubit`, `SocialAuthService`, `AuthenticationController`, `SocialController`, `ProfileController`, `UserResource`, `update-profile`, `update-photo`, `update-password`, `update-fcm-token`, `update-location`, `delete-profile`, `$guarded`, `HasApiTokens`, `MustVerifyEmail` → **01_Authentication.md**

### Profile / User (01 ✅)
`profile`, `edit profile`, `user model`, `full_name`, `total_points`, `fcm_token`, `user_socials`, `user_locations`, `avatar`, `photo`, `gender` → **01_Authentication.md**

### Home (02 ⬜)
`home`, `sliders`, `banners`, `HomeCubit`, `SlidersCubit`, `HomeController`, `show_in_home`, `featured`, `applyDistanceSorting`, `getLocationCoords`, `distance IS NULL`, `home_cache_version`, `homeCacheVersion`, `bumpHomeCacheVersion`, `PlaceObserver`, `update-location`, `updateLocation`, `SafeMailTransport`, `location sorting`, `nearest`, `cache invalidation` → Feature 02

### Trips (03 ⬜)
`trip`, `itinerary`, `prepare`, `reprepare`, `my-trips`, `enhanced trip`, `v2/trips`, `morning/evening`, `TripController`, `EnhancedTripV2Controller`, `EnhancedTripService`, `save-trip-to-email`, `prices`, `days`, `Trip model` → Feature 03

### Places (04 ⬜)
`place`, `places`, `slug`, `PlaceController`, `place_translations`, `places-data-for-map`, `distance sorting`, `haversine`, `lat long` → Feature 04

### Stores (05 ⬜)
`store`, `stores`, `is_online`, `StoreController`, `CategoryOfStore` → Feature 05

### Restaurants / Zad (06 ⬜)
`zad`, `zads`, `restaurant`, `menu`, `menus`, `offer`, `offers`, `food_categories`, `ZadController`, `MenuController`, `OfferController`, `zad_elgadels` → Feature 06

### Tasneef (07 ⬜)
`tasneef`, `classification`, `unified listing`, `applications`, `filters`, `category_id`, `region_id` → Feature 07

### Exploration / Search (08 ⬜)
`search`, `global-search`, `global-map-data`, `map`, `SearchController`, `exploration`, `MapView` → Feature 08

### Events (09 ⬜)
`event`, `events`, `date_from`, `date_to`, `EventController` → Feature 09

### Swalef / Stories (10 ⬜)
`swalef`, `swalefs`, `heritage`, `story`, `stories`, `myLastDayStories`, `SwalefController`, `StoryController`, `شعر`, `قصه` → Feature 10 / 16

### Tour Guides (11 ⬜)
`guide`, `guides`, `tour guide`, `get-guide`, `store-guide`, `update-guide-photo`, `GuideController`, `languages`, `experience` → Feature 11

### Landmarks (12 ⬜)
`landmark`, `landmarks`, `landmark/store`, `myLandMarks`, `LandMarksController` → Feature 12

### Favorites / Saved (13 ⬜)
`favorite`, `favorites`, `get-my-favorites`, `saved`, `get-my-saved`, `bookmark`, `favoritable`, `polymorphic`, `FavoriteController`, `SavedController` → Feature 13

### Rates (14 ⬜)
`rate`, `rates`, `review`, `rating`, `RateController`, `type parent_id` → Feature 14

### My Properties (15 ⬜)
`property`, `properties`, `my-properties`, `PropertiesController`, `ownership_proof_file`, `address_type` → Feature 15

### User Stories (16 ⬜)
`user stories`, `stories/store`, `stories/{id}/delete`, `last day stories`, `total_views`, `total_likes` → Feature 16

### Notifications (17 ⬜)
`notification`, `fcm`, `push`, `firebase messaging`, `MarketingPushDispatcher`, `BroadcastMarketingNewContentJob`, `MarketingNewContentNotification`, `MailNotification` → Feature 17

### Taxonomy (18 ⬜)
`region`, `regions`, `city`, `cities`, `category`, `categories`, `price`, `prices`, `language`, `languages`, `translation`, `astrotomic`, `translatable` → Feature 18

---

## Cross-cutting infra keywords
| Keyword | أين | مرجع |
|---|---|---|
| `dio`, `interceptor`, `baseUrl`, `end_points`, `DioConsumer`, `ApiConsumer` | networking | `AI/API_INDEX.md` |
| `AppEnvironmentManager`, `environment`, `test/prod` | env switching | core/utils |
| `easy_localization`, `RTL`, `accept-language`, `ar/en/ru/zh` | localization | core/locale |
| `get_it`, `service_locator`, `setupServiceLocator` | DI | core/services |
| `go_router`, `StatefulShellRoute`, `RoutesKeys`, `app_router` | routing | core/routing |
| `Either`, `Failure`, `ServerFailure`, `dartz` | error handling | core/errors |
| `spatie media`, `media library`, `UploadService`, `getFirstMedia` | media | Feature 23 |
| `pusher`, `websockets`, `RealTimeMessage`, `broadcast` | real-time | Feature 25 |
| `ApiModalController`, `success()`, `error()`, `{code,message,data}` | response envelope | كل الميزات |
| `monarx-analyzer`, `RCE`, `security`, `CORS`, `vulnerability` | security ⚠️ | Phase-1 report + 01 §29 |

---

## Index maps (روابط الفهارس)
- **علاقات الميزات:** `AI/FEATURE_GRAPH.md`
- **كل الـ endpoints:** `AI/API_INDEX.md`
- **كل الجداول والنماذج:** `AI/DATABASE_INDEX.md`
- **الحالة والأولويات:** `AI/FEATURE_INDEX.md` · `AI/PROJECT_PROGRESS.md`

_آخر تحديث: 2026-07-05._
