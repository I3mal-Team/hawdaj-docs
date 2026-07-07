# Feature 27 — Roles & Permissions (Spatie)

> **Status:** ✅ 100% analyzed · **Priority:** P2 (backend-only) · **Category:** Admin
> **Laravel:** Spatie laravel-permission (PermissionSeeder + RolesService + Dashboard controllers) · **Flutter:** NOT involved (no .dart usage; pure backend authz)
> **One-line:** 145 permissions (29 modules × 5 ops read/create/update/delete/patch, naming `{op}-{module}`) all seeded on the **`web` guard only**. **All 7 roles (root/admin/customer/employee/guard/supervisor/contract_manager) get ALL 145 permissions** — role-based access is effectively cosmetic. Enforced only on the **dashboard** (permission:/role: middleware + `@can`/`@role`); the **mobile API enforces ZERO granular authorization** — every `auth('api')` route is open to any authenticated user. No `Gate::before` super-admin bypass (roles hold all perms explicitly). Seeded users have hardcoded password `123456`. Guard mismatch means the API couldn't use these permissions even if it wanted to.

---

## 1. Feature Name & Identity
- **Name:** Roles & Permissions — Spatie authorization layer.
- **Scope:** dashboard (web) authorization only. API is authentication-only (no authz).
- **Not a Flutter feature** — but critical to understanding the app's security model.

## 2. Business Goal & Rules
- **Goal:** Fine-grained staff authorization on the dashboard.
- **Rules (as implemented):**
  - 145 permissions, `{op}-{module}`, guard `web`.
  - Roles created + granted all permissions by RolesService (no differentiation).
  - Dashboard controllers gate actions by `permission:{op}-{module}`.
  - API: `auth('api')->check()` only — no permission/role checks anywhere.

## 3. User Flow (admin)
- Login (web) → role determines Blade `@can`/`@role` visibility → controller permission middleware allows/denies action. Admin can create roles + sync permissions at runtime (PermissionsController/RolesController).

## 4. Screens / Views
- Dashboard users/roles/permissions Blade views (CRUD + checkbox permission matrix). `@can` (20 uses) + `@role` (3 uses, e.g. sites.blade `@role('guard')`).

## 5. Widgets
- Role-permission checkbox matrix, role assignment dropdowns (Blade).

## 6. Cubits / Blocs
- N/A.

## 7. State Flow
- Standard request → permission middleware → controller. Spatie cache (24h) between changes.

## 8. Models
- `User` (HasRoles trait, User.php:14–22), `Role`, `Permission` (Spatie). guard_name web.

## 9. Repositories
- `RolesService` acts as the seeding/assignment service (not a repo).

## 10. Services / Helpers
- [`RolesService.php`](../../../hawdaj-api/app/Services/RolesService.php): `CreateRole([...])` (78–90), `AssignModelPermissionsToRole()` (140–151), permissions guard web (123,129). Grants ALL permissions to root/admin/customer + created roles.

## 11. API Endpoints
- **None permission-gated.** `routes/api.php` (200+ routes) uses `auth('api')` only. **0 occurrences** of permission:/role:/hasPermissionTo/hasRole/can() in `app/Http/Controllers/Api`.

## 12. Request / Response Models
- Role create/update: `{name, label, permissions[]}`. No API contract (dashboard only).

## 13. Laravel Routes
- Dashboard: permission middleware per controller (web.php + `__construct`). API: no authz middleware.
- Spatie middleware registered Kernel.php:49–50 (`permission`, `role`, `role_or_permission`).

