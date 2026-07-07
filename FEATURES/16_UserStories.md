# Feature 16 — User Stories (قصص المستخدم / آخر 24 ساعة)

> **Status:** ✅ 100% analyzed · **Priority:** P2 · **Category:** Supporting (ephemeral UGC)
> **Flutter:** `lib/features/user_stories/` · **Laravel:** `StoryController` + `Story` model + `stories` table
> **One-line:** An Instagram/WhatsApp-style ephemeral "stories" feature, but **half-built**: the 24h window is enforced only by a read-time `created_at >= now()-1day` query (no `expires_at`, no cron cleanup), there is **NO seen/viewed tracking** (no `story_views` table), the Flutter list endpoint is **mis-mapped to `get-profile-page`** instead of `stories/myLastDayStories`, video mimes are accepted server-side but the client renders images only, and stories are auto-approved (`status='active'`). NOT the Swalef heritage stories ([[10_Swalef]]).

---

## 1. Feature Name & Identity
- **Name:** User Stories — short-lived image (nominally video) posts visible for 24 hours.
- **Distinct from [[10_Swalef]]** (heritage narrative content). This is `Story` model / `stories` table.
- **Reality check:** implemented as a simple grid of the user's own last-day stories with add/download/delete — not a full cross-user story rail + viewer with progress bars + seen state.

## 2. Business Goal & Rules
- **Goal:** Let users post ephemeral media that surfaces for 24h.
- **Rules (as implemented):**
  - `store()` sets `status='active'` (auto-approved, no moderation).
  - `myLastDayStories()` returns own stories with `created_at >= Carbon::now()->subDay()`.
  - Delete allowed only by owner (`user_id == auth()->id`).
  - Media: mimes jpeg/png/jpg/gif/svg/webm, max 2048KB.
  - All endpoints require `auth('api')->check()` (inline, route group unprotected).
- **Not enforced:** true expiry/cleanup (rows persist), seen tracking, video playback, per-user grouping/rail.

## 3. User Flow
1. User opens Stories screen (`UserStoriesView`) → `UserStoriesCubit.fetchMyLastDayStories()`.
2. Grid of own last-24h stories (`UserStoryItem` cards: image + download + delete).
3. "Add Story" → `AddStoryCubit.addStory(file, title)` → POST `stories/store`.
4. Delete own story → `RemoveStoryCubit.removeStory(id)` → DELETE `stories/{id}/delete`.
- **No full-screen viewer, no progress bars, no seen state, no expiry countdown UI.**

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| [`user_stories_view.dart`](lib/features/user_stories/presentation/views/user_stories_view.dart) | 21–313 | GridView of stories, auth gate, "Add Story" button |
| [`user_story_item.dart`](lib/features/user_stories/presentation/widgets/user_story_item.dart) | 16–157 | Card: image + download + delete; no expiry countdown, no video player |

## 5. Widgets
- `UserStoryItem`: image via `Image.network`, download + delete actions. No video rendering, no progress/seen indicator.
- No horizontal circle rail widget (feature is a personal grid, not a social rail).

## 6. Cubits / Blocs
| Cubit | File:Lines | Method |
|-------|-----------|--------|
| UserStoriesCubit | `user_stories/.../user_stories_cubit.dart:13–36` | `fetchMyLastDayStories()` → repo |
| AddStoryCubit | `user_stories/.../add_story_cubit.dart:11–28` | `addStory(file,title)` |
| RemoveStoryCubit | `user_stories/remove/remove_story_cubit.dart:12–20` | `removeStory(storyId)` (ownership enforced server-side) |

## 7. State Flow
- Fetch → cubit calls repo `fetchMyLastDayStories()` → parses `ProfilePageData.myStories` (paginated) → grid renders.
- Add → FormData POST → success → refresh list.
- Remove → DELETE → success → refresh.

