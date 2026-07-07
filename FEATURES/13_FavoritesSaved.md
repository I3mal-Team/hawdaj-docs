# Feature 13 — Favorites & Saved (المفضلة والمحفوظات)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Supporting (engagement)
> **Flutter:** `lib/features/favorites/` + `lib/features/saved/` + `lib/features/profile/` · **Laravel:** `FavoriteController` + `SavedController`
> **One-line:** TWO independent, near-identical toggle systems — **Favorites** (polymorphic `MorphMany` on content models) and **Saved** (a separate flat table, NOT polymorphic-linked). Both use POST-toggle (check→delete/create), share no code, and both add per-item `is_favorite`/`is_saved` flags to content lists via unbatched N+1 helper queries.

---

## 1. Feature Name & Identity
- **Two systems, not one:**
  - **Favorites** = heart/like. Table `favorites`, model `Favorite` (`morphTo` favoritable). Content models declare `morphMany(Favorite,'favoritable')`.
  - **Saved** = bookmark/save-for-later. Table `saves`, model `Save` (`saved_id`/`saved_type`). **No polymorphic relation** — content models do NOT `morphMany(Save,...)`.
- Parallel, copy-pasted implementations. Same UX (toggle icon), separate storage/logic.

## 2. Business Goal & Rules
- **Goal:** Let authenticated users bookmark content (Places/Stores/Swalef/Events/Zad) in two buckets: favorites (hearts) and saved (bookmarks).
- **Rules:**
  - Auth required for all 4 endpoints (toggle + list, ×2).
  - Toggle semantics: POST → if (user,type,id) exists → DELETE; else CREATE. No idempotent add/remove.
  - Allowed types (validation `in:`): `place, store, swalef, event, zad`.
  - Backend normalizes type: `ucwords()` → `Place/Store/Swalef/Event`; special-case `zad` → `Zad_elgadel`.
  - No content-existence / `active` / soft-delete check before toggling → orphaned records possible.

## 3. User Flow
1. On any content card (Places/Stores/etc.), user taps heart → `FavoriteCubit.addToFavorite(id, type)` → POST `favorites` → list re-fetched to reflect state.
2. Tap bookmark → `SavedCubit.addToSaved(id, type)` → POST `saved`.
3. Profile → Favorites tab → `GetFavoritesCubit.getFavorites()` → GET `get-my-favorites`.
4. Profile → Saved tab → `GetSavedCubit.getSaved()` → GET `get-my-saved`.
- Content lists (`GET /places` etc.) carry `is_favorite`/`is_saved` booleans per item (computed server-side per row).

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`favorite_view.dart`](lib/features/profile/presentation/views/favorite_view.dart) | 20–160 | Favorites list, `GetFavoritesCubit`; toggle → re-fetch. Hardcodes `isFavorite:true` on display (line ~66) |
| [`saved_view.dart`](lib/features/profile/presentation/views/widgets/saved_view.dart) | 18–150 | Saved list, `GetSavedCubit`; toggle → re-fetch. Hardcodes `isFavorite:false` (line ~61 — naming inconsistent) |

## 5. Widgets
- Heart/bookmark toggle buttons embedded in content cards (Places/Stores/Zad/etc.), fed by `is_favorite`/`is_saved` from list resources.
- **Type mappers** in `lib/features/exploration/.../map_item_glodal_to_unified_place.dart`:
  - `mapItemGlodalToUnifiedFavorite` (40–73): `FavoriteModel → UnifiedPlaceModel`, hardcodes `isFavorite:true` (48).
  - `mapItemGlodalToUnifiedSaved` (75–108): `SavedItemModel → UnifiedPlaceModel`, `isFavorite:false` (83), `isSaved:null` (97).

## 6. Cubits / Blocs
| Cubit | File:Lines | Method | States |
|-------|-----------|--------|--------|
| FavoriteCubit | `features/favorites/.../favorite_cubit.dart:6–26` | `addToFavorite(favoriteId, favoriteType)` (toggle only) | Loading/Success/Error |
| SavedCubit | `features/saved/.../saved_cubit.dart:6–23` | `addToSaved(savedId, savedType)` (toggle only) | Loading/Success/Error |
| GetFavoritesCubit | `features/profile/.../get_favorites_cubit.dart:8–20` | `getFavorites()` | Loading/Loaded/Error |
| GetSavedCubit | `features/profile/.../get_saved_cubit.dart:8–19` | `getSaved()` | Loading/Loaded/Error |

## 7. State Flow
- **Toggle:** button → cubit `addTo*` → Loading → repo POST → Success → view re-calls `getFavorites()/getSaved()` (full list refetch, not local update).
- **List:** view → GetFavorites/GetSaved cubit → ProfileRepo GET → maps `data['data']['items']` → list of `FavoriteModel`/`SavedItemModel` → render.

