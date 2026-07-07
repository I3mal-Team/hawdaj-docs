# Feature 23 — Media / Gallery / Uploads (الوسائط والمعرض والرفع)

> **Status:** ✅ 100% analyzed · **Priority:** P3 · **Category:** Shared (media infra)
> **Flutter:** `form_data_helper.dart` + `global_network_image.dart` + `gallery_image_carousel.dart` · **Laravel:** Spatie MediaLibrary + custom `Gallery` table + legacy `image` column + `UploadService`
> **One-line:** **THREE coexisting media systems** on the backend: (1) Spatie MediaLibrary (single-file `image`/`photo` collection per model, WebP conversions small 250×250 / medium 500×500, **nonQueued** = blocks upload response), (2) a custom `galleries` table (`parent_id`+`type`, multi-image), and (3) a legacy `image` string column — reconciled by a `getMediaUrl`/`getImageUrl` fallback chain. Flutter uploads via multipart `FormDataHelper`, displays via `CachedNetworkImage` (shimmer + logo fallback, SVG + AVIF support). Client "compression" is only image_picker dimension/quality constraints (no real compression lib). **Server `max:2048` (2KB!) vs client 50MB is a glaring contradiction**; SVG allowed (XSS risk); public disks, no CDN, no orphan cleanup.

---

## 1. Feature Name & Identity
- **Name:** Media / Gallery / Uploads — cross-app image pick, upload, store, convert, serve, display.
- **Three backend systems:** Spatie MediaLibrary (modern, single-file), custom `Gallery` table (multi-image, `parent_id`+`type` like Rate), legacy `image` column (old). All bridged by URL helpers.
- **Client:** image_picker → FormData → CachedNetworkImage display.

## 2. Business Goal & Rules
- **Goal:** Consistent image upload + responsive (small/medium) serving + galleries on detail pages.
- **Rules:**
  - Spatie collections are `singleFile` → one hero image per model; multi-image needs `galleries`.
  - Conversions: small 250×250 (sharpen10, WebP q85), medium 500×500 (WebP q90), original WebP q95. **nonQueued** (synchronous).
  - URL resolution: Spatie `getFirstMediaUrl` → legacy `uploads/` → placeholder `front_assets/imgs/zad1.jpg`.
  - Validation mimes jpeg/png/jpg/gif/svg/webp; server `max:2048` on most endpoints.

## 3. User Flow
1. User taps add-photo → image_picker (camera/gallery) with maxWidth/Height + quality 85.
2. File → `FormDataHelper.addFile('file'/'image', ...)` → multipart POST.
3. Server validates, stores via Spatie `addMediaFromRequest` or `UploadService`, generates conversions.
4. Detail pages fetch content → gallery/image URLs → `CachedNetworkImage` (shimmer → image → logo on error).

## 4. Screens / Views
- Not a screen; cross-cutting. Gallery display: [`gallery_image_carousel.dart`](lib/features/home/presentation/view/widgets/gallery_image_carousel.dart) (carousel_slider + dots, AVIF 50–54, CachedNetworkImage+Shimmer 57–73). Image widget: [`global_network_image.dart`](lib/core/components/global_network_image.dart) (GNImage).

## 5. Widgets
| Widget | File | Notes |
|--------|------|-------|
| GNImage | `global_network_image.dart:13–68` | isSvg check (13), CachedNetworkImage+shimmer+logo fallback (27–40), Image.network/SvgPicture.network fallback (42–68) |
| Gallery carousel | `gallery_image_carousel.dart` | autoplay, ExpandingDotsEffect, AVIF + CachedNetworkImage |

## 6. Cubits / Blocs
- No dedicated media cubit; upload happens inside feature cubits (AddStory, AddProperty, Guide, Landmark, EditProfile) via FormDataHelper.

## 7. State Flow
- Feature cubit picks file → builds FormData → repo POST → Loading→Success/Error. Display driven by content fetch (URLs in resources).

## 8. Models
| Model | File | Fields |
|-------|------|--------|
| GalleryModel | `home/data/model/places_model/gallery_model.dart` | id, file(url), type, mimeType |

## 9. Repositories
- Per-feature repos attach files. `UploadService` (Laravel) is the server-side store/convert/delete service.

## 10. Services / Helpers
| Helper | File:Lines | Role |
|--------|-----------|------|
| FormDataHelper | `utils/form_data_helper.dart:12–55` | addFile (single, MIME-validated), addFiles (loop); MultipartFile.fromBytes; keys 'file'/'file[]'/'files[]' |
| camera_helper | `core/utils/functions/camera_helper.dart:31–35` | image_picker maxW/H 1024, quality 85 |
| image_helper | `core/utils/functions/image_helper.dart` | SVG detection only |
| getImageUrl | `App.php:764–791` | legacy uploads URL (small-/medium- prefix) |
| getMediaUrl | `App.php:793–847` | Spatie first (796), legacy fallback (802–824), fixes malformed `https:/` (831–835) |
| UploadService | `app/Services/UploadService.php:38–191` | server WebP conversions (original 95, small 250×250/85, medium 500×500/90), storeFile raw, delete variants |
| HasImage trait | `app/Traits/HasImage.php:9–15` | fallback chain Spatie→legacy→placeholder |

