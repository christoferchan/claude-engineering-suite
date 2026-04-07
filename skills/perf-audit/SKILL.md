---
name: perf-audit
description: Performance audit pipeline — analyzes bundle size, render performance, memory leaks, image optimization, font loading, lazy loading opportunities, list virtualization, and Core Web Vitals. Use when the user asks to optimize performance, check bundle size, find memory leaks, or improve load times.
---

# Performance Audit Skill

End-to-end performance audit pipeline: Detect → Measure → Analyze → Report → Optimize.

## When to Use

- User says "optimize performance", "check bundle size", "find memory leaks", "why is it slow"
- Before a release to verify performance meets thresholds
- After adding a large dependency or feature
- When users report sluggish behavior or high memory usage

## Phase 0: Pipeline Context

At the start of every run:

1. **Read the pipeline manifest** (if it exists):
   ```
   .claude/pipeline/manifest.md
   ```
   This tells you what feature is being audited, what other skills have already run, and any shared context.

2. **When finished**, write your report to:
   ```
   .claude/pipeline/features/[slug]/perf-audit-report.md
   ```
   Where `[slug]` comes from the manifest. If no manifest exists (standalone run), write to:
   ```
   .claude/pipeline/perf-audit-report.md
   ```

## Phase 0A: User Preferences Interview

Before doing anything, ask these questions. Wait for answers.

### Required Questions

**1. Performance priorities — rank these 1-5:**
"Which matters most to you right now?"
- Initial load time (time to interactive)
- Runtime smoothness (no jank, 60fps)
- Memory usage (no leaks, low footprint)
- Bundle size (smaller download)
- API response time (fast data fetching)

Determines which findings get CRITICAL vs MINOR severity.

**2. Known bottlenecks:**
"Any specific screens or flows that feel slow?"

Test these first with extra instrumentation.

**3. Target devices:**
"What's the lowest-end device your users have?"
- Latest flagship only
- Mid-range (2-3 year old phones)
- Low-end (budget phones, older devices)

Determines acceptable thresholds for memory and render time.

**4. Performance budgets (optional):**
"Do you have specific targets?"
- Bundle size limit: ___ KB
- Time to interactive: ___ ms
- API response budget: ___ ms
- FPS target: 60fps / 30fps acceptable

If not provided, use industry defaults.

**5. Scope:**
- Full app (every screen, every module)
- Specific screens (user lists which)
- Bundle only (just analyze size and dependencies)
- Runtime only (just analyze rendering and memory)

**6. Implementation preference:**
After the audit, do you want me to:
- Just report findings (no code changes)
- Report + propose optimizations with options
- Report + implement easy wins automatically
- Full autopilot (optimize everything, ask only on trade-offs)

### Store Preferences

Save responses to memory for future runs:
```
memory/feedback_perf_audit_preferences.md
```

On subsequent runs, ask: "I have your performance preferences from last time — [summary]. Still accurate, or want to update anything?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after analysis for user review
- STOP after proposals for user approval
- Only optimize what the user explicitly approves

### Autonomous Mode (--dangerously-skip-permissions or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** Fall back to safe defaults:
  - Priority: bundle size > load time > runtime > memory > API
  - Scope: full app
  - Implementation: report + propose only (NO auto-implement)

- **Phase 2: Analysis runs fully — no stop.**

- **Phase 3: Implement — ONLY auto-implement if ALL of these are true:**
  1. User's stored preference is "full autopilot"
  2. The optimization is severity CRITICAL or MAJOR
  3. The change is safe (memoization, lazy import, image compression — not architectural)
  4. The change passes typecheck and tests after applying
  5. The change does not alter application behavior

  Otherwise: generate a report file and notify the user.

## Platform Detection

Auto-detect the tech stack and performance-relevant configuration.

### Step 1: Detect Framework

```bash
# Mobile
ls app.json app.config.js 2>/dev/null && echo "EXPO_PROJECT"
grep -q "react-native" package.json 2>/dev/null && echo "REACT_NATIVE"
ls pubspec.yaml 2>/dev/null && echo "FLUTTER"
ls android/ ios/ 2>/dev/null && echo "NATIVE_DIRS"

# Web
grep -q '"next"' package.json 2>/dev/null && echo "NEXTJS"
grep -q '"vite"\|"@vitejs"' package.json 2>/dev/null && echo "VITE"
grep -q '"webpack"' package.json 2>/dev/null && echo "WEBPACK"
grep -q '"nuxt"' package.json 2>/dev/null && echo "NUXT"
grep -q '"@angular"' package.json 2>/dev/null && echo "ANGULAR"
grep -q '"svelte"' package.json 2>/dev/null && echo "SVELTE"

# Bundler
ls webpack.config.* 2>/dev/null && echo "WEBPACK_CONFIG"
ls vite.config.* 2>/dev/null && echo "VITE_CONFIG"
ls metro.config.* 2>/dev/null && echo "METRO_CONFIG"
```

