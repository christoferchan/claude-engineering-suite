---
name: analytics-audit
description: Analytics audit pipeline — scans all tracking calls, maps user flows to events, verifies event properties, checks for duplicates and gaps, validates funnels, audits PII compliance, and generates a tracking plan. Use when the user asks to audit analytics, check tracking coverage, review events, or create a tracking plan.
---

# Analytics Audit Skill

End-to-end analytics audit pipeline: Detect → Map → Validate → Report → Fix.

## When to Use

- User says "audit analytics", "check tracking", "review events", "create a tracking plan"
- Before a release to verify all key flows are tracked
- After adding new features that need event coverage
- When analytics data looks wrong or incomplete
- When preparing for a product review or investor meeting

## Phase 0: Pipeline Context

At the start of every run:

1. **Read the pipeline manifest** (if it exists):
   ```
   .claude/pipeline/manifest.md
   ```
   This tells you what feature is being audited, what other skills have already run, and any shared context.

2. **When finished**, write your report to:
   ```
   .claude/pipeline/features/[slug]/analytics-audit-report.md
   ```
   Where `[slug]` comes from the manifest. If no manifest exists (standalone run), write to:
   ```
   .claude/pipeline/analytics-audit-report.md
   ```

## Phase 0A: User Preferences Interview

Before doing anything, ask these questions. Wait for answers.

### Required Questions

**1. Analytics provider:**
"Which analytics tool(s) are you using?"
- PostHog
- Segment
- Amplitude
- Mixpanel
- Google Analytics (GA4)
- Custom / self-hosted
- Multiple (list which)
- Not sure / I'll let you detect it

Auto-detect if user says "not sure".

**2. Key metrics:**
"What are the 3-5 most important things you need to measure?"
Examples: onboarding completion, trip creation, subscription conversion, daily active users, feature adoption

Determines which missing events are CRITICAL vs MINOR.

**3. Funnels:**
"What are your critical conversion funnels?"
Examples: signup -> onboarding -> first trip, discovery -> save -> plan trip, free -> paywall -> subscribe

These get funnel completeness validation.

**4. Privacy requirements:**
"What privacy rules apply?"
- GDPR (EU users)
- CCPA (California users)
- COPPA (under-13 users)
- HIPAA (health data)
- None specific / best practices only

Determines PII scanning strictness.

**5. Scope:**
- Full app (every screen, every event)
- Specific flows (user lists which)
- New features only (what was added since last audit)
- Tracking plan review only (no code scanning)

**6. Implementation preference:**
After the audit, do you want me to:
- Just report findings (no code changes)
- Report + propose event additions with options
- Report + add missing events automatically
- Full autopilot (add events, fix properties, ask only on naming)

### Store Preferences

Save responses to memory for future runs:
```
memory/feedback_analytics_audit_preferences.md
```

On subsequent runs, ask: "I have your analytics preferences from last time — [summary]. Still accurate, or want to update anything?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after analysis for user review
- STOP after proposals for user approval
- Only add/modify events the user explicitly approves

### Autonomous Mode (--dangerously-skip-permissions or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** Fall back to safe defaults:
  - Provider: auto-detect
  - Scope: full app
  - Implementation: report + propose only (NO auto-add events)
  - Privacy: GDPR-level strictness

- **Phase 2: Analysis runs fully — no stop.**

- **Phase 3: Implement — ONLY auto-implement if ALL of these are true:**
  1. User's stored preference is "full autopilot"
  2. The change is adding a missing event (not modifying existing)
  3. The event follows existing naming conventions exactly
  4. The change passes typecheck and tests after applying

  Otherwise: generate a report file and notify the user.

## Platform Detection

### Step 1: Detect Analytics Provider

```bash
# PostHog
grep -rq "posthog\|PostHog\|POSTHOG" src/ package.json 2>/dev/null && echo "POSTHOG"

# Segment
grep -rq "analytics\|@segment\|segment" package.json 2>/dev/null && echo "SEGMENT"

# Amplitude
grep -rq "amplitude\|@amplitude" package.json 2>/dev/null && echo "AMPLITUDE"

# Mixpanel
grep -rq "mixpanel" package.json 2>/dev/null && echo "MIXPANEL"

# Google Analytics
grep -rq "gtag\|GA4\|google-analytics\|@google-analytics" src/ package.json 2>/dev/null && echo "GA4"

# Custom
grep -rq "analytics\.track\|trackEvent\|logEvent" src/ --include="*.ts" --include="*.tsx" 2>/dev/null && echo "CUSTOM_TRACKING"
```

### Step 2: Detect Framework

```bash
ls app.json 2>/dev/null && echo "EXPO"
grep -q '"next"' package.json 2>/dev/null && echo "NEXTJS"
grep -q '"react-native"' package.json 2>/dev/null && echo "REACT_NATIVE"
grep -q '"vue"\|"nuxt"' package.json 2>/dev/null && echo "VUE"
grep -q '"@angular"' package.json 2>/dev/null && echo "ANGULAR"
```

