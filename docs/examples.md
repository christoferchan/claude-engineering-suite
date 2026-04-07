# Real Output Examples

What the skills actually produce. These are representative outputs from real pipeline runs.

## `/ship status`

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Pipeline Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Feature:    user-photo-uploads
Branch:     feature/user-photo-uploads
Template:   full-feature
Started:    6 hours ago
Last active: 2 min ago (christofers-macbook)

Progress:   ████████████░░░░ 6 of 9 steps

  1. product-spec      ✅ done   (12 min)
  2. feature-dev       ✅ done   (52 min)
  3. design-audit      ✅ done   (25 min) — 6 issues, 5 fixed
  4. qa-test           ✅ done   (20 min) — 42 passed, 4 failed
  5. security-audit    ✅ done   (9 min)  — 1 major, 0 critical
  6. perf-audit        ✅ done   (7 min)  — bundle +12kb
  7. analytics-audit   ⏳ next
  8. deploy → staging  ○ pending
  9. deploy → prod     ○ pending

Blockers:   none
Open issues: 5 (4 from qa-test, 1 from security-audit)

Last session:
  "Completed perf-audit. Bundle increase is acceptable
   (+12kb for image compression lib). 4 QA failures need
   fixing before deploy: upload timeout, permission denial
   handling, HEIC conversion, empty gallery state."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Quick Actions
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  /ship resume          Continue from analytics-audit
  /ship report          View full audit reports
  /ship run qa-test     Re-run QA after fixing failures
  /ship cancel          Archive and start fresh
```

## `/ship help`

```
/ship — Pipeline Orchestrator

USAGE
  /ship                           Auto-detect intent and propose plan
  /ship [description]             Start pipeline for a task
  /ship [template]                Run a saved template

COMMANDS
  help                            Show this help
  status                          Current pipeline state
  resume                          Resume interrupted pipeline
  cancel                          Cancel and archive current pipeline
  plan [description]              Show execution plan without running
  history                         Recent pipeline runs
  report                          Latest pipeline report
  clean                           Archive old pipeline data

SKILLS
  skills                          List installed skills
  run [skill]                     Run a single skill
  add [skill]                     Install a skill
  remove [skill]                  Unregister a skill

TEMPLATES
  templates                       List all templates
  create-template [name]          Create custom template
  edit-template [name]            Edit template
  delete-template [name]          Delete template
  [template-name]                 Run named template

BUILT-IN TEMPLATES
  full-feature                    spec → dev → audit → deploy
  bug-fix                         dev → test → deploy
  quick-release                   test → security → deploy
  audit-only                      all quality checks, no deploy

INSTALLED SKILLS (12 of 16)
  ✅ design-audit  ✅ qa-test  ✅ security-audit  ✅ perf-audit
  ✅ analytics-audit  ✅ product-spec  ✅ deploy  ✅ api-spec
  ✅ db-migrate  ✅ content-copy  ✅ incident  ✅ infra-setup
  ○ app-store  ○ data-seed  ○ device-test  ○ feature-dev

EXAMPLES
  /ship fix the checkout bug
  /ship quick-release
  /ship run design-audit
  /ship create-template my-release-flow
```

## `/design-audit` Report

```markdown
# Design Audit Report — 2026-04-05

## Summary
- Screens captured: 24 (12 light, 12 dark)
- Issues found: 6
- CRITICAL: 1 | MAJOR: 3 | MINOR: 2

## Issues

