# Feature 15 — My Properties / My Submissions (ممتلكاتي / مساهماتي)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Supporting (UGC management)
> **Flutter:** `lib/features/my_properties/` (+ per-type `my X` screens) · **Laravel:** `PropertiesController` (+ per-type `my*` methods)
> **One-line:** A hybrid: a **unified hub** (`/my-properties`) that aggregates the user's submitted Places/Stores/Events/Zads (created with `added_by_user=1`, `status=pending`, `active=0`, requiring an `ownership_proof_file`), **plus scattered per-type endpoints** (myLandMarks, myStories) with divergent moderation models. Status enum casing is inconsistent (`accepted` client vs `pending`/`active` backend), and Landmarks/Stories skip the ownership-proof + rejected_reason workflow.

---

## 1. Feature Name & Identity
- **Name:** My Properties / My Submissions — where a user views and tracks content THEY submitted, with moderation status (pending/accepted/rejected).
- **Two overlapping mechanisms:**
  - **Unified hub:** `GET /my-properties` → Places + Stores + Events + Zads submitted by the user (`added_by_user=1 AND user_id=auth`).
  - **Per-type endpoints:** `myLandMarks`, `myStories`, `myLastDayStories`, guide-own (`getGuide`) — separate, not part of the hub.

## 2. Business Goal & Rules
- **Goal:** Let users submit content, prove ownership, and track approval status.
- **Rules:**
  - Places/Stores/Events/Zads submission: `added_by_user=1`, `status='pending'`, `active=0`, **`ownership_proof_file` required** (PDF/DOC/IMG).
  - Landmarks: `status='pending'`, **no ownership proof, no rejected_reason** column.
  - Stories: `status='active'` (**auto-approved, no moderation gate**).
  - All `my*` endpoints require `auth('api')->check()`.
  - Moderation (accept/reject + set rejected_reason) happens in the Laravel dashboard, not the API.

## 3. User Flow
1. User submits content (via each feature's create form) → stored pending/inactive.
2. Opens "My Properties" hub (`MyPropertiesView`) → tabs All | Places | Stores | Events | Zads → single API call, local filter by type.
3. Each item shows a **status badge** (accepted/pending/rejected) + rejected_reason if rejected.
4. Landmarks tracked separately via `MyLandMarkView` (`GetMyLandmarksCubit` → `myLandMarks`). Stories via `myStories`.
5. Admin later flips status/active + sets rejected_reason in dashboard → reflected on next fetch.

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`my_properties_view.dart`](lib/features/my_properties/presentation/view/my_properties_view.dart) | 22–208 | Unified hub, tabs All/Places/Stores/Events/Zads, local filter, infinite pagination |
| [`status_badge.dart`](lib/features/my_properties/presentation/view/widgets/status_badge.dart) | 17–73 | Badge: accepted/pending/rejected/unknown; lowercases before match (line ~24) |
| [`my_land_mark_view.dart`](lib/features/landmarks/presentation/view/my_land_mark_view.dart) | 14–72 | Separate per-type "my landmarks" list |

## 5. Widgets
- `StatusBadge`: enum `accepted | pending | rejected | unknown`; **expects `'accepted'`** for approved (line ~32) — mismatch with backend `pending`/`active`.
- Per-item cards with status + (rejected reason). No edit button (no edit API); delete only for stories.

## 6. Cubits / Blocs
| Cubit | File:Lines | Behavior |
|-------|-----------|----------|
| MyPropertiesCubit | `my_properties/.../my_properties_cubit.dart:10–90` | Single paginated `GET /my-properties`, caches full response, filters locally by type (34–35) |
| GetMyLandmarksCubit | `landmarks/.../get_my_landmarks_cubit` | Separate paginated `myLandMarks` (see [[12_Landmarks]]) |

## 7. State Flow
- Hub: `MyPropertiesCubit` fetches once → caches → tab switch filters cached list locally (no refetch per tab). Infinite scroll via `PagingController`.
- Per-type: independent cubit per screen.

