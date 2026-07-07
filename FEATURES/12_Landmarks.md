# Feature 12 — Landmarks (معالم / إضافة مواقع من المستخدم)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Core (UGC content-lite)
> **Flutter:** `lib/features/landmarks/` · **Laravel:** `LandMarksController` + `LandMark` model
> **One-line:** A lightweight user-generated content entity — users submit a landmark (title + description + address + one image + a `type` tag) that is stored `active=0/status=pending` for admin moderation; public list & detail endpoints exist but **do not filter by `active`**.

---

## 1. Feature Name & Identity
- **Name:** Landmarks (`land_marks` table, `LandMark` model, `landmarks` feature folder).
- **Nature:** A *content-lite* UGC feature. Unlike Places/Stores/Zad (rich translatable content entities), Landmarks is a stripped-down submission form. It is the simplest "create content" flow in the app.
- **`type` field** categorizes each landmark into one of the existing content domains: `place | store | swalef | event | zad`. This is a loose text tag, **not** a foreign key.

## 2. Business Goal & Rules
- **Goal:** Let end users contribute points of interest ("معالم") they know about, tied to an existing content category, pending admin approval.
- **Rules:**
  - Must be authenticated to **create** (`store`) and to view **my landmarks** (`myLandMarks`).
  - Public **list** (`index`) and **detail** (`show`) require no auth (auth check absent in `show`).
  - On creation: `status='pending'`, `active='0'` always. Admin must flip `active` in dashboard before it *should* be public.
  - `type` must be one of the 5 allowed strings (validated).
  - One image max per landmark (Spatie single-file media collection).
- **⚠️ Rule violation in code:** Neither `index()` nor `show()` filters `active=1` — so pending/unapproved user submissions are immediately publicly visible. The moderation gate is effectively bypassed. See §18/§24.

## 3. User Flow
1. User opens Landmarks list (`LandMarkView`) → paginated cards (`GetLandmarksCubit`).
2. Taps "add new" → `AddLandmarkView` (`AddLandmarkCubit`).
3. Picks a category (bottom sheet → sets `type`), enters name/description/address, picks image(s).
4. Submits → `POST landmark/store` (multipart FormData) → backend stores `pending/active=0`.
5. Backend returns generic `{code:200, message:"ok"}` — **no landmark id**, so client cannot deep-link to the new item.
6. User can view own submissions via `MyLandMarkView` (`GetMyLandmarksCubit` → `landmark/myLandMarks`).
- **Detail flow is DEAD:** `LandmarkShowCubit` + `getLandmark(id)` + `landmark/show/{id}` all exist but no route/view mounts the show cubit. Tapping a card does not open a detail screen.

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`land_mark_view.dart`](lib/features/landmarks/presentation/view/land_mark_view.dart) | 1–95 | Public list, `PagedListView` infinite scroll, "add" button |
| [`my_land_mark_view.dart`](lib/features/landmarks/presentation/view/my_land_mark_view.dart) | 1–74 | Auth list of user's own landmarks |
| [`add_landmark_view.dart`](lib/features/landmarks/presentation/view/add_landmark_view.dart) | 1–150+ | Create form: multi-image picker, category bottom sheet, name/desc/address fields, validation, FormData upload |

## 5. Widgets
| File | Notes |
|------|-------|
| [`land_mark_items.dart`](lib/features/landmarks/presentation/view/widgets/land_mark_items.dart) | Card: image + gradient overlay + title + address. **Line ~113:** address label hardcoded "location". **Line ~94:** `rating.toStringAsFixed(1)` — hardcoded/placeholder rating, not backed by API (dead). |
| [`landmark_type_bottom_sheet.dart`](lib/features/landmarks/presentation/view/widgets/landmark_type_bottom_sheet.dart) | Category selector → sets `type` (place/store/swalef/event/zad). |

## 6. Cubits / Blocs
| Cubit | File | States | Methods |
|-------|------|--------|---------|
| GetLandmarksCubit | `manager/get_landmarks_cubit/` | Initial, Loaded(pagingController), Error | `_fetchPage()`, `refresh()` |
| GetMyLandmarksCubit | `manager/get_my_landmarks_cubit/` | Initial, Loaded, Error | `_fetchPage()`, `refresh()` — mirrors GetLandmarks |
| AddLandmarkCubit | `manager/add_landmark_cubit/` | Initial, Ready, Loading, Success, Error | `setSelectedCategory()`, `addImages()`, `removeImage()`, `clearImages()`, `validateForm()`, `createLandmark()` |
| **LandmarkShowCubit** | `manager/landmark_show_cubit/` (20 lines) | Initial, Loading, Loaded, Error | `getLandmark(id)` — **ORPHANED, never mounted** |