### Step 2: Check Runtime Status

```bash
# Mobile
xcrun simctl list devices booted 2>&1 | grep -q "Booted" && echo "SIMULATOR_READY"
curl -s http://localhost:8081/status 2>/dev/null && echo "METRO_RUNNING"

# Web
curl -s http://localhost:3000 2>/dev/null && echo "WEB_DEV_RUNNING"
curl -s http://localhost:5173 2>/dev/null && echo "VITE_RUNNING"
```

Present status and ask user to start services if runtime profiling is needed.

## Phase 1: Bundle Size Analysis

### 1.1 Dependency Size

```bash
# Total node_modules size
du -sh node_modules/ 2>/dev/null

# Largest packages
du -sh node_modules/*/ 2>/dev/null | sort -rh | head -20

# Check for duplicate packages
npm ls --all 2>/dev/null | grep -i "deduped" | wc -l
npm ls --all 2>/dev/null | grep -v "deduped" | grep "node_modules.*node_modules" | head -20

# Package analysis (if available)
npx cost-of-modules 2>/dev/null || true
```

### 1.2 Bundle Output Size

```bash
# Next.js build analysis
ls .next/analyze/ 2>/dev/null && echo "BUNDLE_ANALYSIS_EXISTS"

# Expo/React Native bundle
npx react-native-bundle-visualizer 2>/dev/null || true

# Check for source maps (should not be in production)
find .next/ dist/ build/ -name "*.map" 2>/dev/null | head -10
```

### 1.3 Heavy Imports

```bash
# Large libraries imported (check for tree-shaking opportunities)
grep -rn "import .* from ['\"]lodash['\"]" src/ --include="*.ts" --include="*.tsx"
# Should be: import { specific } from 'lodash/specific'

grep -rn "import .* from ['\"]moment['\"]" src/ --include="*.ts" --include="*.tsx"
# Should use date-fns or dayjs instead

grep -rn "import .* from ['\"]@mui\|import .* from ['\"]antd" src/ --include="*.ts" --include="*.tsx"
# Check for barrel import tree-shaking issues

# Barrel exports that might prevent tree-shaking
grep -rn "export \* from" src/ --include="*.ts" --include="*.tsx" | head -20
```

## Phase 2: Render Performance

### 2.1 Unnecessary Re-renders

```bash
# Components without memoization that receive object/array props
grep -rn "export default function\|export function\|const.*=.*(" src/components/ --include="*.tsx" | head -40
# Cross-reference with: are these wrapped in React.memo?

grep -rn "React\.memo\|memo(" src/ --include="*.tsx" | head -20

# Inline object/array creation in JSX (causes re-render on every pass)
grep -rn "style=\{\{" src/ --include="*.tsx" | head -20
grep -rn "\[\]}\|{}\}" src/ --include="*.tsx" | grep "=\s*{" | head -20

# Missing useCallback/useMemo
grep -rn "onPress=\{(" src/ --include="*.tsx" | grep -v "useCallback\|useRef" | head -20
```

### 2.2 Heavy Component Trees

```bash
# Deep nesting — files with many nested components
wc -l src/components/**/*.tsx src/features/**/*.tsx 2>/dev/null | sort -rn | head -20

# ScrollView with many children (should be FlatList/FlashList)
grep -rn "ScrollView" src/ --include="*.tsx" | head -20
# Check: does it contain a .map() rendering a list?

# Conditional rendering that mounts/unmounts heavy components
grep -rn "&&\s*<\|? <" src/ --include="*.tsx" | head -20
```

### 2.3 Animation Performance

```bash
# Animated API — check useNativeDriver
grep -rn "Animated\.\|useAnimatedStyle\|useSharedValue" src/ --include="*.tsx" --include="*.ts" | head -20

# Missing useNativeDriver: true
grep -rn "Animated\.timing\|Animated\.spring\|Animated\.decay" src/ --include="*.tsx" --include="*.ts" | head -20

# Layout animations (can cause jank)
grep -rn "LayoutAnimation\|layoutAnimation" src/ --include="*.tsx" --include="*.ts"

# Reanimated (preferred for RN)
grep -rn "react-native-reanimated" package.json src/ --include="*.ts" --include="*.tsx"
```

