---
name: app-store
description: App Store submission pipeline — generates screenshots via Maestro, formats for required device sizes, writes App Store description/subtitle/keywords, generates privacy policy, prepares review notes, runs compliance checklists (GDPR, COPPA, encryption), and handles ASO optimization. Supports iOS App Store, Google Play, and PWA. Use when the user asks to prepare for App Store submission, generate screenshots, write store listings, or check compliance.
---

# App Store Skill

End-to-end App Store submission pipeline: Interview → Detect → Capture → Generate → Compliance → Deliver.

## When to Use

- User says "prepare for App Store", "generate screenshots", "write store listing"
- User wants to submit to iOS App Store or Google Play
- User wants ASO (App Store Optimization) suggestions
- User needs privacy policy, review notes, or compliance check
- User says "what do I need before I can submit?"

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
  # Read previous skill reports — especially qa-report for known issues
  cat .claude/pipeline/features/$FEATURE_SLUG/*.md 2>/dev/null
fi
```

### At End — Write Output

```bash
# If running inside a pipeline
mkdir -p .claude/pipeline/features/$FEATURE_SLUG
cat > .claude/pipeline/features/$FEATURE_SLUG/app-store-report.md << 'EOF'
[report content]
EOF

# If standalone
mkdir -p .claude/pipeline/features/app-store-submission
# Write report + all generated assets list
```

## Phase 0A: User Interview

Ask these questions before starting. Wait for answers.

### Required Questions

**1. Target platform:**
"Which stores are you submitting to?"
- iOS App Store only
- Google Play only
- Both iOS and Google Play
- Web (PWA manifest + OG images)

→ Determines screenshot sizes, metadata formats, compliance requirements.

**2. App category:**
"What category should this app be listed under?"
Examples: Travel, Lifestyle, Food & Drink, Social Networking
→ Affects keyword strategy and competitor research.

**3. Current submission status:**
"Is this your first submission or an update?"
- First submission (need everything from scratch)
- Update (need What's New text, possibly new screenshots)
- Rejected resubmission (need to address reviewer feedback)

→ If rejected, ask for the rejection reason to address it specifically.

**4. Test account:**
"Do you have a test account with realistic data for App Review?"
→ Apple requires test credentials. If none exists, guide creation.

**5. Scope:**
- Full submission prep (everything: screenshots, copy, compliance, review notes)
- Screenshots only
- Store listing copy only
- Compliance check only
- Specific item (user names what they need)

**6. Monetization:**
"Does the app have in-app purchases, subscriptions, or ads?"
→ Affects compliance (auto-renewable subscription rules, ad tracking disclosure).

**7. Age rating:**
"Is there user-generated content, social features, or location sharing?"
→ Determines age rating questionnaire answers.

### Optional Questions

**8. Competitor apps:**
"Name 2-3 competitor apps in your category."
→ Used for keyword research and screenshot style benchmarking.

**9. Promotional text:**
"Any seasonal or time-sensitive messaging for the store listing?"
→ Promotional text can be updated without review.

**10. Localization:**
"Which languages should the store listing support?"
→ Generates translated metadata if needed.

### Store Preferences

Save to persistent memory:

```
memory/feedback_app_store_preferences.md
```

On subsequent runs: "I have your App Store preferences from last time — [summary]. Still accurate, or want to update?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after screenshot capture for user review
- STOP after copy generation for user approval
- STOP after compliance check for user review
- Only finalize what the user explicitly approves

### Autonomous Mode (invoked by subagent or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** If no stored preferences, use safe defaults:
  - Platform: iOS (most common)
  - Scope: full submission prep
  - Implementation: generate everything, apply nothing without review

- **All phases run fully — no stop.** Produces complete submission package.

- **NEVER auto-submit to App Store Connect** — always require human review of the final package.

## Phase 1: Platform Detection

```bash
# iOS indicators
HAS_IOS=false
ls ios/ 2>/dev/null && HAS_IOS=true
grep -q "expo-dev-client\|react-native" package.json 2>/dev/null && HAS_IOS=true
ls app.json 2>/dev/null && grep -q "ios" app.json && HAS_IOS=true

# Android indicators
HAS_ANDROID=false
ls android/ 2>/dev/null && HAS_ANDROID=true
ls app.json 2>/dev/null && grep -q "android" app.json && HAS_ANDROID=true

# Web / PWA indicators
HAS_WEB=false
ls web/ next.config.* 2>/dev/null && HAS_WEB=true
ls public/manifest.json 2>/dev/null && HAS_WEB=true

# Expo indicators
IS_EXPO=false
grep -q "expo" package.json 2>/dev/null && IS_EXPO=true

# Current app metadata
cat app.json 2>/dev/null | grep -E '"name"|"slug"|"version"|"bundleIdentifier"|"package"'
```

Report what was detected and confirm with user.

## Phase 2: Screenshot Generation

### 2.1 Check Prerequisites

```bash
# Is simulator booted?
xcrun simctl list devices booted 2>&1

# Is Metro running?
curl -s http://localhost:8081/status 2>/dev/null

# Is Maestro available?
which maestro 2>/dev/null || echo "MAESTRO_NOT_FOUND"
export PATH="$HOME/.maestro/bin:$PATH"
maestro --version

# Check for existing Maestro flows
ls maestro/flows/design/*.yaml 2>/dev/null
```

If prerequisites aren't met, guide the user:
- "Start the simulator: `npx expo run:ios`"
- "Start Metro: `npx expo start`"
- "Install Maestro: `curl -Ls https://get.maestro.mobile.dev | bash`"

### 2.2 Capture Screenshots via Maestro

Use existing capture flows if available:

```bash
# Run existing capture-all flow
maestro test maestro/flows/design/capture-all.yaml

# Or capture specific screens for App Store
maestro test maestro/flows/design/capture-app-store.yaml
```

If no App Store-specific flow exists, create one targeting the 5-10 best screens:

**Recommended App Store screens (in order):**
1. Home/discovery feed (hero shot — first impression)
2. Spot detail (shows content quality)
3. Trip planning / itinerary (core value prop)
4. Explore/search (shows breadth)
5. AI feature (Scout chat — differentiator)
6. Map view (if applicable)
7. Social/reviews (shows community)
8. Dark mode variant of hero shot (optional but impressive)

### 2.3 Format for Required Sizes

**iOS App Store required sizes:**

| Device | Resolution | Required? |
|--------|-----------|-----------|
| iPhone 6.7" (15 Pro Max) | 1290 x 2796 | Yes (required) |
| iPhone 6.5" (14 Plus) | 1284 x 2778 | Yes (if supporting older) |
| iPhone 6.1" (15 Pro) | 1179 x 2556 | Yes (required) |
| iPhone 5.5" (8 Plus) | 1242 x 2208 | Only if supporting |
| iPad Pro 12.9" (6th gen) | 2048 x 2732 | Only if iPad app |
| iPad Pro 11" | 1668 x 2388 | Only if iPad app |

**Google Play required sizes:**

| Asset | Resolution | Required? |
|-------|-----------|-----------|
| Phone screenshots | 1080 x 1920 min | Yes (2-8 screenshots) |
| 7" tablet | 1200 x 1920 min | If supporting tablets |
| 10" tablet | 1800 x 2560 min | If supporting tablets |
| Feature graphic | 1024 x 500 | Yes |
| TV banner | 1280 x 720 | Only if Android TV |

### 2.4 Screenshot Enhancement

For each screenshot, suggest:
- **Caption overlay**: Short text explaining the feature shown (e.g., "Discover spots by vibe")
- **Device frame**: Whether to wrap in a device mockup
- **Background color**: Suggest brand-appropriate background behind device frame
- **Ordering**: Which screenshot should be first (most compelling feature)

Note: Actual image editing requires external tools. Generate a spec document listing exactly what each screenshot should show and what caption to overlay.

## Phase 3: Store Listing Copy

### 3.1 Read App Context

```bash
# App metadata
cat app.json 2>/dev/null
cat package.json | grep -E '"name"|"description"|"version"'

# CLAUDE.md for brand context
cat CLAUDE.md 2>/dev/null

# Feature scan
find src/features/ -maxdepth 1 -type d 2>/dev/null
grep -rn "TODO\|FIXME\|coming soon" src/ --include="*.tsx" --include="*.ts" | head -20
```

### 3.2 Generate App Store Metadata (iOS)

```markdown
# iOS App Store Listing

## App Name (30 chars max)
[Name]

## Subtitle (30 chars max)
[Compelling one-liner — include top keyword]

## Promotional Text (170 chars, updatable without review)
[Current seasonal/promotional message]

## Description (4000 chars max)
[Structured as:]
- Hook paragraph (what the app does, why it matters)
- Key features (bullet points with emoji)
- Social proof or differentiator paragraph
- Call to action

## Keywords (100 chars max, comma-separated, no spaces after commas)
[keyword1,keyword2,keyword3,...]

## What's New (for updates)
[Version-specific release notes — user-friendly, not technical]

## Support URL
[from CLAUDE.md or user input]

## Privacy Policy URL
[from CLAUDE.md or user input]

## Category
Primary: [category]
Secondary: [category] (optional)

## Age Rating
[Based on content analysis]
```

### 3.3 Generate Google Play Metadata

```markdown
# Google Play Store Listing

## App Name (50 chars max — more generous than iOS)
[Name]

## Short Description (80 chars max)
[One-liner with primary keyword]

## Full Description (4000 chars max)
[Similar to iOS but can include more formatting]

## Feature Graphic (1024x500)
[Spec: what it should show, text overlay, background]

## Tags
[Up to 5 tags from Google's predefined list]
```

### 3.4 Keyword Research

For the app's category, research keywords:

```
PRIMARY KEYWORDS (high relevance, include in title/subtitle)
| Keyword | Estimated Volume | Competition | Recommendation |
|---------|-----------------|-------------|----------------|
| travel planner | High | High | Use in subtitle |
| trip planning | High | Medium | Use in keywords |

SECONDARY KEYWORDS (medium relevance, include in keywords field)
| Keyword | Estimated Volume | Competition | Recommendation |
|---------|-----------------|-------------|----------------|

LONG-TAIL KEYWORDS (low competition, niche)
| Keyword | Estimated Volume | Competition | Recommendation |
|---------|-----------------|-------------|----------------|
```

Research methods:
- Analyze competitor app subtitles and descriptions
- Check autocomplete suggestions in the App Store
- Cross-reference with web search trends for the category

### 3.5 ASO (App Store Optimization) Suggestions

```markdown
## ASO Recommendations

### Title/Subtitle
- Current: [current]
- Suggested: [improved version with keyword]
- Rationale: [why]

### Keywords
- Added: [new keywords and why]
- Removed: [dropped keywords and why]
- Keyword density in description: [check]

### Screenshots
- Order: [recommended order with rationale]
- First screenshot: [should show primary value prop]
- Caption suggestions: [for each screenshot]

### Competitor Comparison
| App | Subtitle | Top Keywords | Rating |
|-----|----------|-------------|--------|
| [Competitor 1] | ... | ... | ... |
| [Competitor 2] | ... | ... | ... |
| Your App | ... | ... | ... |

### Quick Wins
1. [actionable ASO improvement]
2. [actionable ASO improvement]
```

## Phase 4: Compliance & Review Prep

### 4.1 App Review Notes

Generate review notes that help Apple/Google reviewers test the app:

```markdown
## App Review Notes

### Test Account
Email: [test@example.com]
Password: [password]
Note: This account has pre-populated data including [trips, saved spots, reviews].

### Key Features to Review
1. [Feature]: [How to access it, what it does]
2. [Feature]: [How to access it, what it does]

### In-App Purchases (if applicable)
[List each IAP, what it unlocks, how to test]

### Location-Based Features
The app uses location for [purpose]. To test: [instructions]
If location is denied: [what happens — graceful degradation]

### Third-Party Services
- [Supabase]: Backend database and auth
- [PostHog]: Analytics (no personal data)
- [Sentry]: Error monitoring (no personal data)

### Content Moderation
[How user-generated content is moderated]

### Special Instructions
[Anything the reviewer needs to know — beta features hidden behind flags, etc.]
```

### 4.2 Privacy & Compliance

**App Privacy Details (iOS):**
Scan the codebase to determine actual data collection:

```bash
# Data collection detection
grep -rn "getUserLocation\|requestForegroundPermissions" src/ --include="*.ts" --include="*.tsx"
grep -rn "signIn\|signUp\|email\|password\|createUser" src/ --include="*.ts" --include="*.tsx"
grep -rn "PostHog\|posthog\|analytics\|track(" src/ --include="*.ts" --include="*.tsx"
grep -rn "Sentry\|captureException\|captureMessage" src/ --include="*.ts" --include="*.tsx"
grep -rn "camera\|photo\|image.*upload\|storage.*upload" src/ --include="*.ts" --include="*.tsx"
grep -rn "contacts\|address.*book\|getContacts" src/ --include="*.ts" --include="*.tsx"
grep -rn "push.*notification\|registerForPush\|expo-notifications" src/ --include="*.ts" --include="*.tsx"
grep -rn "IDFA\|AdServices\|ATTrackingManager\|requestTrackingPermission" src/ --include="*.ts" --include="*.tsx"
```

Generate the App Privacy questionnaire answers:

```markdown
## App Privacy Details

### Data Linked to You
| Data Type | Collected? | Purpose | Linked to Identity? |
|-----------|-----------|---------|-------------------|
| Email | Yes | Account creation | Yes |
| Name/Username | Yes | Profile | Yes |
| Photos | Yes | Reviews, profile | Yes |
| Location | Yes | Nearby spots | Yes |
| Usage Data | Yes | Analytics (PostHog) | No |
| Crash Data | Yes | Diagnostics (Sentry) | No |
| Search History | No | — | — |
| Contacts | No | — | — |
| Financial Info | No | — | — |

### Data NOT Collected
[List what you confirmed is not collected]

### Third-Party SDKs
| SDK | Data Shared | Purpose |
|-----|------------|---------|
| PostHog | Anonymous usage events | Product analytics |
| Sentry | Crash reports | Stability monitoring |
| Supabase | User data (encrypted) | Backend services |
```

### 4.3 Compliance Checklists

**GDPR (if serving EU users):**
```
[ ] Privacy policy accessible before signup
[ ] Consent for data collection (not just ToS checkbox)
[ ] Data export capability (user can download their data)
[ ] Data deletion capability (user can delete their account)
[ ] Cookie consent (web only)
[ ] Data Processing Agreement with Supabase
[ ] Under-16 consent mechanism (if applicable)
```

**COPPA (if users could be under 13):**
```
[ ] Age gate at signup
[ ] Parental consent mechanism
[ ] No behavioral advertising to children
[ ] No social features for under-13 users
[ ] Verifiable parental consent for data collection
```

**Encryption (CCATS/ERN for US export):**
```
[ ] App uses HTTPS (standard encryption) — likely exempt under EAR 740.17(b)(1)
[ ] No custom encryption algorithms
[ ] Uses standard libraries (TLS, AES via platform APIs)
[ ] If exempt: select "Yes" for encryption, then "exempt" in App Store Connect
[ ] If NOT exempt: file CCATS with BIS (rare for consumer apps)
```

**Age Rating Questionnaire:**
```
- Cartoon/Fantasy Violence: [None/Infrequent/Frequent]
- Realistic Violence: [None/Infrequent/Frequent]
- Sexual Content: [None/Infrequent/Frequent]
- Nudity: [None/Infrequent/Frequent]
- Profanity: [None/Infrequent/Frequent]
- Drugs/Alcohol/Tobacco: [None/Infrequent/Frequent]
- Horror/Fear: [None/Infrequent/Frequent]
- Gambling: [None/Simulated/Real]
- User-Generated Content: [Yes/No]
- Unrestricted Web Access: [Yes/No]
```

### 4.4 Privacy Policy Generation

Generate a privacy policy based on actual data collection (from 4.2 scan):

```markdown
# Privacy Policy for [App Name]

Last updated: [date]

## What We Collect
[Based on actual code scan — only list what's real]

## How We Use It
[Map each data type to its purpose]

## Third-Party Services
[List each SDK and what data it receives]

## Data Retention
[How long data is kept]

## Your Rights
[GDPR rights if applicable: access, deletion, portability, correction]

## Children's Privacy
[COPPA statement]

## Contact
[Support email from CLAUDE.md]
```

**Flag**: "This is a template. Have a lawyer review before publishing."

## Phase 5: TestFlight / Beta Distribution

### iOS (TestFlight)
```markdown
## TestFlight Checklist
[ ] EAS build: `eas build --platform ios --profile preview`
[ ] Upload to App Store Connect (automatic with EAS)
[ ] Add beta testers (internal: up to 100, external: up to 10,000)
[ ] Write beta test instructions
[ ] Set beta app description
[ ] Enable automatic distribution (or manual per build)
```

### Android (Internal Testing)
```markdown
## Google Play Internal Testing Checklist
[ ] EAS build: `eas build --platform android --profile preview`
[ ] Upload to Google Play Console → Internal testing track
[ ] Add testers via email list or Google Group
[ ] Set up pre-launch report (automated testing by Google)
```

## Phase 6: Deliver

### Output Report Format

```markdown
# App Store Submission Report — [date]

## Platform
[iOS / Android / Both]

## Submission Status
[First submission / Update / Resubmission]

## Screenshots
- Captured: [count] screens
- Formatted for: [device sizes]
- Location: [path to screenshots]
- Enhancement spec: [path to spec file]

## Store Listing
- App name: [name]
- Subtitle: [subtitle]
- Keywords: [count] keywords, [estimated coverage]
- Description: [word count]
- Location: [path to generated copy]

## Compliance
- Privacy questionnaire: [complete / needs review]
- GDPR: [compliant / [N] items remaining]
- COPPA: [N/A / compliant / needs work]
- Encryption: [exempt / needs filing]
- Age rating: [suggested rating]

## Review Notes
- Test account: [ready / needs creation]
- Location: [path to review notes]

## ASO Score
[Estimated optimization level: Low / Medium / High]
- Top keyword gaps: [list]
- Suggested improvements: [list]

## Blockers
[Anything that must be resolved before submission]

## Recommended Next Steps
1. [prioritized action items]
```

### File Outputs

All generated files are saved to:
```
.claude/pipeline/features/[slug]/
├── app-store-report.md         # This report
├── ios-listing.md              # iOS App Store copy
├── google-play-listing.md      # Google Play copy (if applicable)
├── keywords.md                 # Keyword research + recommendations
├── aso-analysis.md             # ASO competitive analysis
├── privacy-policy.md           # Generated privacy policy template
├── review-notes.md             # App Review notes with test account
├── compliance-checklist.md     # GDPR, COPPA, encryption status
├── screenshot-spec.md          # What each screenshot should show
└── screenshots/                # Raw captured screenshots
```

## Anti-Patterns

- DON'T submit without a privacy policy URL — instant rejection
- DON'T use placeholder screenshots — Apple rejects low-quality images
- DON'T keyword-stuff the description — Apple penalizes this
- DON'T include prices in screenshots — they vary by region
- DON'T mention competing platforms ("better than X") — violates guidelines
- DON'T include beta/test/demo language in store listing
- DON'T skip the encryption questionnaire — it blocks the review
- DON'T auto-submit to App Store Connect — always require human review
- DON'T forget to update What's New for version updates
- DON'T use the same screenshots for iPhone and iPad if layouts differ