## 7. State Flow
- **List:** View builds `PagingController` → cubit `_fetchPage(pageKey)` → repo `getLandmarks(pageKey)` → `Either<Failure, LandmarkData>` → on Right append items + set nextPageKey (or mark last page); on Left `pagingController.error`.
- **Create:** field changes → `validateForm()` toggles Ready; `createLandmark()` → Loading → repo `createLandmark(req)` → Success/Error state.
- **My:** identical to list but hits `myLandMarks` endpoint.

## 8. Models
| Model | File:Lines | Fields |
|-------|-----------|--------|
| LandmarkModel | `data/models/landmark_model.dart:1–80` | id, name, description, address, category(ExploreCategoryModel), images[], createdAt, updatedAt; `fromJson/toJson` |
| CreateLandmarkRequest | `landmark_model.dart:82–106` | name, description, address, images[], type; `toJson` maps **name→title** for FormData |
| CreateLandmarkResponse | `landmark_model.dart:108–122` | code, message; `success => code==200` |
| LandmarkResponse / LandmarkData / LandmarkItem | `data/models/list_landmark_response.dart:5–84` | `@JsonSerializable`. LandmarkData = full Laravel paginator shape (currentPage, items, firstPageUrl, from, lastPage, nextPageUrl, perPage, prevPageUrl, to, total). LandmarkItem = id, title, description, address, image |
| (generated) | `list_landmark_response.g.dart` | json_serializable output |

- **⚠️ Model asymmetry:** `LandmarkModel` (with category + images[] + timestamps) is used by create/parse; `LandmarkItem` (flat: id/title/description/address/image) is used by list/show. Two representations of the same entity. Timestamps present in `LandmarkModel` but absent in `LandmarkItem` → list UI loses created_at.

## 9. Repositories
- Abstract: [`landmarks_repository_repo.dart`](lib/features/landmarks/data/repositories/landmarks_repository_repo.dart) (16 lines).
- Impl: [`landmarks_repository_impl.dart`](lib/features/landmarks/data/repositories/landmarks_repository_impl.dart) (82 lines), ctor `apiConsumer: DioConsumer`.

| Method | Signature | HTTP |
|--------|-----------|------|
| createLandmark | `(CreateLandmarkRequest) → Either<Failure, CreateLandmarkResponse>` | POST `landmark/store`, FormDataHelper adds title/description/address/type + `addFile(imageFile,'image')` after file-exists check |
| getLandmarks | `(int pageKey) → Either<Failure, LandmarkData>` | GET `landmark/list?page=` → `json['data']` |
| getMyLandmarks | `(int pageKey) → Either<Failure, LandmarkData>` | GET `landmark/myLandMarks?page=` → `json['data']` |
| getLandmark | `(int id) → Either<Failure, LandmarkItem>` | GET `landmark/show/{id}` → `json['data']` (repo wired, cubit orphaned) |

## 10. Services / Helpers
- `FormDataHelper` (shared) builds multipart body for create.
- `DioConsumer` (shared networking) executes requests.
- No feature-specific service. No location/geocoding service (Landmarks captures no coordinates).

## 11. API Endpoints
| Method | Endpoint | Auth | EndPoints const |
|--------|----------|------|-----------------|
| GET | `landmark/list?page=` | none (route) / **required (controller)** | `landmarkList(page)` `end_points.dart:38–45` |
| GET | `landmark/show/{id}` | **none** | `landmarkShow(id)` |
| GET | `landmark/myLandMarks?page=` | required (controller) | `myLandMarks(page)` |
| POST | `landmark/store` | required (controller) | `landmarkStore` `end_points.dart:120–122` |
| — | `landmarks` (const `→ 'landmarks'`) | — | **unused/dead** |

