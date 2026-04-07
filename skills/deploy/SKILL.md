---
name: deploy
description: Deployment pipeline — detects platform (Vercel, Netlify, AWS, Railway, Fly.io, Heroku, Supabase, EAS), runs pre-deploy checks, executes deploy, verifies post-deploy health, and handles rollback. Use when the user says "deploy", "push to production", "ship to staging", or wants to release code to any environment.
---

# Deploy Skill

End-to-end deployment pipeline: Detect → Check → Deploy → Verify → Log.


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

- User says "deploy", "push to production", "ship to staging"
- User wants to release a new version of the app or web app
- User wants to deploy edge functions
- User wants to promote staging to production
- After a feature passes QA and is ready to ship

## Pipeline Context Protocol

### On Start

1. Read `.claude/pipeline/manifest.md` if it exists — understand what feature is being shipped, what skills ran before
2. Read the feature directory `.claude/pipeline/features/[slug]/` — check qa-report, security-report for blockers
3. If any upstream skill reported CRITICAL issues, STOP and warn: "QA found critical failures. Deploy blocked until resolved."

### On Complete

Write report to `.claude/pipeline/features/[slug]/deploy-report.md`:

```markdown
# Deploy Report — [date]

## Environment
platform: [Vercel | Supabase | EAS | etc.]
target: [staging | production]
commit: [sha]
branch: [branch name]

## Pre-Deploy Checks
- [ ] Tests: [pass/fail — count]
- [ ] Lint: [pass/fail]
- [ ] Type check: [pass/fail]
- [ ] Env vars: [all set / missing: X, Y]
- [ ] Migrations: [applied / pending: X]

## Deploy
status: [success | failed | rolled back]
url: [deployed URL]
duration: [time]

## Post-Deploy Verification
- [ ] Health endpoint: [status]
- [ ] Key pages load: [status]
- [ ] Edge functions respond: [status]

## Rollback Plan
previous_commit: [sha]
rollback_command: [exact command to revert]
```

Update `.claude/pipeline/manifest.md` — mark deploy step complete with environment and URL.

## Phase 0A: User Interview

Ask these questions before starting. Wait for answers.

### Required Questions

**1. Target environment:**
"Where are you deploying? (staging, production, preview)"
→ Determines deploy command flags and verification rigor.

**2. What changed:**
"Is this a feature release, hotfix, or routine deploy?"
→ Hotfixes skip some pre-deploy checks. Feature releases get full checks.

**3. Pre-deploy confidence:**
"Have tests and QA passed? Any known issues going in?"
→ If the user says "tests are broken but ship it anyway", log the override and proceed with warning.

**4. Rollback comfort:**
"If something breaks, should I auto-rollback or notify you first?"
→ Determines rollback behavior.

**5. Notifications:**
"Should I notify anyone after deploy? (Slack webhook, email, etc.)"
→ Post-deploy communication.

### Store Preferences

Save to `.claude/pipeline/preferences.md` so future runs don't re-ask:
```
## Deploy Preferences
default_target: staging
rollback_behavior: notify-first
notification_webhook: [url or none]
```

On subsequent runs: "I have your deploy preferences from last time — deploying to staging, notify-first rollback. Still good?"

## Phase 1: Platform Detection

Detect the deploy platform from config files. Check ALL of these:

```bash
# Vercel
ls vercel.json .vercel/ 2>/dev/null && echo "VERCEL"
grep -q "vercel" package.json 2>/dev/null && echo "VERCEL_DEP"

# Netlify
ls netlify.toml 2>/dev/null && echo "NETLIFY"

# AWS (CDK, SAM, Amplify)
ls cdk.json samconfig.toml amplify/ 2>/dev/null && echo "AWS"

# Railway
ls railway.json .railway/ 2>/dev/null && echo "RAILWAY"

# Fly.io
ls fly.toml 2>/dev/null && echo "FLY"

# Heroku
ls Procfile 2>/dev/null && echo "HEROKU"

# Supabase
ls supabase/config.toml 2>/dev/null && echo "SUPABASE"
ls supabase/functions/ 2>/dev/null && echo "SUPABASE_FUNCTIONS"

# EAS (Expo Application Services)
ls eas.json 2>/dev/null && echo "EAS"
grep -q "expo" package.json 2>/dev/null && echo "EXPO"

# Docker
ls Dockerfile docker-compose.yml 2>/dev/null && echo "DOCKER"

# Firebase
ls firebase.json .firebaserc 2>/dev/null && echo "FIREBASE"

# GitHub Pages
grep -q "gh-pages" package.json 2>/dev/null && echo "GH_PAGES"
```

