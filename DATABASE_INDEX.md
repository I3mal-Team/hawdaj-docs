# Hawdaj — Database Index

> MySQL (utf8mb4). 50+ نموذج / 100+ جدول. لكل جدول: نموذج Laravel + الميزات المستخدِمة.
> ✅ = مغطّى بتحليل feature مكتملة. الترجمة عبر Astrotomic (جداول `*_translations`)، الوسائط عبر Spatie (`media`)، الأدوار عبر Spatie.

---

## Identity & Profile (Feature 01 ✅)
| Table | Model | Features |
|---|---|---|
| `users` | `User` | 01 ✅ + كل ميزة محمية (03,11,12,13,14,15,16,17) |
| `user_socials` | `UserSocial` | 01 ✅ (profile) |
| `user_locations` | `UserLocation` | 01 ✅, 08 Exploration |
| `personal_access_tokens` | (Sanctum) | 01 ✅ + كل مصادقة |
| `roles`, `permissions`, `model_has_roles`, `model_has_permissions`, `role_has_permissions` | Spatie `Role`/`Permission` | 27 Roles/Perms, Dashboard |

## Content — Places / Stores / Zad
| Table | Model | Features |
|---|---|---|
| `places` + `place_translations` | `Place` / `PlaceTranslation` | 04, 03 Trips, 07, 08, 02 |
| `stores` + `store_translations` | `Store` / `StoreTranslation` | 05, 07, 08, 02 |
| `zad_elgadels` + `zad_elgadels_translations` | `ZadElgadel` | 06, 07 |
| `menus` + `menu_translations` | `Menu` | 06 |
| `offers` + `offer_translations` | `Offer` | 06 |
| `food_category_of_zads`, `category_of_zads` | `FoodCategoryOfZad`/`CategoryOfZad` | 06 |
| `food_category_zad_elgadels` (pivot) | — | 06 |
| `category_of_stores` + translations | `CategoryOfStore` | 05 |

## Content — Events / Swalef / Stories / Guides / Landmarks
| Table | Model | Features |
|---|---|---|
| `events` + `event_translations` | `Event` | 09, 07, 08 |
| `swalefs` + `swalef_translations` | `Swalef` | 10, 07 |
| `category_of_swalefs` | `CategoryOfSwalef` | 10 |
| `stories` | `Story` | 16 User Stories, 10, profile page |
| `guides` + `guide_translations` | `Guide` | 11, 07, profile page |
| `landmarks` | `LandMark` | 12 |
| `applications` + `category_of_applications`(+trans) | `Application`/`CategoryOfApplication` | 07 Tasneef |

## Trips (Feature 03)
| Table | Model | Features |
|---|---|---|
| `trips` | `Trip` | 03, profile page |
| `prepare_trips` | `PrepareTrip` | 03 (legacy v1) |

## Engagement — Favorites / Saved / Rates
| Table | Model | Features |
|---|---|---|
| `favorites` (polymorphic `favoritable_*`) | `Favorite` | 13 + محتوى L2 (Place/Store/Event/Swalef/Zad/Guide) |
| `saves` (`saved_id`/`saved_type`) | `Save` | 13 Saved |
| `rates` (pseudo-morph `type`+`parent_id`) | `Rate` | 14 + عرض التقييم في محتوى L2 |

## Taxonomy / Geo (Feature 18 — shared)
| Table | Model | Features |
|---|---|---|
| `regions` + `region_translations` | `Region` | 03,04,05,06,08,09,11,15,18 |
| `cities` + `city_translations` | `City` | نفس ما سبق |
| `prices` + `price_translations` | `Price` | 03 Trips, 04 Places |
| `categories` + `category_translations` | `Category` | 02,03,04,07 |
| `languages` (+trans) | `Language` | 11 Guides, 18 |
| `features` (+trans) | `Feature` | محتوى L2 |
| `most_visits` | `MostVisit` | 08, analytics |

## Media / Gallery / Sliders (shared)
| Table | Model | Features |
|---|---|---|
| `media` (Spatie morph) | — | 01,04,05,06,09,10,11,12 + Sliders |
| `galleries` (`parent_id`+`type`) | `Gallery` | Places/Stores/Events/Swalefs/Guides/Zad |
| `sliders` + `slider_translations` | `Slider` | 02 Home |
| `ceos` (`parent_id`+`type`) | `Ceo` | معلومات تواصل للكيانات |

## Admin / Internal Modules (backend-only)
| Table | Model | Features |
|---|---|---|
| `visits`,`visit_requests`,`visit_activities`,`visitors` | Visit* | 30 Visitor/Guard |
| `materials`,`material_requests`,`material_activities` | Material* | 30 |
| `contracts`,`contract_requests` | Contract* | 26 Dashboard |
| Cars/Plates/Days/Settings (Modules/CarModel) | `Car`,`CarPlate`,`CarDay`,`CarSetting`,`CarRequest`,`Driver` | 28 CarModel |
| `draft_reports`,`archive_files` (Modules/Report) | `DraftReport`,`ArchiveFile` | 29 Report |
| `suggest_places` | `SuggestPlace` | user suggestions |
| `failed_jobs`, `jobs`, `notifications` | (Laravel) | 17 Notifications, queues |

---
## Cross-cutting patterns
- **Soft Deletes** (`deleted_at`): users, places, stores, events, zad_elgadels, swalefs, categories, regions, cities, trips, menus, offers, sliders + المزيد.
- **Translatable** (`*_translations`, locales ar/en/ru/zh): 18+ نموذج.
- **Polymorphic حقيقي:** `favorites`. **Pseudo-morph:** `rates`, `galleries`, `ceos`, `most_visits` (`type`+`parent_id`، بلا قيود FK).
- **Geo:** `lat`/`long` على places/stores/events/zad_elgadels/user_locations/most_visits.

_آخر تحديث: 2026-07-05. الجداول غير المؤكّدة 100% تُثبَّت عند تحليل ميزتها._
