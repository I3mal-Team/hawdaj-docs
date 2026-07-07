# Feature 29 — Report Module (analytics / PDF-Excel export — OUT OF SCOPE)

> **Status:** ✅ 100% analyzed · **Priority:** P4 · **Category:** Admin / Internal (⚠️ **OUT OF SCOPE** for tourism/mobile)
> **Laravel:** `Modules/Report/` (nwidart) + `Modules/CarModel/Services/CarModelReport` · **Flutter:** NOT involved (no mobile API)
> **One-line:** A backend dashboard analytics + archived export (PDF via dompdf / Excel via maatwebsite) system that reports **exclusively on CarModel ANPR data** ([[28_CarModule]]) — plate detections, invitation/no-invitation splits, notice actions, risk durations — per site. **No tourism content, no mobile exposure** (`/api/report` is an auth stub with no business logic). Properly auth + permission-gated (`read-report`/`read-archive_file`). Exports run via cron (`php artisan files:export`) → `ArchiveFile` records → files on **public disk**. **Verdict: OUT OF SCOPE.** Minor flags: exports on public disk, no archive-file cleanup/TTL, potential OOM on very large exports, PII risk from free-text car_notes.

---

## 1. Feature Name & Identity
- **Name:** Report Module — analytics dashboards + queued PDF/Excel export.
- **Reports ON:** CarModel (parking/gate ANPR) metrics only. Not Places/Trips/Stores.
- **nwidart module** `Modules/Report/`; consumes CarModel data.

## 2. Business Goal & Rules
- **Goal:** Operational reporting on vehicle-detection sites (charts + exportable archives).
- **Rules:** dashboard-only, permission-gated; export async via console command; site + date-range filtered; ArchiveFile status 0→1.

## 3. User Flow (dashboard)
1. Admin views report dashboards (charts, drafts, pinned).
2. Export requested → `ArchiveFile` (status=0 pending).
3. Cron `files:export` → `ExportFileService` generates PDF/Excel → sets url + status=1.
4. Admin downloads (permission-checked).

## 4. Screens / Views
- Dashboard Blade report views (charts, draft/pinned reports, archive files list). No mobile.

## 5–7. Widgets / Cubits / State
- N/A (Blade/charts). Flutter uninvolved.

## 8. Models (Entities)
- `ArchiveFile` (export records), `DraftReport`, `PinnedReport`, `Config`, `ChartDetails`, etc. (Modules/Report/Entities).
- CarModel provides data (CarPlate, car_notes).

## 9. Repositories / 10. Services
- `ReportService`, `ExportFileService` (L13–74: fetch CarPlate+car_notes by date+site, chunk 500, render PDF Blade / Excel FromCollection → public storage, update url+status), `ChartService`, `ConfigService`.
- `Modules/CarModel/Services/CarModelReport.php` (report interface), `CarService.handleExport()` (L189–208 creates ArchiveFile).
- `Modules/Report/Console/ExportFile.php` (`files:export` job: queries status=0 → ExportFileService::handle).

## 11. API Endpoints
- **None functional.** `/api/report` is an auth:api stub (returns request user, no logic). Dashboard-only.

## 12. Request / Response Models
- Dashboard/chart JSON + file downloads. No mobile contract.

## 13. Routes
- `Modules/Report/Routes/web.php:7–36` — `/dashboard/report/*` (web auth). ArchiveFilesController download($id) L33–48 / downloadFile($id) L60–73.

## 14. Controllers
- ReportController, ArchiveFilesController (download, permission `read-archive_file`/`delete-archive_file`), DraftController, PinnedController, ConfigController. All permission-gated.

## 15. Services
- ExportFileService (generation), ReportService/ChartService/ConfigService, ExportFiles (maatwebsite Excel class).

## 16. Models
- ArchiveFile + report config/draft/pinned + chart entities. Reads CarModel.

## 17. Database Tables
- `archive_files`: id, start, end, site_id(FK), type(pdf/excel), model_type('CarModel'), url(250), size(250), name(auto `car_file_from_{start}_to_{end}.{type}`), status(0 pending/1 ready), user_id(FK), timestamps.
- + config/draft/pinned/chart tables (Modules/Report/Database/Migrations).

## 18. Validation
- Date range + site + type. Permission checks on read/delete.

## 19. Auth & Permissions
- Web session auth; `permission:read-report`, `read-archive_file`, `delete-archive_file` (Spatie, [[27_RolesPermissions]]). Download permission-checked. `/api/report` stub auth:api only (no data).

