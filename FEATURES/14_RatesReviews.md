# Feature 14 — Rates & Reviews (التقييمات والمراجعات)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Supporting (engagement)
> **Flutter:** `lib/features/rates/` · **Laravel:** `RateController` + `Rate` model
> **One-line:** A single flat `rates` table with a string `type` field acts as a pseudo-polymorphic review store for 7 content types (place/store/zad/swalef/event/app/guide). Content models expose `ratings()` (filtered by type), plus `rate`/`review` accessors that recompute average per access (N+1). **The endpoints are completely UNAUTHENTICATED, expose reviewer email (PII), and enforce no 1–5 range or duplicate check.**

---

## 1. Feature Name & Identity
- **Name:** Rates & Reviews. One combined artifact per submission = star rating (`rate`) + written text (`rateText`) + reviewer identity (name/email or user_id).
- **Pseudo-polymorphic:** NOT Laravel `morphTo`. Uses a flat `rates` table with `parent_id` + string `type`. Each content model declares `hasMany(Rate,'parent_id')->where('type', <literal>)`.
- **Shared across:** Places, Stores, Zad/Restaurants, Swalef, Events, Applications(`app`), Tour Guides(`guide`).

## 2. Business Goal & Rules
- **Goal:** Users (or guests) leave a star rating + comment on content; detail pages show reviews + average.
- **Rules (as implemented):**
  - `rateText` required, min 10 chars.
  - `rate` required — **NO min/max** (accepts 0, negative, >5, non-numeric).
  - `type` required, `in:place,store,zad,swalef,event,app,guide`.
  - Guest must send name+email; authenticated user's name/email/user_id override the payload.
  - **No auth middleware** on POST/DELETE.
  - **No duplicate prevention** — same user can rate the same item unlimited times.
  - **No self-rating block** — user can rate own submitted content.

## 3. User Flow
1. On a detail page, user taps "share your rate" (`ShareYourRateButton`) → `RateCubit.init()` prefills name/email if logged in.
2. Bottom sheet: 5-star `RatingBar` (no half stars) → `setRating()`; comment field; (name+email fields if guest).
3. Submit → `RateCubit.submit(type, parentId)` → validates rating≠0 → rounds to int → repo POST `rates`.
4. Success toast, sheet closes, form resets. User can immediately rate again.
5. Reviews render on detail pages from `content.ratings` (name + text + number; email present in model but not shown in UI).

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`share_your_rate_button.dart`](lib/features/rates/presentation/view/share_your_rate_button.dart) | 18–184 | Rate entry button + `_RateSheet` bottom sheet (stars, comment, guest name/email, submit) |
| [`places_details_items_body.dart`](lib/features/places/presentation/view/widgets/places_details_items_body.dart) | 255–299 | Reviews section on Place detail (name, rateText, rate) — reused by Restaurants/TourGuides |

## 5. Widgets
- `_RateSheet` (in share_your_rate_button.dart): `RatingBar` 5 stars (lines ~111–120), comment field (~138–142), guest name/email (~124–135), submit (~146–157), listener toast+pop (~65–73).
- Review card (name + text + rating) reused across Places/Restaurants/TourGuides detail pages.

## 6. Cubits / Blocs
| Cubit | File:Lines | States | Methods |
|-------|-----------|--------|---------|
| RateCubit | `features/rates/.../rate_cubit.dart:1–107` | Initial, Loading, Ready(name,email,rating), Added, Error | `init()` (29–50, load user), `setRating()` (54–60), `submit(type,parentId)` (63–91: validates rating≠0, rounds int, calls repo) |

- Controllers: `nameController`, `emailController`, `rateTextController`. No client-side duplicate check. `RateAdded` allows immediate re-rate.

## 7. State Flow
Button tap → `init()` (fetch user → Ready) → user sets stars/text → `submit()` → Loading → repo POST → Added (toast) or Error. Detail page reads `content.ratings` list directly from the content fetch (no separate reviews-list cubit).

