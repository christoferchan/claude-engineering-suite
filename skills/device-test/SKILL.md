---
name: device-test
description: Device testing checklist generator — produces comprehensive manual QA checklists for real-device testing covering hardware (camera, GPS, haptics, biometrics), network conditions, OS integration (push notifications, deep links, share sheet), accessibility (VoiceOver, dynamic type), multi-device layouts, edge cases, and dark mode verification. Use when the user asks to test on real devices, prepare for QA, generate a test checklist, or verify hardware-dependent features.
---

# Device Test Skill

Generates structured manual testing checklists for real-device QA: Interview → Scan → Generate → Track.

## When to Use

- User says "test on real device", "generate test checklist", "what should I test manually"
- Before App Store submission to verify hardware-dependent features
- After Maestro/automated tests pass but real-device verification is needed
- When testing accessibility (VoiceOver, TalkBack, dynamic type)
- When testing network edge cases (offline, slow, cellular)

## Pipeline Context Protocol

### At Start — Read Pipeline State

```bash
# Check if running inside a pipeline
if [ -f .claude/pipeline/manifest.md ]; then
  cat .claude/pipeline/manifest.md
fi

# Check for feature context
FEATURE_SLUG=$(grep "^slug:" .claude/pipeline/manifest.md 2>/dev/null | awk '{print $2}')
if [ -n "$FEATURE_SLUG" ] && [ -d ".claude/pipeline/features/$FEATURE_SLUG" ]; then
  ls .claude/pipeline/features/$FEATURE_SLUG/
  # Read previous skill reports for context
  # QA report tells us what's already tested automatically
  cat .claude/pipeline/features/$FEATURE_SLUG/qa-report.md 2>/dev/null
  # Design audit tells us visual issues to verify on real device
  cat .claude/pipeline/features/$FEATURE_SLUG/design-audit.md 2>/dev/null
fi
```

### At End — Write Output

```bash
# If running inside a pipeline
mkdir -p .claude/pipeline/features/$FEATURE_SLUG
cat > .claude/pipeline/features/$FEATURE_SLUG/device-test-checklist.md << 'EOF'
[checklist content]
EOF

# If standalone
mkdir -p .claude/pipeline/features/device-test
# Write checklist
```

## Phase 0A: User Interview

Ask these questions before starting. Wait for answers.

### Required Questions

