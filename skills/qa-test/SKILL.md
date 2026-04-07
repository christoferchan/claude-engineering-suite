---
name: qa-test
description: End-to-end functional QA pipeline — maps user flows, generates test scenarios (happy path, edge cases, destructive, permissions), executes via Maestro/Playwright, reports pass/fail with failure screenshots and suggested fixes. Use when the user asks to test their app, run QA, check if flows work, or verify functionality before release.
---

# QA Test Skill

End-to-end functional testing pipeline: Map → Generate → Execute → Report → Fix.

## When to Use

- User says "test my app", "run QA", "check if everything works"
- Before a release to verify all flows function correctly
- After a major refactor to catch regressions
- When a specific flow is reported broken

## Phase 0A: User Interview

Ask these questions before starting. Wait for answers.

### Required Questions

**1. What broke recently?**
"Any specific flows or features that you know are broken or flaky?"
→ Test these first with extra attention.

**2. Critical paths:**
"Which flows absolutely cannot break? (e.g., auth, payments, trip creation)"
→ These get the most thorough testing.

**3. Scope:**
- Full app (every flow, every edge case)
- Critical paths only (auth + core features)
- Specific flow (user names which one)
- Regression only (just rerun previous failures)

**4. API dependencies:**
"Are all backend services running? (Supabase, edge functions, external APIs)"
→ Some tests need live APIs (Scout chat, itinerary generation). Skip these if services are down.

**5. Test accounts:**
"Do you have a test account with data? (trips, saved spots, reviews, etc.)"
→ Many flows need pre-existing data to test meaningfully.

**6. Implementation preference:**
After the report, do you want me to:
- Just report failures (no code changes)
- Report + suggest fixes
- Report + fix critical failures automatically

### Store Preferences

Save to memory so future runs don't re-ask:
```
memory/feedback_qa_test_preferences.md
```

## Platform Detection

Same as design-audit skill — detect mobile/web/both, check runtime status.

```bash
# Mobile
ls app.json 2>/dev/null && echo "MOBILE"
xcrun simctl list devices booted 2>&1 | grep -q "Booted" && echo "SIMULATOR_READY"
curl -s http://localhost:8081/status 2>/dev/null && echo "METRO_RUNNING"

# Web
ls next.config.* vite.config.* 2>/dev/null && echo "WEB"
curl -s http://localhost:3000 2>/dev/null && echo "WEB_SERVER_RUNNING"
```

Present status and ask user to start any services that aren't running.

## Phase 1: Map User Flows

### Scan the Codebase

**1. Route map — every screen the user can reach:**
```bash
# Expo Router / file-based routing
find app/ -name "*.tsx" -not -name "_*" -not -name "*.test.*" | sort

# Next.js
find app/ -name "page.tsx" -o -name "page.js" | sort
find pages/ -name "*.tsx" -not -name "_*" | sort
```

**2. Mutations — every action that changes data:**
```bash
# React Query mutations
grep -rn "useMutation" src/ --include="*.ts" --include="*.tsx" | grep -v test | grep -v node_modules

# Direct Supabase writes
grep -rn "\.insert\|\.update\|\.delete\|\.upsert\|\.rpc" src/ --include="*.ts" --include="*.tsx" | grep -v test
```

**3. Auth flows — login, signup, logout, token refresh:**
```bash
grep -rn "signIn\|signUp\|signOut\|resetPassword\|onAuthStateChange" src/ --include="*.ts" --include="*.tsx"
```

**4. Navigation flows — how screens connect:**
```bash
# Find all router.push / router.replace calls
grep -rn "router\.push\|router\.replace\|router\.navigate\|router\.back" app/ src/ --include="*.tsx" | head -50
```

**5. Edge function calls — API-dependent flows:**
```bash
grep -rn "invokeProtectedEdgeFunction\|supabase\.functions\.invoke" src/ --include="*.ts" --include="*.tsx"
```

**6. Permission requests — OS-level prompts:**
```bash
grep -rn "requestForegroundPermissionsAsync\|requestCameraPermissionsAsync\|requestMediaLibraryPermissionsAsync\|registerForPushNotificationsAsync" src/ --include="*.ts" --include="*.tsx"
```

### Generate Flow Map

Organize into user journeys:

```
ONBOARDING
  welcome → login/signup → username → interests → first-trip → home

DISCOVERY
  home → explore → spot detail → save/add-to-trip
  home → trending → spot detail
  home → cities → city detail → spot detail
  explore → events → event detail → RSVP/add-to-trip
  explore → vibes → filtered spots

TRIP PLANNING
  trips → plan wizard (6 steps) → generate itinerary → trip detail
  trip detail → day selector → item quick view → edit item
  trip detail → essentials → country info/packing/phrases
  trip detail → tools → share/collaborate

SCOUT AI
  scout → send message → receive response → follow-up actions

SOCIAL
  spot detail → write review → submit
  spot detail → quick tip → submit
  spot detail → save to list → create list
  profile → edit profile → save

SETTINGS
  profile → appearance (dark mode toggle)
  profile → submit spot/event
  profile → help
  profile → upgrade/paywall
```