### Step 3: Find Analytics Configuration

```bash
# Analytics init/config files
grep -rn "posthog\.init\|analytics\.load\|amplitude\.init\|mixpanel\.init\|gtag.*config" src/ --include="*.ts" --include="*.tsx" | head -10

# Analytics wrapper/utility files
find src/ -name "*analytics*" -o -name "*tracking*" -o -name "*telemetry*" -o -name "*events*" 2>/dev/null | grep -v "test\|mock\|node_modules"

# Event type definitions
grep -rn "type.*Event\|interface.*Event\|enum.*Event" src/ --include="*.ts" | grep -i "analytic\|track\|event" | head -10
```

## Phase 1: Event Inventory

### 1.1 Collect All Tracking Calls

```bash
# PostHog
grep -rn "posthog\.capture\|posthog\.identify\|posthog\.group\|posthog\.alias" src/ --include="*.ts" --include="*.tsx" | grep -v test

# Segment
grep -rn "analytics\.track\|analytics\.identify\|analytics\.page\|analytics\.group" src/ --include="*.ts" --include="*.tsx" | grep -v test

# Amplitude
grep -rn "amplitude\.track\|amplitude\.identify\|amplitude\.setUserId\|logEvent" src/ --include="*.ts" --include="*.tsx" | grep -v test

# Mixpanel
grep -rn "mixpanel\.track\|mixpanel\.identify\|mixpanel\.people" src/ --include="*.ts" --include="*.tsx" | grep -v test

# Google Analytics
grep -rn "gtag.*event\|sendGAEvent\|logEvent" src/ --include="*.ts" --include="*.tsx" | grep -v test

# Generic / custom
grep -rn "trackEvent\|track(\|logAnalytics\|captureEvent" src/ --include="*.ts" --include="*.tsx" | grep -v test | grep -v "node_modules"
```

### 1.2 Extract Event Names and Properties

For each tracking call found, extract:
- Event name (string)
- Properties object (keys and value types)
- File and line number
- Which screen/component it's in

Build a complete inventory table.

### 1.3 Check Event Naming Convention

```bash
# Are event names consistent?
# Common conventions: snake_case, camelCase, Title Case, dot.notation
# Extract all unique event names and check for mixed conventions
grep -rn "capture(\|track(\|logEvent(" src/ --include="*.ts" --include="*.tsx" | grep -oP "['\"]([a-zA-Z_.]+)['\"]" | sort -u
```

Flag: mixed naming conventions (some snake_case, some camelCase).

## Phase 2: Flow-to-Event Mapping

### 2.1 Map All User Flows

```bash
# Every screen the user can reach
find app/ -name "*.tsx" -not -name "_*" -not -name "*.test.*" | sort

# Every navigation action
grep -rn "router\.push\|router\.replace\|navigation\.navigate\|Link href\|router\.back" src/ app/ --include="*.ts" --include="*.tsx" | head -40

# Every mutation (user action that changes data)
grep -rn "useMutation" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -30

# Every button/CTA
grep -rn "onPress=\|onClick=\|onSubmit=" src/ app/ --include="*.tsx" | head -40
```

### 2.2 Cross-Reference: Flow vs Events

For each flow/screen/action, check if there's a corresponding tracking event:

- Screen view events for each screen
- Action events for each mutation
- Funnel events for each step in critical funnels
- Error events for each error state
- Timing events for each async operation

### 2.3 Generate Gap Report

List all flows/actions that have NO tracking event. Categorize by importance:
- CRITICAL: Core funnel steps with no tracking
- MAJOR: Key features with no tracking
- MINOR: Secondary flows with no tracking

## Phase 3: Event Validation

### 3.1 Property Type Checking

```bash
# Find events with potentially wrong property types
grep -rn "capture\|track\|logEvent" src/ --include="*.ts" --include="*.tsx" -A 5 | grep -v test
```

For each event, verify:
- Required properties are always present (not undefined/null)
- Property types are consistent across all call sites
- Numeric properties are actually numbers (not strings)
- Timestamps are in a consistent format

### 3.2 Duplicate Event Detection

```bash
# Same event name tracked in multiple places
grep -rn "capture(\|track(\|logEvent(" src/ --include="*.ts" --include="*.tsx" | grep -v test | sort -t: -k3 | uniq -d -f2
```

Flag: same action tracked twice (e.g., both in a component and in a hook).

### 3.3 User Identification

```bash
# identify calls — are they called after auth?
grep -rn "identify\|setUserId\|alias" src/ --include="*.ts" --include="*.tsx" | grep -v test

# Are events before auth anonymous?
# Check: is there a distinct_id or anonymous_id flow?
grep -rn "anonymousId\|anonymous_id\|distinctId\|distinct_id" src/ --include="*.ts" --include="*.tsx"

# Super properties / user properties
grep -rn "register\|setUserProperties\|people\.set\|setPersonProperties" src/ --include="*.ts" --include="*.tsx"
```