## Phase 3: Memory Leaks

### 3.1 Event Listener Cleanup

```bash
# addEventListener without removeEventListener
grep -rn "addEventListener" src/ --include="*.ts" --include="*.tsx" | grep -v "test\|mock"
# Cross-reference: does the same file have a cleanup in useEffect return?

# Subscriptions not cleaned up
grep -rn "\.subscribe\|\.on(" src/ --include="*.ts" --include="*.tsx" | grep -v "test\|mock\|node_modules"
# Check: is there a corresponding .unsubscribe/.off in cleanup?
```

### 3.2 useEffect Cleanup

```bash
# useEffect without cleanup function
grep -rn "useEffect" src/ --include="*.ts" --include="*.tsx" | head -40
# For each: check if it has a return () => {} cleanup when using timers, listeners, or subscriptions

# setInterval / setTimeout without cleanup
grep -rn "setInterval\|setTimeout" src/ --include="*.ts" --include="*.tsx" | grep -v "test\|mock"
# Check: is clearInterval/clearTimeout called in cleanup?
```

### 3.3 Zustand / State Leaks

```bash
# Large state objects that grow without bounds
grep -rn "\.push\|\.concat\|\.set(" src/stores/ --include="*.ts" | head -20
# Check: is there a cleanup/limit mechanism?

# React Query cache — check staleTime and gcTime
grep -rn "staleTime\|gcTime\|cacheTime" src/ --include="*.ts" --include="*.tsx" | head -20
```

## Phase 4: Image Optimization

```bash
# Image files in the project
find assets/ public/ src/ -name "*.png" -o -name "*.jpg" -o -name "*.jpeg" -o -name "*.gif" -o -name "*.svg" -o -name "*.webp" 2>/dev/null | head -40

# Large images (over 500KB)
find assets/ public/ src/ \( -name "*.png" -o -name "*.jpg" -o -name "*.jpeg" \) -size +500k 2>/dev/null

# Images without explicit dimensions (causes layout shift)
grep -rn "<Image\|<img" src/ --include="*.tsx" | grep -v "width\|height\|style" | head -20

# Missing lazy loading on images
grep -rn "<img" src/ --include="*.tsx" | grep -v "loading=" | head -20

# Next.js Image component usage (should use next/image for auto-optimization)
grep -rn '<img ' src/ web/ --include="*.tsx" | grep -v "next/image\|Image from" | head -20

# React Native image caching
grep -rn "FastImage\|expo-image\|Image\.prefetch" src/ --include="*.tsx" --include="*.ts"
```

## Phase 5: Font Loading

```bash
# Font loading strategy
grep -rn "font-display\|fontDisplay" src/ web/ --include="*.css" --include="*.tsx" --include="*.ts"

# Unused font weights loaded
grep -rn "@font-face\|loadAsync\|useFonts\|Google_Fonts" src/ web/ --include="*.css" --include="*.tsx" --include="*.ts"

# Font file sizes
find assets/ public/ -name "*.woff2" -o -name "*.woff" -o -name "*.ttf" -o -name "*.otf" 2>/dev/null | xargs ls -lh 2>/dev/null
```

## Phase 6: Lazy Loading and Code Splitting

```bash
# Dynamic imports (already code-split)
grep -rn "React\.lazy\|lazy(\|import(\|dynamic(" src/ --include="*.ts" --include="*.tsx" | head -20

# Large components that could be lazy loaded
wc -l src/features/**/*.tsx src/components/**/*.tsx 2>/dev/null | sort -rn | head -20
# Components over 300 lines are candidates for code splitting

# Route-level code splitting
grep -rn "loadable\|Suspense\|lazy" app/ src/ --include="*.tsx" | head -20

# Unused exports (dead code)
grep -rn "^export " src/ --include="*.ts" --include="*.tsx" | head -40
```

## Phase 7: Mobile-Specific (React Native / Expo)

```bash
# FlatList vs ScrollView for lists
grep -rn "ScrollView" src/ --include="*.tsx" | head -20
grep -rn "FlatList\|FlashList\|SectionList" src/ --include="*.tsx" | head -20

# FlatList optimization props
grep -rn "FlatList" src/ --include="*.tsx" -A 10 | grep -i "getItemLayout\|removeClippedSubviews\|maxToRenderPerBatch\|windowSize\|initialNumToRender"

# Image caching strategy
grep -rn "expo-image\|FastImage\|Image.prefetch\|cachePolicy" src/ --include="*.tsx" --include="*.ts"

# Hermes engine (should be enabled for RN)
grep -rn "hermes\|jsEngine" app.json metro.config.* 2>/dev/null
```

