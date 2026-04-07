---
name: design-audit
description: Visual design audit pipeline — sets up Maestro screenshot capture, analyzes every screen for UI/UX issues, proposes fixes with design options, generates Figma mockups, and implements approved changes. Use when the user asks to audit their app, review UI quality, capture screenshots, or improve design consistency.
---

# Design Audit Skill

End-to-end visual audit pipeline: Setup → Capture → Analyze → Propose → Implement.

## When to Use

- User says "audit my app", "review the UI", "check the design", "take screenshots"
- User wants to find visual inconsistencies, accessibility issues, or dark mode bugs
- After a feature pass to catch regressions
- Before a release to verify visual quality

## Phase 0A: User Preferences Interview

Before doing anything, ask the user these questions. Wait for answers. Store their responses and use them throughout the audit.

### Required Questions (ask all)

**1. Design reference:**
"What apps or websites do you consider best-in-class for design? (e.g., Airbnb, Arc Browser, Linear, Apple Maps)"
→ This establishes the visual benchmark for the audit.

**2. Design priorities — rank these 1-5:**
"Which matters most to you right now?"
- Consistency (everything follows the same rules)
- Accessibility (readable, usable by everyone)
- Premium feel (Airbnb/Apple-level polish)
- Information density (show more content per screen)
- Simplicity (reduce clutter, fewer elements)

→ This determines which issues get flagged as CRITICAL vs MINOR.

**3. Known pain points:**
"Are there specific screens or flows that bother you?"
→ Focus the audit on what the user already knows is wrong.

**4. Off-limits:**
"Anything I should NOT change? (e.g., specific layouts, colors, features)"
→ Prevents wasting time proposing changes the user will reject.

**5. Scope:**
"Full app audit, or specific screens only?"
- Full app (every screen, light + dark mode)
- Specific screens (user lists which ones)
- Changes only (just audit what changed since last session)

**6. Implementation preference:**
"After the audit, do you want me to:"
- Just report issues (no code changes)
- Report + propose options (you pick which to fix)
- Report + propose + implement approved fixes
- Full autopilot (fix everything I find, ask only on design decisions)

### Optional Questions (ask if relevant)

**7. Dark mode priority:**
"How important is dark mode polish? Ship-blocking or nice-to-have?"

**8. Figma integration:**
"Do you have a Figma file for this project? If so, should I generate mockups for proposed changes?"

**9. Target devices:**
"Which screen sizes matter most? (e.g., iPhone 14 Pro 393pt, iPhone SE 375pt, iPad)"

### How to Use Preferences

- **Design reference** → Compare screenshots against the benchmark apps' patterns
- **Priority ranking** → Weight severity based on what the user cares about. If they rank "premium feel" #1, flag visual roughness as MAJOR. If they rank "accessibility" #1, flag contrast issues as CRITICAL.
- **Known pain points** → Start the audit with these screens, give them extra attention
- **Off-limits** → Skip these in proposals, don't flag them as issues
- **Scope** → Determines how many screenshots to capture/analyze
- **Implementation preference** → Determines whether to stop at reporting or continue to code

### Store Preferences

Save the user's responses to memory so future audit runs don't re-ask:

```
memory/feedback_design_audit_preferences.md
```

On subsequent runs, ask: "I have your design preferences from last time — [summary]. Still accurate, or want to update anything?"

## Execution Modes

The skill adjusts its behavior based on how it's invoked:

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after analysis for user review
- STOP after proposals for user approval
- Only implement what the user explicitly approves

### Autonomous Mode (--dangerously-skip-permissions or user says "full autopilot")
When running autonomously (e.g., invoked by a subagent, cron trigger, or with skip-permissions):

- **Phase 0A: Skip interview IF preferences exist in memory.** If no stored preferences, fall back to safe defaults:
  - Priority: accessibility > consistency > premium feel
  - Scope: full app
  - Implementation: report + propose only (NO auto-implement)
  - Off-limits: none
  
- **Phase 2: Analyze runs fully — no stop.** Produces the audit report.

- **Phase 3: Propose runs fully — no stop.** Generates all options.

- **Phase 4: Implement — ONLY auto-implement if ALL of these are true:**
  1. User's stored preference is "full autopilot"
  2. The fix is severity CRITICAL or MAJOR
  3. The fix confidence is 95%+
  4. The fix is conservative (not a redesign)
  5. The fix passes typecheck and tests after applying
  
  Otherwise: generate a report file at `maestro/screenshots/audit-report.md` with all findings and proposals, and notify the user to review.

### Why This Matters
- A cron-triggered audit at 3am shouldn't redesign the profile page
- A subagent in a parallel workflow should capture + analyze but not rewrite components
- The interview preferences are a guardrail, not a blocker — if they exist, trust them
- Auto-implementation is only safe for clear bugs (wrong color token, missing padding, broken contrast) — never for subjective design choices

### Report File Format (for autonomous runs)
When not implementing, write findings to:

```markdown
# Design Audit Report — [date]

## User Preferences
[from memory or defaults]

## Screenshots Captured
[count] light, [count] dark

## Findings

### CRITICAL
| # | Issue | Screen | File | Confidence | Auto-fixable? |
...

### MAJOR
...

### MINOR
...

## Proposed Fixes
[for each finding, the conservative option with exact file + line + change]

## Recommended Next Steps
[prioritized list]
```

## Platform Detection

Auto-detect what exists, what's runnable, and what the user wants to audit.

### Step 1: Detect What Exists in the Codebase

```bash
# Mobile indicators
HAS_MOBILE=false
(ls app.json app.config.js 2>/dev/null || grep -q "react-native\|expo" package.json 2>/dev/null) && HAS_MOBILE=true
# Flutter
ls pubspec.yaml 2>/dev/null && HAS_MOBILE=true

# Web indicators
HAS_WEB=false
ls next.config.* nuxt.config.* vite.config.* angular.json 2>/dev/null && HAS_WEB=true
grep -q '"react-dom"\|"next"\|"nuxt"\|"vue"\|"svelte"\|"@angular"' package.json 2>/dev/null && HAS_WEB=true
# Check for web directory in monorepo
ls web/package.json web/next.config.* 2>/dev/null && HAS_WEB=true
# Static HTML
(ls index.html 2>/dev/null || ls public/index.html 2>/dev/null) && HAS_WEB=true
```

### Step 2: Check What's Actually Runnable Right Now

Having the code doesn't mean it's running. Check runtime status:

**Mobile:**
```bash
# Is a simulator booted?
xcrun simctl list devices booted 2>&1 | grep -q "Booted" && echo "SIMULATOR_READY" || echo "NO_SIMULATOR"
# Is Metro/dev server running?
curl -s http://localhost:8081/status 2>/dev/null && echo "METRO_RUNNING" || echo "NO_METRO"
# Is the app installed on the simulator?
xcrun simctl listapps booted 2>/dev/null | grep -q "com.chumbing" && echo "APP_INSTALLED" || echo "NO_APP"
```