## 20. Error Handling
- Console job processes pending; failures leave status=0. Standard Laravel.

## 21. Edge Cases
- Very large export (50k+ rows) → all loaded into `$result` before write → OOM risk despite 500-chunk read.
- Old archive files accumulate (no TTL/cleanup).
- Public-disk file accessible if URL guessed (mitigated by permission on the download route, but the raw storage URL is public).
- Only CarModel model_type used (single-domain).

## 22. Dependencies
- nwidart, dompdf, maatwebsite/excel, Spatie permissions.
- **Cross-feature:** [[28_CarModule]] (data source), [[26_DashboardAdmin]]/[[27_RolesPermissions]] (auth). NOT tied to any tourism/Flutter feature.

## 23. Files Involved
`Modules/Report/` (Entities: ArchiveFile/DraftReport/PinnedReport/Config/ChartDetails; Services: ReportService/ExportFileService/ChartService/ConfigService; Controllers; Routes/web.php; Console/ExportFile.php; Exports/ExportFiles.php; Migrations), Modules/CarModel/Services/CarModelReport.php + CarService.handleExport (L189–208), CarModel/Config/config.php (report types L11–38).

## 24. Full Execution Flow (export)
Admin export → `CarService.handleExport()` → `ArchiveFile(status=0)` → cron `php artisan files:export` → `ExportFile` console → `ExportFileService::handle` → query CarPlate+car_notes (date+site, chunk 500) → PDF(dompdf Blade)/Excel(maatwebsite) → save public storage → update url+status=1 → admin downloads (permission-checked).

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Full result array before write (OOM on huge exports) | 🟡 Med | ExportFileService |
| No archive-file cleanup/TTL (storage grows) | 🟡 Med | archive_files |
| Chunked read 500 (good) | 🟢 | ExportFileService |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| Exports on public disk (URL-guessable; download route gated but raw storage public) | 🟡 Med | storage/app/public/CarModel/files/{site_id}/ |
| PII risk from free-text car_notes in exports | 🟢 Low | ExportFileService |
| Auth + permission gating on routes/download ✅ | OK | controllers |
| No API/mobile exposure ✅ | OK | /api/report stub |

## 27. Technical Debt
- Single-domain (CarModel only) despite generic "Report" naming.
- No cleanup of old archives.
- Hardcoded storage path (no config).
- Export trigger console-only (no clear UI action in controllers).
- Public-disk exports.
- Coupled to internal ops, co-resident with tourism.

## 28. Improvement Opportunities
- Move exports to private disk + signed URLs; add retention/cleanup job.
- Stream/queue large exports (avoid full-array OOM).
- Config-driven paths; generalize model_type if more domains report.
- If tourism reporting ever needed, build a separate Tourism report path (none exists today).

---

## 32. Related Features
- **[[28_CarModule]]** — sole data source (ANPR).
- **[[26_DashboardAdmin]] [[27_RolesPermissions]]** — auth/permission host.
- **NOT related to** any Flutter/tourism feature.

## 33. How to Modify This Feature
- Dashboard/internal only. Add report type: extend CarModel Config + ExportFileService/ChartService. Do not expose to mobile.
- Secure: private disk + signed URLs; add cleanup cron.

## 34. Regression Checklist
- [ ] Report dashboards render.
- [ ] Export creates ArchiveFile → cron generates file → status=1.
- [ ] Download permission-gated.
- [ ] Mobile API unaffected.
- [ ] (After fix) old archives cleaned up.

## 35. Common Bugs
- Export never completes (cron not running).
- OOM on huge export.
- Storage fills (no cleanup).
- Public file accessible via raw URL.

## 36. Debug Guide
- **Export stuck status=0:** confirm `files:export` cron running.
- **Download 403:** check `read-archive_file` permission.
- **Mobile can't report:** expected — no API.
- **Large export fails:** memory limit vs result array.

## 37. Search Keywords
`Modules/Report`, `ArchiveFile`, `ExportFileService`, `files:export`, `dompdf`, `maatwebsite excel`, `DraftReport`, `PinnedReport`, `CarModelReport`, `read-report permission`, `archive_files table`, `handleExport`, `report analytics`, `out of scope`, وحدة التقارير, تصدير PDF Excel.

## 38. Future Improvements
- Private secured exports + retention; streaming large exports; generalized multi-domain reporting; separate tourism analytics if product needs it.