## Phase 4: Funnel Validation

For each critical funnel (from user interview):

1. List all steps in the funnel
2. Check each step has a tracking event
3. Verify events have properties that link steps together (e.g., session_id, funnel_id)
4. Check for events that mark funnel drop-off
5. Verify the event sequence can be reconstructed from the data

## Phase 5: Privacy Compliance

### 5.1 PII Scanning

```bash
# Events that might contain PII
grep -rn "capture\|track\|logEvent" src/ --include="*.ts" --include="*.tsx" -A 10 | grep -i "email\|name\|phone\|address\|ip\|location\|birth\|ssn\|password"

# User properties that might be PII
grep -rn "identify\|setUserProperties\|people\.set" src/ --include="*.ts" --include="*.tsx" -A 10 | grep -i "email\|name\|phone\|address"
```

### 5.2 Consent Checking

```bash
# Is there a consent mechanism?
grep -rn "consent\|optOut\|opt_out\|gdpr\|ccpa\|cookie.*banner\|privacy.*modal" src/ --include="*.ts" --include="*.tsx"

# Is tracking conditional on consent?
grep -rn "if.*consent\|consent.*&&\|hasConsent\|isOptedIn" src/ --include="*.ts" --include="*.tsx"
```

### 5.3 Data Retention

```bash
# Check analytics config for retention settings
grep -rn "retention\|ttl\|expire\|purge\|delete.*data" src/ --include="*.ts" --include="*.tsx" | grep -i "analytic\|track\|posthog\|segment"
```

## Phase 6: Report

### Severity Levels

- **CRITICAL**: Core funnel has no tracking, PII in events, user identification broken
- **MAJOR**: Key feature untracked, duplicate events causing inflated metrics, wrong property types
- **MINOR**: Missing screen view event, inconsistent naming, secondary flow untracked

### Report Format

```markdown
# Analytics Audit Report — [date]

## Summary
- **Provider**: [detected provider(s)]
- **Total events found**: [count]
- **Screens tracked**: [X] / [total screens]
- **Flows tracked**: [X] / [total flows]
- **CRITICAL**: [count] | **MAJOR**: [count] | **MINOR**: [count]

## Event Inventory
| Event Name | Category | File | Properties | Call Count |
|------------|----------|------|------------|------------|

## Flow Coverage
| Flow | Steps | Tracked Steps | Coverage | Status |
|------|-------|---------------|----------|--------|

## Funnel Validation
| Funnel | Steps | Tracked | Linkable | Status |
|--------|-------|---------|----------|--------|

## Findings

### CRITICAL
| # | Issue | Category | File | Description |
|---|-------|----------|------|-------------|

### MAJOR
| # | Issue | Category | File | Description |
|---|-------|----------|------|-------------|

### MINOR
| # | Issue | Category | File | Description |
|---|-------|----------|------|-------------|

## Privacy Compliance
| Check | Status | Details |
|-------|--------|---------|
| PII in events | [PASS/FAIL] | |
| Consent mechanism | [PASS/FAIL] | |
| User identification | [PASS/FAIL] | |

## Tracking Plan
[If no tracking plan exists, generate one from the inventory]

## Proposed Changes
[For each finding: what event to add/modify, exact file and location, code snippet]

## Recommended Next Steps
[Prioritized list]
```

## Phase 7: Tracking Plan Generation

If no tracking plan exists, generate one from the audit:

```markdown
# Tracking Plan — [app name]

## Naming Convention
[detected or recommended convention]

## Events

### Core Events
| Event | Trigger | Properties | Required |
|-------|---------|------------|----------|

### Screen Events
| Event | Screen | Properties |
|-------|--------|------------|

### Funnel Events
| Funnel | Step | Event | Properties |
|--------|------|-------|------------|

### User Properties
| Property | Type | Set When |
|----------|------|----------|
```

## Phase 8: Fix Proposals

For each finding, propose a fix with:
1. **What**: Missing event or incorrect event
2. **Where**: Exact file and line to add/modify
3. **Code**: The tracking call to add
4. **Properties**: What properties to include
5. **Naming**: Following the detected convention

### Fix Rules
- NEVER add PII to event properties
- NEVER track before consent is given (if consent mechanism exists)
- Follow the existing naming convention exactly
- Add events at the action site, not deep in utility code
- Include all relevant context properties (screen, source, etc.)

## Pipeline Report Output

When done, write the structured report to the pipeline path (see Phase 0). The report must include:
- Timestamp
- Severity summary (CRITICAL/MAJOR/MINOR counts)
- Complete event inventory
- Flow coverage percentage
- Every finding with file path
- Fix status (proposed / approved / applied / skipped)