**Web:**
```bash
# Is a dev server running?
curl -s http://localhost:3000 2>/dev/null && echo "WEB_SERVER_RUNNING" || echo "NO_WEB_SERVER"
curl -s http://localhost:5173 2>/dev/null && echo "VITE_RUNNING" || echo "NO_VITE"
curl -s http://localhost:4200 2>/dev/null && echo "ANGULAR_RUNNING" || echo "NO_ANGULAR"
# Is it a production build that exists but isn't served?
ls .next/ out/ dist/ build/ 2>/dev/null && echo "BUILD_EXISTS" || echo "NO_BUILD"
```

### Step 3: Present Status and Ask User

Based on detection, present one of these:

**Scenario A: Only mobile exists**
> "I detected a mobile app (React Native/Expo). No web project found.
> Ready to audit: mobile
> Shall I proceed with mobile audit?"

**Scenario B: Only web exists**
> "I detected a web app (Next.js/React/etc). No mobile project found.
> Ready to audit: web
> Shall I proceed with web audit?"

**Scenario C: Both exist, both running**
> "I detected both mobile and web:
> - Mobile: ✅ Simulator booted, Metro running, app installed
> - Web: ✅ Dev server running on localhost:3000
> 
> Audit options:
> 1. Both (comprehensive — takes longer)
> 2. Mobile only
> 3. Web only
> 
> Which would you like?"

**Scenario D: Both exist, only one running**
> "I detected both mobile and web:
> - Mobile: ✅ Simulator booted, Metro running
> - Web: ❌ Dev server not running
> 
> I can audit mobile now. To include web, start the dev server:
> `cd web && npm run dev`
> 
> Proceed with mobile only, or wait for web?"

**Scenario E: Code exists but nothing running**
> "I detected a [mobile/web] project but nothing is running:
> - Mobile: ❌ No simulator booted
> - Web: ❌ No dev server
> 
> To audit mobile: 
> 1. Boot simulator: `xcrun simctl boot "iPhone 16 Pro"`
> 2. Start Metro: `npx expo start --clear`
> 
> To audit web:
> 1. Start dev server: `npm run dev`
> 
> Let me know when ready, or I can attempt to start these for you."

**Scenario F: Nothing detected**
> "I couldn't detect a mobile or web project in this directory.
> 
> What am I looking at?
> - Is this a monorepo? Which subdirectory has the app?
> - Is it a different framework I should look for?
> - Or should I audit static HTML/CSS files?"

### Step 4: Handle "Audit Web" When Web Doesn't Exist

If the user specifically asks to audit web but no web project exists:

> "There's no web project in this codebase yet. I found:
> - `app/` — React Native screens (mobile)
> - `src/` — shared components and features
> - No `web/`, `pages/`, or web framework config
> 
> Options:
> 1. **Audit mobile instead** — I can audit what exists now
> 2. **Scaffold web** — I can help set up a Next.js web app that shares your design system
> 3. **Skip** — audit later when web is built
> 
> What would you prefer?"

### Platform Modes (once confirmed)

**Mobile (React Native / Expo / Flutter)**
- Capture tool: **Maestro**
- Screenshots from: iOS Simulator or Android Emulator
- Screen sizes: iPhone SE (375pt), iPhone Pro (393pt), iPad optional
- Special concerns: safe areas, tab bars, keyboard avoidance, haptics

**Web (Next.js / React / Vue / Svelte / HTML)**
- Capture tool: **Playwright** (preferred) or Puppeteer
- Screenshots from: headless Chromium
- Screen sizes: mobile (375px), tablet (768px), desktop (1440px)
- Special concerns: responsive breakpoints, hover states, scroll behavior, SEO layout shifts

**Both (monorepo with mobile + web)**
- Run both capture pipelines
- Cross-platform consistency check: do shared screens (e.g., spot detail) look consistent between mobile and web?
- Flag divergences that are intentional (adaptive layout) vs bugs

## Phase 0B: Project Design Infrastructure

Before capturing screenshots, verify the project has the design infrastructure needed for a meaningful audit. If anything is missing, create it.

### Check and Create Design System Files

**1. Theme/tokens file**
Search for an existing design system file:
```bash
# Common locations
find src/ -name "theme.*" -o -name "tokens.*" -o -name "design-system.*" -o -name "colors.*" | head -10
# Also check for Tailwind config, styled-components theme, CSS variables
find . -name "tailwind.config.*" -o -name "stitches.config.*" -o -name "globals.css" | head -5
```

If **no theme file exists**, create one based on the project's framework:

- **React Native**: Create `src/constants/theme.ts` with:
  - Color palette (light + dark mode semantic tokens)
  - Spacing scale (4pt grid)
  - Border radius tokens
  - Typography ramp (font families, sizes, line heights, weights)
  - Shadow definitions
  - Export a `useTheme()` hook pattern

- **React/Next.js**: Create CSS variables in `globals.css` or a theme provider with:
  - CSS custom properties for colors, spacing, radius
  - Dark mode via `@media (prefers-color-scheme: dark)` or class toggle
  - Typography scale

- **Any framework**: The theme file should be the single source of truth for all visual tokens. Components should never use hardcoded values.

**Ask the user:** "I didn't find a theme file. Should I create one based on your current codebase? I'll extract colors, fonts, and spacing from your existing components."

If user approves, scan existing components for hardcoded values and extract them into a centralized theme.

**2. Design system documentation (CLAUDE.md or similar)**
Check if design rules are documented:
```bash
cat CLAUDE.md | grep -i "design\|color\|spacing\|button\|font" | head -20
```

If no design rules exist in CLAUDE.md, generate a design system section by:
- Reading the theme file
- Scanning component patterns (button variants, card styles, input styles)
- Documenting the current conventions as rules

**3. Component library inventory**
Check for shared UI components:
```bash
ls src/components/ui/ 2>/dev/null || ls src/components/ 2>/dev/null
```

If the project has no shared component directory, note this in the audit as a systemic issue: "No shared component library — styles are likely inconsistent across screens."

**4. Dark mode support**
Check if dark mode exists:
```bash
grep -r "dark\|darkMode\|colorScheme\|useColorScheme" src/ --include="*.ts" --include="*.tsx" | head -5
```

If no dark mode support, note it but don't create it — that's a feature, not an audit fix. Flag it as a recommendation.

**5. Framework-specific theme detection**
Detect the project's styling approach and adapt the audit accordingly:

```bash
# Tailwind CSS
ls tailwind.config.* 2>/dev/null && echo "TAILWIND"
# CSS Modules
find src/ -name "*.module.css" -o -name "*.module.scss" 2>/dev/null | head -5
# Styled Components / Emotion
grep -rn "styled\.\|styled(" src/ --include="*.tsx" --include="*.ts" 2>/dev/null | head -5
# CSS Variables (custom properties)
grep -rn ":root\|--[a-z]" src/ --include="*.css" 2>/dev/null | head -10
# SwiftUI (iOS native)
find . -name "*.swift" 2>/dev/null | head -3 && grep -rn "Color(" . --include="*.swift" 2>/dev/null | head -5
# Flutter
ls pubspec.yaml 2>/dev/null && grep -rn "ThemeData\|ColorScheme" lib/ --include="*.dart" 2>/dev/null | head -5
```