## 8. Models
| Model | File:Lines | Fields (JSON prefix) |
|-------|-----------|----------------------|
| FavoriteModel | `profile/data/model/favorite_model.dart:1–72` | id, favoritableId, favoritableSlug, favoritableType, favoritableImage, favoritableTitle, favoritableAddress, favoritableLat, favoritableLong, favoritableRatings, favoritableFeatured, favoritableRegion, favoritableCity (keys `favorite_*`) |
| SavedItemModel | `profile/data/model/saved_item_model.dart:4–75` | id, savedId, savedSlug, savedType, savedImage, savedTitle, savedAddress, savedLat, savedLong, savedRatings, savedFeatured, savedRegion, savedCity (keys `saved_*`) |

- Identical shape, different key prefixes — copy-paste divergence.

## 9. Repositories
| Repo | File:Lines | Method | HTTP |
|------|-----------|--------|------|
| FavoriteRepo (abstract) | `favorites/data/repo/favorite_repo.dart:4–9` | `addFavorites(type,id)` | — |
| FavoriteRepoImp | `favorites/data/repo/favorites_repo_imp.dart:14–28` | POST `EndPoints.addFavorites` (`'favorites'`) query `favoritable_type`,`favoritable_id` | POST |
| SavedRepo (abstract) | `saved/data/repo/saved_repo.dart:4–9` | `addSaved(type,id)` | — |
| SavedRepoImp | `saved/data/repo/saved_repo_imp.dart:14–26` | POST `EndPoints.saved` (`'saved'`) query `saved_type`,`saved_id` | POST |
| ProfileRepo | `profile/data/repo/profail_repo_imp.dart:47–65` | `getFavorites()` GET `get-my-favorites` · `getSaved()` GET `get-my-saved` → `data['data']['items']` | GET |

## 10. Services / Helpers
- Laravel global helpers [`App.php`](../../../hawdaj-api/app/Helper/App.php): `isFavorite($id,$type)` (677–697), `isSaved($id,$type)` (699–718). Auth-guarded (return false if guest), else single `.first()` query per call → invoked per list item (N+1).
- Flutter: `DioConsumer`, `Either<Failure,T>`, `UnifiedPlaceModel` mappers.

## 11. API Endpoints
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `favorites` | auth:api | toggle favorite (query: favoritable_type, favoritable_id) |
| GET | `get-my-favorites` | auth:api | list user favorites (opt filter favoritable_type) |
| POST | `saved` | auth:api | toggle saved (query: saved_type, saved_id) |
| GET | `get-my-saved` | auth:api | list user saved |

`routes/api.php:150–153`.

## 12. Request / Response Models
- **Toggle req:** `{favoritable_type:'place', favoritable_id:123}` (or `saved_type/saved_id`).
- **Toggle resp:** `{message:'item_removed_successfully' | 'item_saved_successfully'}`.
- **List resp:** `{data:{items:[{favorite_id, favorite_favoritable_id, favorite_slug, favorite_type, favorite_image, favorite_title, favorite_address, favorite_lat, favorite_long, favorite_ratings, favorite_featured, favorite_region, favorite_city}], ...pagination}}` (saved mirrors with `saved_*`).
- **Content list flag:** each item gets `is_favorite:bool`, `is_saved:bool`.

## 13. Laravel Routes
`routes/api.php:150–153` (inside auth group / auth:api enforced):
```
POST favorites          → FavoriteController@store
GET  get-my-favorites   → FavoriteController@listMyFavorites
POST saved              → SavedController@store
GET  get-my-saved       → SavedController@listMySaved
```

## 14. Laravel Controllers
- [`FavoriteController.php`](../../../hawdaj-api/app/Http/Controllers/Api/FavoriteController.php) `:10–67`
  - `store()` 12–51: auth (16); validate (19–25: type `in:place,store,swalef,event,zad`, id numeric); normalize `ucwords()` (35) + `zad→Zad_elgadel` (37–39); dup check (41–42); if exists DELETE (45) else CREATE (49). **No content existence/active check.**
  - `listMyFavorites()` 53–66: `where user_id=auth`, opt filter type (58), `paginate(10)->onEachSide(2)` → `FavoriteCollection`.
- [`SavedController.php`](../../../hawdaj-api/app/Http/Controllers/Api/SavedController.php) `:10–62` — **IDENTICAL** logic, `saved_*` fields → `SaveCollection`.

## 15. Laravel Services
None. Logic inline. No FormRequest, no service class, no Observer.