## 8. Models
| Model | File:Lines | Key fields |
|-------|-----------|-----------|
| MyPropertiesResponse / PropertyItem | `my_properties/data/model/my_properties_response.dart:6–328` (item 101–167) | id, title, type, image, `status`, `rejectedReason`, `ownershipProofFile`, `active`, address/geo |

- **⚠️** Client status contract = `accepted`/`pending`/`rejected`; backend emits `pending` on create, `active` for stories → normalization gap.

## 9. Repositories
- [`properties_repo_imp.dart`](lib/features/my_properties/data/repo/properties_repo_imp.dart): `GET /my-properties` (list) + POST `/properties` multipart (create, ~line 60+ with ownership_proof_file).

## 10. Services / Helpers
- Multipart upload via `FormDataHelper`/`DioConsumer`.
- Backend `autoSetData` sets status/active/user_id on create (PropertiesController:166–167).

## 11. API Endpoints
| Method | Endpoint | Auth | Controller |
|--------|----------|------|-----------|
| GET | `my-properties` | auth:api (controller) | PropertiesController@index (21–48) |
| POST | `properties` | auth:api | PropertiesController create (proof required, 84) |
| GET | `landmark/myLandMarks` | auth:api | LandMarksController@myLandMarks (29–42) |
| GET | `myStories` | auth:api | StoryController@myStories (31–44) |
| GET | `myLastDayStories` | auth:api | StoryController@myLastDayStories (46–61) |
| DELETE | `stories/{id}/delete` | auth:api | StoryController delete (120–127, ownership-checked) |
| GET/POST | guide own (`getGuide`/`storeGuide`/`updateGuide`) | auth:api | GuideController (see [[11_TourGuides]]) |

Routes: `routes/api.php:76, 169, 175–176, 178, 192–200`.

## 12. Request / Response Models
- **Hub resp:** 4 typed buckets (places/stores/events/zads), each paginated, items carry `status`, `rejected_reason`, `ownership_proof_file`, `active`.
- **Create req (properties):** multipart with content fields + `ownership_proof_file`.
- **PropertyItem status:** string, no enum discipline.

## 13. Laravel Routes
```
GET    my-properties            → PropertiesController@index          (line 76)
GET    landmark/myLandMarks     → LandMarksController@myLandMarks     (169)
GET    myStories                → StoryController@myStories           (175)
GET    myLastDayStories         → StoryController@myLastDayStories    (176)
DELETE stories/{id}/delete      → StoryController@delete              (178)
... guide own endpoints          → GuideController                    (192–200)
```

## 14. Laravel Controllers
- [`PropertiesController.php`](../../../hawdaj-api/app/Http/Controllers/Api/PropertiesController.php)
  - `index()` 21–48: fetch Places/Stores/Events/Zads where `added_by_user=1 AND user_id=auth`, paginate per type, return all 4 (30–40).
  - create: `autoSetData` → `active=0` (166), `status='pending'` (167); requires `ownership_proof_file` (84).
- `LandMarksController@myLandMarks` 29–42: auth (32), `where user_id=auth`, paginate. Create sets `status='pending'` (82), no proof.
- `StoryController@myStories` 31–44, `myLastDayStories` 46–61; create sets `status='active'` (105) auto-approved; `delete` 120–127 ownership-checked (returns 422 if `user_id != auth`).

## 15. Laravel Services
None dedicated. `autoSetData` helper on PropertiesController sets status/active/user_id. Moderation logic lives in dashboard controllers (out of API scope).

## 16. Laravel Models
- Places/Stores/Events/Zads: add `status`, `ownership_proof_file`, `rejected_reason`, `added_by_user`(indexed), `user_id`(FK) — via `2025_11_07_183859_add_proof_of_ownership_to_places_table.php:18–21` (+ parallel migrations per type).
- LandMark: `status`, `user_id` only — **no proof/rejected_reason**.
- Story: `status`, `user_id` — **no moderation fields**.

