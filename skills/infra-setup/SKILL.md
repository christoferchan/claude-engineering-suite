---
name: infra-setup
description: Infrastructure setup and audit pipeline — detects existing infra (Supabase, Vercel, AWS, Firebase), audits secrets management, sets up monitoring and alerting, configures CI/CD, verifies environment parity, and estimates costs. Use when the user says "set up infrastructure", "check my env vars", "configure monitoring", "set up CI/CD", or wants to audit their infrastructure before launch.
---

# Infra Setup Skill

End-to-end infrastructure pipeline: Detect → Audit → Configure → Verify → Document.


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

- User says "set up infrastructure", "configure monitoring", "check my env vars"
- User is preparing for launch and needs infra verified
- User wants CI/CD pipeline set up
- User asks "is my infrastructure production-ready?"
- User wants to add a new environment (staging, production)
- User needs secrets audit before going public

## Pipeline Context Protocol

### On Start

1. Read `.claude/pipeline/manifest.md` if it exists — check if this is part of a larger pipeline
2. Read the project's CLAUDE.md — understand the declared tech stack and deploy config
3. Check `.claude/pipeline/features/[slug]/` for previous skill reports

### On Complete

Write report to `.claude/pipeline/features/[slug]/infra-report.md`:

```markdown
# Infrastructure Report — [date]

## Detected Infrastructure
| Service | Provider | Status | Notes |
|---------|----------|--------|-------|
| Database | Supabase | active | RLS enabled |
| Hosting (web) | Vercel | configured | not deployed |
| Mobile builds | EAS | configured | dev profile only |
| Error tracking | Sentry | partial | DSN not set in prod |
| Analytics | PostHog | partial | key not set |
| Auth | Supabase Auth | active | Apple + Google configured |
| Storage | Supabase Storage | active | 2 buckets |
| Edge functions | Supabase | active | 8 functions |

## Secrets Audit
[pass/fail per environment]

## Monitoring Status
[what's configured, what's missing]

## CI/CD Status
[pipeline exists / needs setup]

## Environment Parity
[dev vs staging vs prod comparison]

## Cost Estimate
[monthly estimate by service]

## Action Items
- [ ] [prioritized list of what to fix]
```

Update `.claude/pipeline/manifest.md` with infra status.

## Phase 0A: User Interview

### Required Questions

**1. What's the goal?**
"Are you setting up from scratch, auditing existing infra, or adding a new environment?"
→ Determines depth of work.

**2. Environments:**
"What environments do you have? (local, dev, staging, production)"
→ Determines what to verify.

**3. Budget constraints:**
"Any budget limits? Free tier only, or willing to pay for production services?"
→ Affects recommendations.

**4. Team size:**
"Just you, or does a team need access? (affects auth, permissions, CI/CD)"
→ Solo dev has different needs than a team.

**5. Launch timeline:**
"When are you launching? (determines urgency of setup)"
→ Prioritizes what to configure first.

**6. Compliance:**
"Any compliance requirements? (GDPR, HIPAA, SOC2, App Store guidelines)"
→ Affects data handling, logging, privacy.

### Store Preferences

Save to `.claude/pipeline/preferences.md`:
```
## Infra Preferences
environments: [local, staging, production]
budget: [free-tier | low | medium | high]
team_size: [solo | small | large]
compliance: [none | gdpr | hipaa]
```

## Phase 1: Infrastructure Detection

Scan the project to detect all infrastructure components.

### Backend / Database

```bash
# Supabase
ls supabase/config.toml 2>/dev/null && echo "SUPABASE"
grep -rn "SUPABASE_URL\|supabase" .env* 2>/dev/null | head -5
ls supabase/migrations/*.sql 2>/dev/null | wc -l
ls supabase/functions/*/index.ts 2>/dev/null | wc -l

# Firebase
ls firebase.json .firebaserc 2>/dev/null && echo "FIREBASE"

# Prisma (indicates SQL database)
ls prisma/schema.prisma 2>/dev/null && echo "PRISMA"

# Drizzle
ls drizzle.config.* 2>/dev/null && echo "DRIZZLE"

# AWS
ls cdk.json samconfig.toml amplify/ 2>/dev/null && echo "AWS"
grep -rn "aws-sdk\|@aws-sdk" package.json 2>/dev/null
```

### Hosting / Deploy

```bash
# Vercel
ls vercel.json .vercel/ 2>/dev/null && echo "VERCEL"

# Netlify
ls netlify.toml 2>/dev/null && echo "NETLIFY"

# Fly.io
ls fly.toml 2>/dev/null && echo "FLY"

# Railway
ls railway.json 2>/dev/null && echo "RAILWAY"

# EAS
ls eas.json 2>/dev/null && echo "EAS"

# Docker
ls Dockerfile docker-compose.yml 2>/dev/null && echo "DOCKER"

# GitHub Pages
grep -q "gh-pages" package.json 2>/dev/null && echo "GH_PAGES"
```

### Monitoring / Error Tracking