**1. Target devices:**
"What devices do you have available for testing?"
Examples:
- iPhone 16 Pro Max (6.7")
- iPhone 16 Pro (6.1")
- iPhone SE 3rd gen (4.7")
- iPad Air
- Android Pixel 8
- Android Samsung Galaxy S24

→ Generates device-specific test items (notch vs no notch, home button vs gesture, screen size).

**2. Minimum OS version:**
"What's your minimum supported OS version?"
- iOS: 16 / 17 / 18
- Android: 12 / 13 / 14

→ Tests features that behave differently on older OS versions.

**3. Feature scope:**
- Full app (every feature, every hardware interaction)
- New feature only (test what just shipped)
- Specific category (accessibility, network, hardware, etc.)
- Regression (retest previously failed items)

**4. Known hardware-dependent features:**
"Which features use device hardware?"
Examples: camera, GPS/location, haptics, biometrics (Face ID/Touch ID), NFC, Bluetooth, motion sensors

→ Auto-detected from code scan, but user confirms what's actually important.

**5. Testing environment:**
"Where will testing happen?"
- Office (reliable WiFi, no GPS movement)
- Field testing (real GPS movement, varying network)
- Both

→ Affects which network/GPS tests are practical.

**6. Accessibility priority:**
"How important is accessibility testing for this release?"
- Ship-blocking (must pass VoiceOver walkthrough)
- Important (test critical paths with VoiceOver)
- Best-effort (note issues but don't block)

### Store Preferences

Save to persistent memory:

```
memory/feedback_device_test_preferences.md
```

On subsequent runs: "I have your device testing preferences from last time — [summary]. Still accurate, or want to update?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after code scan to confirm detected features
- Generate checklist and present for review
- User marks items as they test

### Autonomous Mode (invoked by subagent or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** If no stored preferences, use safe defaults:
  - Scope: full app
  - Accessibility: important (test critical paths)
  - Detect devices from project config (app.json minimum OS, etc.)

- **Generate full checklist without stopping.** Write to pipeline feature directory.

- **NEVER claim tests passed** — this skill generates checklists for humans, it does not execute tests.

## Phase 1: Code Scan — Detect Hardware & OS Dependencies

### 1.1 Hardware Feature Detection

```bash
# Location / GPS
grep -rn "requestForegroundPermissions\|getCurrentPosition\|watchPosition\|useLocation\|expo-location" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v test

# Camera
grep -rn "Camera\|launchCamera\|requestCameraPermissions\|expo-camera\|expo-image-picker" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Haptics
grep -rn "Haptics\|expo-haptics\|ReactNativeHapticFeedback\|impactAsync\|notificationAsync" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Biometrics
grep -rn "LocalAuthentication\|authenticateAsync\|supportedAuthenticationTypes\|expo-local-authentication\|FaceID\|TouchID" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Push notifications
grep -rn "registerForPushNotifications\|expo-notifications\|getExpoPushToken\|scheduleNotification" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Motion / Accelerometer
grep -rn "Accelerometer\|Gyroscope\|DeviceMotion\|expo-sensors" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Clipboard
grep -rn "Clipboard\|setStringAsync\|getStringAsync\|expo-clipboard" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Share sheet
grep -rn "Share\|shareAsync\|expo-sharing" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Deep links
grep -rn "Linking\|useURL\|expo-linking\|deep.*link\|spots://" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Storage / file system
grep -rn "FileSystem\|documentDirectory\|cacheDirectory\|expo-file-system\|AsyncStorage\|MMKV" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Network status
grep -rn "NetInfo\|useNetworkStatus\|isConnected\|expo-network" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Background tasks
grep -rn "BackgroundFetch\|TaskManager\|expo-background-fetch\|expo-task-manager" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules
```

### 1.2 UI Feature Detection

```bash
# Animations
grep -rn "Animated\|useAnimatedStyle\|useSharedValue\|reanimated\|Reanimated\|withSpring\|withTiming" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules | wc -l

# Keyboard interactions
grep -rn "KeyboardAvoidingView\|useKeyboard\|keyboard.*dismiss\|keyboardShouldPersistTaps" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Scrolling / lists
grep -rn "FlatList\|SectionList\|ScrollView\|FlashList" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules | wc -l

# Maps
grep -rn "MapView\|react-native-maps\|GOOGLE_MAPS_API_KEY" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Video / audio
grep -rn "Video\|Audio\|expo-av\|expo-video" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules

# Dark mode
grep -rn "useColorScheme\|isDark\|darkMode\|appearance" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules | wc -l

# Safe area
grep -rn "SafeAreaView\|useSafeAreaInsets\|expo-safe-area" src/ app/ --include="*.ts" --include="*.tsx" | grep -v node_modules | wc -l
```

### 1.3 Permission Detection

```bash
# iOS permission strings (app.json or Info.plist)
grep -A1 "NSLocationWhenInUseUsageDescription\|NSCameraUsageDescription\|NSPhotoLibraryUsageDescription\|NSFaceIDUsageDescription\|NSUserTrackingUsageDescription\|NSMicrophoneUsageDescription" app.json ios/*/Info.plist 2>/dev/null

# Android permissions
grep "PERMISSION\|uses-permission" android/app/src/main/AndroidManifest.xml 2>/dev/null
grep -A5 "permissions" app.json 2>/dev/null
```

### 1.4 Generate Feature Summary

Present detected features to user for confirmation:

```
DETECTED HARDWARE FEATURES:
[x] Location (GPS) — 12 references found
[x] Push notifications — 8 references found
[x] Haptics — 5 references found
[x] Share sheet — 3 references found
[x] Clipboard — 2 references found
[ ] Camera — not detected
[ ] Biometrics — not detected
[ ] Motion sensors — not detected

DETECTED UI FEATURES:
[x] Dark mode — 45 references
[x] Animations — 38 files
[x] Keyboard interactions — 15 references
[x] Maps — 4 references
[x] Scroll lists — 52 references
[ ] Video/audio — not detected

Confirm: Are these correct? Any features to add or remove?
```

## Phase 2: Generate Checklist

Based on detected features + user preferences, generate a comprehensive checklist organized by category.

### Category 1: Hardware

#### Location / GPS
```markdown
### Location / GPS
- [ ] **Fresh install permission prompt**: Open app for first time → location permission dialog appears → text matches app.json description
- [ ] **Permission granted**: Allow location → nearby spots load correctly → distance calculations are accurate
- [ ] **Permission denied**: Deny location → app handles gracefully → shows fallback content (not crash/blank)
- [ ] **Permission "Ask Next Time"**: Select "Allow Once" → location works for session → re-prompts on next launch
- [ ] **Location disabled in Settings**: Turn off Location Services system-wide → app shows appropriate message
- [ ] **Permission revoked mid-use**: Grant location → go to Settings → revoke → return to app → handles state change
- [ ] **Background to foreground**: App uses location → background app → wait 5 min → foreground → location resumes
- [ ] **Indoor vs outdoor accuracy**: Test indoors (less accurate) → map pin placement is reasonable
- [ ] **Airplane mode + WiFi**: Airplane mode on, WiFi on → location via WiFi still works (less accurate)
- [ ] **Rapid location changes**: Simulate movement (drive/walk) → map/distance updates in real time (if applicable)
```

#### Camera / Photos
```markdown
### Camera / Photos (if detected)
- [ ] **Camera permission prompt**: First camera access → permission dialog → text matches app.json description
- [ ] **Photo library permission prompt**: First photo picker → permission dialog → limited vs full access options
- [ ] **Take photo**: Camera opens → take photo → photo appears in app correctly (right orientation)
- [ ] **Choose from library**: Photo picker opens → select photo → uploads correctly
- [ ] **Large photo handling**: Select a 12MP+ photo → app handles without crash → resizes appropriately
- [ ] **Camera denied**: Deny camera permission → app shows helpful message with Settings link
- [ ] **Photo library denied**: Deny photo access → app handles gracefully
- [ ] **Limited photo access (iOS 14+)**: Select "Limit Photos" → only selected photos available → app handles correctly
- [ ] **Front/back camera**: If camera UI allows switching → both cameras work
- [ ] **Low light**: Take photo in dim environment → reasonable result
```

#### Haptics
```markdown
### Haptics (if detected)
- [ ] **Haptic feedback fires**: Perform haptic-triggering action → feel the vibration
- [ ] **Haptic timing**: Feedback occurs at the right moment (not delayed)
- [ ] **Haptic intensity**: Appropriate for the action (light for selection, medium for success, heavy for error)
- [ ] **Haptics disabled**: System Settings → Accessibility → turn off haptics → app doesn't crash
- [ ] **Silent mode**: Phone on silent → haptics still work (they should)
```

#### Biometrics
```markdown
### Biometrics (if detected)
- [ ] **Face ID prompt**: Biometric action triggers → Face ID dialog appears → text is correct
- [ ] **Face ID success**: Authenticate → action proceeds
- [ ] **Face ID failure**: Cover camera → fails → shows fallback (password entry)
- [ ] **Face ID disabled**: Settings → Face ID off → app offers password fallback
- [ ] **Touch ID (older devices)**: Same tests as Face ID on Touch ID-capable device
- [ ] **Biometrics not enrolled**: No face/fingerprint enrolled → app handles gracefully
```

### Category 2: Network Conditions

```markdown
### Network — WiFi
- [ ] **Strong WiFi**: All screens load quickly → images render → no timeouts
- [ ] **Slow WiFi (simulate)**: Network Link Conditioner → 3G profile → app shows loading states → doesn't timeout too fast
- [ ] **WiFi disconnect mid-request**: Start loading → turn WiFi off → error handled → retry works

### Network — Cellular
- [ ] **4G/5G**: Full app works on cellular → no WiFi-only restrictions
- [ ] **3G/LTE (slow)**: Network Link Conditioner → slow profile → skeleton screens appear → content eventually loads
- [ ] **Edge/2G**: Very slow network → app doesn't hang → shows timeout message

### Network — Offline
- [ ] **Launch offline**: Airplane mode → open app → offline banner/message appears
- [ ] **Go offline during use**: Using app → airplane mode → immediate offline indicator
- [ ] **Cached content**: Go offline → previously loaded screens still show cached data (if applicable)
- [ ] **Queue actions offline**: Perform save/action while offline → queues for retry → succeeds when back online (if applicable)
- [ ] **Come back online**: Offline → reconnect → app recovers → pending actions complete
- [ ] **Offline → specific screen**: Navigate to each key screen while offline → no crashes, graceful degradation

### Network — Airplane Mode
- [ ] **Airplane mode on**: All network features show appropriate offline state
- [ ] **Airplane mode + WiFi**: Turn on airplane mode then enable WiFi → app works on WiFi
```

### Category 3: OS Integration

```markdown
### Push Notifications
- [ ] **Permission prompt**: First-time notification request → dialog appears → description is accurate
- [ ] **Notification received (foreground)**: App open → receive notification → in-app banner appears (not system alert)
- [ ] **Notification received (background)**: App in background → notification → system notification appears with correct title/body
- [ ] **Notification tap (background)**: Tap notification → app opens to correct screen (deep link works)
- [ ] **Notification tap (killed)**: Force-quit app → tap notification → app launches to correct screen
- [ ] **Notification permission denied**: Deny notifications → app functions normally, notification features show "enable in Settings"
- [ ] **Badge count**: Receive notifications → badge count updates → opening app clears badge (if implemented)
- [ ] **Do Not Disturb**: DND on → notifications still delivered when DND ends
- [ ] **Notification settings per type**: If app has notification categories → each can be toggled independently in Settings

### Deep Links
- [ ] **Universal link (web → app)**: Open deep link URL in Safari → app opens to correct screen
- [ ] **Universal link (email)**: Tap link in email → app opens correctly
- [ ] **Universal link (Messages)**: Tap link in iMessage → app opens correctly
- [ ] **Deep link with app not installed**: Tap link → goes to App Store / web fallback
- [ ] **Deep link with app in background**: App backgrounded → tap link → app foregrounds to correct screen
- [ ] **Custom scheme**: Open `spots://` URL → app handles correctly
- [ ] **Invalid deep link**: Open malformed URL → app handles gracefully (not crash)

### Share Sheet
- [ ] **Share content**: Tap share → system share sheet opens → all expected options present
- [ ] **Share to Messages**: Share → select Messages → content formats correctly
- [ ] **Share to social apps**: Share → select Twitter/Instagram/WhatsApp → content appears correctly
- [ ] **Copy link**: Share → Copy → paste elsewhere → link is valid
- [ ] **AirDrop**: Share → AirDrop to another device → content received correctly

### Clipboard
- [ ] **Copy action**: Tap copy → paste elsewhere → content is correct
- [ ] **Paste into app**: Copy text elsewhere → paste into app input → works correctly
- [ ] **iOS paste permission banner**: Paste triggers "App pasted from..." banner → acceptable UX
```

### Category 4: Battery & Performance

```markdown
### Battery Impact
- [ ] **Background GPS drain**: Use location features → background app for 30 min → check battery usage in Settings → acceptable
- [ ] **Animation performance**: Scroll through animated screens → no frame drops → 60fps feel
- [ ] **Heavy list scrolling**: Scroll long lists quickly → smooth performance → no jank
- [ ] **Image loading**: Screen with many images → load efficiently → no memory warnings
- [ ] **Extended use**: Use app for 15+ minutes continuously → battery drain is reasonable vs similar apps

### Memory Pressure
- [ ] **Many apps open**: Open 10+ apps → switch to this app → no crash → state preserved
- [ ] **Low memory warning**: Simulate with Instruments → app handles gracefully → doesn't crash
- [ ] **Low storage**: Fill device storage near capacity → app handles photo/data saves gracefully
- [ ] **Background kill recovery**: System kills app for memory → user opens → state restored or graceful restart

### Performance
- [ ] **Cold start time**: Force-quit → launch → time to interactive (target: under 3 seconds)
- [ ] **Warm start time**: Background → foreground → time to interactive (target: under 1 second)
- [ ] **Screen transition smoothness**: Navigate between screens → transitions are smooth → no flash/flicker
- [ ] **Scroll performance**: Scroll any long list → smooth at 60fps → no dropped frames
- [ ] **Image load time**: Navigate to image-heavy screen → images load progressively → no blank spaces lingering
```

### Category 5: Accessibility

```markdown
### VoiceOver (iOS) / TalkBack (Android)
- [ ] **Enable VoiceOver**: Settings → Accessibility → VoiceOver on → app is navigable
- [ ] **Every interactive element announced**: Swipe through each screen → every button, link, input is read aloud
- [ ] **Meaningful labels**: Elements read with descriptive labels (not "button" or "image")
- [ ] **Reading order**: VoiceOver reads elements in logical order (not random DOM order)
- [ ] **Custom actions**: Complex elements (cards, swipeable rows) have custom VoiceOver actions
- [ ] **Modal announcements**: Open modal → VoiceOver announces it → focus moves to modal
- [ ] **Dismiss modal**: Close modal → focus returns to triggering element
- [ ] **Navigation**: Can navigate entire app flow using only VoiceOver gestures
- [ ] **Form inputs**: Input fields announce label, current value, and keyboard type
- [ ] **Error announcements**: Form validation errors are announced by VoiceOver
- [ ] **Loading states**: Loading indicators are announced ("Loading, please wait")
- [ ] **Toast messages**: Toast announcements are read by VoiceOver

### Dynamic Type / Text Scaling
- [ ] **Largest text size**: Settings → Accessibility → Larger Text → max → app is usable
- [ ] **Smallest text size**: Settings → smallest → text is still readable → layout not broken
- [ ] **Accessibility text sizes (xxxLarge)**: Enable "Larger Accessibility Sizes" → app handles extreme sizes
- [ ] **Text truncation**: Large text → no text overlapping other elements → truncation with "..." where needed
- [ ] **Button targets still large enough**: Large text → buttons still have 44x44pt minimum touch target
- [ ] **Scroll to see all content**: If content overflows at large text → user can scroll to see everything

### Color & Visual Accessibility
- [ ] **Color inversion**: Settings → Accessibility → Smart Invert → app is usable → images not inverted
- [ ] **Reduce Transparency**: Enable Reduce Transparency → app still looks good → no invisible elements
- [ ] **Reduce Motion**: Enable Reduce Motion → animations simplified or removed → app still functional
- [ ] **Bold Text**: Enable Bold Text → text is bolder → layout not broken
- [ ] **Increase Contrast**: Enable Increase Contrast → colors adjust → no invisible elements
- [ ] **Color blind simulation**: Use Accessibility Inspector → check for info conveyed only by color
- [ ] **WCAG AA contrast**: Check all text against background → 4.5:1 for body, 3:1 for large text
```

### Category 6: Multi-Device & Layout

```markdown
### Screen Sizes
- [ ] **Small phone (iPhone SE / mini)**: All screens fit → no cut-off content → touch targets accessible
- [ ] **Standard phone (iPhone 15/16)**: Baseline layout looks correct → optimal spacing
- [ ] **Large phone (iPhone 15/16 Pro Max)**: Extra space used well → no awkward gaps → content scales
- [ ] **Dynamic Island / notch**: Content not hidden behind Dynamic Island → status bar area clear
- [ ] **Home indicator area**: Bottom content not obscured by home indicator → safe area respected

### iPad (if applicable)
- [ ] **iPad portrait**: Layout adapts → not just stretched phone layout
- [ ] **iPad landscape**: Layout adapts → uses horizontal space meaningfully
- [ ] **Split view**: App in split view → still usable at half width
- [ ] **Slide Over**: App in Slide Over → still usable at compact width
- [ ] **Stage Manager**: Multiple windows → app handles correctly

### Orientation
- [ ] **Portrait lock**: App locked to portrait → rotation doesn't break layout
- [ ] **Landscape (if supported)**: Rotate → layout adapts → no cut-off content
- [ ] **Rotation during interaction**: Rotate while modal is open → modal repositions correctly
```

### Category 7: Edge Cases & Interruptions

```markdown
### App Lifecycle
- [ ] **Incoming call**: Using app → receive phone call → answer → return to app → state preserved
- [ ] **Siri activation**: Using app → activate Siri → dismiss → return to app → state preserved
- [ ] **Control Center**: Swipe down Control Center → dismiss → app state preserved
- [ ] **Notification Center**: Swipe down notifications → dismiss → app state preserved
- [ ] **Screenshot**: Take screenshot → share sheet → dismiss → app still works
- [ ] **Screen recording**: Start screen recording → app functions normally → DRM content hidden (if applicable)
- [ ] **App switcher**: Double-tap home / swipe up → app appears in switcher → tap to return → state preserved
- [ ] **Force quit and relaunch**: Force quit → relaunch → app recovers gracefully (login state, last screen, etc.)

### Input Edge Cases
- [ ] **Emoji in text fields**: Enter emoji → displays correctly → saves correctly → doesn't crash
- [ ] **Very long text**: Enter 1000+ characters → handles gracefully → truncation or scroll
- [ ] **Special characters**: Enter `<script>`, quotes, backslashes → no injection → displays correctly
- [ ] **Paste large content**: Paste a very long string → app handles → no freeze
- [ ] **Right-to-left text**: Enter Arabic/Hebrew text → text displays correctly (if i18n supported)

### Permission Edge Cases
- [ ] **Revoke permission during use**: Using location → Settings → revoke location → return → app handles
- [ ] **Upgrade permission**: Limited photo access → go to Settings → give full access → return → app sees new photos
- [ ] **Fresh install (no permissions)**: Delete and reinstall → all permission prompts appear correctly → no stale state

### Time & Date
- [ ] **Timezone change**: Change timezone in Settings → dates in app update correctly
- [ ] **24-hour vs 12-hour**: Toggle time format → app displays times correctly in both formats
- [ ] **Calendar system**: If supporting non-Gregorian calendars → dates display correctly
- [ ] **Past dates**: Any date picker → can it select past dates when it shouldn't? (or vice versa)
```

### Category 8: Dark Mode (Real Device)

```markdown
### Dark Mode — Real Device Verification
- [ ] **System dark mode**: Settings → Display → Dark → app switches to dark theme
- [ ] **System light mode**: Settings → Display → Light → app switches to light theme
- [ ] **In-app toggle (if exists)**: Toggle dark mode in app settings → theme changes immediately
- [ ] **Transition**: Switch modes → no flash of wrong theme → smooth transition
- [ ] **Every screen**: Navigate all major screens in dark mode → no white flash / wrong backgrounds
- [ ] **Text readability**: All text readable in dark mode → sufficient contrast
- [ ] **Image/icon visibility**: Icons and images visible against dark backgrounds → no invisible elements
- [ ] **Status bar**: Status bar text is light in dark mode → readable
- [ ] **Keyboard theme**: Keyboard matches app theme (dark keyboard in dark mode)
- [ ] **Alert dialogs**: System alerts match theme (iOS 13+)
- [ ] **OLED display check**: On OLED device → true black (#000000) vs near-black → power saving vs readability
- [ ] **Screenshots comparison**: Compare dark mode on real device vs simulator → note any differences in color rendering
```

## Phase 3: Completion Tracking

### Checklist Format

The generated checklist uses markdown checkboxes that can be marked as tested:

```markdown
# Device Test Checklist — [App Name] v[version]

Generated: [date]
Tester: _______________
Device: _______________
OS Version: _______________

## Summary
Total items: [count]
Tested: [ ] / [count]
Passed: [ ] / [count]
Failed: [ ] / [count]
Skipped: [ ] / [count]

## How to Use
1. Work through each category in order
2. Mark items: [x] passed, [!] failed, [~] skipped, [-] N/A
3. For failures: note the exact behavior and screenshot it
4. When done: count results and fill in Summary above

---

[Categories from Phase 2, filtered to detected features]
```

### Failure Tracking

For each failed item, the tester should record:

```markdown
### FAILURE: [item description]
- **Expected**: [what should happen]
- **Actual**: [what actually happened]
- **Steps to reproduce**: [exact steps]
- **Device**: [device model + OS version]
- **Screenshot**: [path or reference]
- **Severity**: CRITICAL / MAJOR / MINOR
- **Blocking release?**: Yes / No
```

## Output Report Format

```markdown
# Device Test Report — [date]

## Test Environment
- Device: [model]
- OS: [version]
- App version: [version]
- Network: [WiFi / cellular / both]
- Tester: [name]

## Results Summary
| Category | Total | Passed | Failed | Skipped | N/A |
|----------|-------|--------|--------|---------|-----|
| Hardware | X | X | X | X | X |
| Network | X | X | X | X | X |
| OS Integration | X | X | X | X | X |
| Battery & Perf | X | X | X | X | X |
| Accessibility | X | X | X | X | X |
| Multi-Device | X | X | X | X | X |
| Edge Cases | X | X | X | X | X |
| Dark Mode | X | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** | **X** |

## Failures
[List of all failed items with details]

## Blockers (must fix before release)
[CRITICAL failures that block App Store submission]

## Recommended Fixes
[For each failure, suggested fix or investigation path]

## Sign-Off
- [ ] All CRITICAL items passed or have approved workarounds
- [ ] All P0 flows work on target devices
- [ ] Accessibility passes for critical paths
- [ ] Dark mode verified on real device
- [ ] Approved for release by: _______________
```

## Anti-Patterns

- DON'T claim a test passed without human verification — this skill generates checklists, not results
- DON'T skip accessibility testing — it's required for App Store compliance and the right thing to do
- DON'T test only on the newest device — older/smaller devices reveal layout issues
- DON'T test only on WiFi — cellular and offline reveal different bugs
- DON'T test only in light mode — dark mode on real OLED displays looks different than simulator
- DON'T skip interruption tests — phone calls and notifications during use are extremely common
- DON'T generate tests for features the app doesn't have — only include detected features
- DON'T make the checklist so long it's impractical — prioritize by risk, mark optional items