### Identify Critical Paths

Rank flows by business impact:
- **P0 (must work):** auth, trip creation, itinerary generation, save/unsave
- **P1 (should work):** reviews, Scout chat, search, notifications
- **P2 (nice if works):** share, collaborate, submit spot/event, help

## Phase 2: Generate Test Scenarios

For each flow, generate scenarios across 4 categories:

### Happy Path
The standard user journey — everything works, data is valid, network is good.

Example for "Plan Trip":
```
1. Tap "+" FAB on trips tab
2. Enter destination "Tokyo"
3. Enter country "Japan"
4. Tap Next
5. Toggle flexible dates
6. Tap Next
7. Select "Solo"
8. Tap Next
9. Select "Balanced"
10. Tap Next
11. Select "Mid-range"
12. Tap Next
13. Tap "Build my trip"
14. VERIFY: loading state appears
15. WAIT: itinerary generates (up to 120s)
16. VERIFY: trip detail shows with itinerary days
```

### Edge Cases
Unexpected but valid inputs:

- **Empty states:** user with zero trips, zero saves, zero reviews
- **Long text:** 100-character city name, 500-character review, 280-character tip
- **Rapid actions:** double-tap save, tap back during loading, switch tabs mid-mutation
- **Boundary values:** rating of 1 star, 0 search results, 999 saved spots
- **Null data:** spot with no image, event with no date, trip with no itinerary
- **Timeout:** API takes >30s — does the UI show loading/error correctly?

### Background/Foreground Transitions

Mobile-specific lifecycle tests for app suspension and restoration:

```yaml
# Maestro: send app to background and bring back
- pressKey: home
- waitForAnimationToEnd:
    timeout: 3000
- launchApp
# Verify state preserved
- assertVisible:
    id: "expected-element-still-there"
```

**Test scenarios:**

| Scenario | Expected Behavior |
|----------|-------------------|
| Background mid-form → return | Form data preserved, no reset |
| Background mid-mutation → return | Mutation completes or shows retry |
| Background for 5+ minutes → return | Session still valid or re-auth prompt shown |
| Memory pressure (simulate via Xcode) | App restores correctly, no data loss |
| Background during network request → return | Request completes or fails gracefully |
| Background with unsaved changes → return | Changes still present, not discarded |

```yaml
# Maestro: Background mid-form test
- launchApp
- runFlow:
    file: ../shared/auto-login.yaml
- tapOn:
    id: "tab-profile"
- tapOn:
    id: "profile-edit-button"
- tapOn:
    id: "edit-profile-bio-input"
- inputText: "Testing background persistence"
- hideKeyboard
# Go to background
- pressKey: home
- waitForAnimationToEnd:
    timeout: 5000
# Return to app
- launchApp
# Verify form data preserved
- assertVisible:
    text: "Testing background persistence"
- takeScreenshot: test-background-form-preserved
```

Note: Simulating memory pressure and extended background time is limited in Maestro. Flag these as "manual QA recommended" for thorough testing via Xcode Instruments.

### Destructive Paths
Actions that remove or break things:

- **Delete trip** → confirm dialog → trip gone from list
- **Unsave spot** → removed from saved list → can re-save
- **Cancel mid-wizard** → no half-created trip left behind
- **Sign out** → all cached data cleared, redirected to login
- **Delete account** → confirm, verify data removal
- **Remove from list** → spot gone, list count decrements
- **Block user** → their content hidden everywhere

### Permission Flows
OS-level permission denials:

- **Location denied** → "Near me" shows helpful message, not crash
- **Camera denied** → photo upload shows permission prompt, not crash
- **Notifications denied** → app works normally without push
- **Network offline** → offline banner shows, cached data readable, mutations queue

### Multi-User / Collaboration Testing

For apps with real-time or collaborative features, test concurrent usage scenarios.

**1. Detect if the app has collaboration features:**
```bash
grep -rn "realtime\|collaboration\|shared\|invite\|collaborator\|presence\|broadcast\|subscribe" src/ --include="*.ts" --include="*.tsx" | head -10
```

**2. If collaboration features exist, test these scenarios:**

| Scenario | Test Approach |
|----------|---------------|
| Concurrent edits — two sessions modifying same entity | Run two simulator instances or use a scripted API client as the second user |
| Conflict resolution — both change the same field | One session via Maestro, second via direct API call, verify merge/last-write-wins behavior |
| Permission levels — editor vs viewer | Login as viewer, attempt edit action, verify blocked. Login as editor, verify allowed |
| Invite flow — share with another user | Send invite, verify recipient sees the shared entity |
| Revoke access — remove collaborator | Remove user, verify they lose access immediately |
| Stale data — one user deletes while another views | Delete entity via API, verify viewing user sees error/refresh, not crash |