## 8. Models
| Model | File:Lines | Fields |
|-------|-----------|--------|
| UserStoryModel | `user_stories/data/models/user_story_model.dart:1–56` | id, type(text/file), file, text, status, totalViews, totalLikes, totalComments, totalShares. **No created_at/expires_at/seen** |
| UserStoriesResponse / ProfilePageData | `user_stories/data/models/user_stories_response.dart:1–150` | wraps whole profile page (myStories paginated + PersonalData + social + trips) — over-broad payload |

- **⚠️** Client cannot compute expiry (no timestamp parsed). Response leaks full profile data (see §26).

## 9. Repositories
| Method | File:Lines | HTTP |
|--------|-----------|------|
| fetchMyLastDayStories | `user_stories_repository_impl.dart:18–23` | GET `EndPoints.myLastDayStories` (**mis-mapped to `get-profile-page`**) |
| addStory | `:26–69` | POST `stories/store` FormData (type='file', file, optional text=title) |
| removeStory | `:72–80` | DELETE `stories/{id}/delete` |

## 10. Services / Helpers
- `FormDataHelper`/`DioConsumer` for upload.
- Laravel `getImageUrl()` resource helper wraps `file` (small variant).
- 24h boundary: `Carbon::now()->subDay()` at query time (StoryController:53).

## 11. API Endpoints
| Method | Endpoint | Auth | Notes |
|--------|----------|------|-------|
| GET | `stories/list` | auth:api | all stories, no pagination boundary |
| GET | `stories/myStories` | auth:api | own, no time filter |
| GET | `stories/myLastDayStories` | auth:api | own, 24h window ✅ (correct query) |
| POST | `stories/store` | auth:api | auto status='active' |
| DELETE | `stories/{id}/delete` | auth:api | ownership-checked ✅ |
| GET | `stories/show/{id}` | auth:api | ⚠️ returns `LandMarkResource` (wrong resource) |

**Flutter EndPoints** `end_points.dart:108–113`: `myLastDayStories = 'get-profile-page'` ❌ (wrong); `storiesmyLastDayStories = 'stories/myLastDayStories'` (defined, unused); `addStory='stories/store'`; `deleteStory(id)='stories/{id}/delete'`.

## 12. Request / Response Models
- **Add req:** multipart `{type:'file', file, text}`.
- **Add resp:** `{message}`.
- **List resp:** `ProfilePageData` → `myStories` paginated: `[{id, order_id, type, text, file, status, total_views, total_likes, total_comments, total_shares}]` — **no user, no created_at/expires_at**.
- **Show resp:** LandMarkResource shape (mismatch).

## 13. Laravel Routes
`routes/api.php:172–178`, inside unprotected `front.` group (auth inline in controller):
```
GET    stories/list              → StoryController@index
GET    stories/myStories         → StoryController@myStories
GET    stories/myLastDayStories  → StoryController@myLastDayStories
POST   stories/store             → StoryController@store
DELETE stories/{id}/delete       → StoryController@delete
GET    stories/show/{id}         → StoryController@show
```

## 14. Laravel Controller
[`StoryController.php`](../../../hawdaj-api/app/Http/Controllers/Api/StoryController.php) `:1–129`.
| Method | Line | Logic |
|--------|------|-------|
| index() | ~ | auth; `Story::query()` all; no bounded pagination |
| myStories() | 31–44 | auth; `where user_id=auth`; no time filter |
| **myLastDayStories()** | ~46–61 (query 53) | auth; `where user_id=auth`, `where created_at >= now()->subDay()` ✅ |
| store() | ~85–105 | auth; validate type in [file,text], file mimes+max2048; `status='active'` (105) |
| delete() | ~112–127 (check 120) | auth; if `user_id != auth()->id` → 422; else hard delete ✅ |
| show($id) | ~ | auth; returns `LandMarkResource` ⚠️ wrong |

## 15. Laravel Services
None. No expiry service, no cleanup cron, no view-tracking service.