Adapt the audit based on what's found:
- **Tailwind**: Check `tailwind.config.js` for custom theme values vs defaults. Flag screens using default Tailwind colors (blue-500, gray-200) instead of custom theme tokens. Check for arbitrary values (`text-[#ff0000]`) that bypass the config.
- **CSS Modules**: Scan `.module.css` files for hardcoded color/spacing values instead of CSS variable references.
- **Styled Components**: Check if `styled.` usages reference a theme provider (`${props => props.theme.X}`) vs hardcoded values.
- **CSS Variables**: Scan `:root` block in `globals.css` for the variable definitions. Check components reference `var(--token)` vs hardcoded values.
- **SwiftUI**: Scan for `Color(` with hex literals vs named colors from an asset catalog or extension.
- **Flutter**: Scan for `ThemeData` usage and check if widgets reference `Theme.of(context)` vs hardcoded `Color(0xFF...)`.

If none of these are detected, fall back to the React Native `theme.ts` approach.

**6. Maestro testIDs**
Check if components have testIDs for automation:
```bash
grep -r "testID" src/ app/ --include="*.tsx" | wc -l
```

If fewer than 20 testIDs exist, the capture flows will be limited. Note which screens can't be automated and flag them for manual capture.

### Summary
After this phase, you should have:
- A theme file (existing or newly created) as the source of truth
- Design rules documented (existing or newly generated)
- Knowledge of the component library structure
- Dark mode status
- testID coverage level

## Phase 0C: Capture Tool Setup

Set up the appropriate screenshot capture tool based on platform.

### Mobile: Maestro Setup

Before capturing, verify Maestro and the simulator are ready.

### Maestro Installation

```bash
# Check if Maestro is installed
which maestro || echo "not installed"

# If not installed:
curl -Ls "https://get.maestro.mobile.dev" | bash

# Maestro requires Java — check:
java -version 2>&1 || echo "Java not installed"

# If Java missing, install Azul Zulu JDK 17:
brew install --cask zulu@17
```

### Simulator Check

```bash
# iOS simulator must be booted
xcrun simctl list devices booted

# If no device booted, boot one:
xcrun simctl boot "iPhone 16 Pro"
```

### Metro / Dev Server Check

The app must be running on the simulator. Check if Metro is serving:
- Ask the user: "Is Metro running? If not, run `npx expo start --clear`"
- The auto-login flow requires `EXPO_PUBLIC_MAESTRO_AUTO_LOGIN=true` in `.env`

### Auto-Login Flow

Check if an auto-login flow exists at the project's Maestro flows directory. If not, create one:

```yaml
# maestro/flows/shared/auto-login.yaml
appId: ${APP_ID}
---
# Dismiss any dev menu popups
- tapOn:
    text: "Continue"
    optional: true
- tapOn:
    text: "Go home"
    optional: true
# Wait for main navigation to appear
- extendedWaitUntil:
    visible:
      id: "tab-explore"  # Adjust to project's main tab testID
    timeout: 15000
```

### Capture Flow Generation

If comprehensive capture flows don't exist, **generate them from the codebase:**

1. **Scan the route structure:**
   ```bash
   # For Expo Router / file-based routing:
   find app/ -name "*.tsx" -not -name "_*" | sort
   ```

2. **Identify all screens** from the route files:
   - Tab screens (bottom navigation)
   - Push screens (detail pages)
   - Modal screens
   - Auth screens

3. **Identify all interactions** by searching for:
   - `testID` props — these are automatable touch targets
   - Sheet/modal state toggles (`setShowX(true)`)
   - Tab switches, filter toggles, search inputs

4. **Generate capture flows** organized by area:
   - `full-capture-1-tabs-details.yaml` — all tabs + main detail screens (spot, trip, event)
   - `full-capture-2-browse-screens.yaml` — trending, cities, saved, lists, submissions, help
   - `full-capture-3-modals-wizard.yaml` — all modals, wizard steps, interactions
   - `full-capture-4-dark-mode.yaml` — switch to dark, capture all key screens, restore light

   Each flow should:
   - Start with `launchApp` + `runFlow: auto-login.yaml`
   - Use `testID` selectors over text selectors where possible
   - Take screenshots with descriptive numbered names: `001-home-top.png`
   - Scroll through long screens (multiple swipe + screenshot pairs)
   - Use `optional: true` on elements that may not exist for all data states
   - Use deep links (`openLink: appscheme://path`) when scroll-to-find is unreliable

5. **For dark mode flow:**
   - Navigate to settings/profile → toggle dark mode
   - Capture the same key screens as light mode
   - Restore light mode at the end

6. **Handle screens that need specific data:**
   - AI-generated content (itinerary, Scout response) — use `extendedWaitUntil` with long timeouts
   - Auth-gated screens — the auto-login flow handles this
   - Screens needing specific navigation — use deep links

### Output of Phase 0
- Maestro installed and working
- Simulator booted with app running
- Auto-login flow verified
- Capture flows generated for every screen and interaction
- Test run: execute one flow to verify screenshots save correctly

### Web: Playwright Setup

If the project is web-based, use Playwright for screenshot capture.

```bash
# Check if Playwright is installed
npx playwright --version 2>/dev/null || echo "not installed"

# If not installed:
npm install -D @playwright/test
npx playwright install chromium
```

#### Generate Web Capture Script

Create a Playwright script that captures every route:

1. **Scan routes:**
   ```bash
   # Next.js App Router
   find app/ -name "page.tsx" -o -name "page.js" | sort
   # Next.js Pages Router
   find pages/ -name "*.tsx" -o -name "*.js" | grep -v "_app\|_document\|api/" | sort
   # React Router — grep for route definitions
   grep -rn "path=\|<Route" src/ --include="*.tsx" | head -30
   # Static HTML
   find . -name "*.html" -not -path "*/node_modules/*" | sort
   ```

2. **Generate capture script** at `tests/visual-capture.ts`:
   ```typescript
   import { test } from '@playwright/test';

   const routes = [
     { path: '/', name: '001-home' },
     { path: '/explore', name: '002-explore' },
     // ... generated from route scan
   ];

   const viewports = [
     { name: 'mobile', width: 375, height: 812 },
     { name: 'tablet', width: 768, height: 1024 },
     { name: 'desktop', width: 1440, height: 900 },
   ];

   for (const viewport of viewports) {
     test.describe(viewport.name, () => {
       test.use({ viewport: { width: viewport.width, height: viewport.height } });
       
       for (const route of routes) {
         test(`capture ${route.name}`, async ({ page }) => {
           await page.goto(route.path);
           await page.waitForLoadState('networkidle');
           await page.screenshot({
             path: `screenshots/${viewport.name}/${route.name}.png`,
             fullPage: true,
           });
         });
       }
     });
   }

   // Dark mode captures
   test.describe('dark-mode', () => {
     test.use({ colorScheme: 'dark' });
     // ... repeat key routes
   });
   ```