**3. Implementation approach:**
Since Maestro controls one simulator at a time, simulate the second user via direct API calls:
```bash
# Second user makes an edit via API while first user is viewing
curl -X PATCH "$SUPABASE_URL/rest/v1/trips?id=eq.$TRIP_ID" \
  -H "Authorization: Bearer $SECOND_USER_TOKEN" \
  -H "apikey: $ANON_KEY" \
  -d '{"city": "Updated by second user"}'
```
Then verify the first user's UI reflects the change (via realtime subscription or on next refresh).

**4. If the app has no collaboration features, skip this section automatically.**

### Deep Link Testing

Test that every deep link route works correctly across app states.

**1. Discover deep link routes:**
```bash
# Find deep link configuration
grep -rn "scheme\|deeplink\|linking\|openURL" app.json app.config.js src/ --include="*.ts" --include="*.tsx" | head -20

# Find all route patterns
find app/ -name "*.tsx" -not -name "_*" -not -name "*.test.*" | sed 's|app/||;s|\.tsx||;s|\[|:|g;s|\]||g' | sort
```

**2. Test scenarios:**

| Scenario | Expected Behavior |
|----------|-------------------|
| Valid deep link while logged in | Navigates to correct screen with correct data |
| Valid deep link while logged out | Auth prompt appears → after login, navigates to target |
| Deep link to deleted/non-existent entity | Error state shown (e.g., "Not found"), no crash |
| Malformed deep link | Graceful handling — fallback to home or error screen |
| Deep link with query params | Params parsed correctly, screen state matches |

**3. Maestro deep link tests:**
```yaml
# Test: Valid deep link to spot detail
- openLink: spots://spot/valid-spot-id
- assertVisible:
    id: "spot-detail-header"
- takeScreenshot: test-deeplink-spot-valid

# Test: Deep link to non-existent entity
- openLink: spots://trip/nonexistent-id
- assertVisible:
    text: "not found"  # or similar error state
- takeScreenshot: test-deeplink-trip-notfound

# Test: Malformed deep link
- openLink: spots://invalid/route/here
- assertVisible:
    id: "tab-home"  # Should fallback to home
- takeScreenshot: test-deeplink-malformed
```

**4. Web deep links (if applicable):**
```typescript
// Playwright: test web deep link redirect
test('web deep link redirects to app or shows content', async ({ page }) => {
  await page.goto('/spot/valid-spot-id');
  // Should show spot detail or "Open in app" CTA
  await expect(page.locator('[data-testid="spot-detail"]')).toBeVisible();
});
```

### Push Notification Handling

Full push notification testing requires physical devices. Generate a manual QA checklist instead.

**Manual QA checklist (mark as "manual QA required" in the report):**

- [ ] Tap notification from lock screen → app opens to correct screen
- [ ] Tap notification while app is in foreground → in-app navigation works
- [ ] Tap notification mid-flow (e.g., mid-wizard) → doesn't lose current state
- [ ] Tap notification when logged out → auth prompt → then navigate to target
- [ ] Notification badge count updates correctly after receiving notifications
- [ ] Notification badge count decrements after reading/clearing
- [ ] Dismiss notification → no side effects in the app
- [ ] Multiple notifications → tapping each navigates to the correct entity
- [ ] Notification with deleted entity → graceful error, not crash
- [ ] Notification permissions denied → app works normally without push

**Include in the report as:**
```markdown
## Manual QA Required: Push Notifications
The following scenarios cannot be automated via Maestro and require manual testing on a physical device:
[checklist above]
Status: NOT TESTED (requires manual QA)
```

### Test Coverage Map

After generating test scenarios, produce a coverage map before execution.

**1. List every flow/screen in the app:**
```bash
# All screens
find app/ -name "*.tsx" -not -name "_*" -not -name "*.test.*" -not -name "layout.tsx" | sort

# All mutations (user actions that change data)
grep -rn "useMutation" src/ --include="*.ts" --include="*.tsx" | grep -v test | grep -v node_modules | wc -l
```

**2. Build the coverage table:**

```markdown
## Test Coverage Map

| Flow / Screen | Has Tests | Priority | Risk |
|--------------|-----------|----------|------|
| Auth: login | Yes | P0 | - |
| Auth: signup | Yes | P0 | - |
| Auth: forgot password | No | P1 | MEDIUM |
| Trip: create wizard | Yes | P0 | - |
| Trip: edit itinerary | Yes | P0 | - |
| Trip: delete | Yes | P1 | - |
| Trip: share/collaborate | No | P2 | LOW |
| Spot: detail view | Yes | P0 | - |
| Spot: save/unsave | Yes | P0 | - |
| Scout: send message | Yes | P1 | - |
| Profile: edit | No | P1 | MEDIUM |
| Settings: dark mode | No | P2 | LOW |
| ...           | ...       | ...      | ...  |

**Coverage: 32 of 45 flows have tests (71%)**
```

**3. Flag untested critical paths:**
- Any P0 flow without tests → **HIGH RISK** — must add tests before release
- Any P1 flow without tests → **MEDIUM RISK** — should add tests
- P2 flows without tests → **LOW RISK** — nice to have

**4. Present to user for review before execution:**
> "Here's what I plan to test. 32 of 45 flows are covered (71%). 2 critical paths are untested [list them]. Want me to add those before running?"