## 16. Laravel Models
- [`Favorite.php`](../../../hawdaj-api/app/Models/Favorite.php) `:8–22`: fillable `favoritable_id, favoritable_type, user_id`; `morphTo()` favoritable. No soft-delete/status.
- [`Save.php`](../../../hawdaj-api/app/Models/Save.php) `:8–17`: fillable `saved_id, saved_type, user_id`; **no morphTo/relationship**. No soft-delete/status.
- Content models (Place/Store/Event/Swalef/ZadElgadel): each `morphMany(Favorite::class,'favoritable')`. **None link to Save** — saved model hydrated manually via type-switch in resource.

## 17. Database Tables & Relationships
- [`favorites`](../../../hawdaj-api/database/migrations/2024_03_31_235139_fix_favorites_table.php) `:14–27`: id, favoritable_id (unsignedBigInt), favoritable_type (string), user_id (unsignedBigInt FK→users onDelete set null), timestamps. **No unique constraint, no index** on favoritable_type/id.
- [`saves`](../../../hawdaj-api/database/migrations/2024_06_05_234657_create_saves_table.php) `:14–25`: id, saved_id, saved_type, user_id (FK set null), timestamps. **No unique constraint, no index.**

## 18. Validation
`store()` inline: `favoritable_type` required|in:place,store,swalef,event,zad; `favoritable_id` numeric. (Saved: `saved_type`/`saved_id`.) No FormRequest. No verification the referenced content row exists.

## 19. Auth & Permissions
- All 4 endpoints auth:api (Sanctum bearer). Controller re-checks `auth('api')->check()`.
- Helpers `isFavorite/isSaved` return false for guests (no crash on public lists).
- **IDOR-adjacent:** no explicit `DELETE /favorites/{id}` — removal only via toggle scoped to `user_id`, so cross-user delete not directly possible. But no ownership check beyond the query filter.

## 20. Error Handling
- Flutter `Either<Failure,T>`; cubits emit Error state.
- Laravel: validation → 422; success → envelope message.
- **Resource fallback:** if favoritable/saved content missing (deleted), resource returns a stub `{message:'Resource not found'}` instead of omitting → dead/ghost rows surface in list.

## 21. Edge Cases
- Content soft-deleted but favorite/save row remains → list shows "not found" stub.
- No DB unique constraint → concurrent double-tap can create duplicate rows (toggle logic assumes single row).
- Type casing mismatch: client sends `store`; stored `Store`; helper `isFavorite($id,'Place')` uses `Place::TYPE_PLACE` constant. If constants and normalization drift → flag reads false while row exists.
- Flutter mapper compares against `'restaurant'`, `'story'` — strings that don't match backend (`Swalef`, no `story`) → potential mismap.
- Guest content lists → all flags false (expected).

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dartz, dio, `UnifiedPlaceModel`, ProfileRepo.
- **Laravel:** Eloquent morphTo, `ApiModalController`, global helpers `App.php`.
- **Cross-feature:** consumed by [[04_Places]], [[05_Stores]], [[06_Restaurants]], [[09_Events]], [[10_Swalef]] (each list resource calls the flag helpers). Rendered in [[21_ProfileSettings]] tabs.

## 23. Files Involved
**Flutter:** favorites (cubit+state, repo abstract+imp), saved (cubit+state, repo abstract+imp), profile (get_favorites_cubit, get_saved_cubit, favorite_view, saved_view, FavoriteModel, SavedItemModel, ProfileRepo), exploration mappers, service_locator, end_points, routes (kFavoriteView/kSavedView).
**Laravel:** Favorite model, Save model, FavoriteController, SavedController, FavoriteResource, FavoriteCollection, SaveResource, SaveCollection, isFavorite/isSaved in App.php, 2 migrations, routes/api.php:150–153, morphMany decls on 5 content models, `is_favorite`/`is_saved` in PlaceListResource (76–77) + Store/Swalef/Event/Zad resources.

## 24. Full Execution Flow (UI → DB → UI)
**Toggle favorite:**
Card heart tap → `FavoriteCubit.addToFavorite(id,'store')` → `FavoriteRepoImp` POST `favorites?favoritable_type=store&favoritable_id=123` → `FavoriteController@store` → auth → validate → normalize `store→Store` → `Favorite::where(user_id,'Store',123)->first()` → exists? DELETE : CREATE → `{message}` → cubit Success → view `getFavorites()` refetch.
**Content list flag:**
`GET /places` → `PlaceListResource` per item → `is_favorite => isFavorite($id, Place::TYPE_PLACE)` → `Favorite::where(user_id,type,id)->first()` (1 query/item) → bool. N items = 1 + 2N queries.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| **N+1 on flags** — isFavorite + isSaved per list row | 🔴 HIGH | App.php:677–718, PlaceListResource:76–77 (100 items ⇒ 201 queries) |
| No composite index on (user_id,type,id) | 🟡 Med | both migrations — every toggle/flag scans |
| Full list re-fetch after each toggle | 🟢 Low | favorite_view/saved_view |
| Resource `.first()` per favorite in collection (hydrate content) | 🟡 Med | FavoriteResource:47–66 |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| No content-existence/active/soft-delete check before toggle | 🟡 Med | FavoriteController/SavedController store() — orphaned + can favorite inactive content |
| Deleted content returns stub in list (info about existence) | 🟢 Low | FavoriteResource/SaveResource fallback |
| No unique constraint → duplicate rows race | 🟡 Med | migrations |
| No explicit ownership check beyond user_id filter | 🟢 Low | store() toggle (mitigated by query scope) |