## 8. Models
| Model | File:Lines | Fields |
|-------|-----------|--------|
| RatingModel | `features/tasneef/data/models/rating_model.dart:1–55` | id, name, **email**, rateText, `rate` (**String**, not int), type, parentId, createdAt, updatedAt, userId. Direct fromJson mapping (26–39) |

- **⚠️** `rate` typed as String on client; email carried in model though not rendered.

## 9. Repositories
| Repo | File:Lines | Method | HTTP |
|------|-----------|--------|------|
| RateRepo (abstract) | `features/rates/data/repo/rate_repo.dart:1–13` | `addRate() → Either<Failure,String>` | — |
| RateRepoImp | `features/rates/data/repo/rate_repo_imp.dart:8–36` | POST `EndPoints.addRate` (`'rates'`) query: name, rate, email, rateText, type, parent_id → `data['message']` | POST |

## 10. Services / Helpers
- `DioConsumer` (attaches auth token if logged in). No dedicated rating service.
- Laravel: average computed in model accessors (`getRateAttribute`, `getReviewAttribute`) — no helper/service.

## 11. API Endpoints
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `rates` | **NONE** | create rating (guest or user) |
| DELETE | `rates/{id}` | **NONE** | delete rating (only if user_id matches) |

`routes/api.php:147–148` — both in unprotected group.

## 12. Request / Response Models
- **POST req:** `{name, email, rateText, rate, parent_id, type}`.
- **POST resp:** `{message:'rating_added_successfully'}`.
- **Ratings in content resource** (e.g. PlaceResource:74) — raw array:
```json
"ratings":[{"id":1,"name":"Ahmed","email":"ahmed@example.com","rateText":"...","rate":"5","type":"place","parent_id":123,"user_id":42,"created_at":"...","updated_at":"..."}]
```
- Content also exposes computed `rate` (avg) + `review` (count).

## 13. Laravel Routes
`routes/api.php:147–148`:
```
POST   rates       → RateController@store   (no auth)
DELETE rates/{id}  → RateController@delete  (no auth)
```

## 14. Laravel Controller
[`RateController.php`](../../../hawdaj-api/app/Http/Controllers/Api/RateController.php) `:1–51`.
- `store()` 17–42: validate (22–29: `rateText` required|min:10, `rate` required [no range], `parent_id` required, `type` in:place,store,zad,swalef,event,app,guide, email/name conditional for guest); if authenticated override user_id/name/email (35–38); `Rate::create($data)` (41); no dup check; return `{message}`.
- `delete()` 45–49: deletes only where `user_id` matches — but no auth, so guest (null user_id) path is meaningless/blocked.

## 15. Laravel Services
None. Average logic lives in content-model accessors.

## 16. Laravel Models
[`Rate.php`](../../../hawdaj-api/app/Models/Rate.php) `:1–22`: `$guarded=[]` (all fillable), `$appends=['created_at']` (custom format). No relationships, scopes, or validation.
- Content relations (pseudo-morph): `Place::ratings()` 132–134 `hasMany(Rate,'parent_id')->where('type','place')`; `Store::ratings()` 90–92 type `store`; `ZadElgadel::ratings()` 90 type `zad`; similar for swalef/event/guide.
- **Average accessors:** `Place` 142–149 uses `$this->ratings` (loaded collection) `sum/count`; `Store` 95–102 uses `$this->ratings()` (**query** — re-queries every access → worse N+1).

## 17. Database Tables & Relationships
- [`rates`](../../../hawdaj-api/database/migrations/2022_10_06_130715_create_rates_table.php) `:14–26`: id, name, email, rateText, rate, type (all string), parent_id (unsignedBigInt), timestamps. **No FK, no soft-delete, no index on parent_id/type.**
- [add user_id](../../../hawdaj-api/database/migrations/2023_01_11_213437_add_user_id_to_rates_table.php) `:17`: nullable `user_id`, **no FK**.
- Relationship: content `hasMany` rates via `parent_id` + type literal. `user_id` links to users only by convention.