```bash
# Sentry
grep -rn "@sentry\|sentry" package.json 2>/dev/null
grep -rn "SENTRY_DSN\|NEXT_PUBLIC_SENTRY_DSN" .env* src/ web/ 2>/dev/null | head -5

# PostHog
grep -rn "posthog" package.json 2>/dev/null
grep -rn "POSTHOG_KEY\|NEXT_PUBLIC_POSTHOG_KEY" .env* src/ web/ 2>/dev/null | head -5

# LogRocket
grep -rn "logrocket" package.json 2>/dev/null

# Datadog
grep -rn "datadog\|dd-trace" package.json 2>/dev/null
```

### CI/CD

```bash
# GitHub Actions
ls .github/workflows/*.yml 2>/dev/null

# CircleCI
ls .circleci/config.yml 2>/dev/null

# GitLab CI
ls .gitlab-ci.yml 2>/dev/null

# Bitbucket Pipelines
ls bitbucket-pipelines.yml 2>/dev/null
```

### Auth

```bash
# Auth providers in code
grep -rn "signInWithOAuth\|signInWithApple\|GoogleSignin\|apple.*auth\|google.*auth" src/ --include="*.ts" --include="*.tsx" | head -10

# Auth config
grep -rn "auth\.\|Auth\." supabase/ 2>/dev/null | head -10
```

Present findings as a table:
```
Detected Infrastructure:
| Service | Provider | Detected From | Status |
|---------|----------|---------------|--------|
| Database | Supabase | supabase/config.toml | 14 migrations, 8 edge functions |
| Web hosting | Vercel | vercel.json | configured, not deployed |
| Mobile builds | EAS | eas.json | dev + preview profiles |
| Error tracking | Sentry | package.json | SDK installed, DSN not set |
| Analytics | PostHog | package.json | SDK installed, key not set |
| Auth | Supabase Auth | src/stores/authStore.ts | Apple + Google + email |
| CI/CD | None detected | — | needs setup |
```

## Phase 2: Secrets Audit

### Check for Leaked Secrets

```bash
# Secrets in git history (dangerous!)
git log --all --diff-filter=A -- "*.env" ".env*" 2>/dev/null | head -10

# Check .gitignore covers secrets
grep -q "\.env" .gitignore && echo "GITIGNORE_OK" || echo "GITIGNORE_MISSING"
grep -q "\.env\.local" .gitignore && echo "ENV_LOCAL_OK"

# Check for hardcoded secrets in source
grep -rn "sk_live\|sk_test\|api_key.*=.*['\"]" src/ --include="*.ts" --include="*.tsx" | head -10
grep -rn "password.*=.*['\"]" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -10
grep -rn "secret.*=.*['\"]" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -10

# Check for .env files that shouldn't exist
ls .env .env.local .env.production .env.staging 2>/dev/null
```

### Environment Variable Audit

```bash
# Find all env var references in code
grep -rn "process\.env\.\|import\.meta\.env\.\|NEXT_PUBLIC_\|EXPO_PUBLIC_" src/ web/ --include="*.ts" --include="*.tsx" | grep -oP "[A-Z_]+" | sort -u

# Compare against .env.example
cat .env.example 2>/dev/null | grep -v "^#" | cut -d= -f1 | sort

# Check which vars are set vs missing
```

Report:
```
Secrets Audit:
[PASS] .gitignore covers .env files
[PASS] No secrets in source code
[WARN] .env committed to git history (commit abc123) — recommend rotating
[FAIL] NEXT_PUBLIC_SENTRY_DSN not set in production
[FAIL] NEXT_PUBLIC_POSTHOG_KEY not set in production
[PASS] Supabase keys properly in .env.local
```

## Phase 3: Monitoring Setup

### Error Tracking (Sentry)

If Sentry SDK detected but not configured:

```markdown
## Sentry Setup Checklist
- [ ] Create Sentry project at sentry.io
- [ ] Set SENTRY_DSN / NEXT_PUBLIC_SENTRY_DSN in environment
- [ ] Verify Sentry.init() is called on app start
- [ ] Test: throw a test error, verify it appears in Sentry
- [ ] Configure alert rules: notify on new errors, error rate spikes
- [ ] Set up release tracking: tag deploys with git commit
```

### Analytics (PostHog)

If PostHog SDK detected but not configured:

```markdown
## PostHog Setup Checklist
- [ ] Create PostHog project
- [ ] Set NEXT_PUBLIC_POSTHOG_KEY in environment
- [ ] Verify posthog.init() is called on app start
- [ ] Test: trigger a test event, verify it appears in PostHog
- [ ] Set up key funnels: onboarding, core feature usage, conversion
- [ ] Configure session recording (web) if desired
```

### Uptime Monitoring

```markdown
## Uptime Monitoring Recommendations
| What to Monitor | Check Interval | Alert Threshold |
|----------------|----------------|-----------------|
| Homepage (web) | 1 min | 2 consecutive failures |
| API health endpoint | 1 min | 2 consecutive failures |
| Edge functions | 5 min | 3 consecutive failures |
| Database | 5 min | 1 failure |

Recommended tools: UptimeRobot (free), Better Uptime, Pingdom
```

### Alerting Rules

