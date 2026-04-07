---
name: incident
description: Incident response pipeline — reads error tracking (Sentry, Bugsnag, LogRocket), traces errors to source code, performs root cause analysis, suggests fixes with confidence levels, assesses severity, and generates post-mortems. Use when the user says "something broke", "check Sentry", "we have an incident", "debug this error", or wants to investigate production issues.
---

# Incident Skill

End-to-end incident response: Detect → Diagnose → Fix → Postmortem.


## Update Check

Before starting, check if the skill suite has updates available:

```bash
if [ -d ~/.claude/skills/.git ]; then
  git -C ~/.claude/skills fetch --quiet 2>/dev/null
  LOCAL=$(git -C ~/.claude/skills rev-parse HEAD 2>/dev/null)
  REMOTE=$(git -C ~/.claude/skills rev-parse origin/master 2>/dev/null)
  [ "$LOCAL" != "$REMOTE" ] && [ -n "$REMOTE" ] && echo "Update available: cd ~/.claude/skills && git pull"
fi
```

If update available, notify the user once (don't block execution):
> "Skill suite update available. Run: cd ~/.claude/skills && git pull"


## When to Use

- User says "something broke", "users are seeing errors", "check Sentry"
- User pastes an error message or stack trace
- User reports a production incident
- After a deploy when something seems wrong
- User wants to investigate a specific error or crash

## Pipeline Context Protocol

### On Start

1. Read `.claude/pipeline/manifest.md` if it exists — check if this incident is related to a recent deploy
2. Read `.claude/pipeline/features/[slug]/deploy-report.md` — check what was last deployed, rollback info
3. Cross-reference the error with recently changed files

### On Complete

Write report to `.claude/pipeline/features/[slug]/incident-report.md`:

```markdown
# Incident Report — [date]

## Summary
severity: [CRITICAL | MAJOR | MINOR]
status: [investigating | identified | fixed | monitoring]
started: [timestamp]
resolved: [timestamp or "ongoing"]

## Error
message: [error message]
source: [file:line]
frequency: [count in last hour/day]
users_affected: [estimate]

## Root Cause
[1-3 sentences explaining why this happened]

## Fix Applied
commit: [sha or "pending"]
confidence: [HIGH | MEDIUM | LOW]
files_changed: [list]

## Rollback Info
should_rollback: [yes/no]
rollback_command: [if applicable]

## Prevention
[what to change to prevent recurrence]
```

Update `.claude/pipeline/manifest.md` with incident status.

## Phase 0A: User Interview

### Required Questions

**1. What's happening?**
"Describe what users are seeing. Error message, broken flow, blank screen?"
→ If user pastes a stack trace, skip to diagnosis.

**2. When did it start?**
"When did you first notice? Did anything deploy recently?"
→ Correlate with deploy history.

**3. Scope:**
"Is it affecting all users or specific ones? (e.g., only iOS, only new users, only one city)"
→ Helps narrow root cause.

**4. Severity:**
"How bad is it?"
- CRITICAL: core flow broken for all users (auth, payments, main feature)
- MAJOR: significant feature broken, workaround exists
- MINOR: cosmetic or edge case, most users unaffected

**5. Fix preference:**
"Should I:"
- Diagnose only (find the bug, don't touch code)
- Diagnose + suggest fix (you review before applying)
- Diagnose + fix + open PR
- Diagnose + recommend rollback (if safer than hotfix)

### Store Preferences

Save to `.claude/pipeline/preferences.md`:
```
## Incident Preferences
default_fix_behavior: suggest-fix
escalation_threshold: CRITICAL
```

## Phase 1: Error Tracking Detection

Detect which error tracking tools are in use:

```bash
# Sentry
grep -rn "sentry\|@sentry" package.json 2>/dev/null
grep -rn "Sentry.init\|sentry.init\|SENTRY_DSN\|NEXT_PUBLIC_SENTRY_DSN" src/ web/ .env* 2>/dev/null | head -10

# Bugsnag
grep -rn "bugsnag\|@bugsnag" package.json 2>/dev/null

# LogRocket
grep -rn "logrocket\|LogRocket" package.json 2>/dev/null

# PostHog (has error tracking)
grep -rn "posthog\|POSTHOG" package.json .env* 2>/dev/null | head -5

# Custom error handling
grep -rn "ErrorBoundary\|errorBoundary\|onError\|captureException" src/ --include="*.ts" --include="*.tsx" | head -20

# Console error patterns
grep -rn "console\.error" src/ --include="*.ts" --include="*.tsx" | head -20
```

Report: "Found: Sentry (web + mobile), PostHog (analytics). No Bugsnag or LogRocket."

## Phase 2: Error Parsing & Source Tracing

### From Stack Trace

If the user provides a stack trace or error message:

1. **Extract the error type and message**
2. **Extract file paths and line numbers** from the stack
3. **Read the source files** at those locations
4. **Trace the call chain** — what called the failing function?
5. **Check recent git changes** to those files

```bash
# Find the source file
grep -rn "[error message or key phrase]" src/ --include="*.ts" --include="*.tsx" | head -10

# Check recent changes to the failing file
git log --oneline -10 -- [file-path]
git diff HEAD~5 -- [file-path]
```

### From Error Tracking Dashboard

If the user asks to check Sentry/error tracking:

```bash
# Check for Sentry CLI
which sentry-cli 2>/dev/null

# If Sentry CLI available
sentry-cli issues list --project [project] 2>/dev/null | head -20

# Otherwise, guide user to check dashboard
# "Open Sentry at [project-url]. What are the top errors in the last 24h?"
```

### From Supabase Logs

For Supabase-backed projects:

```bash
# Edge function logs
supabase functions logs [function-name] 2>/dev/null | tail -50

# Database logs (if accessible)
supabase db logs 2>/dev/null | tail -50

# Check edge function source
ls supabase/functions/*/index.ts 2>/dev/null
```

### From Application Logs

```bash
# Check for log files
ls logs/ *.log 2>/dev/null

# Check console output patterns
grep -rn "console\.error\|console\.warn" src/ --include="*.ts" --include="*.tsx" | head -20

# Check error boundary implementations
grep -rn "componentDidCatch\|ErrorBoundary" src/ --include="*.tsx" | head -10
```

## Phase 3: Root Cause Analysis

### Step-by-Step Diagnosis

1. **Reproduce the path**: Trace the user flow that triggers the error
   ```bash
   # Find the screen/route
   grep -rn "[screen or component name]" app/ --include="*.tsx" | head -5
   
   # Find the data flow
   grep -rn "[hook or query name]" src/ --include="*.ts" --include="*.tsx" | head -10
   ```

2. **Check data dependencies**: Is the error from bad data, missing data, or wrong types?
   ```bash
   # Check the query/mutation
   grep -rn "useQuery\|useMutation" [failing-file] | head -5
   
   # Check the API call
   grep -rn "supabase\.\|fetch\(" [failing-file] | head -5
   ```

3. **Check environment**: Is it env-specific?
   ```bash
   # Missing env vars
   grep -rn "process\.env\|import\.meta\.env" [failing-file] | head -5
   
   # Platform-specific code
   grep -rn "Platform\.OS\|Platform\.select" [failing-file] | head -5
   ```

4. **Check recent changes**: Did a recent commit introduce this?
   ```bash
   git log --oneline -20
   git log --oneline -10 -- [failing-file]
   git bisect start  # Guide user through bisect if needed
   ```

5. **Check dependencies**: Did a dependency update break something?
   ```bash
   git diff HEAD~10 -- package.json | head -40
   git diff HEAD~10 -- package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null | head -20
   ```

### Confidence Assessment

Rate the diagnosis:

- **HIGH confidence** (90%+): Clear stack trace, obvious code bug, reproducible
- **MEDIUM confidence** (60-89%): Likely cause identified but can't fully reproduce, or multiple possible causes
- **LOW confidence** (< 60%): Intermittent, no clear stack trace, could be infrastructure

Always state confidence: "Root cause identified with HIGH confidence: the `useTrip` hook doesn't handle null `trip.itinerary` when the trip was created before the itinerary feature shipped."

## Phase 4: Severity Assessment

### Impact Analysis

```bash
# How many users could be affected?
# Check if the failing code is in a core flow
grep -rn "[failing-component]" app/ --include="*.tsx" | head -10

# Is it in a frequently visited screen?
# Check navigation — is this a tab screen, detail screen, or settings screen?
grep -rn "[screen-name]" app/ --include="*.tsx" | head -5
```

### Severity Matrix

| Factor | CRITICAL | MAJOR | MINOR |
|--------|----------|-------|-------|
| Users affected | All / most | Segment | Few / edge case |
| Flow affected | Auth, payments, core feature | Secondary feature | Settings, profile, cosmetic |
| Data risk | Corruption, loss | Incorrect display | None |
| Workaround | None | Exists but awkward | Easy |
| Time sensitivity | Fix NOW | Fix today | Fix this week |

Report: "Severity: MAJOR. Affects users who try to view trip itineraries created before March 27. ~15% of trips. Workaround: users can recreate the trip."

## Phase 5: Fix Suggestion

### Fix Format

For each identified issue:

```markdown
## Suggested Fix

**Confidence:** HIGH
**Risk:** LOW (isolated change, no side effects)

**File:** src/features/trips/hooks/useTrip.ts
**Line:** 47

**Current code:**
const items = trip.itinerary.days[0].items;

**Fixed code:**
const items = trip.itinerary?.days?.[0]?.items ?? [];

**Explanation:** The itinerary field is null for trips created before the itinerary feature. Adding optional chaining prevents the crash.

**Test:** Add a test case for trip without itinerary in `__tests__/tripCrud.test.ts`
```

### Fix Categories

- **Safe fix** (apply with confidence): null checks, type guards, default values, missing imports
- **Targeted fix** (review recommended): logic changes, query modifications, state management
- **Architectural fix** (plan needed): data model changes, migration needed, breaking change

## Phase 6: Rollback Guidance

### When to Rollback

Recommend rollback when:
- Error rate > 5% of requests
- Core auth or payment flow broken
- Data corruption occurring
- Fix will take > 30 minutes and severity is CRITICAL

### Rollback Commands

```bash
# Vercel
vercel rollback 2>&1

# Supabase edge functions
git checkout [last-good-commit] -- supabase/functions/
supabase functions deploy 2>&1

# EAS OTA
eas update --branch production --message "rollback: [reason]" 2>&1

# Git revert (for hotfix branch)
git revert [bad-commit] --no-edit
```

### When to Hotfix Instead

- Fix is < 10 lines and HIGH confidence
- Rollback would revert other needed changes
- The fix is safer than the rollback (e.g., adding a null check vs reverting a migration)

## Phase 7: Post-Mortem

Generate a post-mortem document after the incident is resolved.

```markdown
# Post-Mortem: [incident title]

## Date
[date]

## Duration
[time from detection to resolution]

## Severity
[CRITICAL | MAJOR | MINOR]

## Summary
[1-2 sentences: what happened and who was affected]

## Timeline
- HH:MM — [event: user reported issue / alert fired / deploy happened]
- HH:MM — [investigation started]
- HH:MM — [root cause identified]
- HH:MM — [fix applied / rollback executed]
- HH:MM — [verified fix in production]
- HH:MM — [incident closed]

## Root Cause
[Technical explanation of why this happened]

## What Was Done
- [Action 1: e.g., "Added null check in useTrip hook"]
- [Action 2: e.g., "Added test case for trips without itinerary"]
- [Action 3: e.g., "Deployed hotfix to production"]

## What Went Well
- [e.g., "Error was caught within 10 minutes of deploy"]
- [e.g., "Rollback plan was documented and ready"]

## What Could Be Better
- [e.g., "No automated alert for this error type"]
- [e.g., "Missing test coverage for null itinerary case"]

## Action Items
- [ ] [Preventive action 1: e.g., "Add Sentry alert for >10 errors/min"]
- [ ] [Preventive action 2: e.g., "Add integration test for legacy trip data"]
- [ ] [Preventive action 3: e.g., "Add null safety lint rule"]

## Prevention
[What systemic change would prevent this class of bug]
```

Write to `.claude/pipeline/features/[slug]/postmortem-[date].md`

## Phase 8: Communication Template

Generate a user-facing status update:

```markdown
## Status Update Template

### During Incident
"We're aware of an issue affecting [feature]. Some users may experience [symptom]. We're investigating and will update shortly."

### Fix Deployed
"We've resolved the issue affecting [feature]. Everything should be working normally now. Thank you for your patience."

### For App Store Review Notes (if relevant)
"Fixed: [brief description of the bug and fix]"
```

## Autonomous Mode

When running autonomously (invoked by /ship or a subagent):

- **Skip interview** if user provided error details or if preferences exist
- **Run full diagnosis** — trace error, read source, check recent changes
- **Assess severity automatically** using the severity matrix
- **For CRITICAL**: STOP and escalate to human immediately, do not auto-fix
- **For MAJOR**: Generate fix suggestion, write incident-report.md, flag for human review
- **For MINOR**: Generate fix suggestion with HIGH confidence fixes only, write report
- **Never auto-deploy a fix** — always write the report and let deploy skill handle shipping
- **Always write incident-report.md** to the pipeline feature directory

## Anti-Patterns

- DON'T guess at root cause without reading the actual failing code
- DON'T apply a fix without understanding why the bug exists
- DON'T skip the severity assessment — it determines response urgency
- DON'T auto-rollback without checking if rollback is safe (migrations may make rollback dangerous)
- DON'T write a post-mortem that blames people — focus on systems and processes
- DON'T dismiss intermittent errors — they often indicate race conditions or data edge cases
- DON'T ignore the timeline — correlating the error start with deploys is the fastest diagnostic