3. **Auth handling for protected routes:**
   ```typescript
   // Save auth state
   test('login', async ({ page }) => {
     await page.goto('/login');
     await page.fill('[name="email"]', process.env.TEST_EMAIL);
     await page.fill('[name="password"]', process.env.TEST_PASSWORD);
     await page.click('button[type="submit"]');
     await page.context().storageState({ path: 'auth-state.json' });
   });

   // Reuse in all subsequent tests
   test.use({ storageState: 'auth-state.json' });
   ```

4. **Interaction captures:**
   ```typescript
   // Hover states (web-specific)
   await page.hover('.card');
   await page.screenshot({ path: 'screenshots/hover-card.png' });

   // Modal/dialog open
   await page.click('[data-testid="open-modal"]');
   await page.screenshot({ path: 'screenshots/modal-open.png' });

   // Dropdown expanded
   await page.click('[data-testid="filter-dropdown"]');
   await page.screenshot({ path: 'screenshots/dropdown-expanded.png' });
   ```

#### Web-Specific Checks
In addition to the standard visual checklist, web audits should check:
- **Responsive breakpoints** — does the layout reflow correctly at 375/768/1440?
- **Hover states** — do interactive elements have hover feedback?
- **Focus states** — can you tab through the page? Are focus rings visible?
- **Scroll behavior** — sticky headers, parallax, scroll-to-top
- **Loading performance** — does content shift after load (CLS)?
- **SEO layout** — is the above-the-fold content meaningful without JS?
- **Print styles** — does the page look reasonable when printed? (if applicable)
- **Browser compatibility** — capture in Chrome + Firefox + Safari if critical

## Phase 1: Capture

Run all capture flows and collect screenshots:

```bash
export PATH="$PATH":"$HOME/.maestro/bin"

# Run each flow, move screenshots to organized directory
mkdir -p maestro/screenshots/full-app

maestro test maestro/flows/design/full-capture-1-tabs-details.yaml
mv *.png maestro/screenshots/full-app/

maestro test maestro/flows/design/full-capture-2-browse-screens.yaml
mv *.png maestro/screenshots/full-app/

# ... repeat for all flows
```

### Handling Failures
- If a flow fails mid-run, move captured screenshots and continue with the next flow
- Log which screens weren't captured — note as "not captured" in the audit
- Common failure causes:
  - Element not found: the testID may have changed or the element isn't visible
  - Scroll not reaching content: use `scrollUntilVisible` instead of blind swipes
  - App not loading: Metro needs restart (`npx expo start --clear`)
  - Keyboard blocking elements: use swipe gestures to dismiss

### Screenshot Naming Convention

Enforce a consistent naming convention across all capture runs so before/after comparisons work across sessions:

**Format:** `[NNN]-[screen]-[state].png`

**Rules:**
- `NNN` = zero-padded 3-digit sequence number (e.g., `001`, `042`, `107`)
- `screen` = kebab-case screen name matching the route (e.g., `home`, `spot-detail`, `trip-wizard-step3`)
- `state` = optional suffix describing the UI state being captured

**State suffixes (standardized):**
- `-top` — screen at scroll position 0
- `-scroll` — screen after scrolling down
- `-bottom` — screen scrolled to the end
- `-empty` — empty state (no data)
- `-error` — error state
- `-loading` — loading/skeleton state
- `-hover` — hover state (web only)
- `-focus` — input focus state
- `-expanded` — sheet/modal/accordion expanded
- `-dark` — dark mode variant (prefix with same number as light equivalent)

**Examples:**
```
001-home-top.png
002-home-scroll.png
003-spot-detail-top.png
003-spot-detail-scroll.png
004-trip-wizard-step1.png
078-dark-home-top.png
079-dark-spot-detail-top.png
```

**Why this matters:**
- Consistent naming lets Phase 5 (before/after) diff the same screen across runs by matching filenames
- The sequence numbers create a stable ordering so new screens slot in without renumbering
- State suffixes make it clear what each screenshot represents without opening it

When generating capture flows (Phase 0C), use these naming conventions in the `takeScreenshot` commands. On subsequent runs, reuse the same numbering scheme so filenames stay comparable.

### Final Screenshot Count
Report: "X screenshots captured — Y light mode, Z dark mode. N screens not captured: [list]"

## Phase 2: Analyze

### Read Design System First
Before looking at any screenshot, read these files to understand the rules:
- Design tokens: `src/constants/theme.ts`
- Color system: `src/constants/vibeColors.ts`  
- Design standards: `CLAUDE.md` (look for Design System section)
- Component library: `src/components/ui/` (understand what components exist)

### Read Every Screenshot
Use the Read tool on every `.png` file in the screenshots directory. Don't skip any.

### Design Token Coverage Metric

Before evaluating screenshots, calculate the design system adoption rate as a baseline metric:

```bash
# Count total style-related declarations in components
TOTAL_STYLES=$(grep -rn "color:\|backgroundColor:\|fontSize:\|fontFamily:\|margin:\|padding:\|borderRadius:\|borderColor:" src/ app/ --include="*.tsx" --include="*.ts" | grep -v "node_modules\|theme\.\|constants/" | wc -l)

# Count hardcoded values (not using theme tokens)
HARDCODED_COLORS=$(grep -rn "'#[0-9A-Fa-f]\{3,8\}'" src/ app/ --include="*.tsx" | grep -v "theme\|constants\|vibeColors" | wc -l)
HARDCODED_SPACING=$(grep -rn "margin.*: [0-9]\|padding.*: [0-9]" src/ app/ --include="*.tsx" | grep -v "StyleSheet\|makeStyles\|theme\.\|spacing" | wc -l)
HARDCODED_FONTS=$(grep -rn "fontFamily.*'" src/ app/ --include="*.tsx" | grep -v "theme\|constants\|typography" | wc -l)
HARDCODED_TOTAL=$((HARDCODED_COLORS + HARDCODED_SPACING + HARDCODED_FONTS))

# Calculate adoption rate
ADOPTION=$(( (TOTAL_STYLES - HARDCODED_TOTAL) * 100 / TOTAL_STYLES ))
echo "Design system adoption: ${ADOPTION}% — ${HARDCODED_TOTAL} hardcoded values out of ${TOTAL_STYLES} total"
```

Include this metric at the top of every audit report. Track it across runs (Phase 6) to measure whether the codebase is converging on or drifting from the design system.

