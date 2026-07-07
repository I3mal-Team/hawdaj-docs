# Feature 28 — CarModel Module (fleet / vehicle logistics — OUT OF SCOPE)

> **Status:** ✅ 100% analyzed · **Priority:** P4 · **Category:** Admin / Internal (⚠️ **OUT OF SCOPE** for the tourism product)
> **Laravel:** `Modules/CarModel/` (nwidart) + `app/Models/Car|CarRequest|Driver` + Dashboard/Cars + GuardController · **Flutter:** NOT involved (zero API exposure)
> **One-line:** A **fleet / license-plate-detection / vehicle-logistics** system bolted into the same Laravel install but **completely unrelated to tourism** and **never touched by the mobile app** (its `Routes/api.php` is fully commented out; no car controllers in `Api/`). It does camera-based plate detection + guard check-in/out + daily risk analytics, dashboard-only. Shares only the `User` model + `guard`/`contract_manager` roles. **Verdict: OUT OF SCOPE** for hawdaj mobile work — documented for completeness. Security notes: `cars.{siteId}` broadcast channel authorizes `fn()=>true` (no auth), the `takeAction` permission check is commented out, and `car_requests` has a missing migration.

---

## 1. Feature Name & Identity
- **Name:** CarModel Module — internal vehicle/fleet logistics + ANPR (automatic number-plate recognition) gate monitoring.
- **Not tourism:** distinct from Places/Trips/etc. Orthogonal domain sharing the codebase.
- **nwidart module** at `Modules/CarModel/` + supporting app-level models.

## 2. Business Goal & Rules
- **Goal (inferred):** Monitor vehicles at sites via camera plate detection; track invitation vs no-invitation, risk duration; log guard check-in/out of car requests; daily analytics + notes/audit.
- **Rules:** dashboard-only (auth + localeSessionRedirect + maintanis middleware); per-site config (start/end time/date, notification, screenshot); real-time push on detection/check-in.

## 3. User Flow (dashboard/guard)
1. Camera detects plate → `car_plates` record.
2. Dashboard live-mode shows detections per site (real-time via `cars.{siteId}`).
3. Guard checks car request in/out → `RealTimeMessage` broadcast + host notification.
4. Admin marks action success/error/pending; adds notes with attachments; exports to Report module.

## 4. Screens / Views
- Dashboard Blade: cars list per site, live-mode, car-notes CRUD, DataTables. No mobile screens.

## 5–7. Widgets / Cubits / State
- N/A (backend/Blade). Flutter uninvolved.

## 8. Models
- Module Entities: `CarPlate` (detections: camID, plate_card, detection_status, notice_time, last_risk, site_id), `CarSetting` (per-site config), `CarDay` (daily aggregates), `CarNotes` (audit + files).
- App models: `Car` (plate_ar/en, license, type, status), `CarRequest` (car_id, driver_id, delivery_date/from/to, checkin/checkout, host_id, site_id), `Driver` (contact, id_number, phone, license), `CarOfRequest` (pivot).

## 9. Repositories / 10. Services
- `Modules/CarModel/Services/CarService.php`: getStatus, getCarDay, getCamera, updateDuration (risk vs safe time, invitation flags, first_row/last_risk), handleTable (DataTables), handleExport (→ Report module ArchiveFile), noteWithAction (notes+files).

## 11. API Endpoints
- **NONE (mobile).** `Modules/CarModel/Routes/api.php` fully commented (6–8). No `Api/` car controllers. `routes/api.php` has zero car refs.

## 12. Request/Response
- Dashboard forms + DataTables JSON only. No mobile contract.

## 13. Routes
- `Modules/CarModel/Routes/web.php:4–17`: `/dashboard/cars`, `/dashboard/cars/{site}`, `/dashboard/car-table/{site}`, `/dashboard/cars/takeAction/{car}` (success|error|pending), `/dashboard/car-notes` CRUD, `/dashboard/cars/live-mode`. Middleware auth+localeSessionRedirect+maintanis.
- `Modules/CarModel/Routes/channel.php`: `Broadcast::channel('cars.{siteId}', fn()=>true)` (**no auth**).

## 14. Controllers
- `CarModelController` (index, filtered, car-table, takeAction, live-mode) — line 20 permission middleware **commented out**.
- `GuardController::carAction()` (app, ~250+): guard check-in/out CarRequest → CarRequestNotification to host.
- `CarRequestDetailsController:99`: dispatches `RealTimeMessage` on check-in (see [[25_Realtime]]).

## 15. Services
- CarService (above). Ties to Report module ([[29_ReportModule]]) via export.

## 16. Models (summary)
- See §8. Shares `User` (host/guard). No tourism relation.

## 17. Database Tables
- `car_plates` (2021_09_15_075023), `car_settings` (2021_09_15_075055), `car_days` (2021_10_12_135001), `car_notes` (2021_10_31_135157), `drivers` (2022_03_14), `cars` (2022_03_15). **`car_requests` migration MISSING** (model exists, no migration file).