## 14. Laravel Controllers
- Dashboard/* (EventController:18–26, SwalefController, 15+): `$this->middleware('permission:{op}-{module}')`.
- Users/RolesController (givePermissionTo 68, syncPermissions 108 — no escalation check, see [[26_DashboardAdmin]]), UsersController (assignRole/syncRoles), PermissionsController (runtime perm→role), VendorController (assignRole).
- API controllers: **no authz** (PlaceController, ProfileController, etc.).

## 15. Laravel Services
- RolesService (catalog + assignment). AuthServiceProvider **EMPTY** (no Gate::before/define, no policies).

## 16. Laravel Models
- Spatie Role/Permission + pivots. User HasRoles. All guard web.

## 17. Database Tables & Relationships
- `permissions`, `roles`, `model_has_permissions`, `model_has_roles`, `role_has_permissions` (`2021_04_19_011226_create_permission_tables.php`).
- 145 permissions rows (29 modules × 5 ops). 7 roles, each linked to all permissions.

## 18. Validation
- Role/permission create via manual `$request->except([...])`. Permissions granted unvalidated (any id → any role).

## 19. Auth & Permissions
- **Dashboard:** web guard + Spatie enforcement ✅.
- **API:** Sanctum authentication ONLY — no role/permission gate. Any logged-in user hits any API route (subject to per-controller `auth('api')` checks, some of which are also missing per [[14_RatesReviews]]/[[21_ProfileSettings]]).
- **Guard mismatch:** permissions guard=web → cannot be checked on api guard even if added.
- No `Gate::before` bypass; root/admin get all perms explicitly.

## 20. Error Handling
- Spatie middleware → 403 on missing permission (dashboard). API never 403s on authz (none exists).

## 21. Edge Cases
- All roles identical (all permissions) → assigning a "limited" role grants everything.
- Permission cache 24h → new permission/role changes may not take effect until flush.
- Guard mismatch → adding api-guard checks requires re-seeding permissions for api guard.
- Runtime permission grants unvalidated → escalation.
- Seeded hardcoded creds if not changed in prod.

## 22. Dependencies
- **Laravel:** spatie/laravel-permission.
- **Cross-feature:** [[26_DashboardAdmin]] (uses these gates), [[01_Authentication]] (User identity), all Dashboard-managed content.

## 23. Files Involved
**Laravel:** config/permission.php (96 guard, 109–142 cache), PermissionSeeder.php, RoleSeeder.php (28–77 users), RolesService.php, User model, Role/Permission models, Kernel.php:49–50, Dashboard controllers (permission middleware), Users/RolesController + PermissionsController + UsersController, AuthServiceProvider (empty), Blade @can/@role, routes/web.php (dashboard), routes/api.php (no authz).

## 24. Full Execution Flow (authz)
**Dashboard:** login (web) → request to `dashboard.places.store` → PlacesController `__construct` `permission:create-places` → Spatie checks user roles→permissions (cached) → allow/403.
**API:** app request → `auth('api')->check()` in controller → if authenticated, proceed (NO permission check) → response. Same for all users regardless of role.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Permission cache 24h (stale after changes) | 🟡 Med | config/permission.php:109–142 |
| Loading all-permissions roles (large pivot) | 🟢 Low | RolesService |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **API has ZERO authorization** (auth-only, any user any route) | 🔴 CRITICAL | app/Http/Controllers/Api/* |
| **All 7 roles get all 145 permissions** (RBAC ineffective) | 🔴 HIGH | RolesService/PermissionSeeder |
| **Guard mismatch** (perms web-only, unusable on api) | 🔴 HIGH | PermissionSeeder:96, RolesService:123 |
| **Hardcoded seeded creds `123456`** (root/admin/…) | 🔴 HIGH | RoleSeeder:28–77 |
| Runtime permission grant no escalation check | 🟡 Med | RolesController:68,108 |
| Cache staleness on changes | 🟡 Med | config |
| No Gate::before misuse (explicit grants) ✅ | OK | AuthServiceProvider empty |
| Direct user→permission not used (via roles) ✅ | OK | controllers |

## 27. Technical Debt
- Roles undifferentiated (all-permissions) → RBAC exists on paper only.
- API entirely unauthorized beyond authentication.
- Web-guard-only permissions block any future api authz without re-seed.
- Empty AuthServiceProvider (no policies/gates).
- Hardcoded demo credentials in seeder.
- Unvalidated runtime permission grants.

## 28. Improvement Opportunities
- Differentiate roles (least privilege per role) instead of granting all.
- Add API authorization where sensitive (e.g., delete/moderate) — but note API is user-facing; introduce a proper policy for admin-ish API actions or keep them dashboard-only.
- Seed permissions for both guards (or use a guard-agnostic approach) if API authz desired.
- Rotate/randomize seeded credentials; force change on first login; never ship 123456 to prod.
- Validate permission grants against granter's own permissions (escalation guard).
- Reduce cache TTL or flush on change.

---

## 32. Related Features
- **[[26_DashboardAdmin]]** — consumer of these gates; moderation actions.
- **[[01_Authentication]]** — user identity (both guards).
- **[[15_MyProperties]]** — moderation authz.
- All Dashboard-managed content features.

## 33. How to Modify This Feature
- **Add a permission:** add module to PermissionSeeder loop (op-module); assign to intended roles (edit RolesService to stop granting all).
- **Restrict a role:** stop `AssignModelPermissionsToRole` blanket grant; assign explicit subset.
- **Add API authz:** re-seed permissions for `api` guard + add `permission:` middleware (Sanctum) OR policies; decide which API actions need it.
- **Flush cache after change:** `php artisan permission:cache-reset`.

## 34. Regression Checklist
- [ ] Dashboard action blocked without permission (403).
- [ ] Blade hides unauthorized controls.
- [ ] Role change reflects after cache flush.
- [ ] (After fix) roles have differentiated permissions.
- [ ] (After fix) seeded creds changed in prod.
- [ ] (If added) API authz enforced on sensitive routes.

## 35. Common Bugs
- Permission change not taking effect (24h cache).
- "Limited" role can do everything (all-perms seeding).
- Cannot enforce permission on API (guard mismatch).
- Escalation via runtime grants.
- Default creds still active.

## 36. Debug Guide
- **Permission not enforced (API):** expected — API has no authz.
- **Role has too much access:** check RolesService blanket grant.
- **Change not applied:** run `permission:cache-reset`.
- **403 on dashboard:** verify role→permission link + guard web.
- **Login with 123456 works:** seeded creds not rotated.

## 37. Search Keywords
`Spatie permission`, `HasRoles`, `permission:read-places`, `@can`, `@role`, `PermissionSeeder`, `RoleSeeder`, `RolesService`, `AssignModelPermissionsToRole`, `guard_name web`, `assignRole`, `syncPermissions`, `givePermissionTo`, `AuthServiceProvider`, `permission:cache-reset`, `123456 seeded`, `api no authorization`, الأدوار, الصلاحيات.

## Known Incidents (live)
- **[[KNOWN_ISSUES#K3]] — الصلاحيات الممنوحة لا تسري:** `config/permission.php` كاش 24h، و**لا `forgetCachedPermissions()` في الكود**. أعطاء دور/صلاحية لا يظهر إلا بعد 24h. حلّ فوري: `php artisan permission:cache-reset`؛ جذري: استدعاء `forgetCachedPermissions()` بعد أي تعديل (RolesController/PermissionsController/UsersController). دَين [[TECH_DEBT#D46]].

## 38. Future Improvements
- True least-privilege RBAC with differentiated roles + policies.
- استدعاء `forgetCachedPermissions()` بعد كل تعديل أدوار/صلاحيات (يزيل التباس الكاش).
- Guard-consistent permissions + optional API authorization layer for admin-grade API actions.
- Secure credential seeding + rotation; escalation-safe permission management; shorter/flushed cache.