## 27. Technical Debt
- Two fully parallel copy-pasted systems (Favorite vs Save) — no shared abstraction.
- Save is NOT polymorphic (no morphMany) → manual type-switch hydration duplicated.
- Type casing normalization (`ucwords`, `zad→Zad_elgadel`) fragile & duplicated in 2 controllers.
- Flutter mapper compares stale type strings (`restaurant`, `story`).
- No add/remove idempotent endpoints; toggle-only forces full refetch.
- Hardcoded `isFavorite` flags in views/mappers.

## 28. Improvement Opportunities
- Add composite unique index + index on (user_id, type, id) for both tables.
- Batch flags: single `whereIn` per list (collect ids, one query for favorites + one for saved) instead of per-item helpers.
- Merge Favorite+Save into one polymorphic `interactions` table with a `kind` column, or at least share a trait.
- Validate content existence + active before toggle; omit ghost rows from lists.
- Return the new state (favorited:true/false) from toggle so client updates locally (no refetch).
- Unify type-string mapping in one enum/config used by both repos + resources.

---

## 32. Related Features
- **[[04_Places]] [[05_Stores]] [[06_Restaurants]] [[09_Events]] [[10_Swalef]]** — content that can be favorited/saved; their list resources call the flag helpers.
- **[[21_ProfileSettings]]** — Favorites/Saved tabs live under profile.
- **[[08_Exploration]]** — `UnifiedPlaceModel` mappers convert favorite/saved items.
- **[[14_RatesReviews]]** — sibling engagement system (also per-content, also polymorphic-ish).

## 33. How to Modify This Feature
- **Add favoritable content type:** add to validation `in:` list (both controllers), add case in `getFavoritable()`/`getSavedModel()` resource switch, add `morphMany(Favorite)` on the model, add type mapping in Flutter mapper.
- **Fix N+1:** replace per-item `isFavorite()`/`isSaved()` with batched `whereIn` — collect page ids, one query each, hydrate a lookup set in the collection.
- **Add unique constraint:** migration `unique(['user_id','favoritable_type','favoritable_id'])` — first dedupe existing rows.

## 34. Regression Checklist
- [ ] Favorite toggle on/off updates list.
- [ ] Saved toggle independent of favorite.
- [ ] Guest sees flags=false, no crash.
- [ ] Content lists show correct is_favorite/is_saved for logged-in user.
- [ ] zad type stores as `Zad_elgadel` and reads back correctly.
- [ ] Deleted content doesn't crash favorites list.
- [ ] Double-tap doesn't create duplicate rows (after unique-constraint fix).

## 35. Common Bugs
- Flag reads false though item favorited → type casing mismatch (`store` vs `Store`).
- zad items never show favorited → `zad`→`Zad_elgadel` normalization drift.
- Duplicate favorite rows from rapid taps (no unique constraint).
- Ghost "Resource not found" entries in favorites list (soft-deleted content).
- Slow content lists at scale → N+1 flag queries.

## 36. Debug Guide
- **Flag wrong:** check stored `favoritable_type` casing vs helper's `TYPE_*` constant.
- **Duplicate rows:** query `favorites` group by (user,type,id) having count>1.
- **List slow:** enable query log on `GET /places` — count isFavorite/isSaved calls.
- **Toggle no effect:** verify auth token present; check normalized type matches stored.
- **zad issues:** confirm controller lines 37–39 special-case applied.

## 37. Search Keywords
`favorite`, `Favorite`, `favoritable_type`, `favoritable_id`, `isFavorite`, `is_favorite`, `saved`, `Save`, `saved_type`, `saved_id`, `isSaved`, `is_saved`, `FavoriteController`, `SavedController`, `FavoriteCubit`, `SavedCubit`, `get-my-favorites`, `get-my-saved`, `morphMany Favorite`, `Zad_elgadel`, `ucwords type`, المفضلة, المحفوظات, حفظ.

## 38. Future Improvements
- Unify into single polymorphic interactions table (kind: favorite|saved|...).
- Batched flag resolution + DB indexes for scale.
- Idempotent add/remove + optimistic local UI (no full refetch).
- Content-existence validation + soft-delete-aware lists.
- Shared type-mapping source of truth across Flutter + Laravel.