## 18. Validation
Inline in `store()`: `rateText` required|min:10; `rate` required (**no numeric|min:1|max:5**); `parent_id` required (no exists check); `type` in whitelist; guest name/email conditional. No FormRequest.

## 19. Auth & Permissions
- **CRITICAL:** POST + DELETE both public (no `auth:api`). Anyone can create ratings; identity is whatever payload/guest sends unless a token happens to be attached.
- Delete guarded only by `user_id` match — but with no auth there's no reliable authenticated user_id.

## 20. Error Handling
- Flutter `Either<Failure,String>`; cubit Error state → toast.
- Laravel: validation → 422; success message envelope.
- **Silent failure risk:** `rate` non-numeric → accessor arithmetic (`sum('rate')`) misbehaves.

## 21. Edge Cases
- `rate` = 0 / negative / >5 / "abc" accepted by API (only client blocks rating==0).
- Same user rates same item repeatedly → inflates count + skews average.
- Self-rating own content allowed.
- Rating soft-deleted/inactive content allowed (no parent active check).
- Guest email spoofable (any string).
- `rate` stored as string → arithmetic + sort inconsistencies.

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dartz, dio, `flutter_rating_bar`, RatingModel (lives under tasneef).
- **Laravel:** Eloquent, `ApiModalController`.
- **Cross-feature:** feeds `rate`/`review`/`ratings` into [[04_Places]], [[05_Stores]], [[06_Restaurants]], [[09_Events]], [[10_Swalef]], [[11_TourGuides]], and `applications` (Tasneef). Referenced in Favorite/Save/Search resources (avg).

## 23. Files Involved
**Flutter:** rate_cubit(+state), rate_repo(+imp), share_your_rate_button, RatingModel (tasneef), review display in places_details_items_body (+restaurants/tourguides detail), service_locator:76–77, end_points(addRate).
**Laravel:** Rate model, RateController, routes:147–148, ratings() relations + rate/review accessors on Place/Store/Zad/Swalef/Event/Guide, PlaceResource:74 (+ListResource, StoreResource:61, FavoriteResource:29, SaveResource:29, SearchResource:55), 2 migrations.

## 24. Full Execution Flow (UI → DB → UI)
**Submit:** `ShareYourRateButton` → `RateCubit.submit('place',123)` → validate rating≠0 → round → `RateRepoImp.addRate()` POST `rates?name&rate&email&rateText&type=place&parent_id=123` → `RateController@store` (no auth) → validate → maybe override user fields → `Rate::create` → `rates` row → `{message}` → cubit Added → toast.
**Display:** content fetch (e.g. `GET /places/{slug}`) → `PlaceResource` → `ratings` raw array + `rate` (avg via accessor: `ratings->sum/count`) + `review` (count) → Flutter `RatingModel[]` → review cards render name+text+stars.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| N+1 avg — accessors recompute per item; Store uses `ratings()` query (re-query each call) | 🟡 Med | Store.php:95–102, Place.php:142–149 (20 items ⇒ 40–80+ queries) |
| No index on `parent_id`/`type` | 🟡 Med | rates migration — every ratings() scans |
| Raw full ratings array shipped in every list item | 🟡 Med | PlaceListResource:74 |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **No auth on POST/DELETE `rates`** | 🔴 CRITICAL | routes/api.php:147–148 |
| **Reviewer email (PII) in raw ratings array** | 🔴 HIGH | PlaceResource:74 + all content resources |
| **No rate range (1–5) enforcement** | 🔴 HIGH | RateController:26 |
| **No duplicate-rating prevention** | 🔴 HIGH | store() — rating stuffing/skew |
| Self-rating own content allowed | 🟡 Med | no parent ownership check |
| Rating inactive/soft-deleted content | 🟡 Med | no parent active check |
| Guest email/name spoofable | 🟡 Med | conditional validation only |