## 16. Laravel Models
[`Story.php`](../../../hawdaj-api/app/Models/Story.php) `:8–29`: fillable type, file, text, status, user_id, total_views/likes/comments/shares, order_id. `user()` belongsTo(User). **No expires_at, no seen/viewed, no story_views pivot.**

## 17. Database Tables & Relationships
[`stories`](../../../hawdaj-api/database/migrations/2024_05_14_011357_create_stories_table.php) `:14–30`: id, type, file, text, status, total_views/likes/comments/shares (all nullable), user_id (FK→users onDelete set null), timestamps.
- **Missing indexes:** none on user_id or created_at → `myLastDayStories` range scan.
- **No expires_at, no story_views table.**

## 18. Validation
`store()`: `type` in [file,text]; `file` mimes jpeg/png/jpg/gif/svg/webm max:2048. No FormRequest. `status` free string (only 'active' used). Client can't set status.

## 19. Auth & Permissions
- All 6 endpoints auth:api (inline). Route group itself unprotected.
- Delete ownership-checked (422 on non-owner) ✅.
- `stories/list` returns all users' stories to any authed user (no privacy scoping beyond auth).

## 20. Error Handling
- Flutter `Either<Failure,T>`; cubits Error state.
- Laravel: validation 422; delete non-owner 422; auth check.
- **Latent bug:** list endpoint mis-map returns whole profile page — silently "works" but ships excess data.

## 21. Edge Cases
- Stories never physically expire (persist in DB); only filtered at read → `stories/list`/`myStories` still return old ones.
- No seen tracking → nothing to re-show logic (feature-personal only anyway).
- Video mime `webm` accepted but no client player → uploaded video won't render.
- SVG allowed → stored-XSS risk if served with wrong MIME.
- Timezone: `Carbon::now()->subDay()` uses app tz (config/app.php) — boundary drift if tz not UTC.
- `show` returns wrong resource type → malformed detail.

## 22. Dependencies
- **Flutter:** flutter_bloc, get_it, dartz, dio, image_picker, FormDataHelper.
- **Laravel:** Eloquent, Carbon, `ApiModalController`, getImageUrl helper.
- **Cross-feature:** overlaps [[15_MyProperties]] (myStories/myLastDayStories listed there), [[21_ProfileSettings]] (profile page payload). Distinct from [[10_Swalef]].

## 23. Files Involved
**Flutter:** user_stories (models: user_story_model, user_stories_response; cubits: user_stories_cubit, add_story_cubit, remove_story_cubit; views: user_stories_view, user_story_item; repo impl), end_points:108–113, service_locator, routes.
**Laravel:** StoryController, Story model, stories migration, StoryListResource (+StoryCollection), routes/api.php:172–178.

## 24. Full Execution Flow (UI → DB → UI)
**Add:** `AddStoryCubit.addStory(file,title)` → POST `stories/store` multipart → `StoryController@store` → auth → validate mimes/size → `Story::create(status='active', user_id=auth)` → `{message}` → refresh list.
**View:** `UserStoriesView` → `UserStoriesCubit.fetchMyLastDayStories()` → GET (mis-mapped `get-profile-page`) → returns ProfilePageData incl. `myStories` (should be `stories/myLastDayStories` → `where user_id=auth AND created_at>=now-1day`) → grid of `UserStoryItem`.
**Delete:** `RemoveStoryCubit` → DELETE `stories/{id}/delete` → `StoryController@delete` → ownership check → hard delete → refresh.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| No index on user_id / created_at → range scan | 🔴 HIGH | stories migration |
| `stories/list` unbounded (all rows) | 🟡 Med | StoryController@index |
| Over-broad profile payload for a stories list | 🟡 Med | UserStoriesResponse / get-profile-page |
| No cleanup → table grows unbounded | 🟡 Med | no cron / no expires_at |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| SVG upload allowed (stored-XSS if wrong MIME) | 🟡 Med | store() mimes |
| Profile PII (PersonalData+social) returned in stories fetch | 🟡 Med | get-profile-page mis-map |
| `stories/list` exposes all users' stories to any authed user | 🟢 Low | index() |
| Delete IDOR-protected ✅ | OK | StoryController:120 |
| Endpoints auth-gated but route group unprotected (fragile) | 🟢 Low | routes group |