## 18. Validation
- Dashboard form validation; takeAction status enum success|error|pending.

## 19. Auth & Permissions
- Web auth on routes. **`permission:update-car_plate` on takeAction commented out** → no authz. Broadcast channel `fn()=>true` → anyone (any authed) can subscribe to any site's car channel. Uses `guard`/`contract_manager` roles (from [[27_RolesPermissions]], which grant all perms anyway).

## 20. Error Handling
- Standard Laravel. Real-time on null driver (per [[25_Realtime]]) → broadcasts discarded currently.

## 21. Edge Cases
- Missing `car_requests` migration → table must exist ad-hoc (fresh install would break).
- Broadcast channel open → info leak of site car activity.
- Module isolated → no tourism impact.

## 22. Dependencies
- nwidart/laravel-modules, Spatie roles, Report module (export). Shares User.
- **Cross-feature:** [[25_Realtime]] (RealTimeMessage/cars channel — only live event source), [[26_DashboardAdmin]] (dashboard Cars/Guard), [[29_ReportModule]] (export), [[30_VisitorTracking]] (guard workflows, separate).

## 23. Files Involved
`Modules/CarModel/` (Entities: CarPlate/CarSetting/CarDay/CarNotes; Services/CarService; Http/CarModelController; Routes web/api[dead]/channel), app/Models/Car|CarRequest|Driver|CarOfRequest, GuardController, CarRequestDetailsController, migrations (car_plates/settings/days/notes/drivers/cars; car_requests MISSING), RoleSeeder (guard/contract_manager).

## 24. Full Execution Flow
Camera → plate detection → `car_plates` → CarService.updateDuration (risk/invitation) → dashboard live-mode via `cars.{siteId}` broadcast. Guard check-in → GuardController/CarRequestDetailsController → RealTimeMessage + host notification. Admin takeAction/notes/export.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| DataTables server-side (fine) | 🟢 | CarService.handleTable |
| Export queued to Report module | 🟢 | handleExport |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| `cars.{siteId}` broadcast `fn()=>true` (no auth on channel) | 🟡 Med | channel.php |
| takeAction permission check commented out | 🟡 Med | CarModelController:20 |
| Missing `car_requests` migration (fragile deploy) | 🟢 Low | migrations |
| Mixed unrelated domain in tourism codebase | 🟢 Arch | Modules/CarModel |

## 27. Technical Debt
- Unrelated logistics domain co-resident with tourism (confusing scope).
- Dead API routes (commented).
- Missing migration.
- Commented-out authz + open broadcast.
- No tests.

## 28. Improvement Opportunities
- Extract to a separate repo/app OR deprecate if unused; add README clarifying non-tourism scope.
- Enforce takeAction permission + secure broadcast (validate user↔site).
- Restore `car_requests` migration.
- Add tests if kept.

---

## 32. Related Features
- **[[25_Realtime]]** — RealTimeMessage/cars channel (only live event source).
- **[[26_DashboardAdmin]] [[27_RolesPermissions]]** — dashboard + guard/contract_manager roles.
- **[[29_ReportModule]]** — export target.
- **[[30_VisitorTracking]]** — sibling guard/visitor workflows (also internal, not tourism/mobile).
- **NOT related to** any Flutter/tourism feature.

## 33. How to Modify This Feature
- Dashboard-only: edit Modules/CarModel controllers/services/views. **Do not expose to mobile** unless product scope changes.
- Secure it: uncomment permission middleware; add channel auth; restore migration.

## 34. Regression Checklist
- [ ] Dashboard cars list/live-mode loads.
- [ ] Guard check-in triggers notification/broadcast.
- [ ] takeAction records status (+ after fix: permission-gated).
- [ ] Mobile API unaffected (no car endpoints).

## 35. Common Bugs
- Fresh deploy fails (missing car_requests migration).
- Broadcasts vanish (null driver, [[25_Realtime]]).
- Unauthorized takeAction (permission commented).

## 36. Debug Guide
- **Mobile can't see cars:** expected — no API.
- **Deploy migrate fails:** missing car_requests migration.
- **Channel leak:** `cars.{siteId}` open — expected until secured.

## 37. Search Keywords
`Modules/CarModel`, `CarPlate`, `CarSetting`, `CarDay`, `CarNotes`, `CarService`, `CarModelController`, `Car`, `CarRequest`, `Driver`, `cars.{siteId}`, `takeAction`, `GuardController carAction`, `car_plates`, `plate detection ANPR`, `fleet logistics`, `out of scope`, وحدة السيارات, لوجستيات.

## 38. Future Improvements
- Decide fate: spin out as separate product or remove. If kept, secure authz + channels, restore migration, add tests, document non-tourism scope.