## 27. Technical Debt
- Pseudo-polymorphic (string type + parent_id) instead of real morphs — fragile, unindexed.
- `rate` stored/typed as string on both sides.
- Average computed in accessors, inconsistently (`$this->ratings` vs `$this->ratings()`).
- RatingModel lives under `tasneef` feature, not `rates`.
- Raw ratings collection leaks PII — needs a slim RateResource.
- No FormRequest; no auth; no unique/range constraints.

## 28. Improvement Opportunities
- Add `auth:api` to POST/DELETE; derive user_id server-side; require login to rate.
- Add `numeric|min:1|max:5` on `rate`; cast column to unsignedTinyInt.
- Add unique(user_id, parent_id, type) → one rating per user per item (upsert to edit).
- Introduce `RateResource` exposing only {id,name,rate,rateText,created_at} — drop email/user_id.
- Precompute avg/count via `withCount` + `withAvg` (or a cached `rating_avg`/`rating_count` column updated on write) to kill N+1.
- Index (parent_id, type); validate parent exists + active + not owned by rater.

---

## 32. Related Features
- **[[04_Places]] [[05_Stores]] [[06_Restaurants]] [[09_Events]] [[10_Swalef]] [[11_TourGuides]]** — all display `rate`/`review`/`ratings` from this system.
- **[[13_FavoritesSaved]]** — sibling engagement; Favorite/Save resources also compute avg rating.
- **[[08_Exploration]]** — SearchResource exposes `ratings_count`.
- **[[01_Authentication]]** — user identity backfills name/email/user_id.

## 33. How to Modify This Feature
- **Add content type:** extend validation `in:` list + add `ratings()` relation + `rate`/`review` accessors on the new model (copy Place pattern).
- **Secure it:** wrap routes in `auth:api`; in store() use `auth()->id()`; add unique index; add `numeric|between:1,5`.
- **Stop PII leak:** replace `'ratings'=>$this->ratings` with `RateResource::collection($this->ratings)` exposing safe fields only.
- **Fix N+1:** eager `->withCount('ratings')->withAvg('ratings','rate')` in index queries; use loaded values in accessors.

## 34. Regression Checklist
- [ ] Submit rating (logged in) → appears on detail, user_id set.
- [ ] Guest rating requires name+email.
- [ ] rate=0 blocked (client); after fix, API rejects <1 or >5.
- [ ] After unique fix: second rating by same user edits/rejects, not duplicates.
- [ ] Average + count correct after add/delete.
- [ ] Email NOT present in API response (after slim resource).
- [ ] Delete only own rating.

## 35. Common Bugs
- Average skewed by out-of-range or duplicate ratings.
- `rate` string vs int causes sort/compare bugs.
- PII (email) visible in network responses.
- Slow content lists from N+1 rating accessors.
- Guest can spam ratings / delete confusion (no auth).

## 36. Debug Guide
- **Rating not saving:** check `rateText` ≥10 chars, `type` in whitelist, `parent_id` present.
- **Wrong average:** inspect for out-of-range/duplicate rows in `rates` for that parent_id+type.
- **PII in response:** confirm resource still ships raw `$this->ratings`.
- **List slow:** query-log a content index; count ratings() calls.
- **Store avg differs from Place:** Store re-queries (`ratings()`), Place uses collection.

## 37. Search Keywords
`rate`, `Rate`, `rateText`, `rates table`, `parent_id`, `type place store zad`, `RateController`, `RateCubit`, `addRate`, `ShareYourRateButton`, `RatingModel`, `getRateAttribute`, `getReviewAttribute`, `ratings array email PII`, `rate range 1-5`, `duplicate rating`, التقييم, المراجعة, نجوم.

## 38. Future Improvements
- Migrate to real polymorphic `rateable` morphs + indexed columns, or keep flat but index + constrain.
- Cached rating_avg/rating_count columns updated on write (Observer) → O(1) reads.
- One-rating-per-user with edit; moderation/report on reviews.
- Slim public RateResource (no PII); optional avatar/verified-visitor badge.
- Auth-gated rating tied to actual visit/favorite for trust.