**Thresholds:**
- 90%+ = excellent — flag individual hardcoded values for cleanup
- 70-89% = good — flag as a systemic improvement opportunity
- Below 70% = poor — recommend a dedicated centralization pass before auditing individual screens

### Evaluate Each Screen Against This Checklist

**Consistency:**
- [ ] Horizontal screen padding matches `screenPaddingH` token
- [ ] Buttons use the correct radius token (not pill for action buttons, pill only for tags/chips)
- [ ] Button heights are consistent (48px for md/lg)
- [ ] Typography hierarchy: serif for editorial headings, sans-serif for UI
- [ ] Back buttons and close buttons use the shared NavBackButton/ModalCloseButton components
- [ ] Section headers follow the project's label pattern
- [ ] Cards have consistent border/radius/padding treatment
- [ ] Status badges use the correct color coding per the design system

**Accessibility:**
- [ ] Text contrast meets WCAG AA (4.5:1 body, 3:1 large text)
- [ ] Touch targets minimum 44x44pt
- [ ] Interactive elements have clear visual affordance (not just color)
- [ ] Error messages use error color tokens, not just red text
- [ ] Focus states visible on inputs

**Dark Mode (check every dark screenshot):**
- [ ] No light-mode hardcoded colors leaking (watch for bright whites, creams, ambers)
- [ ] Borders visible — borderLight should have enough opacity
- [ ] Cards distinguishable from background
- [ ] Badge/chip tints at sufficient opacity
- [ ] Text hierarchy preserved (primary/secondary/tertiary all readable)
- [ ] Placeholder images/icons visible against dark background

**Cross-Theme Consistency (compare light vs dark):**
After checking light and dark mode separately, compare them side by side:
- [ ] Every element visible in light mode has a corresponding dark mode version — nothing disappears
- [ ] Interactive elements (buttons, links, chips) have both light and dark styles defined — not falling back to invisible defaults
- [ ] Icons and illustrations that use hardcoded fill colors remain visible in both themes
- [ ] Shadows that provide separation in light mode have a dark equivalent (border, background shift, or elevated surface)
- [ ] Charts, graphs, or data visualizations are readable in both themes
- [ ] Placeholder/empty-state illustrations are adapted for dark backgrounds (not white-background PNGs on dark surfaces)

To check systematically: match light and dark screenshots by screen name (using the naming convention) and compare each pair. Flag any element that appears in one theme but not the other.

**Layout:**
- [ ] No content clipped by floating elements (FABs, tab bars, keyboards)
- [ ] No overlapping UI elements (toasts stacking on banners on CTAs)
- [ ] Safe area respected (content not behind notch or home indicator)
- [ ] Scroll content has bottom padding for last items
- [ ] Empty states are actionable, not just informational
- [ ] Modals don't show content bleeding through from behind

**Data States:**
- [ ] Image placeholders look intentional, not broken
- [ ] Loading states use skeletons, not spinners
- [ ] Error states are human-readable
- [ ] Empty states inspire action ("Be the first to...") not state facts ("No data")
- [ ] Validation errors only show after user attempts submission

**Performance & Assets:**
- [ ] Images are appropriately sized for the device — not loading 4K originals on a phone. Check for resize parameters in image URLs (Supabase storage `?width=`, Cloudinary transforms, `srcSet` on web).
  ```bash
  # Check for image optimization patterns
  grep -rn "width=\|height=\|resize\|transform\|srcSet\|sizes=" src/ app/ --include="*.tsx" | head -20
  # Check for image components with loading states
  grep -rn "placeholder\|blurhash\|progressive\|onLoadStart\|onLoad=" src/ app/ --include="*.tsx" | head -10
  # Check for image caching
  grep -rn "cache\|FastImage\|Image.prefetch\|next/image" src/ app/ --include="*.tsx" --include="*.ts" | head -10
  ```
- [ ] Images have loading states (blur-up, placeholder, skeleton) — not a blank space that pops in
- [ ] Image caching is implemented (react-native-fast-image, next/image, or equivalent)
- [ ] Placeholder/fallback images display correctly when the source fails to load

**Typography Consistency:**
Scan for all font sizes and families used across the app and compare against the defined typography ramp:
```bash
# Find all fontSize values in components
grep -rn "fontSize.*:" src/ app/ --include="*.tsx" | grep -v "node_modules\|theme\.\|constants/" | sed 's/.*fontSize[:\s]*//' | sort | uniq -c | sort -rn
# Find all fontFamily values in components
grep -rn "fontFamily.*:" src/ app/ --include="*.tsx" | grep -v "node_modules\|theme\.\|constants/" | head -20
# Find all Text style presets from theme
grep -rn "typography\.\|textStyle\.\|font\." src/constants/theme.ts | head -20
```
- [ ] Every font size used in the app maps to a defined text style in the theme (e.g., `typography.body`, `typography.h2`)
- [ ] No raw `fontSize: 13` or `fontSize: 17` that doesn't match a ramp value — flag each orphan size
- [ ] No direct `fontFamily: 'Plus Jakarta Sans'` that bypasses the theme's typography tokens — should use `typography.fontFamily.sans` or equivalent
- [ ] Display font (Fraunces/serif) only used for editorial moments (h1, h2, hero text), not body/UI text
- [ ] Line heights are proportional and consistent (not mixing `lineHeight: 1.2` and `lineHeight: 24` arbitrarily)

**Visual Quality:**
- [ ] No duplicate actions (same button appearing twice on one screen)
- [ ] Visual hierarchy is clear — primary action stands out
- [ ] Information density is appropriate (not too cramped, not too sparse)
- [ ] Animations and transitions feel intentional
- [ ] Color usage is purposeful (teal for interactive, not for prices)

### Output Format

Produce a structured table grouped by severity:

```
CRITICAL — Broken functionality or accessibility violations
MAJOR — Visible to users, degrades trust or usability  
MINOR — Polish items, noticeable but not harmful
NICE-TO-HAVE — Subjective improvements
```

