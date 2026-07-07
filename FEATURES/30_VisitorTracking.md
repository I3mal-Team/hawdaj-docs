# Feature 30 — Visitor Tracking / Guard Check-in / Task Activity

> **Status:** ✅ 100% analyzed · **Priority:** P4 · **Category:** Hidden / Internal (mixed scope)
> **Laravel:** `views_num` + VisitSite/MostVisit (mobile-facing but broken) + GuardController/VisitRequest/Visitor (internal ops) · **Flutter:** parses `views_num` but doesn't display it
> **One-line:** TWO unrelated things under this name. **(A) Mobile-facing content view tracking is REAL-but-BROKEN:** `views_num` columns exist on Place/Store/Zad/Event and are exposed in API resources + parsed by Flutter, but **nothing ever increments them** — so "topVisitedPlaces"/"topVisitedStores" sort by a constant 0 = **fake popularity** (effectively id/random order). IP logging (`VisitSite`) fires only on the home page (not per-content, no user_id/content_id, random-IP fallback); `MostVisit` table + `visit_item()` helper are **dead/gutted**. **(B) Guard/Visitor/Contract/Material/Task ops are internal dashboard-only** (PII-heavy visitor check-in/out, health checks) — **OUT OF SCOPE**, zero mobile exposure.

---

## 1. Feature Name & Identity
- **(A) Content view tracking** — `views_num` on content, "top visited" ranking. Tourism-visible, currently non-functional.
- **(B) Physical visitor/guard management** — VisitRequest/Visitor/GuardController/TaskController. Internal ops, out of scope.

## 2. Business Goal & Rules
- **(A) Goal:** count/rank content by views. **Reality:** counter never incremented → ranking fake.
- **(B) Goal:** guards check visitors in/out at sites (health/ID/vehicle), task workflows. Dashboard-only, auth + check_site middleware.

## 3. User Flow
- **(A)** App loads home → HomeController `visit()` logs one home-page row to VisitSite → topVisited* sorts by `views_num DESC` (all 0) or distance. Content detail (`show`) does NOT increment.
- **(B)** Guard searches VisitorRequest → checkin/checkout/reject with notes → History logged + host notified. Task dashboard manages visitors/contractors/materials/cars.

## 4. Screens / Views
- **(A)** No dedicated view-count UI in app (parsed, not shown).
- **(B)** Dashboard guard/task Blade views. No mobile.

## 5–7. Widgets / Cubits / State
- **(A)** views_num in models only. **(B)** N/A (Blade). Flutter uninvolved in (B).

## 8. Models
- **(A)** Place/Store/ZadElgadel/Event: `views_num` bigInteger default 0. VisitSite (ip, page, visits), MostVisit (ip+geo, dead).
- **(B)** VisitRequest (visit date/time/reason/host), VisitorRequest (pivot: checkin/checkout/health_status/status), Visitor (PII: id_number, id_copy, personal_photo, mobile, nationality, vehicle), VisitorHealthCheck, History (morph activity).

## 9–10. Repositories / Services
- **(A)** `visit()` helper (App.php:170–185) logs VisitSite; `visit_item()` (App.php:16–26) **gutted/returns early**.
- **(B)** GuardController/TaskController logic inline.

## 11. API Endpoints
- **(A)** `views_num` exposed in resources (PlaceResource:59, Store/Zad/Event). `/api/home` calls `visit()`. No increment endpoint.
- **(B)** NONE (dashboard-only, auth+check_site).

## 12. Request / Response
- **(A)** content responses include `views_num` (stale/0). **(B)** dashboard forms.

## 13. Routes
- **(A)** part of content GETs. **(B)** dashboard guard/task/visit routes (web auth+check_site).

## 14. Controllers
- **(A)** HomeController (30 visit(), 66–79 topVisitedPlaces sort views_num DESC/distance), PlaceController (98 same, 147–151 show() no increment).
- **(B)** GuardController (search 50–87, takeAction 132–226: checkin/checkout/reject + History + notify), TaskController (index 30–144, taskAction 182–223).

## 15. Services
- App.php helpers. No dedicated tracking service (visit_item dead).

## 16. Models (summary)
- **(A)** view columns + VisitSite/MostVisit. **(B)** Visitor domain (PII).

## 17. Database Tables
- **(A)** `views_num` col (2022_10_07 places + parallel), `visit_sites` (ip,page,visits), `most_visits` (ip+geo, 2023_01_28, +lat/long 2024_08_18, **never populated**).
- **(B)** visit_requests, visitor_requests, visitors, visitor_health_checks, histories.

## 18. Validation
- **(A)** none (read). **(B)** guard action validation (status/notes).

## 19. Auth & Permissions
- **(A)** content public; visit() logs IP without consent (mitigated: sparse, no user linkage).
- **(B)** web auth + check_site + Spatie permissions (guard role). No mobile.

## 20. Error Handling
- **(A)** silent (no increment). **(B)** standard + notifications.