## 12. Request / Response Models
- **Create request (multipart):** `title`, `description`, `address`, `type`, `image` (single file).
- **Create response:** `{code:200, message:"ok"}` — no data, no id.
- **List/My response:** `{status, message, data:{ current_page, items:[{id,title,description,address,image}], first_page_url, from, last_page, last_page_url, next_page_url, path, per_page, prev_page_url, to, total }}`.
- **Show response:** `{...,data:{ id, title, description, address, image, image_small, image_medium }}`.

## 13. Laravel Routes
`routes/api.php` (~lines 164–167), inside `Route::group(['as'=>'front.','namespace'=>'Front'])` — **no route middleware**; auth enforced inline in controller.
```
POST  landmark/store        → LandMarksController@store
GET   landmark/list         → LandMarksController@index
GET   landmark/myLandMarks   → LandMarksController@myLandMarks
GET   landmark/show/{id}    → LandMarksController@show
```

## 14. Laravel Controller
[`LandMarksController.php`](../../../hawdaj-api/app/Http/Controllers/Api/LandMarksController.php) `:1–95` (extends `ApiModalController`).
| Method | Line | Auth | Logic |
|--------|------|------|-------|
| `index()` | ~16–22 | `auth('api')->check()` required | `LandMark::paginate(10)->onEachSide(2)` — **no `active=1` filter** → returns `LandMarkCollection` |
| `myLandMarks()` | ~31–37 | required | `where user_id = auth()->id`, `paginate(10)->onEachSide(2)` → `LandMarkCollection` |
| `show($id)` | ~47 | **NONE** | `findOrFail($id)` — **no `active=1` filter** → `LandMarkResource` |
| `store()` | ~56–92 | required | validate → set `user_id`, `status='pending'`, `active='0'` → create → `addMediaFromRequest('image')` → return generic 200 "ok" |

## 15. Laravel Services
None. All logic inline in controller. No FormRequest, no Observer, no dedicated service class (contrast: Places has Observer + helpers).

## 16. Laravel Models
[`LandMark.php`](../../../hawdaj-api/app/Models/LandMark.php) `:1–56` — `implements HasMedia`, uses `HasFactory, InteractsWithMedia`.
- **fillable:** title, description, address, address_type, image, user_id, type, status, active.
- **Relationships:** `user()` → belongsTo(User)->withDefault(). Nothing else.
- **Media:** `image` collection, singleFile, mimes jpeg/png/jpg/gif/svg/webp; conversions `small` 250×250, `medium` 500×500, nonQueued.
- **NOT present:** Translatable, SoftDeletes, scopes, accessors, lat/long, Gallery, Rate, Favorite, Ceo.

## 17. Database Tables & Relationships
[`2024_05_13_215028_create_land_marks_table.php`](../../../hawdaj-api/database/migrations/2024_05_13_215028_create_land_marks_table.php) `:14–30`.
```
land_marks
  id PK
  title       string nullable
  description  text   nullable
  address     text   nullable
  address_type string nullable   -- UNUSED in controller (schema drift)
  image       string nullable   -- legacy column; real image in media table
  user_id     bigint nullable → FK users.id ON DELETE SET NULL
  type        string nullable   -- place/store/swalef/event/zad
  status      string nullable   -- default 'pending'
  active      tinyint default 0 -- always 0 on create
  timestamps
```
- **Indexes:** only PK + FK. No index on `type`, `active`, or `user_id` (myLandMarks filters user_id unindexed).
- **Relationships:** belongsTo users. Media stored in Spatie `media` polymorphic table (`model_type=LandMark`).

## 18. Validation
Inline `Validator::make` in `store()` (~lines 60–72):
```
title       => required|string
description => required|string
address     => nullable|string
type        => required|string|in:place,store,swalef,event,zad
image       => nullable|image|mimes:jpeg,png,jpg,gif,svg|max:2048
```
- **Gaps:** no max length on title/description; no dedicated FormRequest (inconsistent with Places/Events); Flutter allows multi-image but API validates a single `image`.

## 19. Auth & Permissions
- **Create / list / myLandMarks:** `auth('api')->check()` inline (Sanctum bearer).
- **show:** unauthenticated, and no `active` gate → any id publicly readable.
- Route group has no `auth:api` middleware — auth is controller-side only (inconsistent with rest of API best practice).