This allows the user to adjust scope, add missing critical flows, or deprioritize low-risk gaps before spending time on execution.

## Phase 2B: Test Infrastructure

Before executing tests, set up the infrastructure they need to run reliably.

### Data Seeding (framework-agnostic)

Tests need data. Rather than hardcoding fixtures, detect what the app needs:

**1. Detect the backend framework:**
```bash
BACKEND="unknown"
ls Gemfile 2>/dev/null && BACKEND="rails"
ls requirements.txt manage.py 2>/dev/null && BACKEND="django"
ls go.mod 2>/dev/null && BACKEND="go"
ls pom.xml build.gradle 2>/dev/null && BACKEND="java"
ls Package.swift 2>/dev/null && BACKEND="swift"
ls Cargo.toml 2>/dev/null && BACKEND="rust"
ls composer.json artisan 2>/dev/null && BACKEND="laravel"
grep -q "prisma\|supabase\|firebase\|mongoose" package.json 2>/dev/null && BACKEND="node"
echo "Detected backend: $BACKEND"
```

**2. Discover the data model (framework-aware):**

For each backend, use the appropriate data model discovery:

| Backend | Discovery Command |
|---------|------------------|
| **Rails** | `grep -rn "has_many\|belongs_to\|has_one" app/models/` |
| **Django** | `grep -rn "class.*models.Model" */models.py` |
| **Prisma** | `cat prisma/schema.prisma \| grep "^model"` |
| **Go** | `grep -rn "type.*struct" internal/models/` |
| **Laravel** | `grep -rn "class.*extends Model" app/Models/` |
| **Supabase/Postgres** | `grep -rn "\.from(" src/ --include="*.ts" --include="*.tsx" \| sed "s/.*\.from('\([^']*\)').*/\1/" \| sort -u` |
| **Mongoose/MongoDB** | `grep -rn "\.find(\|\.create(\|\.findOne(" src/ --include="*.ts" \| head -20` |
| **Firebase** | `grep -rn "collection(\|doc(" src/ --include="*.ts" \| head -20` |

For JS/TS apps, also check the client-side data layer:
```bash
# Supabase / Postgres
grep -rn "\.from(" src/ --include="*.ts" --include="*.tsx" | sed "s/.*\.from('\([^']*\)').*/\1/" | sort -u
# Prisma
grep -rn "prisma\." src/ --include="*.ts" | sed "s/.*prisma\.\([a-zA-Z]*\)\..*/\1/" | sort -u
# Mongoose / MongoDB
grep -rn "\.find(\|\.create(\|\.findOne(" src/ --include="*.ts" | head -20
# Firebase
grep -rn "collection(\|doc(" src/ --include="*.ts" | head -20
```

**2. Discover what data each flow needs:**
- Scan test scenarios from Phase 2
- For each flow, identify required entities: "Edit profile" needs a user. "Edit trip" needs a trip. "Write review" needs a user + a spot.
- Build a dependency tree: user → trip → itinerary items, user → saved spots, user → reviews

**3. Generate a seed script based on the app's ORM/database:**

For Supabase:
```sql
-- Auto-generated test fixtures
-- Run before test suite, clean up after

INSERT INTO [table] ([columns]) VALUES ([test data])
ON CONFLICT DO NOTHING;
```

For Prisma:
```typescript
await prisma.[model].create({ data: { ... } })
```

For any backend:
```bash
# Use the app's own API/mutations to create test data
# This ensures validation rules are respected
curl -X POST /api/trips -d '{"city": "Test City", ...}'
```

**4. Seeding strategy — ask the user:**
> "Your tests need data to work. Options:
> 1. **Use existing data** — test against your real account (risky: destructive tests modify real data)
> 2. **Create test fixtures** — I'll generate seed data using your app's API/database (safe: isolated test data)
> 3. **Test account** — do you have a dedicated test account? (best: real-ish data, no risk to production)
> 
> Which approach?"

**5. For "create test fixtures" — auto-generate from the data model:**
- Read the database schema or type definitions
- Generate minimal valid records for each entity
- Respect foreign key relationships (create user before creating their trips)
- Use obviously fake data ("Test City", "test@example.com") so it's easy to clean up
- Tag test data with a marker (e.g., `source: 'qa_test'`) for cleanup

### State Reset Between Tests

Each test should be independent. Options (detect which is possible):

**Option A: Database transaction rollback (ideal)**
```sql
BEGIN;
-- run test
ROLLBACK;
```
Works with Postgres/Supabase. Each test runs in a transaction that's rolled back.

**Option B: App state reset**
```yaml
# Maestro: relaunch app between tests
- launchApp:
    clearState: true
```
Works for any mobile app. Clears AsyncStorage/MMKV/local state.

**Option C: Cleanup after each test**
Delete any data created during the test:
```bash
# Delete test-created records
DELETE FROM trips WHERE notes LIKE '%[QA_TEST]%';
DELETE FROM reviews WHERE body LIKE '%[QA_TEST]%';
```