## 21. Edge Cases
- **(A)** views_num永远 0 → topVisited fake; random-IP fallback pollutes VisitSite; MostVisit empty → dashboard map shows nothing real; Flutter parses 0 (harmless as not shown).
- **(B)** PII exposure risk in visitor records; internal only.

## 22. Dependencies
- **(A)** part of content stack. **(B)** nwidart-ish internal ops, Spatie, notifications.
- **Cross-feature:** [[02_Home]]/[[19_SplashOnboarding]] (topVisited* in home), [[04_Places]]/[[05_Stores]]/[[06_Restaurants]]/[[09_Events]] (views_num), [[28_CarModule]]/[[29_ReportModule]] (sibling internal ops), [[26_DashboardAdmin]].

## 23. Files Involved
**(A)** Place/Store/Zad/Event models (views_num), HomeController:30/66–79, PlaceController:98/147–151, PlaceResource:59 (+Store/Zad/Event), App.php:16–26 (dead visit_item)/170–185 (visit), migrations (views_num, visit_sites, most_visits), Flutter place_model.dart (parses views_num).
**(B)** VisitRequest/VisitorRequest/Visitor/VisitorHealthCheck/History models, GuardController, TaskController, dashboard views.

## 24. Full Execution Flow
**(A):** app → `/api/home` → HomeController.visit() → `visit_sites` row (ip,page=home,visits=1) → topVisitedPlaces `orderBy views_num DESC` (all 0) → resources include stale views_num → Flutter parses, doesn't render. Content `show()` → NO increment.
**(B):** guard scans → GuardController.takeAction → VisitorRequest status+timestamps → History + host notification. Dashboard-only.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| No N+1 (no per-row increment) | 🟢 | — |
| VisitSite grows (home-only, small) | 🟢 Low | visit_sites |
| topVisited sort on constant → meaningless index use | 🟢 Low | HomeController |

## 26. Security / Privacy
| Issue | Severity | Location |
|-------|----------|----------|
| IP logged without consent (mitigated: sparse, no user link, random fallback) | 🟢 Low | App.php visit() |
| **(B) Visitor PII** (id_number, id_copy, personal_photo) — sensitive, dashboard-only | 🟡 Med (compliance) | Visitor model |
| views_num exposed but stale (misleading) | 🟢 Low | resources |

## 27. Technical Debt
- **views_num dead column** — exposed, parsed, never incremented → misleading "top visited".
- `visit_item()` gutted; `MostVisit` table unpopulated (dead).
- VisitSite logs home only, no per-content attribution, random-IP fallback.
- Guard/visitor ops (PII) co-resident with tourism (scope confusion).

## 28. Improvement Opportunities
- **Fix (A):** increment views on content `show()` (or a dedicated `visits` table with user_id/content_id/created_at, no raw IP) → real topVisited; OR relabel as "featured" if manual.
- Remove dead visit_item/MostVisit or implement properly.
- If views_num stays 0, don't ship it in resources / don't rank by it.
- **(B):** audit PII handling (retention, access) for compliance; keep internal.

---

## 32. Related Features
- **[[02_Home]] [[19_SplashOnboarding]]** — topVisitedPlaces/Stores consumed in home.
- **[[04_Places]] [[05_Stores]] [[06_Restaurants]] [[09_Events]]** — views_num field.
- **[[28_CarModule]] [[29_ReportModule]] [[26_DashboardAdmin]]** — sibling internal ops.

## 33. How to Modify This Feature
- **Make view counts real:** add `Place::increment('views_num')` in PlaceController::show() (+ Store/Zad/Event) or a visits table + aggregation; update topVisited query.
- **Stop misleading ranking:** if not incrementing, order topVisited by a real signal or rename "featured".
- **(B):** dashboard-only; edit GuardController/TaskController; do not expose to mobile.

## 34. Regression Checklist
- [ ] (After fix) viewing content increments views_num.
- [ ] topVisited reflects real counts.
- [ ] Home visit logging works (or removed).
- [ ] Guard checkin/checkout records History + notifies.
- [ ] Mobile unaffected by (B).

## 35. Common Bugs
- "Top visited" identical/random (views never incremented).
- views_num always 0 on app.
- Empty dashboard visit map (MostVisit dead).
- Random IPs in visit_sites.

## 36. Debug Guide
- **Top visited wrong:** confirm views_num values (all 0?) + no increment code.
- **Views 0 in app:** expected — never incremented.
- **Visit map empty:** MostVisit unpopulated (visit_item dead).
- **Guard action not saving:** check status_action + History write + check_site middleware.

## 37. Search Keywords
`views_num`, `topVisitedPlaces`, `topVisitedStores`, `visit()`, `visit_item`, `visit_sites`, `most_visits`, `VisitSite`, `MostVisit`, `VisitRequest`, `VisitorRequest`, `Visitor`, `GuardController takeAction`, `TaskController`, `checkin checkout`, `views tracking`, المشاهدات, الزوار, الحرّاس.

## 38. Future Improvements
- Real, privacy-respecting view analytics (dedicated visits table, dedupe, trending window) feeding topVisited; drop dead columns/tables.
- Compliant visitor-PII lifecycle for internal ops; keep separated from tourism.