| # | Severity | Screen | Issue | Description |
|---|----------|--------|-------|-------------|
| 1 | CRITICAL | Upload modal (dark) | Touch target too small | Upload button is 32x32pt — below 44pt minimum. Users will mis-tap, especially on smaller devices. |
| 2 | MAJOR | Photo gallery | Inconsistent card radius | Gallery thumbnails use 8px radius while all other cards use 12px (`radius.button`). Breaks visual consistency. |
| 3 | MAJOR | Upload progress | Missing skeleton state | Upload screen shows a raw spinner instead of a skeleton placeholder. Every other loading state in the app uses skeletons. |
| 4 | MAJOR | Gallery (dark) | Border invisible | Gallery card border uses `borderLight` at 6% opacity — invisible on `bgSurface` (#1E1E1E). Should be 10% per dark mode spec. |
| 5 | MINOR | Empty gallery | Generic empty state | Shows "No photos" with a gray icon. Should follow the app pattern: inspirational copy ("Be the first to share a photo of this spot") with an illustration. |
| 6 | MINOR | Photo detail | Caption font mismatch | Caption uses system font instead of Plus Jakarta Sans. Only visible on Android. |

## Proposed Fixes

### Issue 1 — Upload button touch target (CRITICAL)
**File:** `src/features/spots/components/PhotoUploadButton.tsx:47`
**Current:** `width: 32, height: 32`
**Fix:** `width: 48, height: 48` — matches `spacing[12]` and the 48px button height standard.
**Risk:** None. Increases tap area without changing visual size (use `padding` to maintain icon size).

### Issue 2 — Gallery thumbnail radius (MAJOR)
**File:** `src/features/spots/components/SpotPhotoGallery.tsx:83`
**Current:** `borderRadius: 8`
**Fix:** `borderRadius: theme.radius.button` (12px)
**Risk:** None.

### Issue 3 — Upload skeleton (MAJOR)
**File:** `src/features/spots/components/PhotoUploadProgress.tsx:22`
**Current:** `<ActivityIndicator />`
**Fix:** Replace with `<SkeletonCard width="100%" height={200} />` — same pattern used in SpotCard, EventListRow, and ListCard loading states.
**Risk:** None.

## Screenshots
  Before: .claude/pipeline/features/user-photo-uploads/screenshots/gallery-dark-before.png
  After:  .claude/pipeline/features/user-photo-uploads/screenshots/gallery-dark-after.png
```

## `/qa-test` Report

```markdown
# QA Test Report — 2026-04-05

## Summary
- Flows tested: 42
- Passed: 38 ✅
- Failed: 4 ❌
- Skipped: 0
- Duration: 18 min 42s
- Platform: iOS Simulator (iPhone 16 Pro, iOS 18.4)

## Results by Category

### Auth (5/5 passed)
  ✅ Email signup → onboarding → home
  ✅ Apple Sign-In → returning user
  ✅ Google OAuth → new user → onboarding
  ✅ Logout → login screen
  ✅ Token refresh → silent re-auth

### Photo Upload (4/8 passed)
  ✅ Select photo from library → preview → upload → appears in gallery
  ✅ Take photo with camera → upload → appears in gallery
  ✅ Upload 5 photos sequentially → all appear in correct order
  ✅ Cancel upload midway → no partial upload in gallery
  ❌ Upload 15MB photo → timeout with no error message
  ❌ Deny camera permission → app shows blank screen instead of permission prompt
  ❌ Upload HEIC photo → conversion fails silently, gallery shows broken image
  ❌ View gallery with 0 photos → blank white screen (no empty state)

### Spot Detail (6/6 passed)
  ✅ View spot → all sections render
  ✅ Save spot → appears in saved list
  ✅ Share spot → share sheet opens with correct URL
  ✅ Report spot → confirmation shown
  ✅ Write review → appears in reviews section
  ✅ View on map → map centers on spot

### Trip Planning (8/8 passed)
  ✅ Create trip → wizard completes → trip detail loads
  ✅ Add spot to trip → appears in itinerary
  ✅ Remove spot from trip → removed from itinerary
  ✅ Generate AI itinerary → items populate
  ✅ Share trip → link copied
  ✅ Edit trip dates → itinerary adjusts
  ✅ Delete trip → removed from trips list
  ✅ Duplicate trip → new trip with same items

### Discovery (7/7 passed)
  ✅ Home feed loads → modules render
  ✅ Explore tab → spots, events, vibes tabs work
  ✅ Search spots → results appear
  ✅ Filter by vibe → results filtered
  ✅ City guide → sections render
  ✅ Trending section → spots load
  ✅ For You → personalized results appear

### Social (8/8 passed)
  ✅ Follow user → follower count updates
  ✅ Unfollow user → follower count updates
  ✅ Like review → count updates
  ✅ Comment on review → appears in thread
  ✅ View profile → sections render
  ✅ Edit profile → changes saved
  ✅ Block user → content hidden
  ✅ Report content → confirmation shown

## Failure Details

### ❌ F1: Upload 15MB photo — timeout
**Flow:** Select oversized photo → tap Upload → wait
**Expected:** Photo compressed and uploaded, or error shown if too large
**Actual:** Spinner runs for 60s, then nothing. No error toast, no timeout message.
**Screenshot:** `.claude/pipeline/features/user-photo-uploads/screenshots/upload-timeout.png`
**Suggested fix:** Add a 30s timeout to `useUploadPhoto` mutation. Show toast: "Photo is too large. Try a smaller image."
**File:** `src/features/spots/hooks/useUploadPhoto.ts:34`

### ❌ F2: Camera permission denied — blank screen
**Flow:** Tap "Take Photo" → deny permission in iOS dialog
**Expected:** Permission explanation screen with "Open Settings" button
**Actual:** Blank white screen. Back button still works.
**Screenshot:** `.claude/pipeline/features/user-photo-uploads/screenshots/camera-denied.png`
**Suggested fix:** Check `Camera.getPermissionsAsync()` result, show `StateCard` with tone="neutral" and "Open Settings" CTA.
**File:** `src/features/spots/components/CameraCapture.tsx:18`

### ❌ F3: HEIC upload — silent conversion failure
**Flow:** Select HEIC photo from library → tap Upload
**Expected:** Auto-convert to JPEG, upload succeeds
**Actual:** Upload completes but gallery shows broken image icon. No error surfaced.
**Suggested fix:** Add HEIC→JPEG conversion in `compressImage()` using `expo-image-manipulator`. Fall back to original format if conversion fails.
**File:** `src/utils/compressImage.ts:12`

### ❌ F4: Empty gallery — blank screen
**Flow:** Navigate to spot with 0 user photos → scroll to gallery section
**Expected:** Empty state: "Be the first to share a photo" with upload CTA
**Actual:** Section header renders, but content area is blank white.
**Suggested fix:** Add empty state check in `SpotPhotoGallery` — when `photos.length === 0`, render `StateCard` with upload CTA.
**File:** `src/features/spots/components/SpotPhotoGallery.tsx:55`
```

## `/security-audit` Report

```markdown
# Security Audit Report — 2026-04-05

## Summary
- Scanned: 247 files, 14 API endpoints
- Framework: React Native (Expo) + Supabase
- CRITICAL: 0 | MAJOR: 1 | MINOR: 3

## OWASP Top 10 Coverage
| Category | Status | Findings |
|----------|--------|----------|
| A01: Broken Access Control | PASS | 0 |
| A02: Cryptographic Failures | PASS | 0 |
| A03: Injection | PASS | 0 |
| A04: Insecure Design | PASS | 0 |
| A05: Security Misconfiguration | WARN | 1 |
| A06: Vulnerable Components | WARN | 1 |
| A07: Auth Failures | PASS | 0 |
| A08: Data Integrity Failures | PASS | 0 |
| A09: Logging Failures | WARN | 2 |
| A10: SSRF | PASS | 0 |

## Findings

### MAJOR
| # | Issue | OWASP | File | Line | Description |
|---|-------|-------|------|------|-------------|
| 1 | Upload endpoint missing file type validation | A05 | `supabase/functions/upload-photo/index.ts` | 23 | Accepts any file type. An attacker could upload executable files or SVGs with embedded scripts. Should validate MIME type against allowlist (image/jpeg, image/png, image/webp, image/heic). |

### MINOR
| # | Issue | OWASP | File | Line | Description |
|---|-------|-------|------|------|-------------|
| 2 | Verbose error in upload handler | A09 | `supabase/functions/upload-photo/index.ts` | 67 | `catch(e) { return new Response(e.message) }` exposes internal error details. Should return generic "Upload failed" message. |
| 3 | Console.error not DEV-guarded | A09 | `src/features/spots/hooks/useUploadPhoto.ts` | 41 | `console.error(err)` runs in production. Wrap with `if (__DEV__)`. |
| 4 | Outdated `sharp` dependency | A06 | `package.json` | — | `sharp@0.32.1` has a known DoS vulnerability (CVE-2024-29041). Update to `>=0.33.3`. |

## Proposed Fixes

### Finding 1 — File type validation (MAJOR)
**Current:**
```typescript
const file = await req.blob()
await supabase.storage.from('photos').upload(path, file)
```
**Fix:**
```typescript
const file = await req.blob()
const allowed = ['image/jpeg', 'image/png', 'image/webp', 'image/heic']
if (!allowed.includes(file.type)) {
  return new Response('Invalid file type', { status: 400 })
}
await supabase.storage.from('photos').upload(path, file)
```
**Risk:** Could reject valid images with unusual MIME types. Mitigated by including all common image types.
```

## `/ship history`

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Pipeline History
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ⏳ user-photo-uploads   in progress   6/9 steps   started 6 hours ago
  ✅ event-rsvp-flow       shipped       7/7 steps   2h 10m total
  ✅ dark-mode-fixes       shipped       3/3 steps   45m total
  ✅ trip-collaboration    shipped       9/9 steps   3h 50m total
  ❌ push-notifications    cancelled     1/8 steps   "blocked on APNs certificate"

Totals (last 30 days):
  Shipped: 11 features
  Avg cycle: 2.1 hours
  Issues caught: 63 (before production)

Details: /ship report [feature-name]
```

## Session Log

What gets written to `.claude/pipeline/history/` after each session.

```markdown
# Session Log — 2026-04-05

## Feature: user-photo-uploads
## Branch: feature/user-photo-uploads
## Duration: 2h 15m (10:00 — 12:15)
## Device: christofers-macbook

## Steps Completed
1. design-audit — 25 min — 6 issues found, 5 fixed
2. qa-test — 20 min — 42 flows, 38 passed, 4 failed
3. security-audit — 9 min — 1 major, 3 minor

## Key Decisions
- 10:32 — Gallery thumbnails: 12px radius (matching card standard, not 8px)
- 10:41 — Upload button: increased to 48px touch target
- 11:15 — HEIC conversion: will use expo-image-manipulator, not server-side
- 11:48 — File type validation: allowlist approach, not blocklist

## Open Issues (4)
- Upload timeout (no error shown after 60s)
- Camera permission denial (blank screen)
- HEIC conversion (broken image)
- Empty gallery (no empty state)

## Context for Next Session
- 4 QA failures need code fixes before re-running qa-test
- Security finding #1 (file type validation) approved — ready to implement
- Next pipeline step: analytics-audit (after QA fixes)
- Upload uses Supabase Storage — test with files >10MB after timeout fix

## Files Changed
- src/features/spots/components/SpotPhotoGallery.tsx
- src/features/spots/components/PhotoUploadButton.tsx
- src/features/spots/components/PhotoUploadProgress.tsx
- src/constants/theme.ts (added radius.thumbnail token)

## Git Commits
- a1b2c3d design-audit: fix gallery dark mode border and touch targets
- d4e5f6a design-audit: add skeleton state to upload progress
- g7h8i9j security-audit: report — 1 major, 3 minor findings
```