**Option D: Dedicated test environment**
If the app supports multiple environments (dev/staging/prod), run tests against a disposable environment.

Ask the user which approach works for their stack.

### Flaky Test Detection

A test that fails intermittently is worse than one that always fails — it erodes confidence.

**Strategy:**
- On first failure, mark as RETRY
- Rerun the specific failed test up to 3 times
- Results:
  - 3/3 fail → FAILED (real bug)
  - 1/3 or 2/3 fail → FLAKY (intermittent, investigate)
  - 0/3 fail after initial failure → FLAKY (timing-sensitive)

**Common flaky causes:**
- Animation not finished before assertion → add `waitForAnimationToEnd`
- Network latency varies → increase timeout
- Previous test left dirty state → add cleanup
- Element rendered after assertion fires → add `extendedWaitUntil`

Track flaky tests across runs. If a test is flaky 3+ times, flag it for investigation.

### Performance Budgets (optional)

While testing flows, measure timing:

```yaml
# Maestro timing example
- startTimer: "trip-creation"
- tapOn:
    id: "plan-trip-build-button"
- extendedWaitUntil:
    visible:
      id: "trip-tab-plan"
    timeout: 120000
- stopTimer: "trip-creation"
# Budget: < 60s
```

**Default budgets (customize per app):**
| Operation | Budget |
|-----------|--------|
| Screen navigation | < 500ms |
| List load | < 2s |
| Search results | < 1s |
| Form submission | < 3s |
| AI generation | < 90s |
| Image upload | < 10s |

Pass = within budget. Warn = 1.5x budget. Fail = 2x budget.

Ask the user: "Do you want performance timing? If yes, what are your acceptable thresholds?"

### Subscription / Paywall State

For apps with premium tiers, test that gating works correctly.

**1. Detect if the app has subscription/paywall logic:**
```bash
grep -rn "subscription\|paywall\|premium\|isPremium\|tier" src/ --include="*.ts" --include="*.tsx" | head -10
```