## Phase 8: Web-Specific (Next.js / Vite)

```bash
# Core Web Vitals hints
# LCP — largest contentful paint candidates
grep -rn "priority\|fetchPriority\|preload" src/ web/ --include="*.tsx" | head -10

# CLS — cumulative layout shift (missing dimensions)
grep -rn "<img\|<Image" src/ web/ --include="*.tsx" | grep -v "width\|height\|fill" | head -10

# FID — first input delay (heavy JS on load)
grep -rn "useEffect.*\[\]" src/ web/ --include="*.tsx" | head -20
# Check: do any of these run expensive operations on mount?

# SSR vs CSR — check if data fetching is server-side where possible
grep -rn "getServerSideProps\|getStaticProps\|generateStaticParams\|fetch.*cache" web/ --include="*.tsx" --include="*.ts"
```

## Phase 9: API Performance

```bash
# N+1 query patterns
grep -rn "for.*await\|forEach.*await\|map.*await" src/ --include="*.ts" --include="*.tsx" | head -20
# Awaiting in a loop is usually an N+1 problem

# Unbounded queries (no LIMIT)
grep -rn "\.select(\|\.from(" src/ --include="*.ts" --include="*.tsx" | grep -v "\.limit\|\.range\|\.single" | head -20

# Missing pagination
grep -rn "useInfiniteQuery\|hasNextPage\|fetchNextPage" src/ --include="*.ts" --include="*.tsx"
# Compare against: how many list screens exist vs how many use infinite query

# Waterfall requests (sequential when could be parallel)
grep -rn "await.*await" src/ --include="*.ts" --include="*.tsx" | head -20
# Check: could these be Promise.all?
```

## Phase 10: Report

### Severity Levels

- **CRITICAL**: Visible to users, degrades UX significantly (bundle > 2MB, render > 100ms, memory leak in core flow)
- **MAJOR**: Impacts performance measurably but not immediately visible (missing memoization on hot path, large unoptimized images)
- **MINOR**: Best practice violation, marginal impact (could lazy load, missing font-display, unused export)

### Report Format

```markdown
# Performance Audit Report — [date]

## Summary
- **Framework**: [detected stack]
- **Bundle size**: [total JS size]
- **CRITICAL**: [count] | **MAJOR**: [count] | **MINOR**: [count]

## Performance Budgets
| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Bundle size (JS) | < [X] KB | [Y] KB | [PASS/FAIL] |
| Time to interactive | < [X] ms | [Y] ms | [PASS/FAIL] |
| Largest image | < 500 KB | [Y] KB | [PASS/FAIL] |
| FlatList coverage | 100% | [Y]% | [PASS/FAIL] |

## Findings

### CRITICAL
| # | Category | Issue | File | Impact | Estimated Savings |
|---|----------|-------|------|--------|-------------------|

### MAJOR
| # | Category | Issue | File | Impact | Estimated Savings |
|---|----------|-------|------|--------|-------------------|

### MINOR
| # | Category | Issue | File | Impact | Estimated Savings |
|---|----------|-------|------|--------|-------------------|

## Proposed Optimizations
[For each finding: what to change, expected improvement, risk/trade-offs]

## Recommended Next Steps
[Prioritized by impact]
```

## Phase 11: Fix Proposals

For each finding, propose an optimization with:
1. **What**: Exact file, current code
2. **Why**: Measured or estimated performance impact
3. **Fix**: The specific code change
4. **Expected improvement**: Bundle size reduction, render time reduction, etc.
5. **Trade-offs**: Any downsides (e.g., lazy loading adds a loading state)

### Fix Rules
- NEVER remove features to improve performance (optimize, don't cut)
- NEVER disable error handling for speed
- Prefer non-breaking changes (memoization, lazy loading) over architectural rewrites
- After implementing, re-measure to verify improvement
- Present options when there are multiple valid approaches

## Pipeline Report Output

When done, write the structured report to the pipeline path (see Phase 0). The report must include:
- Timestamp
- Severity summary (CRITICAL/MAJOR/MINOR counts)
- Every finding with file path and estimated impact
- Fix status (proposed / approved / applied / skipped)
- Before/after measurements where available