## 11. API Endpoints
- No standalone media endpoint; uploads ride content endpoints (update-photo, properties, stories/store, store-guide, update-guide-photo, landmark/store) + GalleryController for gallery items.

## 12. Request / Response Models
- **Upload req:** multipart, file key `image`/`file`/`images[]`.
- **Media in resources:** `image` (full), `image_small`, `image_medium` (Spatie/legacy resolved); gallery items `{id,file,type,mime_type}`.

## 13. Laravel Routes
- Gallery: GalleryController (store) + content upload routes. (No dedicated media route group.)

## 14. Laravel Controllers
- GuideController:143 `addMediaFromRequest('image')->toMediaCollection('image')`, validation :342 `image|mimes:...|max:2048`.
- GalleryController:40–46 `UploadService::storeFile()` → `Gallery::create()`.
- Story/Landmark/Property/Profile controllers similar (Spatie or Gallery).

## 15. Laravel Services
- `UploadService` (store/convert/delete via Intervention\Image; WebP; fallback to raw if GD/Imagick missing).

## 16. Laravel Models
- Spatie: Place (`:182–205` collections+conversions nonQueued), Guide, User, Event, Store, Application, Landmark, Slider, Swalef, ZadElgadel — `HasMedia`+`InteractsWithMedia`, singleFile.
- Gallery (`:18–26`): `getFileAttribute` fallback to `front_assets/imgs/zad1.jpg`; `Place::galleries()` `:127–130` HasMany + type filter.
- HasImage trait for legacy column.

## 17. Database Tables & Relationships
- `media` (Spatie polymorphic: model_type/model_id, collection, conversions).
- [`galleries`](../../../hawdaj-api/database/migrations/2022_07_25_195259_create_galleries_table.php): id, parent_id, type(guides/places…), file(path), active, mime_type, timestamps. **No FK/cascade** → orphans on parent delete.
- Legacy `image` column on many content tables.

## 18. Validation
- Server: `image|mimes:jpeg,png,jpg,gif,svg,webp|max:2048` (**2048 = 2MB via Laravel KB units? actually 2048KB=2MB**; agent flagged as possibly-too-small/typo vs client). Stories add webm.
- Client: allowed jpg/jpeg/png/gif/svg/webm/mp4/mov/avi, **max 50MB** (add_story_bottom_sheet:138–145).
- **Mismatch:** client 50MB vs server 2048KB → large files rejected server-side.
- No dimension validation (1×1 accepted).

## 19. Auth & Permissions
- Uploads inherit their content endpoint's auth (mostly auth:api). Files served from public disks (no auth on serving).

## 20. Error Handling
- Flutter: FormDataHelper MIME check throws; CachedNetworkImage errorWidget → logo; Shimmer placeholder.
- Laravel: validation 422; UploadService fallback to raw store if image lib missing; getMediaUrl patches malformed URLs.

## 21. Edge Cases
- Parent deleted → orphaned `galleries` rows + files (no cascade).
- Legacy vs Spatie vs Gallery: same content may resolve via different chain → inconsistent URLs.
- SVG accepted → potential stored-XSS if served inline.
- nonQueued conversions → slow upload response (blocks until small+medium built).
- Missing image → placeholder `zad1.jpg` (Gallery) or logo (client).
- Malformed `https:/domain` needs regex fix (indicates URL-build fragility).
- Client picks large file (up to 50MB) → server rejects (2MB).

## 22. Dependencies
- **Flutter:** image_picker, cached_network_image, flutter_svg, cached_network_avif_image, carousel_slider, smooth_page_indicator, shimmer, dio (multipart).
- **Laravel:** spatie/laravel-medialibrary, intervention/image, custom UploadService.
- **Cross-feature:** every content feature (Places/Stores/Zad/Events/Swalef/Guides/Landmarks), [[16_UserStories]], [[15_MyProperties]] (proof file), [[21_ProfileSettings]] (photo). RTL-agnostic.

## 23. Files Involved
**Flutter:** form_data_helper, global_network_image, gallery_image_carousel, camera_helper, image_helper, gallery_model, per-feature upload repos, add_story_bottom_sheet.
**Laravel:** UploadService, HasImage trait, Gallery model + migration, GalleryController, ItemGalleryResource, App.php (getImageUrl 764–791, getMediaUrl 793–847), Spatie config, content models with media collections/conversions, config/filesystems.php:51–62.

## 24. Full Execution Flow (upload → serve)
**Upload:** pick (image_picker, 1024²/q85) → `FormDataHelper.addFile` (MultipartFile, MIME check) → POST content endpoint → controller validate (`mimes|max:2048`) → Spatie `addMediaFromRequest->toMediaCollection('image')` (nonQueued builds small/medium WebP) OR `UploadService::store` (gallery/small-/medium- WebP) → DB row (media or galleries or image col).
**Serve:** content fetch → resource resolves URL via `getMediaUrl` (Spatie→legacy→placeholder) / `getImageUrl` → `image`/`image_small`/`image_medium` → Flutter `CachedNetworkImage` (shimmer→image→logo).