**2. If found, ask the user:**
> "Your app has subscription/paywall logic. How should I test premium features?
> - **Option A:** Test account has premium enabled (I'll test both free and premium paths by toggling)
> - **Option B:** Mock the subscription check — point me to the function that checks premium status
> - **Option C:** Use a sandbox/test purchase flow (Apple StoreKit Testing, Google Play test tracks)"

**3. Generate tests for subscription states:**

| Scenario | Expected Behavior |
|----------|-------------------|
| Free user taps gated feature | Paywall appears, feature blocked |
| Free user hits usage limit | Limit message shown, upgrade CTA visible |
| Premium user accesses gated feature | Feature works normally, no paywall |
| Premium user sees no upgrade CTAs | Upgrade prompts hidden for premium users |
| Subscription expires mid-session | Graceful downgrade, no crash |
| Restore purchases | Previously purchased tier restored |

```yaml
# Maestro: Free user hits paywall
- launchApp
- runFlow:
    file: ../shared/auto-login.yaml  # Login as free-tier user
- tapOn:
    id: "tab-scout"
# Send messages until limit hit
- repeat:
    times: 10
    commands:
      - tapOn:
          id: "scout-input"
      - inputText: "Test message"
      - tapOn:
          id: "scout-send-button"
      - extendedWaitUntil:
          visible:
            id: "scout-response"
          timeout: 30000
# Verify paywall appears
- assertVisible:
    id: "paywall-modal"
- takeScreenshot: test-free-user-paywall
```

If the app has no subscription/paywall logic detected, skip this section automatically.

### Data Integrity Verification

After mutations, verify data actually persisted — not just that the UI showed success:

**Strategy (detect based on backend):**

For Supabase:
```bash
# After "create trip" test passes UI check:
npx supabase db query --linked "SELECT id FROM trips WHERE city = 'Test City' ORDER BY created_at DESC LIMIT 1"
# If empty → UI showed success but data didn't persist = critical bug
```

For any REST API:
```bash
curl -s /api/trips?city=TestCity | jq '.length'
# Should be > 0 after creation test
```

For apps without direct DB access:
- Navigate to the entity's list view and verify the item appears
- This is less reliable but works universally

### Mock/Stub Guidance

When to use real vs mocked services:

| Use Real APIs When | Use Mocks When |
|--------------------|---------------|
| Testing the full integration (E2E) | Testing UI behavior in isolation |
| Verifying data persistence | Testing error states reliably |
| Testing auth flows | Testing rate limits / edge cases |
| Final pre-release validation | CI runs that need speed + determinism |

**For apps using common patterns, suggest mock setup:**

- **React Query:** `QueryClient` with prefilled cache — override `queryClient.setQueryData` to inject test state without hitting the network
- **Supabase:** point to a test project or use local Supabase (`npx supabase start` for a local instance)
- **REST APIs:** suggest MSW (Mock Service Worker) for web, or mock the fetch layer for mobile
- **GraphQL:** suggest mock resolvers with tools like `@graphql-tools/mock`
- **Firebase:** use the Firebase emulator suite (`firebase emulators:start`)

**Ask user before proceeding:**
> "Do you want tests to hit real APIs or use mocks?
> - **Real** = more realistic but slower and flakier. Best for final validation.
> - **Mocks** = faster and deterministic but may miss integration bugs. Best for CI and rapid iteration.
> - **Hybrid** = real APIs for P0 critical paths, mocks for edge cases and error states."

If user chooses mocks, detect the app's data layer and generate the appropriate mock setup:
```bash
# Detect data layer
grep -rn "createClient\|supabase" src/lib/ --include="*.ts" | head -3  # Supabase
grep -rn "prisma" src/ --include="*.ts" | head -3                       # Prisma
grep -rn "initializeApp\|getFirestore" src/ --include="*.ts" | head -3  # Firebase
grep -rn "setupServer\|setupWorker" src/ --include="*.ts" | head -3     # MSW already set up
```

### Cross-Flow Dependency Graph

Generate execution order from dependencies:

```
# Independent (can parallelize)
auth_login
auth_signup
explore_browse
explore_search

# Depends on auth
save_spot → requires: auth_login
create_trip → requires: auth_login
write_review → requires: auth_login

# Depends on created data
edit_trip → requires: create_trip
delete_trip → requires: create_trip
add_to_trip → requires: create_trip + explore_browse
generate_itinerary → requires: create_trip

# Must run last (destructive)
delete_account → requires: auth_login (run LAST)
```

Execute independent flows in parallel, dependent flows sequentially, destructive flows last.

### Rollback Verification (optimistic updates)

For apps that use optimistic UI updates:

```yaml
# Test: Save spot → API fails → UI rolls back
- tapOn:
    id: "spot-save-button"
# Verify optimistic update showed
- assertVisible:
    text: "Saved"
# Now simulate failure (disconnect network if possible)
# Or: verify that on next app launch, the unsaved state is restored
- launchApp
- runFlow:
    file: ../shared/auto-login.yaml
- tapOn:
    id: "tab-explore"
- tapOn:
    id: "explore-spot-row-0"
# Verify the save didn't actually persist
- assertVisible:
    text: "Save"  # Not "Saved"
```

Note: Network simulation is limited in Maestro. For thorough rollback testing, flag for manual QA with specific instructions.

## Phase 3: Generate & Execute Test Flows

### Mobile: Maestro Flows

For each scenario, generate a `.yaml` Maestro flow:

```yaml
appId: com.example.app
---
# Test: Plan Trip Happy Path
- launchApp
- runFlow:
    file: ../shared/auto-login.yaml

# Navigate to trips
- tapOn:
    id: "tab-trips"
- assertVisible:
    id: "trips-plan-trip-fab"

# Start wizard
- tapOn:
    id: "trips-plan-trip-fab"
- assertVisible:
    text: "Where to?"

# Step 1: Destination
- tapOn:
    id: "plan-trip-destination-input"
- inputText: "Tokyo"
- hideKeyboard
- tapOn:
    id: "plan-trip-country-input"
- inputText: "Japan"
- hideKeyboard
- tapOn:
    id: "plan-trip-next-button"

# ... continue through all steps

# Verify result
- assertVisible:
    id: "trip-tab-plan"
- takeScreenshot: test-plan-trip-success
```

### Assertion Types

Use these Maestro assertions to verify behavior:

```yaml
# Element exists
- assertVisible:
    id: "element-testid"

# Text content
- assertVisible:
    text: "Expected text"

# Element NOT visible (for destructive tests)
- assertNotVisible:
    text: "Deleted item name"

# Wait for async operations
- extendedWaitUntil:
    visible:
      id: "result-element"
    timeout: 60000
```

### Web: Playwright Tests

```typescript
test('plan trip happy path', async ({ page }) => {
  await page.goto('/trips');
  
  // Start wizard
  await page.click('[data-testid="plan-trip-fab"]');
  await expect(page.locator('text=Where to?')).toBeVisible();
  
  // Fill destination
  await page.fill('[data-testid="destination-input"]', 'Tokyo');
  await page.fill('[data-testid="country-input"]', 'Japan');
  await page.click('[data-testid="next-button"]');
  
  // ... continue through steps
  
  // Verify
  await expect(page.locator('[data-testid="trip-tab-plan"]')).toBeVisible({ timeout: 120000 });
  await page.screenshot({ path: 'test-results/plan-trip-success.png' });
});
```

### Framework-Aware Selectors

Different frameworks use different selector patterns. Auto-detect which pattern the app uses and generate selectors accordingly.

**Selector patterns by framework:**

| Framework | Selector Attribute | Maestro Usage | Playwright Usage |
|-----------|-------------------|---------------|-----------------|
| React Native | `testID` | `id: "my-id"` | N/A |
| Flutter | `Key('my-key')` | `id: "my-key"` | N/A |
| iOS Native | `accessibilityIdentifier` | `id: "my-id"` | N/A |
| Android Native | `contentDescription` | `id: "my-id"` | N/A |
| React Web | `data-testid` | N/A | `[data-testid="my-id"]` |
| Vue | `data-test` | N/A | `[data-test="my-id"]` |
| Angular | `data-cy` or `data-testid` | N/A | `[data-testid="my-id"]` |

**Auto-detect the app's selector pattern:**
```bash
grep -rn "testID\|data-testid\|data-test\|data-cy\|Key(" src/ app/ --include="*.tsx" --include="*.jsx" --include="*.vue" --include="*.dart" | head -5
```

**Based on detection:**
- If `testID` found → React Native app → generate Maestro flows with `id:` selectors
- If `data-testid` found → React/Vue/Angular web app → generate Playwright tests with `[data-testid="..."]`
- If `Key(` found → Flutter app → generate Maestro flows with `id:` selectors
- If mixed (both `testID` and `data-testid`) → hybrid app → generate both Maestro and Playwright tests
- If no selectors found → warn user: "No test selectors detected. Tests will rely on text matching, which is brittle. Consider adding testID/data-testid attributes to interactive elements."

Generate selectors in the correct format for the detected framework. When generating new test flows, use the same naming convention observed in existing selectors (e.g., if the app uses `kebab-case-ids`, don't generate `camelCaseIds`).

### Execution Strategy

**Run order:**
1. P0 critical paths first (fail fast on broken core flows)
2. P1 flows
3. Edge cases
4. Destructive paths (run last — they modify data)
5. Permission flows (may need simulator reset)

**On failure:**
- Take screenshot of the failure state
- Log which step failed and what was expected vs actual
- Continue to next test (don't stop the entire suite)
- Mark the flow as FAILED in the report

**Parallel execution:**
- Independent flows can run in parallel (save spot + plan trip)
- Sequential flows must wait (create trip → edit trip → delete trip)
- Use Maestro's `--shard` for parallel mobile tests
- Use Playwright's built-in parallelism for web

### Handling API-Dependent Tests

Some flows require live backend responses:

```yaml
# For Scout chat — needs Anthropic API
- inputText: "Best coffee in Tokyo?"
- tapOn:
    id: "scout-send-button"
- extendedWaitUntil:
    visible:
      text: "Browse"  # Any response indicator
    timeout: 90000
# If timeout: mark as SKIPPED (API unavailable), not FAILED
```

For tests that need API calls, add a timeout and mark as SKIPPED (not FAILED) if the API is unavailable. Distinguish between "the app broke" and "the API was slow."

## Phase 4: Report

### Report Format

```markdown
# QA Test Report — [date]

## Summary
- Total flows: 42
- Passed: 38 ✅
- Failed: 3 ❌
- Skipped: 1 ⏭️ (API unavailable)

## Critical Path Status
| Flow | Status | Notes |
|------|--------|-------|
| Auth: login | ✅ Pass | |
| Auth: signup | ✅ Pass | |
| Trip: create | ✅ Pass | |
| Trip: generate itinerary | ✅ Pass | 45s generation time |
| Save: toggle | ✅ Pass | |
| Scout: send message | ⏭️ Skip | API timeout after 90s |

## Failures

### ❌ FAIL: Edit Profile → Save
**Step failed:** Step 8 — Tap "Save" button
**Expected:** Toast "Profile updated" + navigate back
**Actual:** Nothing happened. Button appeared disabled.
**Screenshot:** test-results/edit-profile-fail.png
**Suggested fix:** Check if the save mutation is firing. The button may be gated on form validation that's too strict.
**File:** `app/(modals)/edit-profile.tsx`

### ❌ FAIL: Delete Trip → Confirm
...

## Edge Case Results
| Scenario | Status |
|----------|--------|
| Empty state: zero trips | ✅ |
| Long city name (40 chars) | ✅ |
| Rapid double-tap save | ❌ — double mutation fired |
| Offline: save spot | ✅ — queued correctly |

## Regression Delta
(compared to previous run)
- Previously failing: 5
- Now fixed: 3
- New failures: 1
- Net: improved by 2
```

### Save Report

Write to `maestro/test-results/[date]-qa-report.md`

Save test result history for regression tracking:
```
maestro/test-results/
├── 2026-04-07-qa-report.md
├── 2026-04-06-qa-report.md
└── test-screenshots/
    ├── edit-profile-fail.png
    └── double-tap-save-fail.png
```

## Phase 5: Fix (if approved)

For each failure, propose a fix:

**For code bugs:**
1. Read the failing component
2. Identify the root cause from the failure description
3. Propose a fix with exact file + line + change
4. Implement if user approves
5. Rerun the specific test to verify

**For test issues (not app bugs):**
- Flaky selector: the testID changed or element loads slowly → update the test flow
- Data dependency: test assumes data that doesn't exist → add setup step
- Timing: async operation takes longer than timeout → increase timeout
- Simulator state: previous test left dirty state → add cleanup step

**For API issues:**
- Mark as SKIPPED, not FAILED
- Note the expected behavior and suggest manual testing
- If the API is consistently down, flag as a deployment issue

## Execution Modes

### Interactive (default)
- Ask all interview questions
- Present flow map for review before generating tests
- STOP after report for user to review
- Only fix what user approves

### Autonomous (skip-permissions / cron)
- Use stored preferences
- Run all P0 + P1 tests
- Generate report file
- Only auto-fix if:
  1. User stored "auto-fix critical" preference
  2. Failure is in app code (not API/timing)
  3. Fix is obvious (missing null check, wrong testID)
  4. Fix passes typecheck + tests after applying

### CI Mode
Generate a GitHub Actions workflow:

```yaml
name: QA Tests
on: pull_request
jobs:
  qa:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm install
      - name: Install Maestro
        run: curl -Ls "https://get.maestro.mobile.dev" | bash
      - name: Boot Simulator
        run: xcrun simctl boot "iPhone 16 Pro"
      - name: Build & Run
        run: npx expo run:ios --simulator
      - name: Run QA Tests
        run: |
          export PATH="$PATH:$HOME/.maestro/bin"
          maestro test maestro/flows/qa/ --format junit
      - name: Upload Results
        uses: actions/upload-artifact@v4
        with:
          name: qa-results
          path: maestro/test-results/
```

### Smart Test Selection for CI

For PR-level testing, map changed files to affected flows instead of running the full suite:

**1. Parse changed files:**
```bash
# Get files changed in this PR
CHANGED_FILES=$(git diff --name-only origin/master...HEAD)
echo "$CHANGED_FILES"
```

**2. Map files to affected test flows:**

| Changed File Pattern | Affected Tests |
|---------------------|---------------|
| `src/features/trips/*` | Trip-related flows only |
| `src/features/spots/*` | Spot detail, save, explore flows |
| `src/features/scout/*` | Scout AI chat flows |
| `src/features/auth/*` | Auth flows (login, signup, logout) |
| `src/features/reviews/*` | Review, tip, comment flows |
| `src/features/home/*` | Home feed flows |
| `src/components/ui/*` | ALL flows (shared UI) |
| `src/constants/theme.ts` | ALL flows (theme affects everything) |
| `app/(tabs)/*` | Tab-level flows for the changed tab |
| `app/(modals)/*` | Modal-specific flows |
| `app/spot/*` | Spot detail flows |
| `app/trip/*` | Trip detail flows |

**3. Selection logic:**
```bash
# If shared components or theme changed → run everything
if echo "$CHANGED_FILES" | grep -q "src/components/ui/\|src/constants/theme\|src/hooks/"; then
  echo "Shared code changed → running full suite"
  TEST_SCOPE="full"
else
  # Map feature directories to test flows
  AFFECTED_FLOWS=""
  echo "$CHANGED_FILES" | grep -q "features/trips\|app/trip\|app/(tabs)/trips" && AFFECTED_FLOWS="$AFFECTED_FLOWS trip"
  echo "$CHANGED_FILES" | grep -q "features/spots\|app/spot" && AFFECTED_FLOWS="$AFFECTED_FLOWS spot"
  echo "$CHANGED_FILES" | grep -q "features/auth\|app/(auth)" && AFFECTED_FLOWS="$AFFECTED_FLOWS auth"
  echo "$CHANGED_FILES" | grep -q "features/scout\|app/(tabs)/scout" && AFFECTED_FLOWS="$AFFECTED_FLOWS scout"
  echo "$CHANGED_FILES" | grep -q "features/reviews" && AFFECTED_FLOWS="$AFFECTED_FLOWS review"
  echo "$CHANGED_FILES" | grep -q "features/home\|app/(tabs)/index" && AFFECTED_FLOWS="$AFFECTED_FLOWS home"
  TEST_SCOPE="partial"
fi
```

**4. Output to user:**
> "3 files changed → 8 of 42 tests affected → running those 8"

**5. Full suite still runs on main/master merges:**
```yaml
on:
  pull_request:
    # Smart selection — only affected flows
  push:
    branches: [master, main]
    # Full suite — catch anything missed
```

## Companion Skills

This skill pairs with:
- **design-audit** — checks how it looks (visual QA)
- **qa-test** (this skill) — checks if it works (functional QA)

Run design-audit first (visual pass), then qa-test (functional pass). Together they cover the full quality spectrum.

## Limitations

- Cannot test real push notifications (needs physical device + APNs)
- Cannot test deep links from external sources (email, SMS)
- Cannot test payment flows (App Store sandbox requires manual setup)
- Cannot test multi-device sync scenarios
- API-dependent tests may be flaky if services are slow
- Maestro has limitations with complex gestures (pinch-zoom, 3D touch)
- Cannot verify data persistence after app kill + restart (Maestro relaunches fresh)

## Anti-Patterns

- DON'T test UI appearance — that's design-audit's job
- DON'T run destructive tests before happy-path tests
- DON'T mark API timeouts as app failures
- DON'T test with empty/fresh state only — many bugs only show with data
- DON'T skip edge cases — they're where most production bugs hide
- DON'T run all tests in sequence when they can parallelize
- DON'T ignore flaky tests — a test that sometimes fails is a bug waiting to happen