```markdown
## Suggested Alert Rules

### CRITICAL (page immediately)
- Error rate > 5% of requests
- API response time p95 > 5s
- Database connection failures
- Auth service down

### WARNING (notify within 1 hour)
- Error rate > 1% of requests
- API response time p95 > 2s
- Storage usage > 80%
- Edge function cold starts > 3s

### INFO (daily digest)
- New error types seen
- Unusual traffic patterns
- Approaching free tier limits
```

## Phase 4: CI/CD Pipeline Setup

If no CI/CD detected, generate a GitHub Actions workflow:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [master, main]
  pull_request:
    branches: [master, main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm test -- --no-coverage

  # Add deploy job if platform supports it
  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/master'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Platform-specific deploy steps
```

Customize based on detected platform:
- **Vercel**: use `vercel-action` or Vercel's native GitHub integration
- **Supabase**: add `supabase db push` and `supabase functions deploy` steps
- **EAS**: add `eas build` step for mobile
- **Netlify**: use `netlify-cli` deploy

## Phase 5: Environment Parity

Verify all environments have the same configuration structure.

```markdown
## Environment Parity Check

| Variable | Local | Staging | Production |
|----------|-------|---------|------------|
| SUPABASE_URL | set | set | set |
| SUPABASE_ANON_KEY | set | set | set |
| SENTRY_DSN | set | missing | missing |
| POSTHOG_KEY | — | — | missing |
| GOOGLE_MAPS_API_KEY | set | — | missing |

| Config | Local | Staging | Production |
|--------|-------|---------|------------|
| RLS enabled | yes | yes | yes |
| Auth providers | 3 | 3 | 3 |
| Edge functions | 8 | 8 | ? |
| Storage buckets | 2 | 2 | 2 |
```

Flag mismatches: "Production is missing SENTRY_DSN and POSTHOG_KEY. These are in your P0 checklist."

## Phase 6: Database & Backup Verification

### Supabase

```markdown
## Database Health
- [ ] RLS enabled on all tables
- [ ] Automated backups: [enabled/disabled] (Pro plan required)
- [ ] Point-in-time recovery: [enabled/disabled]
- [ ] Database size: [current] / [limit]
- [ ] Connection pooling: [enabled/disabled]

## RLS Audit
| Table | SELECT | INSERT | UPDATE | DELETE |
|-------|--------|--------|--------|--------|
| spots | public read | auth | owner | owner |
| trips | owner+collab | auth | owner | owner |
| ...
```

### Migration Status

```bash
# List all migrations
ls supabase/migrations/*.sql 2>/dev/null | sort

# Check which are applied
supabase migration list 2>/dev/null
```

## Phase 7: SSL / DNS Verification

```bash
# Check SSL
curl -vI https://[domain] 2>&1 | grep -E "SSL|certificate|expire|issuer"

# Check DNS
dig [domain] +short
dig www.[domain] +short

# Check HTTPS redirect
curl -sI http://[domain] | head -5

# Check HSTS
curl -sI https://[domain] | grep -i strict-transport
```

Report:
```
SSL/DNS:
[PASS] SSL certificate valid (expires 2027-01-15)
[PASS] DNS resolves to Vercel
[PASS] HTTP → HTTPS redirect active
[WARN] HSTS header not set (recommended)
```

## Phase 8: Cost Estimation

```markdown
## Monthly Cost Estimate

| Service | Tier | Monthly Cost | Usage Limit |
|---------|------|-------------|-------------|
| Supabase | Free | $0 | 500MB DB, 1GB storage, 2M edge invocations |
| Vercel | Free | $0 | 100GB bandwidth, 100 deploys |
| EAS | Free | $0 | 30 builds/month |
| Sentry | Free | $0 | 5K errors/month |
| PostHog | Free | $0 | 1M events/month |
| Domain | — | ~$12/year | — |
| **Total** | | **~$1/month** | |

### When to Upgrade
- Supabase Pro ($25/mo): when you need >500MB DB, daily backups, or >2M edge calls
- Vercel Pro ($20/mo): when you need >100GB bandwidth or advanced analytics
- EAS Production ($99/mo): when you need priority builds or >30 builds
```

## Autonomous Mode

When running autonomously (invoked by /ship or a subagent):

- **Skip interview** if preferences exist in `.claude/pipeline/preferences.md`
- **Run full detection and audit** — every phase
- **Do NOT create resources** (Sentry projects, PostHog projects) — only audit and recommend
- **Do NOT modify environment variables** — report what's missing, let the user set them
- **Generate CI/CD config file** but do NOT commit it — present for review
- **Write infra-report.md** to the pipeline feature directory
- **Flag CRITICAL issues** (secrets in git, RLS disabled, no backups) for immediate human attention

## Anti-Patterns

- DON'T set environment variables without user confirmation — you might overwrite production values
- DON'T commit .env files to git — ever
- DON'T assume free tier limits are sufficient for production
- DON'T skip the RLS audit for Supabase — an open table is a security incident waiting to happen
- DON'T set up monitoring without alerting — monitoring you don't check is useless
- DON'T create duplicate CI/CD pipelines — check what exists first
- DON'T recommend paid services without noting the free tier alternative