## 20. Error Handling
- Flutter: `Either<Failure, T>` throughout; list/my push errors into `pagingController.error`; create emits Error state.
- Laravel: `findOrFail` → 404; validation failure → 422 envelope; success envelope via `ApiModalController`.
- **Weak spot:** create returns generic 200 "ok" with no id/body — client cannot confirm what was created or navigate to it.

## 21. Edge Cases
- Multi-image selected in Flutter but only one `image` key sent/accepted → silent data loss of extra images.
- `selectedCategory` can be null if bottom sheet skipped — `type` may be sent empty → 422.
- `show` on a `pending` landmark returns it (no gate) → moderation bypass.
- Legacy `image` column vs media table: Resource falls back between them → possible stale/orphaned image URL.
- `address_type` never written → always null.
- Newly created item not returned → list must refresh to see it; possible race where pending item appears publicly to everyone.

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dartz, infinite_scroll_pagination (PagedListView), image_picker, dio, `ExploreCategoryModel` (shared), `FormDataHelper`, `DioConsumer`.
- **Laravel:** spatie/laravel-medialibrary, Sanctum, `ApiModalController` base.
- **Cross-feature:** `type` values reference Places/Stores/Swalef/Events/Zad domains (loose coupling, string only). Shares `ExploreCategoryModel` with Exploration feature.

## 23. Files Involved
**Flutter (18):** views (land_mark_view, my_land_mark_view, add_landmark_view), widgets (land_mark_items, landmark_type_bottom_sheet), cubits+states (get_landmarks, get_my_landmarks, add_landmark, landmark_show[orphaned]), models (landmark_model, list_landmark_response + .g.dart), repos (repo abstract + impl), registration in `core/di` (getIt ~154–155), routes in `core/routing/routes.dart`, `RoutesKeys` (~line 38 kAddLandmarkView), `end_points.dart`.
**Laravel (6):** LandMark model, LandMarksController, LandMarkResource, LandMarkCollection, migration, routes/api.php:164–167.

## 24. Full Execution Flow (UI → DB → UI)
**Create:**
`AddLandmarkView` submit → `AddLandmarkCubit.createLandmark()` (Loading) → `LandmarksRepositoryImpl.createLandmark(req)` → `FormDataHelper` builds multipart (title/description/address/type + file 'image') → `DioConsumer.post(landmark/store)` → **[network]** → `routes/api.php` → `LandMarksController@store` → `auth('api')->check()` → `Validator::make` → `LandMark::create({...,user_id,status:'pending',active:'0'})` → `addMediaFromRequest('image')` (Spatie generates small/medium) → return `{code:200,message:'ok'}` → repo `Right(CreateLandmarkResponse)` → cubit Success → UI shows success, clears form.
**List:**
`LandMarkView` `PagingController._fetchPage(pageKey)` → `GetLandmarksCubit._fetchPage` → repo `getLandmarks(pageKey)` → GET `landmark/list?page=` → `LandMarksController@index` → `auth check` → `LandMark::paginate(10)` (no active filter) → `LandMarkCollection` → `json['data']` → `LandmarkData` → append `LandmarkItem`s to controller → cards render.

## 25. Performance
| Issue | Severity | Location | Note |
|-------|----------|----------|------|
| No `active=1` filter in index | Med | Controller ~20 | scans/serves all rows incl. pending |
| N+1 on media/user | Med | LandMarkResource | no eager load of media/user; `getMediaUrl()` per row |
| Unindexed `user_id` in myLandMarks | Med | migration | full scan for user's list |
| `onEachSide(2)` | Low | Controller | extra count/window queries |
| Hardcoded rating render | Low | land_mark_items.dart:94 | dead interpolation |

## 26. Security
| Issue | Severity | Location | Impact |
|-------|----------|----------|--------|
| `show()` no auth + no active filter | **HIGH** | Controller ~47 | any id publicly readable incl. unapproved UGC |
| `index()` no active filter | **HIGH** | Controller ~20 | moderation gate bypassed; pending content public |
| Route-level auth missing (controller-only) | Med | api.php:164–167 | fragile; easy to forget on new methods |
| Legacy image + media dual system | Med | model/Resource | orphaned-file risk |
| No FormRequest / no length caps | Med | Controller validation | inconsistent, unbounded text |