Each item needs:
- Issue description (what's wrong)
- Screenshot reference (which file)
- Affected file/component (where to fix)
- Confidence level (how sure you are this is actually an issue, not stale data or Maestro artifact)
- Is it systemic (appears on multiple screens) or isolated?

### Critical Rules for Analysis
- **Be honest** — don't say something looks good if it doesn't
- **Distinguish code bugs from data gaps** — missing image = data, broken placeholder = code
- **Distinguish app bugs from Maestro bugs** — identical scroll frames = Maestro issue
- **Check if "broken" items were already fixed** — grep the codebase before flagging
- **Flag systemic issues once, not per-screen** — "all cards have wrong radius" is one issue
- **Compare to design system rules, not subjective preference**

## Phase 3: Propose

**STOP and present the audit to the user before proposing fixes.**

After user reviews the audit, for each approved issue propose 1-3 options:

### Option Levels
- **Conservative** — fix the specific bug, nothing else
- **Moderate** — fix the bug + improve surrounding context  
- **Redesign** — rethink the component/section

### For Each Option
- What changes visually (before → after)
- Which files to modify
- Exact token values from theme.ts
- Does this change cascade to other screens?
- Effort: trivial (< 5 min) / small (< 30 min) / medium (< 2 hr) / large (> 2 hr)

### Figma Mockup Generation
If the Figma MCP is connected and the user wants visual mockups:
1. Use `figma:figma-use` skill (mandatory prerequisite)
2. Use `figma:figma-generate-design` to create proposed designs in Figma
3. Generate both light and dark mode variants
4. This Figma file becomes the source of truth

If no Figma:
- Describe changes with exact token values
- Reference existing app screenshots as visual anchors

### User Approval Gate
**STOP after proposals. Present options. Wait for user to choose.** Never implement without explicit approval.

## Phase 4: Implement

Only after user approves:

1. **Read target files** — understand current state before changing
2. **Make surgical changes** — only what's needed for the approved fix
3. **Use the project's design system** — theme tokens, makeStyles pattern, existing components
4. **Never introduce new libraries** for visual changes
5. **Verify:**
   - `tsc --noEmit` — zero new TypeScript errors
   - `jest --no-coverage` — no new test failures
   - Both light and dark mode impact checked
6. **Retake screenshots** of changed screens via Maestro
7. **Compare** new vs old — did the fix actually land?
8. **If fix didn't land** — check if Metro needs `--clear`, or if the change needs a different approach

## Phase 5: Before/After Verification

After implementing fixes, verify they actually landed:

### Screenshot Diffing
1. **Save previous screenshots** before retaking:
   ```bash
   cp -r maestro/screenshots/full-app maestro/screenshots/full-app-before-fixes
   ```
2. **Retake screenshots** after code changes + Metro restart (`npx expo start --clear`)
3. **Compare key screens** side-by-side — read both old and new screenshot for each fixed item
4. **Report results:**
   - "Fix landed ✅" — visual change matches expected
   - "Fix didn't land ❌" — screenshot unchanged, likely stale bundle or wrong file
   - "Unexpected change ⚠️" — something changed that wasn't part of the fix (regression)

### If a Fix Didn't Land
- Check Metro was restarted with `--clear`
- Check the edit is in the correct file (not a duplicate or _shared copy)
- Check the style isn't being overridden by a more specific selector
- Check the component is actually rendered on this screen (not a different variant)

## Phase 6: Regression Tracking

### Store Audit History
After each audit run, save a summary to:
```
maestro/screenshots/audit-history/[date]-audit.md
```

Contents:
```markdown
# Audit — [date]
Screenshots: [count]
Issues found: [count by severity]
Issues fixed since last audit: [count]
New issues since last audit: [count]
Net change: [+/- count]
```

### Cross-Audit Comparison
On subsequent runs, read the previous audit file and compare:
- Which issues from last time are still present?
- Which were fixed?
- What's new?
- Is the trend improving or degrading?

Report this as "Audit Delta" at the top of the analysis output.

## Phase 7: Extended Checks (optional, user can skip)

These go beyond static screenshots. Ask the user if they want these.

### Screen Size Variants
Test on multiple device sizes to catch responsive issues:
```bash
# Boot iPhone SE simulator (375pt width)
xcrun simctl boot "iPhone SE (3rd generation)"
# Run capture flows on SE
# Compare against standard captures — look for:
# - Text truncation
# - Cards overflowing
# - Buttons wrapping
# - Touch targets too small
```

Flag any screen that looks broken on the smaller size.

### Edge Case Data
Create a Maestro flow that tests visual edge cases:
- **Long text**: City name "São Paulo do Interior do Brasil" — does the card clip?
- **Empty states**: User with zero trips, zero saves, zero reviews
- **Overflow text**: Review with 500 words — does the card handle it?
- **Missing data**: Spot with no tags, no category, no price
- **Numeric extremes**: Rating of 1.0, rating count of 99999

Generate a test flow that navigates to key screens with edge case data and screenshots each.

### Network State Simulation
Note in the audit report which screens need manual testing for:
- **Slow network**: Do loading skeletons appear? Or does the screen freeze?
- **No network**: Does the offline banner show? Are cached screens readable?
- **Timeout**: Do error states render correctly with retry buttons?
- **Partial failure**: Does the home feed render if 1 of 5 data sources fails?

The skill can't automate network simulation via Maestro, so flag these for manual QA with specific instructions.

### Interaction Quality (manual check list)
Generate a checklist for things screenshots can't capture:
- [ ] Scroll performance — 60fps, no jank
- [ ] Press animations — spring feedback on buttons/cards
- [ ] Sheet/modal transitions — smooth slide up
- [ ] Pull-to-refresh — spring physics feel natural
- [ ] Tab switching — instant, no flash
- [ ] Keyboard appearance — input scrolls into view
- [ ] Navigation transitions — push/pop feel native
- [ ] Dark mode toggle — instant switch, no flash of wrong theme

### Multi-Language / RTL Support

Detect whether the app has internationalization support:
```bash
# Check for i18n libraries and patterns
grep -rn "i18n\|intl\|locale\|t('\|t(\")\|useTranslation\|FormatMessage\|defineMessages\|react-intl\|i18next\|expo-localization" src/ app/ --include="*.ts" --include="*.tsx" | head -20
# Check for RTL-specific handling
grep -rn "RTL\|rtl\|isRTL\|I18nManager\|direction.*rtl\|writingDirection" src/ app/ --include="*.ts" --include="*.tsx" | head -10
# Check for locale-aware formatting
grep -rn "toLocaleDateString\|toLocaleString\|Intl\.NumberFormat\|Intl\.DateTimeFormat\|formatCurrency\|formatDate" src/ app/ --include="*.ts" --include="*.tsx" | head -10
```

**If i18n support exists**, generate this checklist:
- [ ] RTL layout: Do flexbox rows reverse correctly? Check `flexDirection: 'row'` containers — in RTL these should flip.
- [ ] Icons with directional meaning (arrows, chevrons) flip appropriately in RTL
- [ ] Translated strings don't overflow their containers — German/Finnish text is often 40% longer than English
- [ ] Date formats are locale-aware (MM/DD/YYYY vs DD/MM/YYYY vs YYYY-MM-DD)
- [ ] Currency formatting uses locale conventions (€1.000,00 vs $1,000.00)
- [ ] Number formatting respects locale (decimal separators, thousands grouping)
- [ ] Text alignment (`textAlign: 'left'`) uses `'start'`/`'end'` instead for RTL compatibility
- [ ] Padding/margin that's asymmetric (e.g., `paddingLeft: 16, paddingRight: 8`) uses `paddingStart`/`paddingEnd`

**If no i18n support exists**, flag as a recommendation:
> "No internationalization detected. If this app will serve non-English markets, consider adding i18n support. Key concerns: RTL layout support, string externalization, locale-aware date/currency formatting."

### State Persistence Across App Restart

Screenshots capture a single session, but users experience the app across restarts. Generate a manual checklist for persistence testing:

- [ ] **Dark mode preference**: Toggle dark mode → kill app → relaunch. Does the theme persist?
- [ ] **Auth state**: Log in → kill app → relaunch. Is the user still logged in? Or shown a login screen flash?
- [ ] **Onboarding completion**: Complete onboarding → kill app → relaunch. Does onboarding show again?
- [ ] **Cached data**: Load the home feed → kill app → enable airplane mode → relaunch. Does cached content appear immediately, or is it a blank/loading screen?
- [ ] **Form state**: Start filling a multi-step form → kill app → relaunch → navigate back. Is partial input preserved?
- [ ] **Scroll position**: Scroll deep into a list → switch tabs → return. Is the position retained?
- [ ] **Tab memory**: Navigate into a sub-screen on Tab 2 → switch to Tab 1 → switch back to Tab 2. Is the sub-screen still showing, or reset to root?

Check what persistence mechanism the app uses:
```bash
# AsyncStorage
grep -rn "AsyncStorage\|MMKV\|SecureStore\|expo-secure-store" src/ --include="*.ts" --include="*.tsx" | head -10
# Check what's being persisted
grep -rn "\.set(\|\.setItem(\|\.save(" src/ --include="*.ts" --include="*.tsx" | head -20
```

Flag any preference or state that is visually impactful (dark mode, auth, onboarding) but not persisted.

### Animation / Motion Audit

Screenshots can't assess animation quality, but code analysis can flag issues. Run these checks:

```bash
# Count animation usage
grep -rn "withSpring\|withTiming\|withDecay\|useAnimatedStyle\|useSharedValue" src/ app/ --include="*.tsx" --include="*.ts" | wc -l
# Framer Motion (web)
grep -rn "motion\.\|animate=\|variants=\|transition=" src/ web/ --include="*.tsx" --include="*.ts" | wc -l
# CSS transitions/animations
grep -rn "transition:\|@keyframes\|animation:" src/ web/ --include="*.css" --include="*.tsx" | wc -l
```

**Check for consistent spring configs:**
```bash
# Extract all spring/timing config values
grep -rn "withSpring(\|damping:\|stiffness:\|mass:" src/ app/ --include="*.tsx" --include="*.ts" | head -30
grep -rn "withTiming(\|duration:\|easing:" src/ app/ --include="*.tsx" --include="*.ts" | head -30
```
- Flag if there are more than 3 distinct spring configs — the app should use a small set of motion presets (e.g., `springGentle`, `springSnappy`, `springBouncy`)
- Flag any `withSpring` using the default config — these should be intentional, not accidental defaults

**Check if reduce motion is respected:**
```bash
grep -rn "useReducedMotion\|reduceMotion\|prefersReducedMotion\|accessibilityReduceMotion" src/ --include="*.tsx" --include="*.ts" | head -5
```
- If animations exist but no reduce motion check: flag as accessibility issue
- Recommendation: wrap all non-essential animations in a `useReducedMotion()` guard

**Manual motion quality checklist:**
- [ ] Spring animations don't bounce (per CLAUDE.md: "no bounce animations")
- [ ] Animations run at 60fps with `useNativeDriver: true` or Reanimated's native thread
- [ ] Sheet/modal open/close transitions are smooth, not janky
- [ ] Skeleton shimmer animations are subtle, not distracting
- [ ] Page transitions feel platform-native (iOS: push/pop slide, Android: material motion)
- [ ] No animations on initial render that delay content visibility
- [ ] Long press / hold animations have clear progress indication

### Platform Conventions (iOS)
Check against Human Interface Guidelines:
- [ ] Back swipe gesture works on all push screens
- [ ] Tab bar uses SF Symbols or equivalent weight icons
- [ ] Sheets use the system presentation style (pageSheet)
- [ ] Safe areas respected (notch, home indicator)
- [ ] Status bar text color adapts to content (light on dark hero)
- [ ] Haptic feedback on destructive actions

## Phase 8: Copy & Microcopy Audit (optional)

Scan all user-facing strings for consistency and quality.

### Automated String Scan
```bash
# Find all user-facing text in components
grep -rn "label=\|title=\|placeholder=\|body=\|message=\|text:" src/ app/ --include="*.tsx" | head -200
# Find toast messages
grep -rn "toast\.\(success\|error\|info\|warn\)" src/ app/ --include="*.tsx" --include="*.ts"
# Find empty state text
grep -rn "No.*yet\|No.*found\|empty\|Nothing" src/ app/ --include="*.tsx"
```

### Check For
- **Terminology inconsistency**: "Sign in" vs "Log in" vs "Login" — pick one
- **Capitalization inconsistency**: "Add to Trip" vs "Add to trip" vs "ADD TO TRIP"
- **Tone inconsistency**: formal "No results found" vs casual "Nothing here yet!"
- **Brand name usage**: "Scout" (AI) vs "Spots" (app) — verify CLAUDE.md naming rules
- **Action verb consistency**: "Save" vs "Keep" vs "Bookmark" for the same action
- **Error message quality**: raw errors like "Error: 500" vs human-readable "Something went wrong — try again"
- **Placeholder text quality**: "Enter text" (lazy) vs "Search spots, vibes, cities..." (helpful)
- **Empty state quality**: "No data" (bad) vs "Be the first to add a photo" (good)

### Output
Add a "Copy Issues" section to the audit report with each inconsistency and recommended standard.

## Phase 9: Accessibility Deep Dive (optional)

Goes beyond contrast checks.

### Color Blindness Simulation
Check if the UI relies on color alone to communicate state:
- Save/unsave: is there a shape change (filled vs outline icon) in addition to color?
- Error/success: is there an icon or text label alongside the color?
- Status badges: do they have text labels, not just colored dots?
- Charts/graphs: are they distinguishable without color?

Flag any element where removing color makes the state indistinguishable.

### Dynamic Type / Font Scaling
Check if the app respects system font size settings:
```bash
# Check if text uses system-scalable values
grep -rn "allowFontScaling\|maxFontSizeMultiplier" src/ app/ --include="*.tsx" | head -10
```

Common issues:
- Fixed-height containers that clip when text scales up
- Layouts that break when font is 1.5x default
- Touch targets that become too small relative to scaled text

Note: This requires manual testing with the simulator's accessibility settings. Generate a checklist:
- [ ] Set Dynamic Type to "Extra Large" in simulator settings
- [ ] Check: Home feed — text readable? Cards clip?
- [ ] Check: Profile — name and bio wrap correctly?
- [ ] Check: Trip detail — itinerary items don't overlap?
- [ ] Check: Buttons — labels still fit inside button bounds?

### VoiceOver Reading Order
Check that interactive elements have accessibility labels:
```bash
# Count elements with accessibilityLabel
grep -rn "accessibilityLabel" src/ app/ --include="*.tsx" | wc -l
# Count interactive elements (Pressable, Button, TouchableOpacity)
grep -rn "<Pressable\|<Button\|<TouchableOpacity" src/ app/ --include="*.tsx" | wc -l
```

Flag the gap — if there are 200 interactive elements but only 50 have labels, that's a problem.

### Reduce Motion
Check if animations respect the system's reduce motion setting:
```bash
grep -rn "useReducedMotion\|reduceMotion\|prefersReducedMotion" src/ --include="*.tsx" --include="*.ts" | head -5
```

If no reduce motion checks exist, flag as a recommendation.

## Phase 10: Component Drift Detection (optional)

Measure how well the codebase follows its own design system.

### Inline Style Count
```bash
# Count inline styles (anti-pattern)
grep -rn 'style={{' src/ app/ --include="*.tsx" | wc -l
# Count hardcoded colors (anti-pattern)  
grep -rn "'#[0-9A-Fa-f]\{3,8\}'" src/ app/ --include="*.tsx" | grep -v "theme\|constants\|vibeColors" | wc -l
# Count hardcoded spacing (anti-pattern)
grep -rn "margin.*: [0-9]\|padding.*: [0-9]" src/ app/ --include="*.tsx" | grep -v "StyleSheet\|makeStyles\|theme\." | wc -l
```

### Component Inventory vs Actual Usage

Check whether shared components are actually being used, or if screens are rolling their own:

```bash
# List all shared UI components
SHARED_COMPONENTS=$(ls src/components/ui/*.tsx 2>/dev/null | sed 's/.*\///' | sed 's/\.tsx//')

# For each shared component, count how many files import it
for comp in $SHARED_COMPONENTS; do
  COUNT=$(grep -rn "import.*$comp\|from.*$comp" src/ app/ --include="*.tsx" | grep -v "node_modules" | wc -l)
  echo "$comp: $COUNT usages"
done
```

**Then check for component bypass — screens building their own versions instead of using shared ones:**

```bash
# If a Button component exists, how many screens use raw Pressable/TouchableOpacity for actions?
SHARED_BUTTON_IMPORTS=$(grep -rn "import.*Button" src/ app/ --include="*.tsx" | grep "components/ui" | wc -l)
RAW_PRESSABLE=$(grep -rn "<Pressable\|<TouchableOpacity" src/ app/ --include="*.tsx" | grep -v "components/ui\|node_modules" | wc -l)
echo "Shared Button used: $SHARED_BUTTON_IMPORTS times. Raw Pressable/TouchableOpacity: $RAW_PRESSABLE times."

# If an Input component exists, how many screens use raw TextInput?
SHARED_INPUT_IMPORTS=$(grep -rn "import.*Input" src/ app/ --include="*.tsx" | grep "components/ui" | wc -l)
RAW_TEXTINPUT=$(grep -rn "<TextInput" src/ app/ --include="*.tsx" | grep -v "components/ui\|node_modules" | wc -l)
echo "Shared Input used: $SHARED_INPUT_IMPORTS times. Raw TextInput: $RAW_TEXTINPUT times."

# If a Card component exists, how many screens create custom card Views?
# (this one is harder to detect — look for View with border/radius/shadow patterns)
grep -rn "borderRadius.*12\|borderWidth.*1\|shadowColor" src/ app/ --include="*.tsx" | grep -v "components/ui\|makeStyles\|theme" | wc -l
```

**Output a usage table:**
```
Component       | Shared Usage | Bypass Usage | Adoption Rate
----------------|-------------|--------------|---------------
Button          | 47          | 12           | 80%
Input           | 18          | 3            | 86%
Tag/Chip        | 23          | 8            | 74%
Card (pattern)  | 31          | 15           | 67%
NavBackButton   | 22          | 0            | 100%
```

Flag specific files that bypass shared components — these are the highest-value cleanup targets. Example output:
> "Button component exists but `app/spot/[id].tsx` and `app/trip/[id].tsx` use custom `<Pressable>` for their primary CTAs instead."

### Design System Adoption Score
Calculate a health score:
- Total style declarations in components
- % using theme tokens vs hardcoded values
- % of screens using `makeStyles` pattern vs inline
- % of interactive elements using shared Button/Tag/Input vs custom Pressables

Output: "Design system adoption: 78% — 22% of styles bypass the theme"

### Trend Over Time
Compare against previous drift detection runs:
- "Last audit: 85% adoption. This audit: 78%. Drift increasing — 12 new hardcoded values added."

## Phase 11: CI Integration (optional setup)

Help the user set up automated visual audits in their CI pipeline.

### GitHub Actions Workflow
Generate a workflow file that:
1. Boots an iOS simulator in CI
2. Builds the app
3. Runs Maestro capture flows
4. Compares screenshots against a baseline
5. Posts a comment on the PR with any visual diffs

```yaml
# .github/workflows/visual-audit.yml
name: Visual Audit
on: pull_request
jobs:
  visual-audit:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm install
      - name: Install Maestro
        run: curl -Ls "https://get.maestro.mobile.dev" | bash
      - name: Boot Simulator
        run: xcrun simctl boot "iPhone 16 Pro"
      - name: Build App
        run: npx expo run:ios --simulator
      - name: Capture Screenshots
        run: |
          export PATH="$PATH:$HOME/.maestro/bin"
          maestro test maestro/flows/design/full-capture-1-tabs-details.yaml
          # ... more flows
      - name: Compare Against Baseline
        run: |
          # pixel-diff against maestro/screenshots/baseline/
          # generate report
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            // Post visual diff report as PR comment
```

### Baseline Management
- Store baseline screenshots in git (or git-lfs for large repos)
- Update baseline when design changes are intentional: `npm run update-visual-baseline`
- Flag pixel diffs above a threshold (e.g., >2% change on any screen)

### Scheduled Audits
Use the `schedule` skill or cron to run weekly audits:
- Every Monday at 9am: capture + analyze + report
- Email or Slack the report to the team
- Track the design system adoption score over time

## Limitations

Be explicit about what this skill CANNOT do:
- Cannot assess animation smoothness from static screenshots
- Cannot test real network conditions
- Cannot test on physical devices (simulator only)
- Cannot verify App Store guideline compliance
- Cannot test push notifications or deep link behavior from external sources
- Cannot test concurrent/multi-user scenarios
- Maestro scroll may not match real user scroll — identical scroll frames may be Maestro issue not app issue

Note these limitations in every audit report so the user knows what to test manually.

## Anti-Patterns

- DON'T audit from memory — always read actual screenshots
- DON'T propose changes that violate the project's CLAUDE.md
- DON'T implement without user approval
- DON'T change working code for subjective reasons
- DON'T skip dark mode
- DON'T confuse Maestro automation failures with app bugs
- DON'T add new dependencies for visual fixes
- DON'T flag the same systemic issue on every screen — flag it once
- DON'T assume screenshots are current — check git log for recent changes that may not be reflected