Present findings: "Detected: Vercel (web), Supabase (backend + edge functions), EAS (mobile). Which should I deploy?"

If nothing detected: "I couldn't detect a deploy platform. What are you using?"

## Phase 2: Pre-Deploy Checklist

Run ALL checks before deploying. Report results as a checklist.

### 2a. Tests

```bash
# Detect test runner
ls jest.config.* vitest.config.* 2>/dev/null
grep -q "\"test\"" package.json && npm test 2>&1 | tail -20

# Or specific runner
npx jest --no-coverage --passWithNoTests 2>&1 | tail -20
npx vitest run 2>&1 | tail -20
```

If tests fail: "X tests failing. Deploy anyway? (not recommended)"

### 2b. Lint & Type Check

```bash
# Lint
npx eslint . --quiet 2>&1 | tail -20

# TypeScript
npx tsc --noEmit 2>&1 | tail -20
```

### 2c. Environment Variables

```bash
# Check .env.example vs actual env
# Compare required vars against what's set
cat .env.example 2>/dev/null | grep -v "^#" | cut -d= -f1 | while read var; do
  [ -z "${!var}" ] && echo "MISSING: $var"
done

# Platform-specific env check
vercel env ls 2>/dev/null
supabase secrets list 2>/dev/null
```

### 2d. Database Migrations

```bash
# Supabase
ls supabase/migrations/*.sql 2>/dev/null | sort
supabase db diff 2>/dev/null  # Check for unapplied changes

# Prisma
npx prisma migrate status 2>/dev/null

# Drizzle
npx drizzle-kit check 2>/dev/null
```

If pending migrations: "Found unapplied migrations. Run them before deploying? [list migration files]"

### 2e. Build Test

```bash
# Verify the project builds before deploying
npm run build 2>&1 | tail -30
```

### Summary

Present checklist:
```
Pre-Deploy Checklist:
[PASS] Tests: 555/556 passing (1 known float precision issue)
[PASS] Lint: no errors
[PASS] TypeScript: no errors
[WARN] Env vars: NEXT_PUBLIC_POSTHOG_KEY not set (non-blocking)
[WARN] Migrations: 2 pending (blocked_users, source_trip_id)
[PASS] Build: successful

Proceed with deploy? (2 warnings noted)
```

## Phase 3: Deploy Execution

Execute the deploy for each detected platform.

### Vercel

```bash
# Preview deploy
vercel 2>&1

# Production deploy
vercel --prod 2>&1
```

### Supabase Edge Functions

```bash
# List functions
ls supabase/functions/*/index.ts 2>/dev/null

# Deploy all functions
supabase functions deploy --no-verify-jwt 2>&1

# Or deploy specific function
supabase functions deploy [function-name] 2>&1
```

### Supabase Migrations

```bash
supabase db push 2>&1
```

### EAS (Expo)

```bash
# Development build
eas build --platform ios --profile development 2>&1

# Preview
eas build --platform ios --profile preview 2>&1

# Production + submit
eas build --platform ios --profile production 2>&1
eas submit --platform ios 2>&1
```

### Netlify

```bash
netlify deploy 2>&1          # Preview
netlify deploy --prod 2>&1   # Production
```

### Fly.io

```bash
fly deploy 2>&1
```

### Railway

```bash
railway up 2>&1
```

### Multi-Platform Deploy

If multiple platforms detected (common: Vercel for web + Supabase for backend):

1. Deploy backend first (migrations, edge functions)
2. Deploy frontend second (it depends on backend)
3. Verify both

## Phase 4: Post-Deploy Verification

After deploy succeeds, verify the deployment is healthy.

### Health Checks

```bash
# Hit the deployed URL
curl -s -o /dev/null -w "%{http_code}" [DEPLOY_URL] 2>&1

# Check specific endpoints
curl -s [DEPLOY_URL]/api/health 2>&1
curl -s -o /dev/null -w "%{http_code}" [DEPLOY_URL]/api/[key-endpoint] 2>&1
```

### Edge Function Verification (Supabase)

```bash
# Test each edge function
curl -s -X POST [SUPABASE_URL]/functions/v1/[function-name] \
  -H "Authorization: Bearer [ANON_KEY]" \
  -H "Content-Type: application/json" \
  -d '{"test": true}' 2>&1
```

### Web Page Verification