## 17. Database Tables & Relationships
| Table | Submission cols | Moderation cols |
|-------|-----------------|-----------------|
| places/stores/events/zad_elgadels | added_by_user, user_id, status, active | ownership_proof_file, rejected_reason |
| land_marks | user_id, status, active | — (none) |
| stories | user_id, status | — (auto active) |

- `added_by_user` indexed; user_id FK. No status enum/index.

## 18. Validation
- Properties create: content validation + `ownership_proof_file` required (PropertiesController:84). Status/active NOT client-settable (server sets).
- Landmarks/Stories: no proof; status server-set.
- No FormRequest; inline validators.

## 19. Auth & Permissions
- All `my*` + create + delete endpoints auth:api (controller `auth('api')->check()`).
- Story delete verifies ownership (`user_id != auth → 422`). ✅
- Places/Stores/Events/Zads: **no API edit/delete-own endpoints found** (managed in dashboard) → mobile users can't edit/withdraw submissions.

## 20. Error Handling
- Flutter `Either<Failure,T>`; hub cubit Error state.
- Laravel: validation 422; auth check → unauthorized; story delete 422 on non-owner.

## 21. Edge Cases
- Status casing mismatch: badge expects `accepted`; create sets `pending`; stories `active` → approved stories may render as "unknown"/wrong badge.
- Landmarks lack rejected_reason → user never sees why rejected.
- Stories auto-approved → different trust model, no pending state.
- No edit endpoint → user can't fix a rejected property, only resubmit.
- Hub loads all types in one call → large payload; local-only filtering.
- ownership_proof_file exposed in list resources (see §26).

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dartz, dio, infinite_scroll_pagination, FormDataHelper.
- **Laravel:** Eloquent, Spatie Media (proof file), `ApiModalController`.
- **Cross-feature:** aggregates [[04_Places]], [[05_Stores]], [[09_Events]], [[06_Restaurants]] (zad); relates [[12_Landmarks]], [[16_UserStories]], [[11_TourGuides]]. Status ties to [[26_DashboardAdmin]] moderation.

## 23. Files Involved
**Flutter:** my_properties (view, status_badge widget, MyPropertiesCubit+state, MyPropertiesResponse/PropertyItem, properties_repo_imp), my_land_mark_view, routes (kMyPropertiesView), end_points, service_locator.
**Laravel:** PropertiesController, LandMarksController@myLandMarks, StoryController (myStories/myLastDayStories/delete), GuideController own, routes/api.php:76/169/175–176/178/192–200, proof-of-ownership migrations (per type), PlaceListResource:78–80 / StoreListResource:68–70 / ZadListResource:73–75 / EventListResource:73–75 (expose status/proof/reason), LandMarkResource (no status), StoryResource:17 (status only).

## 24. Full Execution Flow (UI → DB → UI)
**Hub view:** `MyPropertiesView` → `MyPropertiesCubit` → `GET /my-properties` → `PropertiesController@index` → auth → 4× `where(added_by_user=1, user_id=auth)->paginate` → typed buckets w/ status+proof+reason → PropertyItem[] → tabs filter locally → cards + StatusBadge.
**Submit:** create form → POST `/properties` multipart (+ownership_proof_file) → autoSetData sets user_id/status='pending'/active=0 → row created → appears in hub as pending.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Hub returns all 4 types in one response (4 paginated queries + large payload) | 🟡 Med | PropertiesController@index |
| No index on status | 🟢 Low | migrations |
| Local filtering means full dataset fetched | 🟢 Low | MyPropertiesCubit |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| `ownership_proof_file` + `rejected_reason` in **list** resources (shipped in list payloads, not just owner detail) | 🟡 Med | PlaceListResource:78–80 (+Store/Zad/Event) — leaks if any list not user-scoped |
| No API edit/delete-own for properties → withdraw impossible; but also no IDOR surface | 🟢 Low | controllers |
| Story delete ownership-checked ✅ | OK | StoryController:120 |
| Landmarks no rejected_reason → opaque moderation | 🟢 Low | schema |
| Inconsistent status enum (accepted/pending/active) | 🟡 Med | client vs backend |