## 27. Technical Debt
- `LandmarkShowCubit` + `getLandmark` + `landmark/show` fully built but **never wired** (dead detail flow).
- Hardcoded placeholder rating in card (no rating backend).
- `address_type` column dead.
- `landmarks` endpoint constant unused.
- Two model shapes (`LandmarkModel` vs `LandmarkItem`) for one entity.
- Multi-image UI vs single-image API mismatch.
- Inline validation instead of FormRequest.
- Generic "ok" create response (no id).

## 28. Improvement Opportunities
- Add `where('active',1)` to `index()` and `show()`; require auth or public-but-active on `show`.
- Return created resource (with id) from `store()` for client navigation.
- Wire `LandmarkShowCubit` to a real detail route, or delete the dead cubit.
- Support true multi-image (align API to Flutter) or restrict Flutter to single.
- Add real rating/favorite if Landmarks should behave like content entities — or remove placeholder.
- Extract `StoreLandMarkRequest` FormRequest; add length caps + index on `type`/`user_id`/`active`.

---

## 32. Related Features
- **[[04_Places]] / [[05_Stores]] / [[06_Restaurants]] / [[09_Events]] / [[10_Swalef]]** — `type` tags a landmark to one of these domains (string only, no FK).
- **[[08_Exploration]]** — shares `ExploreCategoryModel`.
- **[[15_MyProperties]]** — sibling UGC-submission pattern (compare status/active moderation).
- **[[23_MediaGallery]]** — Spatie small/medium conversions.
- **Contrast with Places (reference template):** Landmarks lacks Translatable, SoftDeletes, Rate, Favorite, Ceo, Gallery, lat/long.

## 33. How to Modify This Feature
- **Add a field:** migration on `land_marks` → add to `$fillable` → validation in `store()` → `LandMarkResource` → Flutter `LandmarkModel`/`CreateLandmarkRequest` + form field.
- **Fix moderation leak:** add `->where('active',1)` in `index()`/`show()` (Controller ~20/47). Confirm admin dashboard sets `active=1` on approve.
- **Enable detail screen:** add route mounting `LandmarkShowCubit`, tap handler on `land_mark_items.dart`.
- **Add rating:** need backend Rate relation (mirror Places) + real value in card (remove hardcoded line ~94).

## 34. Regression Checklist
- [ ] Create landmark (auth) → appears in myLandMarks.
- [ ] Unauth create → rejected.
- [ ] Invalid `type` → 422.
- [ ] Image >2MB → 422.
- [ ] List paginates (page 2 loads).
- [ ] After moderation fix: pending landmark NOT in public list/show.
- [ ] myLandMarks shows only own rows.
- [ ] Multi-image selection behavior defined (not silent drop).

## 35. Common Bugs
- Extra selected images silently dropped (single `image` key).
- Empty `type` (skipped category) → 422 with unclear UI message.
- Card shows fake rating "0.0"/placeholder — confuses users.
- New landmark not visible until manual refresh (no id returned).
- Pending items leaking into public list (active filter missing).

## 36. Debug Guide
- **Create fails silently:** check `type` non-null + image ≤2MB; inspect FormData keys (`title` not `name`).
- **Item not public/visible:** currently ALL are visible (no active filter). If a fix added filter, verify `active=1` in DB.
- **Image missing:** check Spatie media row (`model_type=App\Models\LandMark`) vs legacy `image` column fallback in Resource.
- **Detail screen blank:** expected — `LandmarkShowCubit` not mounted.
- **404 on show:** `findOrFail` — bad id.

## 37. Search Keywords
`landmark`, `LandMark`, `land_marks`, `landmark/store`, `landmark/list`, `landmark/myLandMarks`, `landmark/show`, `LandMarksController`, `AddLandmarkCubit`, `GetLandmarksCubit`, `LandmarkShowCubit`, `CreateLandmarkRequest`, `LandmarkItem`, `LandMarkCollection`, `address_type`, `type in place,store,swalef,event,zad`, `active=0 pending`, معالم, إضافة موقع.

## 38. Future Improvements
- Promote Landmarks to a full content entity (Translatable + Rate + Favorite + coords) OR keep it as a lightweight "suggest a place" intake feeding admin review.
- Proper moderation queue UI + notification to submitter on approve/reject.
- Geolocation capture (map picker) so landmarks can appear on the Exploration map.
- Deduplicate model representations; return created resource.
