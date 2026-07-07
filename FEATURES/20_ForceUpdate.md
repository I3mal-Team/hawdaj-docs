# Feature 20 — Force Update (التحديث الإجباري)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Supporting (startup gate)
> **Flutter:** `lib/features/force_update/` · **Laravel:** NOT involved (confirmed — no version/force_update endpoints)
> **One-line:** A **Firebase Firestore-only** version gate (collection `app_versions`, doc `latest_version`, fields `version` + `isMandatory`). Uses the `version` package for correct **semantic comparison** (1.10 > 1.9 handled). Mandatory update → truly non-bypassable dialog (`PopScope canPop:false` + `barrierDismissible:false`); optional → dismissable with "remember my choice" (`ignored_update_version` pref, optional-only). **Fail-OPEN** on Firebase error (user proceeds). Store URLs **hardcoded per platform**. Build number ignored in comparison.

---

## 1. Feature Name & Identity
- **Name:** Force Update — the first boot gate ([[19_SplashOnboarding]] boots into it).
- **Backend:** Firebase Firestore, NOT Laravel (grep confirmed no server version endpoint).
- **Nature:** Client-side version check + blocking/dismissable dialog + store deep-link.

## 2. Business Goal & Rules
- **Goal:** Force users off outdated (mandatory) versions; nudge on optional ones.
- **Rules:**
  - Mandatory required ⇔ `latest > current AND isMandatory==true`.
  - Optional available ⇔ `latest > current AND !isMandatory` (and not previously ignored).
  - Mandatory dialog cannot be dismissed/back-navigated.
  - Optional dialog: "Update Now" + "Maybe Later" + optional "remember choice" → store version in `ignored_update_version`.
  - On Firebase error → proceed (fail-open).
  - Build number (`+40`) ignored; only X.Y.Z compared.

## 3. User Flow
1. Boot → `ForceUpdateIntermediateScreen` `initState()` → `ForceUpdateCubit.checkForUpdate()`.
2. Fetch Firebase `app_versions/latest_version` + `PackageInfo.fromPlatform()` current.
3. Decide: Required → blocking dialog; Available → dismissable; NotRequired/Error → continue to Splash.
4. Dialog "Update Now" → open platform store via `url_launcher` (external app).
5. Optional "Maybe Later" (+remember) → ignore version → continue.

## 4. Screens / Views
| File | Lines | Purpose |
|------|-------|---------|
| `force_update_intermediate_screen.dart` | 8–90 | Initial route; triggers check; spinner; routes to Splash on done/error |
| `force_update_dialog.dart` | 35–180 | Mandatory (blocking) vs optional (dismissable) dialog; store open |

## 5. Widgets
- `ForceUpdateDialog`: mandatory → `PopScope(canPop:false)` (37) + `barrierDismissible:false` (39), single "Update Now". Optional → back+outside enabled, "Update Now" + "Maybe Later" (138–157) + "remember my choice" checkbox (113–134).
- Loading spinner on intermediate screen (79–86).

## 6. Cubits / Blocs
| Cubit | File:Lines | States |
|-------|-----------|--------|
| ForceUpdateCubit | `force_update_cubit.dart:17–76` | ForceUpdateRequired, ForceUpdateAvailable, ForceUpdateNotRequired, ForceUpdateError |

- `checkForUpdate()` (17–76), `ignoreVersion()` (99–126), `_isNewerVersion()` fallback (79–96, optional-only).

## 7. State Flow
- checkForUpdate → fetch latest + current → check ignored (optional only, 30–34) → `isUpdateRequired()` (42) → emit Required(50–57) | Available(58–66) | NotRequired(70) | Error(72–75).
- Intermediate screen: Required/Available → show dialog; NotRequired/Error → `_navigateToNextScreen()` (72–74).

## 8. Models
| Model | File:Lines | Fields |
|-------|-----------|--------|
| AppVersionModel | `app_version_model.dart:1–14` | `version` (String semver), `isMandatory` (bool). **No storeUrl/changelog/platform urls** |