## 27. Technical Debt
- Hybrid architecture: unified hub for 4 types + scattered per-type endpoints for landmarks/stories/guides → no single "my content" contract.
- Status values not standardized (accepted vs active vs pending) across types.
- Moderation fields present only on some types (proof/reason on properties; none on landmarks/stories).
- No edit-own-content API; resubmission only.
- Proof/reason exposed in list resources.

## 28. Improvement Opportunities
- Standardize a status enum (pending/accepted/rejected) + normalize on serialize; map stories' `active`→`accepted`.
- Add moderation fields uniformly OR document per-type intent.
- Add edit-own + soft-withdraw endpoints with ownership checks (mirror story delete).
- Move ownership_proof_file/rejected_reason to owner-only detail resource, strip from public lists.
- Truly unify: one `/my-content?type=` endpoint covering all UGC types.

---

## 32. Related Features
- **[[04_Places]] [[05_Stores]] [[09_Events]] [[06_Restaurants]]** — submittable types in the hub.
- **[[12_Landmarks]] [[16_UserStories]] [[11_TourGuides]]** — separate "my X" flows.
- **[[26_DashboardAdmin]]** — where moderation (accept/reject/reason) happens.
- **[[01_Authentication]]** — user_id ownership.
- **[[23_MediaGallery]]** — ownership_proof_file storage.

## 33. How to Modify This Feature
- **Add a type to the hub:** ensure model has added_by_user/user_id/status; add its query to `PropertiesController@index`; add tab + local filter in `my_properties_view.dart` + PropertyItem mapping.
- **Fix badge mismatch:** normalize backend status in resource, or map in `status_badge.dart` (add `active→accepted`).
- **Add edit-own:** new `PUT /properties/{id}` with `where user_id=auth` guard; decide if edit resets status→pending.

## 34. Regression Checklist
- [ ] Submitted property appears in hub as pending.
- [ ] Approved item shows correct badge (after casing fix).
- [ ] Rejected item shows rejected_reason (where supported).
- [ ] Story auto-active shows appropriately.
- [ ] Non-owner cannot delete a story (422).
- [ ] my* endpoints reject unauthenticated.
- [ ] Landmarks appear in their own "my" list.

## 35. Common Bugs
- Approved item shows "unknown"/pending badge (status casing).
- User confused by missing rejection reason on landmarks.
- Large hub payload / slow load with many submissions.
- Can't edit a rejected submission (no endpoint).
- Proof file URL visible where lists aren't user-scoped.

## 36. Debug Guide
- **Badge wrong:** log raw `status` string from API vs `status_badge.dart` cases.
- **Item missing from hub:** verify `added_by_user=1` AND `user_id` set on create.
- **Story not deletable:** confirm auth token → `auth()->id()` matches `story.user_id`.
- **Proof leak:** grep list resources for `ownership_proof_file`.
- **Landmark status:** landmarks lack rejected_reason — expected.

## 37. Search Keywords
`my-properties`, `MyPropertiesView`, `MyPropertiesCubit`, `PropertiesController`, `added_by_user`, `ownership_proof_file`, `rejected_reason`, `status pending accepted rejected active`, `myLandMarks`, `myStories`, `myLastDayStories`, `StatusBadge`, `PropertyItem`, ممتلكاتي, مساهماتي, حالة الطلب, مرفوض.

## 38. Future Improvements
- One unified `/my-content` endpoint + uniform status/moderation schema across all UGC types.
- Edit/withdraw own submissions + notify on accept/reject.
- Owner-only exposure of proof/reason; audit trail of moderation.
- Standard status enum shared Flutter↔Laravel.