## 25. Performance
| Issue | Severity | Location |
|-------|----------|----------|
| **nonQueued conversions block upload response** | 🟡 Med | Place.php:196,203 (+other models) |
| No real client-side compression (only picker constraints) | 🟡 Med | camera_helper |
| No CDN; direct public-disk serving | 🟡 Med | filesystems.php |
| Large gallery load (multi-image carousel) memory | 🟢 Low | gallery_image_carousel |
| No cache headers server-side (browser cache only) | 🟢 Low | UploadService |

## 26. Security
| Issue | Severity | Location |
|-------|----------|----------|
| **SVG allowed → stored-XSS if served inline** | 🟡 Med | mimes whitelist (Guide/Story/etc.) |
| Public disks expose all uploads (no auth on serve) | 🟡 Med | filesystems public+media |
| No dimension/content validation (1×1, disguised files) | 🟢 Low | validation |
| No upload rate limiting | 🟢 Low | controllers |
| Orphaned gallery files after parent delete | 🟢 Low | galleries (no cascade) |

## 27. Technical Debt
- **Three coexisting media systems** (Spatie + Gallery table + legacy column) → complex fallback chain, inconsistent URLs, orphan risk.
- Client/server size-limit contradiction (50MB vs 2MB).
- Malformed-URL regex patch signals fragile URL construction.
- nonQueued conversions (should queue).
- No compression lib; picker-only constraints.
- Gallery placeholder hardcoded (`zad1.jpg`).

## 28. Improvement Opportunities
- Consolidate on Spatie (multi-file collections replace custom Gallery + legacy column); migrate + backfill; add cascade cleanup.
- Queue conversions (`->queued()`) + return fast; process async.
- Align size limits (raise server max to match intended, e.g. 10–20MB) + add dimension checks.
- Sanitize/deny SVG or serve as attachment; add CDN + cache headers; private disk + signed URLs for sensitive (proof files).
- Real client compression (flutter_image_compress) before upload.

---

## 32. Related Features
- **[[04_Places]] [[05_Stores]] [[06_Restaurants]] [[09_Events]] [[10_Swalef]] [[11_TourGuides]] [[12_Landmarks]]** — content images + galleries.
- **[[16_UserStories]]** — story media (SVG/webm).
- **[[15_MyProperties]]** — ownership_proof_file (public-disk exposure risk).
- **[[21_ProfileSettings]]** — profile photo.
- **[[24_Networking]]** — multipart via Dio.

## 33. How to Modify This Feature
- **Add image field to a model:** implement HasMedia + registerMediaCollections/Conversions (copy Place:182–205) OR add to galleries (parent_id+type). Add URL to resource via getMediaUrl. Flutter: FormDataHelper key + GNImage display.
- **Fix size limit:** raise server `max:` (KB) to match client; update client cap consistently.
- **Queue conversions:** change nonQueued→queued + run queue worker.
- **Add multi-image to a model:** use galleries table (HasMany + type) or Spatie multi-file collection.

## 34. Regression Checklist
- [ ] Pick + upload image (each feature) → stored, small/medium generated.
- [ ] Gallery carousel renders multiple images.
- [ ] Broken URL → logo/placeholder, no crash.
- [ ] SVG handled (decide: allow/deny).
- [ ] Large file behavior consistent client↔server.
- [ ] Deleted content cleans up media/gallery files.

## 35. Common Bugs
- Upload fails silently for files >2MB (server) though client allows 50MB.
- Inconsistent image URL (Spatie vs legacy vs gallery) for same content.
- Slow upload (nonQueued conversions).
- Orphaned gallery images after delete.
- Broken `https:/` URLs pre-regex-fix.
- SVG rendering/security surprises.

## 36. Debug Guide
- **Image not showing:** trace getMediaUrl chain (Spatie media row? legacy image col? placeholder?).
- **Upload rejected:** check server `max:2048` vs file size + mime.
- **Slow upload:** conversions nonQueued — check UploadService/Spatie.
- **Orphan files:** audit galleries rows vs existing parents.
- **Malformed URL:** check getMediaUrl regex patch input.

## 37. Search Keywords
`FormDataHelper`, `MultipartFile`, `image_picker`, `CachedNetworkImage`, `GNImage`, `global_network_image`, `gallery_image_carousel`, `Gallery model`, `galleries table parent_id type`, `Spatie MediaLibrary`, `addMediaFromRequest`, `registerMediaConversions small medium`, `nonQueued`, `getImageUrl`, `getMediaUrl`, `UploadService`, `HasImage`, `max:2048`, الوسائط, المعرض, رفع الصور.

## 38. Future Improvements
- Single Spatie-based media pipeline (multi-file collections) + backfill/migrate off Gallery+legacy; cascade cleanup.
- Queued conversions + CDN + signed private URLs for sensitive files.
- Real compression + dimension validation + upload throttling.
- Consistent size policy + SVG sanitization.