## 9. Repositories
- [`firebase_force_update_repo_impl.dart`](lib/features/force_update/data/repo/firebase_force_update_repo_impl.dart) `:9–62`: fetch Firestore doc → `AppVersionModel`; version compare via `Version.parse` (41–62): `isRequired = (latest > current) && isMandatory`.

## 10. Services / Helpers
- `cloud_firestore` (fetch), `package_info_plus` (`PackageInfo.fromPlatform()`), `version` package (semver compare), `url_launcher` (store open), SharedPreferences (`ignored_update_version`), `dart:io Platform` (iOS/Android detect).

## 11. API Endpoints
- **None (Laravel).** Data source = Firebase Firestore `app_versions/latest_version`.

## 12. Request / Response Models
- Firestore doc: `{version:"5.10.24", isMandatory:true}`.
- No HTTP request/response; SDK read.
- Store URLs (hardcoded, `app_store_links.dart`): Android `https://play.google.com/store/apps/details?id=com.hawdaj`; iOS `https://apps.apple.com/us/app/hawdaj/id6752390026`.

## 13. Laravel Routes
- **None.** Confirmed no backend involvement.

## 14. Laravel Controllers
- N/A.

## 15. Laravel Services
- N/A. (If migrating off Firebase, a `/app-version` endpoint would live here.)

## 16. Laravel Models
- N/A.

## 17. Database Tables & Relationships
- Firebase Firestore only: collection `app_versions`, doc `latest_version` (fields version, isMandatory). No MySQL table.

## 18. Validation
- Version parse via `Version.parse` (throws → caught → Error → fail-open). No server validation.

## 19. Auth & Permissions
- Public Firestore read (no auth). Runs before login. No permissions.

## 20. Error Handling
- `catch(e) → emit ForceUpdateError` (72–75). Intermediate screen treats Error like NotRequired → **fail-open** (proceeds to Splash). No retry/backoff.

## 21. Edge Cases
- Firebase down → all users bypass gate (mandatory unenforceable network-wide).
- Build-number-only bump (`+40`→`+41`) → not detected (no forced hotfix).
- Optional version ignored → never reshown even if `isMandatory` later flips true (unless pref cleared).
- Non-iOS/Android platform → store open silently no-ops.
- App killed mid-check → next launch rechecks (stateless).
- Semver edge (1.10 vs 1.9) → handled correctly by `version` lib.

## 22. Dependencies
- cloud_firestore, firebase_core, package_info_plus, version, url_launcher, shared_preferences, flutter_bloc, get_it.
- **Cross-feature:** [[19_SplashOnboarding]] (boot gate order), [[25_Realtime]]/[[17_Notifications]] (also Firebase), routing.

## 23. Files Involved
**Flutter:** force_update_cubit, firebase_force_update_repo_impl, force_update_dialog, force_update_intermediate_screen, app_version_model, app_store_links, service_locator:174–177, main.dart:165–167 (BlocProvider), app_router.dart (initialLocation kForceUpdateIntermediateScreen), RoutesKeys.
**Laravel:** none.

## 24. Full Execution Flow (boot gate)
`app_router` initialLocation `kForceUpdateIntermediateScreen` → `initState` → `ForceUpdateCubit.checkForUpdate()` → `FirebaseForceUpdateRepoImpl.getLatestVersion()` (Firestore) + `PackageInfo.fromPlatform()` → check `ignored_update_version` (optional path) → `Version.parse` compare → emit state → intermediate screen: Required → blocking `ForceUpdateDialog` (Update Now → `launchUrl(store, externalApplication)`); Available → dismissable dialog (Maybe Later → `ignoreVersion`); NotRequired/Error → `_navigateToNextScreen()` → SplashViewBody.

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| Single Firestore read at boot (blocks intermediate screen) | 🟢 Low | repo |
| No retry/backoff on transient failure | 🟡 Med | cubit catch |
| Fixed gate before splash adds latency | 🟢 Low | app_router |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **Fail-open: Firebase outage bypasses mandatory gate for all** | 🟡 Med | intermediate screen (Error→continue) |
| Hardcoded store URLs (no region/rotation) | 🟢 Low | app_store_links |
| Public Firestore read (fine for version data) | 🟢 OK | — |
| Mandatory dialog non-bypassable ✅ (PopScope+barrier) | OK | dialog:37,39 |