## 27. Technical Debt
- **Endpoint mis-mapping** (`get-profile-page` vs `stories/myLastDayStories`) — correct constant exists but unused.
- No expiry column / cron → "ephemeral" only at read; DB accretes.
- No seen/viewed tracking, no cross-user rail, no full-screen viewer w/ progress → feature is a stub of a stories system.
- `show()` returns `LandMarkResource` (copy-paste bug).
- Video accepted but not rendered.
- `total_*` counters present but no like/comment/share/view endpoints wire them.
- `status` free-form string.

## 28. Improvement Opportunities
- Fix endpoint mapping to `stories/myLastDayStories`; slim the response to stories only.
- Add `expires_at` (created_at+24h) + cleanup cron; return it; render countdown.
- Add `story_views` table + `mark-seen` endpoint + unseen query for a real rail.
- Include poster `user:{id,name,photo}` in StoryResource; build cross-user grouped rail + viewer.
- Add video player for webm; drop SVG or sanitize.
- Index (user_id, created_at); bound `stories/list`; fix `show()` resource.

---

## 32. Related Features
- **[[15_MyProperties]]** — myStories/myLastDayStories also surface there.
- **[[21_ProfileSettings]]** — get-profile-page payload shared.
- **[[10_Swalef]]** — different "stories" (heritage), not this.
- **[[23_MediaGallery]]** — story media upload/serving.
- **[[01_Authentication]]** — user_id ownership + profile data.

## 33. How to Modify This Feature
- **Fix list endpoint:** point `EndPoints.myLastDayStories` at `'stories/myLastDayStories'`; update response model to parse StoryCollection not ProfilePageData.
- **Add expiry:** migration `expires_at`; set on create; return it; Flutter countdown widget; cron `where('expires_at','<',now())->delete()`.
- **Add seen:** `story_views` migration + `mark-seen` route + cubit; unseen leftJoin query.
- **Fix show:** swap `LandMarkResource` → a `StoryResource`.

## 34. Regression Checklist
- [ ] Add story (image) → appears in last-day list.
- [ ] Story older than 24h excluded from myLastDayStories.
- [ ] Delete own story works; non-owner → 422.
- [ ] Unauthenticated → rejected.
- [ ] After endpoint fix: list returns only stories, not full profile.
- [ ] webm handling defined (render or reject).
- [ ] SVG handling decided.

## 35. Common Bugs
- List returns full profile instead of stories (mis-map).
- Uploaded video shows blank (no player).
- Old stories linger in `stories/list`/`myStories` (no cleanup).
- `show` returns landmark-shaped JSON.
- Slow list at scale (no index).

## 36. Debug Guide
- **List returns odd data:** check `EndPoints.myLastDayStories` value (`get-profile-page`?).
- **Story not expiring:** expected — only `myLastDayStories` filters; no deletion.
- **Delete fails:** confirm token → `auth()->id()` == `story.user_id`.
- **Video blank:** client has no player; check mime.
- **Slow query:** add index on (user_id, created_at); inspect explain.

## 37. Search Keywords
`story`, `Story`, `stories`, `StoryController`, `myLastDayStories`, `myStories`, `stories/store`, `stories/{id}/delete`, `UserStoriesCubit`, `AddStoryCubit`, `RemoveStoryCubit`, `UserStoryModel`, `get-profile-page`, `Carbon subDay`, `expires_at`, `story_views`, `status active`, قصص, آخر ٢٤ ساعة.

## 38. Future Improvements
- Full ephemeral system: expires_at + cron, seen tracking, cross-user grouped rail, full-screen viewer with progress bars, reactions/replies (wire total_* counters).
- Video-first support; media sanitization.
- Slim, correct StoryResource with poster info; correct endpoint + indexes.