```bash
# Check key pages return 200
for path in "/" "/explore" "/login"; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "[DEPLOY_URL]${path}")
  echo "$path: $STATUS"
done
```

### SSL/DNS Verification

```bash
# Check SSL cert
curl -vI https://[DOMAIN] 2>&1 | grep -i "ssl\|certificate\|expire"

# Check DNS
dig [DOMAIN] +short
nslookup [DOMAIN]
```

### Zero-Downtime Verification

```bash
# If the platform supports it, check for zero-downtime
# Vercel/Netlify: atomic deploys by default
# Fly.io: rolling deploys
# Custom: check if old instances are still serving during deploy
```

Report results:
```
Post-Deploy Verification:
[PASS] Homepage returns 200
[PASS] /explore returns 200
[PASS] API health endpoint: OK
[PASS] Edge function generate-itinerary: responds
[PASS] SSL certificate valid (expires 2027-01-15)
[FAIL] /api/webhooks: 500 — investigate
```

## Phase 5: Rollback Plan

Always document how to revert BEFORE deploying.

### Vercel

```bash
# List recent deployments
vercel ls 2>&1

# Rollback to previous
vercel rollback [deployment-url] 2>&1

# Or promote a previous deployment
vercel promote [deployment-url] 2>&1
```

### Supabase Edge Functions

```bash
# Redeploy previous version
git checkout [previous-commit] -- supabase/functions/
supabase functions deploy 2>&1
```

### EAS

```bash
# Mobile: can't rollback a published binary
# Use OTA updates (expo-updates) for JS-only changes
eas update --branch production --message "rollback to [version]" 2>&1
```

### Database Migration Rollback

```bash
# Supabase: write a reverse migration
# Document the exact SQL to undo each migration in the deploy report
```

### When to Rollback vs Hotfix

- **Rollback** if: error rate spiked, core flow broken, data corruption risk
- **Hotfix** if: minor bug, non-blocking, affects small percentage of users
- **Neither** if: cosmetic issue, can wait for next regular deploy

## Phase 6: Deploy Log

Record every deployment for history.

```markdown
## Deploy Log Entry

| Field | Value |
|-------|-------|
| Date | 2026-04-05T14:30:00Z |
| Commit | abc1234 |
| Branch | feature/photo-gallery |
| Platform | Vercel (web) + Supabase (functions) |
| Target | production |
| Pre-checks | all pass |
| Status | success |
| URL | https://usespots.com |
| Verified | homepage, explore, API health |
| Duration | 3m 42s |
| Deployed by | Chris via /deploy |
```

Append to `.claude/pipeline/features/[slug]/deploy-log.md` (or `.claude/pipeline/history/deploys.md` for non-feature deploys).

## Feature Flag Management

If feature flags detected in the codebase:

```bash
# Check for common feature flag patterns
grep -rn "featureFlag\|feature_flag\|FEATURE_\|isEnabled\|useFeatureFlag" src/ --include="*.ts" --include="*.tsx" | head -20

# Check for LaunchDarkly, Statsig, PostHog feature flags, Unleash
grep -rn "launchdarkly\|statsig\|posthog.*feature\|unleash" package.json 2>/dev/null
```

If found: "Feature flags detected. Should I enable/disable any flags as part of this deploy?"

## Multi-Environment Promotion

Support dev → staging → production promotion:

```
1. Deploy to staging
2. Run post-deploy verification on staging
3. Ask user: "Staging looks good. Promote to production?"
4. Deploy to production
5. Run post-deploy verification on production
6. Log both deployments
```

## Autonomous Mode

When running autonomously (invoked by /ship or a subagent):

- **Skip interview** if preferences exist in `.claude/pipeline/preferences.md`
- **Run all pre-deploy checks** — if any FAIL, STOP and report (do not auto-deploy with failures)
- **Deploy to staging only** — never auto-deploy to production
- **Run post-deploy verification** — report pass/fail
- **Write deploy-report.md** to the pipeline feature directory
- **Production promotion requires explicit human approval** — always

## Anti-Patterns

- DON'T deploy with failing tests unless the user explicitly overrides
- DON'T deploy to production without deploying to staging first (if staging exists)
- DON'T skip post-deploy verification — always check the deploy actually works
- DON'T deploy database migrations and code at the same time without ordering (migrations first)
- DON'T force-push or deploy from a dirty working tree without warning
- DON'T deploy without documenting the rollback plan
- DON'T auto-rollback without notifying the user (unless they configured auto-rollback)
