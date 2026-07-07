# Feature 26 — Dashboard / Admin CRUD & Content Moderation (backend web)

> **Status:** ✅ 100% analyzed · **Priority:** P2 (backend-only) · **Category:** Admin
> **Laravel:** `app/Http/Controllers/Dashboard/*` + `routes/web.php` + Blade views · **Flutter:** NOT involved (session/Blade web dashboard, separate from the Sanctum API the app uses)
> **One-line:** Session-based (`web` guard) Blade admin dashboard with per-entity CRUD (Places/Stores/Events/Zad/Swalef/Guides/Sliders/Users/Roles…), Spatie Permission (`read/create/update/delete-{entity}`) applied **per-controller in `__construct()`** (route group has **no global `auth` wrapper** — fragile), and content moderation via `updateStatus` (status pending/accepted/rejected + rejected_reason) and `activate` (active 0/1). CSRF ✅. Key risks: the **`monarx-analyzer.php` RCE artifact** (CRITICAL, remote-code eval — flagged for deletion), `Place` model `guarded=[]` (mass-assignment), **status vs active conflated** (marketing push fires on `active=1` regardless of moderation status), **no moderation audit trail**, and **no permission-escalation check** in RolesController.

---

## 1. Feature Name & Identity
- **Name:** Admin Dashboard — Laravel web CRUD + moderation, NOT exposed to mobile.
- **Guard split:** `web` (session) = dashboard; `api` (Sanctum) = Flutter. Clean separation, no cross-contamination.
- **Not a Flutter feature** — included for completeness (moderation drives F15 submissions + F17 broadcast).

## 2. Business Goal & Rules
- **Goal:** Staff manage all content, moderate user submissions, manage users/roles/permissions, sliders, notifications, and internal modules (guard/visits/cars).
- **Rules:**
  - Auth via web session; permissions gate actions per entity.
  - Moderation: set `status` (pending/accepted/rejected) + `rejected_reason` (required if rejected); `active` toggled separately.
  - Activation (`active=1`) triggers marketing broadcast ([[17_Notifications]]) regardless of `status`.

## 3. User Flow (admin)
1. Login (`/login`, web guard) → dashboard index.
2. Navigate entity list (DataTables/Blade) → create/edit/delete (permission-gated).
3. Moderate submission: open pending item → set status accepted/rejected (+reason) and/or activate → save.
4. Manage users/roles/permissions, sliders, notifications, internal modules.

## 4. Screens / Views (Blade)
`resources/views/dashboard/`: index, per-entity (places/stores/events/…), users (CRUD+roles/permissions), settings (categories/regions/cities), tasks/guard/notifactions. Moderation UI in entity show/edit views.

## 5. Widgets
- Blade + DataTables (server-side lists), forms with CSRF tokens, approve/reject forms, role-permission checkbox matrices.

## 6. Cubits / Blocs
- N/A (server-rendered, no Flutter).

## 7. State Flow
- Standard MVC request→controller→model→Blade. No client state store.

## 8. Models
- All content models (Place/Store/Event/ZadElgadel/Swalef/Guide/LandMark/Slider), User, Spatie Role/Permission, Notification, AuditLog/History (exist but not populated by moderation).
- **Mass-assignment:** `Place` `$guarded=[]` (21) 🔴; `Store` `$guarded;` (malformed) ⚠️; `Event` uses `$fillable` (36) ✅.

## 9. Repositories
- None (direct Eloquent in controllers).

## 10. Services / Helpers
- `MarketingPushDispatcher` (53–72): onCreated `qualifiesForCreated` (active=1), onUpdated `qualifiesForActivation` (active→1) → dispatch broadcast. **No status check.**
- `handleActivation()`/`handleDeactivation()` (UsersPlacesController trait) set active + points reward.

## 11. API Endpoints
- N/A — web routes, not API. (Dashboard `NotificationController@index`/`destroy` are web, auth:web.)

## 12. Request / Response Models
- Blade forms → controller; responses are redirects/views. No JSON envelope.

## 13. Laravel Routes
[`routes/web.php`](../../../hawdaj-api/routes/web.php) `:1–312`:
- `Auth::routes(['verify'=>false])` (24).
- Dashboard group `['as'=>'dashboard.','namespace'=>'Dashboard']` (34–36) — **NO `middleware=>['auth']` at group level**; localization + `maintanis` only.
- Moderation/action routes: `users_places/{id}/approve` (52), `update_status/{id}` (54,63,74,88,114…), activate, destroy_selected.
- Auth enforced per-controller `__construct` `$this->middleware(['auth'])` (27–30) + `permission:*`.