## 27. Technical Debt
- Firebase single point of failure + fail-open → mandatory gate not guaranteed.
- Build number ignored → can't force hotfix-level updates.
- Optional-ignore is permanent (no expiry / no re-prompt if severity rises).
- Store URLs + Firebase config hardcoded.
- No changelog/release-notes surface.
- Version gate on Firebase, decoupled from Laravel content backend.

## 28. Improvement Opportunities
- Add retry/backoff; consider fail-closed (or cached-last-known) for mandatory to prevent bypass on outage.
- Include build number / min-supported-version + changelog + storeUrl fields in the version doc.
- Move (or mirror) version gate to Laravel (`/app-version`) for a single source of truth + easier ops.
- Expire ignored-version or re-prompt when a newer optional arrives.
- Region/platform store-URL resolution; graceful desktop/web handling.

---

## 32. Related Features
- **[[19_SplashOnboarding]]** — Force Update is the first boot gate before splash.
- **[[17_Notifications]] / [[25_Realtime]]** — also Firebase-dependent.
- **[[24_Networking]]** — env/version context.
- **[[26_DashboardAdmin]]** — if migrated, admin would set required version.

## 33. How to Modify This Feature
- **Push a mandatory update:** set Firestore `app_versions/latest_version` = `{version:"X.Y.Z", isMandatory:true}` (> live version).
- **Change store links:** edit `app_store_links.dart` (requires app release).
- **Add changelog:** extend `AppVersionModel` + Firestore doc + dialog UI.
- **Make fail-closed:** in intermediate screen, treat Error separately (retry/block) instead of proceeding.

## 34. Regression Checklist
- [ ] Live version < latest & mandatory → blocking dialog, no dismiss/back.
- [ ] Optional newer → dismissable; "Maybe Later"+remember → not reshown.
- [ ] Current == latest → proceeds silently.
- [ ] Firebase unreachable → app still opens (fail-open) — decide if desired.
- [ ] 1.10 vs 1.9 compared correctly.
- [ ] Store opens correct platform URL.

## 35. Common Bugs
- Mandatory update not enforced during Firebase outage (fail-open).
- Hotfix (+build) not triggering update.
- Ignored optional never reappears.
- Wrong/broken store URL after store id change (hardcoded).
- Stuck spinner if Firestore hangs without timeout.

## 36. Debug Guide
- **Dialog not showing:** check Firestore doc `version`/`isMandatory` vs `PackageInfo` current; check `ignored_update_version` pref (optional).
- **App bypasses gate:** confirm Firebase reachable; error path is fail-open.
- **Wrong compare:** verify `Version.parse` inputs (semver, no stray build issues).
- **Store won't open:** check Platform + `launchUrl` mode + URL validity.
- **Stuck:** add timeout around Firestore read.

## 37. Search Keywords
`forceUpdate`, `ForceUpdateCubit`, `force_update`, `app_versions`, `latest_version`, `isMandatory`, `AppVersionModel`, `Version.parse`, `PackageInfo.fromPlatform`, `ignored_update_version`, `app_store_links`, `launchUrl store`, `ForceUpdateIntermediateScreen`, `PopScope canPop false`, التحديث الإجباري, تحديث التطبيق.

## 38. Future Improvements
- Backend-owned version policy (min-supported + latest + changelog + per-platform store urls), cached for offline, fail-closed for mandatory.
- Build-number awareness for hotfixes; staged rollout flags.
- Rich update dialog (changelog, size, screenshots) + analytics on update funnel.