## 14. Laravel Controllers
`app/Http/Controllers/Dashboard/`:
- PlacesController (CRUD + `updateStatus` 135–160, `destroy_selected` 424–436), StoreController, EventController, ZadElgadelController, SwalefController, GuidesController, SliderController, MenuController, UsersPlacesController (approve 79–101, activate 126–141), Users/UsersController (CRUD+assignRole 79–100), Users/RolesController (create 60–120, syncPermissions 108), Users/PermissionsController, NotificationController (1–30), GuardController, TaskController, MailTemplateController. Nested: Settings/Visits/Materials/Contracts/Cars.
- Per-controller permission middleware pattern: `permission:read-{entity}` etc. (PlacesController 27–31).

## 15. Laravel Services
- MarketingPushDispatcher (broadcast on activate). Auth via Laravel `AuthenticatesUsers` (LoginController 9–28, redirect `/` localized ar, no role check).

## 16. Laravel Models
- Spatie: Role, Permission (+ pivot). Seeded roles: admin (admin@hawdaj.net), supervisor, employee, guard, contract_manager (RoleSeeder 22,73,76). **No explicit super-admin/root hierarchy.**

## 17. Database Tables & Relationships
- Spatie: permissions, roles, model_has_permissions, model_has_roles, role_has_permissions (`2021_04_19_011226_create_permission_tables.php`).
- Moderation fields `rejected_reason`/`ownership_proof_file`/`status`/`active` via `2025_11_07_*_add_proof_of_ownership_to_*` migrations (places/events/stores/zad_elgadels).
- AuditLog/History tables exist, unused by moderation.

## 18. Validation
- `updateStatus`: status in (pending,accepted,rejected); `rejected_reason` required if rejected.
- User/Role create: manual `$request->except([...])` whitelist. `destroy_selected`: accepts array OR json_decode string, **no `*.integer` rule**.

## 19. Auth & Permissions
- Web session guard. Permission middleware per controller/action (Spatie).
- **Gaps:** no group-level auth (fragile if a controller omits `__construct` middleware); **no escalation check** — an admin can create a role and `givePermissionTo`/`syncPermissions` ANY permission (RolesController 68,108) → self-elevation; `assignRole` unvalidated (UsersController 93).

## 20. Error Handling
- Standard Laravel validation redirects + flash. Moderation lacks audit logging.

## 21. Edge Cases
- `active=1` without `status=accepted` (or vice versa) → inconsistent state; marketing push still fires on active.
- Role created with `delete-user`/`update-role` assigned to low role → privilege escalation.
- destroy_selected with non-integer ids → type juggling risk.
- Controller missing `__construct` auth → exposed route (no group guard).

## 22. Dependencies
- **Laravel:** spatie/laravel-permission, Blade, DataTables, Auth scaffolding.
- **Cross-feature:** moderates [[15_MyProperties]] submissions (status/active/rejected_reason); triggers [[17_Notifications]] marketing broadcast; manages [[04_Places]]…[[12_Landmarks]], [[18_Taxonomy]], [[27_RolesPermissions]] (its own deep-dive), sliders ([[02_Home]]).

## 23. Files Involved
**Laravel:** Dashboard/* controllers, routes/web.php, LoginController, config/auth.php (38–49), config/permission.php, Kernel.php (78–80 middleware, 40 CSRF), RoleSeeder, permission-tables migration, proof-of-ownership migrations, MarketingPushDispatcher, resources/views/dashboard/*, **monarx-analyzer.php (RCE artifact)**.

## 24. Full Execution Flow (moderation)
Admin opens pending Place (submitted via [[15_MyProperties]] with status=pending/active=0) → `PlacesController@updateStatus($id)` → validate status(+reason) → `$place->update(['status'=>'accepted','rejected_reason'=>null])` → (separately) `activate` sets `active=1` → model `updated` event → `MarketingPushDispatcher::onUpdated` → `qualifiesForActivation` (active→1) → `BroadcastMarketingNewContentJob` → FCM (if configured; currently null-driver-ish for realtime but FCM channel real per [[17_Notifications]]). **No approved_by/at recorded.**

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Bulk delete + activate unthrottled | 🟢 Low | destroy_selected, activate |
| DataTables server-side (fine) | 🟢 | dashboard lists |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **`monarx-analyzer.php` RCE artifact** (fetches+evals remote, self-deleting) | 🔴 CRITICAL | monarx-analyzer.php:1–104 |
| No group-level `auth` on dashboard routes (per-controller only) | 🟠 HIGH | web.php:34–36 |
| **Place `guarded=[]` mass-assignment** (any column via admin update) | 🟠 HIGH | Place.php:21 |
| **Permission escalation** — no check who can grant which permission | 🟡 Med | RolesController:68,108 |
| No moderation audit trail (who/when approved/rejected) | 🟡 Med | moderation controllers |
| status vs active conflated; push on active regardless of status | 🟡 Med | MarketingPushDispatcher |
| destroy_selected ids not integer-validated | 🟢 Low | PlacesController:424–436 |
| CSRF enabled ✅ | OK | Kernel:40 |
| Store `$guarded;` malformed | 🟢 Low | Store.php:22 |

## 27. Technical Debt
- RCE artifact must be removed (see [[MEMORY]]-worthy security item).
- Fragile per-controller auth (no group middleware).
- Mass-assignment on Place (+ malformed Store guarded).
- Moderation lacks audit + conflates status/active.
- Role system lacks hierarchy/escalation guard.
- AuditLog/History tables unused.
- Mixed internal modules (guard/visits/cars/contracts) in same dashboard.

## 28. Improvement Opportunities
- **Delete `monarx-analyzer.php` immediately.**
- Wrap dashboard route group in `middleware(['auth', 'localization', ...])`.
- Replace Place `guarded=[]` with explicit `$fillable`; fix Store.
- Log approved_by/at + rejected_by/at to AuditLog; separate status (moderation) from active (visibility); gate marketing on status=accepted.
- Add escalation check: assigner must hold a permission to grant it; introduce super-admin role.
- Validate destroy_selected ids as `array|*.integer`; throttle bulk actions.

---

## 32. Related Features
- **[[15_MyProperties]]** — submissions moderated here.
- **[[17_Notifications]]** — activation triggers marketing broadcast.
- **[[27_RolesPermissions]]** — Spatie roles/permissions (own deep-dive).
- **[[04_Places]]…[[12_Landmarks]] [[18_Taxonomy]] [[02_Home]]** — content managed here.
- **[[28_CarModule]] [[29_ReportModule]] [[30_VisitorTracking]]** — internal dashboard modules.

## 33. How to Modify This Feature
- **Add a moderated entity:** create Dashboard controller with permission middleware + updateStatus/activate; add routes; Blade views; ensure status/active/rejected_reason columns.
- **Harden auth:** add group middleware in web.php.
- **Add audit:** in updateStatus, write to AuditLog with actor + timestamp.
- **Fix mass-assignment:** set explicit fillable on Place/Store.

## 34. Regression Checklist
- [ ] Login required for all dashboard routes.
- [ ] Permission gates block unauthorized actions.
- [ ] Approve sets status=accepted; reject requires reason.
- [ ] Activation triggers broadcast.
- [ ] CSRF enforced on forms.
- [ ] (After fix) audit records actor.
- [ ] (After fix) monarx artifact gone.

## 35. Common Bugs
- Route exposed if controller misses `__construct` auth.
- Inconsistent status/active (approved-but-inactive or active-but-pending).
- Admin self-escalates via role permissions.
- Marketing fires unexpectedly on activation.
- Mass-assignment injecting unintended columns.

## 36. Debug Guide
- **Unauthorized action allowed:** check controller permission middleware + user role.
- **Broadcast not firing:** check active flip + MarketingPushDispatcher qualifies + FCM config.
- **Moderation not saving:** validate status enum + rejected_reason.
- **Route open:** confirm `__construct` middleware present.
- **RCE concern:** confirm monarx-analyzer.php removed.

## 37. Search Keywords
`Dashboard controllers`, `routes/web.php`, `auth:web`, `Spatie Permission`, `permission:read-places`, `updateStatus`, `rejected_reason`, `status accepted rejected`, `activate active`, `UsersPlacesController approve`, `RolesController syncPermissions`, `guarded []`, `MarketingPushDispatcher`, `monarx-analyzer.php`, `AuditLog`, لوحة التحكم, الإدارة, المراجعة, الأدوار.

## Known Incidents (live)
- **[[KNOWN_ISSUES#K4]] — إنشاء المستخدم/المشرف لا تصله كلمة المرور:** `userMailNotification implements ShouldQueue` (يحتاج `queue:work`) + `UsersController@store` يبتلع فشل البريد صامتًا (~L95–98) + احتمال SMTP ([[KNOWN_ISSUES#K1]]). تذكرة: `hawdaj-api/TICKET_supervisor_login_email.md`.
- **[[KNOWN_ISSUES#K3]] — تعديل الأدوار/الصلاحيات لا يسري:** كاش Spatie 24h بلا flush → استخدم `permission:cache-reset`. دَين [[TECH_DEBT#D46]].

## 38. Future Improvements
- Remove RCE artifact; harden route auth; audit-logged moderation; status/active separation; role hierarchy + escalation guards; explicit fillables; throttled bulk ops; move internal modules to a distinct admin area.
- بريد إنشاء المستخدم: سجّل الفشل بدل ابتلاعه + اعرض كلمة المرور للأدمن كخطة بديلة + تأكّد queue worker.
