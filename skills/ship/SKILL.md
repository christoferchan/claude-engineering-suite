---
name: ship
description: Pipeline orchestrator — coordinates all engineering skills (design-audit, qa-test, security-audit, perf-audit, deploy, etc.) to ship features end-to-end. Detects intent, picks the right skills in the right order, manages context between them, and tracks pipeline state. Use when the user says "ship this", "build this feature", "release", or wants the full pipeline.
---

# Ship — Pipeline Orchestrator

Coordinates all engineering skills to ship features end-to-end.

## When to Use

- User says "ship this", "build this feature", "do a release"
- User wants the full quality pipeline, not just one skill
- User is starting a new feature and wants structured execution
- User wants to know "what's left before we can ship?"

## Pipeline Context Protocol

All skills in the suite communicate through files in `.claude/pipeline/`.

### Directory Structure

```
.claude/pipeline/
├── manifest.md                    # Current pipeline state (the orchestrator's brain)
├── preferences.md                 # User's stored preferences across all skills
├── features/
│   ├── [feature-slug]/
│   │   ├── spec.md               # Product spec (from /product-spec)
│   │   ├── design-audit.md       # Design audit report
│   │   ├── qa-report.md          # QA test report
│   │   ├── security-report.md    # Security findings
│   │   ├── perf-report.md        # Performance report
│   │   ├── analytics-report.md   # Analytics coverage
│   │   ├── deploy-log.md         # Deploy history
│   │   ├── changelog.md          # Auto-generated changelog
│   │   └── screenshots/          # Before/after from audits
│   └── ...
└── history/
    ├── [date]-session.md          # Daily session summaries
    └── ...
```

### Manifest Format

```markdown
# Pipeline Manifest

## Current Feature
name: photo-gallery
slug: photo-gallery
branch: feature/photo-gallery
started: 2026-04-07T10:00:00Z
status: qa-testing

## Pipeline Progress
- [x] product-spec — completed 2026-04-07T10:15:00Z
- [x] feature-dev — completed 2026-04-07T11:30:00Z — 3 files changed
- [x] design-audit — completed 2026-04-07T11:45:00Z — 2 issues, both fixed
- [ ] qa-test — IN PROGRESS
- [ ] security-audit — pending
- [ ] perf-audit — pending
- [ ] analytics-audit — pending
- [ ] deploy — pending

## Context for Next Skill
- Feature: photo gallery on spot detail page
- Files changed: SpotPhotoGallery.tsx, SpotDetail.tsx, useSpotPhotos.ts
- Design audit: found dark mode placeholder issue (fixed)
- QA focus areas: upload flow, permission handling, empty state

## Blockers
(none)

## Decision Log
- 2026-04-07T10:05:00Z — User chose: gallery grid layout, not horizontal scroll
- 2026-04-07T11:40:00Z — Design audit: user approved conservative fix for placeholder
```

### Context Protocol Rules

Every skill in the suite MUST:

1. **Read manifest.md at start** — understand what's happening, what came before
2. **Read the feature directory** — check previous skill reports for context
3. **Write its report** to the feature directory when done
4. **NOT modify other skills' reports** — append-only per skill
5. **Update manifest.md** — mark its step complete, add context for next skill

The orchestrator:
- Creates the feature directory and initial manifest
- Calls each skill in order
- Updates manifest between steps
- Git commits after each phase
- Stops at human approval gates

## Skill Discovery

At the start of every pipeline run, discover which skills are installed:

```bash
# Scan for installed skills
INSTALLED_SKILLS=""
for dir in ~/.claude/skills/*/; do
  if [ -f "$dir/SKILL.md" ]; then
    name=$(basename "$dir")
    [ "$name" != "ship" ] && INSTALLED_SKILLS="$INSTALLED_SKILLS $name"
  fi
done
echo "Installed skills:$INSTALLED_SKILLS"
```

### Adaptive Pipeline

The orchestrator only calls skills that are installed. If a skill is missing, skip it — don't fail.

**Full suite installed (all 15):**
```
spec → dev → design-audit → qa-test → security-audit → perf-audit → analytics-audit → deploy
```

**Minimal install (just design-audit + qa-test):**
```
dev → design-audit → qa-test → (done — no deploy skill installed)
```

**Custom mix (design-audit + security-audit + deploy):**
```
dev → design-audit → security-audit → deploy
```

### Required vs Optional Skills

Some skills are required for certain pipeline types:

| Pipeline | Required | Optional (run if installed) |
|----------|----------|---------------------------|
| New Feature | (none — all optional) | product-spec, design-audit, qa-test, security-audit, perf-audit, analytics-audit, deploy |
| Bug Fix | (none) | qa-test, deploy |
| Release | (none) | qa-test, security-audit, deploy |
| Audit Only | (none) | design-audit, qa-test, security-audit, perf-audit, analytics-audit |

If zero skills are installed besides `/ship`, tell the user:
> "No engineering skills installed. Install the full suite or individual skills:
> ```
> # Full suite
> git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills
> 
> # Or cherry-pick
> git clone https://github.com/christoferchan/design-audit-skill.git ~/.claude/skills/design-audit
> git clone https://github.com/christoferchan/qa-test-skill.git ~/.claude/skills/qa-test
> ```"

### Show Installed Skills

When the user invokes `/ship`, always show what's available:
> "Pipeline ready. Installed skills:
> ✅ design-audit, qa-test, security-audit, deploy
> ❌ Not installed: perf-audit, analytics-audit, product-spec, ...
> 
> I'll run: design-audit → qa-test → security-audit → deploy
> Missing skills will be skipped. Install more with `git clone`."

## Intent Detection

When the user invokes `/ship`, detect what they want:

### New Feature
Trigger: "build X", "add X", "implement X", "I want X"
Pipeline: spec → dev → design-audit → qa-test → security → perf → analytics → deploy

### Bug Fix
Trigger: "fix X", "X is broken", "bug in X"
Pipeline: dev → qa-test (focused) → deploy (hotfix)

### Release / Ship
Trigger: "ship it", "release", "deploy", "push to production"
Pipeline: qa-test (regression) → security → perf → deploy

### Refactor
Trigger: "refactor X", "clean up X", "improve X"
Pipeline: dev → qa-test (regression) → perf → deploy

### Audit Only
Trigger: "check everything", "full audit", "pre-release check"
Pipeline: design-audit → qa-test → security → perf → analytics

### Content / Data
Trigger: "seed data for X", "write copy for X", "App Store prep"
Pipeline: data-seed / content-copy / app-store (as needed)

Ask the user if intent is ambiguous: "Are you building a new feature, fixing a bug, or preparing a release?"

## Project Bootstrap (existing projects)

When `/ship` runs on a project for the first time, detect and create what's missing:

### Step 1: Check project essentials (search by PURPOSE, not filename)

Projects name things differently. Search for the concept, not the exact file.

**Pipeline directory:**
```bash
ls .claude/pipeline/manifest.md 2>/dev/null || echo "NO_PIPELINE"
```

**Design system / tokens / theme (any of these count):**
```bash
# React Native
find . -path ./node_modules -prune -o -name "theme.*" -print -o -name "tokens.*" -print -o -name "colors.*" -print -o -name "design-system.*" -print -o -name "designTokens.*" -print 2>/dev/null | head -5
# Tailwind
find . -name "tailwind.config.*" 2>/dev/null | head -1
# CSS variables / custom properties
grep -rl ":root" --include="*.css" --include="*.scss" . 2>/dev/null | head -3
# Styled Components / Emotion theme
grep -rl "ThemeProvider\|createTheme\|theme:" src/ --include="*.ts" --include="*.tsx" --include="*.js" 2>/dev/null | head -3
# Flutter
find . -name "*.dart" -exec grep -l "ThemeData" {} \; 2>/dev/null | head -3
# SwiftUI
find . -name "*.swift" -exec grep -l "Color(\|\.foregroundColor\|\.background" {} \; 2>/dev/null | head -3
# Android
find . -name "themes.xml" -o -name "colors.xml" -o -name "styles.xml" 2>/dev/null | head -3
# Rails / Django — CSS/SCSS
find . -name "application.css" -o -name "application.scss" -o -name "base.css" -o -name "variables.scss" -o -name "_variables.scss" 2>/dev/null | head -3
```
If ANY of these return results → `HAS_THEME=true`. Store the path.

**Project documentation (any of these count):**
```bash
ls CLAUDE.md claude.md .claude/CLAUDE.md 2>/dev/null  # Claude Code
ls CONTRIBUTING.md DEVELOPMENT.md ARCHITECTURE.md 2>/dev/null  # Standard docs
ls docs/design-system.md docs/style-guide.md 2>/dev/null  # Design docs
ls .cursor/rules .windsurf/rules 2>/dev/null  # Other AI tools
```
If none exist → generate CLAUDE.md. If non-Claude docs exist, READ them first to understand existing conventions before generating CLAUDE.md.

**Test infrastructure (any of these count):**
```bash
ls jest.config.* vitest.config.* karma.conf.* .mocharc.* pytest.ini setup.cfg tox.ini 2>/dev/null  # Config files
ls spec/ test/ tests/ __tests__/ cypress/ e2e/ 2>/dev/null  # Test directories
find . -name "*.test.*" -o -name "*.spec.*" -o -name "*_test.*" 2>/dev/null | head -3  # Test files
ls pubspec.yaml 2>/dev/null && grep -q "flutter_test" pubspec.yaml && echo "FLUTTER_TESTS"  # Flutter
ls Gemfile 2>/dev/null && grep -q "rspec\|minitest" Gemfile && echo "RUBY_TESTS"  # Rails
```

**CI/CD (any of these count):**
```bash
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null  # GitHub Actions
ls .gitlab-ci.yml 2>/dev/null  # GitLab CI
ls Jenkinsfile 2>/dev/null  # Jenkins
ls .circleci/config.yml 2>/dev/null  # CircleCI
ls bitbucket-pipelines.yml 2>/dev/null  # Bitbucket
ls .travis.yml 2>/dev/null  # Travis
ls vercel.json netlify.toml fly.toml railway.json 2>/dev/null  # Platform-specific
ls Procfile 2>/dev/null  # Heroku
ls Dockerfile docker-compose.yml 2>/dev/null  # Docker
```

**Git safety:**
```bash
ls .gitignore 2>/dev/null || echo "NO_GITIGNORE"
# Check secrets aren't committed regardless of .gitignore name
git ls-files | grep -iE "\.env$|\.env\.local|credentials|secret" | head -5
```

### How to handle "similar but named differently"

The bootstrap should NEVER say "theme.ts not found" if the project has `tailwind.config.js` serving the same purpose. The check is:

**"Does this project have centralized design tokens?"** — not "does theme.ts exist?"
**"Does this project have test infrastructure?"** — not "does jest.config.js exist?"
**"Does this project have CI/CD?"** — not "does .github/workflows exist?"

When a file is found with a non-standard name, log it:
> "Found design tokens in `src/styles/variables.scss` (not the typical `theme.ts`). I'll use this as the design system source of truth."

When generating CLAUDE.md for a project that has existing conventions in differently-named files, reference those files:
```markdown
## Design System
- Tokens defined in: `src/styles/variables.scss`
- Component library: `src/ui/` (12 components)
- Test config: `vitest.config.ts`
```

### Step 2: Create what's missing

**If no `.claude/pipeline/`:**
```bash
mkdir -p .claude/pipeline/features .claude/pipeline/history
```
Create `manifest.md` with initial state.

**If no `CLAUDE.md`:**
Generate one by scanning the codebase:
- Detect framework, language, directory structure
- Read existing README.md for project context
- Scan component patterns (button variants, card styles, input styles)
- Scan color usage (extract palette from existing code)
- Scan spacing patterns
- Document all findings as design system rules
- Write to `CLAUDE.md`

Tell the user: "I created a CLAUDE.md with design system rules based on your codebase. Review it — this becomes the source of truth for all audits."

**If no theme file:**
Delegate to `/design-audit` Phase 0B which handles theme creation per framework (React Native, Tailwind, CSS variables, etc.)

**If no test config:**
Note in manifest: "No test infrastructure detected. /qa-test will generate test config if invoked."

**If no CI:**
Note in manifest: "No CI pipeline detected. /deploy can generate GitHub Actions workflow if invoked."

**If no `.gitignore` or missing entries:**
Check that these are gitignored:
- `.env`, `.env.local` (secrets)
- `node_modules/` (dependencies)
- `.claude/pipeline/` (optional — some teams want this tracked, some don't)

### Step 3: First-run summary

Present to user:
> "First time running /ship on this project. Here's what I found:
> 
> ✅ Exists: React Native + Expo, Supabase backend, 287 testIDs
> ✅ Created: .claude/pipeline/, CLAUDE.md (review this)
> ❌ Missing: CI pipeline, test config (will create when needed)
> 
> Installed skills: design-audit, qa-test, security-audit, deploy
> 
> Ready to proceed?"

### Subsequent runs

On future runs, skip bootstrap. Just read manifest.md and continue:
> "Welcome back. Last session: shipped photo-gallery feature.
> What are we working on today?"

## Edge Case Guards

Before executing the pipeline, detect and handle these edge cases. Check ALL of them on every run.

### 1. Monorepo Detection

```bash
# Check for monorepo indicators
ls lerna.json turbo.json nx.json pnpm-workspace.yaml 2>/dev/null
ls packages/ apps/ services/ 2>/dev/null
grep -q "workspaces" package.json 2>/dev/null
```

If monorepo detected:

**1. Map the workspace:**
```bash
# Find all apps/packages
find packages/ apps/ services/ -maxdepth 1 -name "package.json" 2>/dev/null | while read f; do
  dir=$(dirname "$f")
  name=$(grep -o '"name": "[^"]*"' "$f" | head -1 | cut -d'"' -f4)
  # Detect type
  type="unknown"
  grep -q "react-native\|expo" "$f" && type="mobile"
  grep -q "next\|nuxt\|vite\|react-dom" "$f" && type="web"
  grep -q "express\|fastify\|hono\|koa" "$f" && type="api"
  # Check if it's a shared lib (imported by others, not runnable)
  grep -q '"main"\|"exports"' "$f" && ! grep -q '"start"\|"dev"' "$f" && type="library"
  echo "$dir | $name | $type"
done
```

**2. Ask the user:**
> "This is a monorepo. I found:
> 
> **Apps:**
> 1. `packages/web` — Next.js (web)
> 2. `packages/mobile` — React Native (mobile)
> 3. `packages/api` — Express (api)
> 
> **Shared libraries:**
> - `packages/ui` — shared component library
> - `packages/utils` — shared utilities
> 
> Options:
> 1. **Single app** — target one app (fastest)
> 2. **All apps** — audit each app + shared code (thorough)
> 3. **Changed apps only** — detect which apps are affected by recent changes"

**3. Scope handling:**

**Single app mode:**
```markdown
## Manifest
scope: packages/web
scope_type: single
shared_dependencies: packages/ui, packages/utils
```
- All skills scan `packages/web/` as the root
- Also scan shared dependencies (`packages/ui/`) since changes there affect the target app
- Maestro/Playwright runs against this app only
- Reports go to `.claude/pipeline/features/[slug]/web/`

**All apps mode:**
```markdown
## Manifest  
scope: all
apps:
  - path: packages/web, type: web, platform: playwright
  - path: packages/mobile, type: mobile, platform: maestro
  - path: packages/api, type: api, platform: none
shared:
  - packages/ui
  - packages/utils
```
- Skills run per-app in sequence (or parallel if independent)
- Each app gets its own report subdirectory
- Shared code gets scanned once, findings attributed to all apps
- Combined summary at the end: "Web: 3 issues, Mobile: 5 issues, API: 1 issue, Shared: 2 issues"

**Changed apps only mode:**
```bash
# Detect which packages changed
git diff --name-only HEAD~5 | cut -d'/' -f1-2 | sort -u
# Map to affected apps via dependency graph
# If packages/ui changed → both web and mobile are affected
```
Only run skills on affected apps. This is the fastest for CI.

**4. Shared library handling:**

When a shared library (`packages/ui`) changes:
- Run `/design-audit` on every app that imports from it (they might render differently)
- Run `/qa-test` on every app (shared component behavior may differ per platform)
- Run `/security-audit` once on the library itself (no need per-app)
- Run `/perf-audit` per-app (bundle impact differs)

Detect dependencies:
```bash
# Which apps import from packages/ui?
grep -rl "from.*packages/ui\|@acme/ui" packages/web/src packages/mobile/src 2>/dev/null
```

**5. Cross-app consistency checks:**

For monorepos with web + mobile sharing a component library:
- Do the same components render consistently across platforms?
- Are design tokens (colors, spacing) identical or intentionally different?
- Are API contracts the same (both apps hit the same endpoints)?

Add a cross-app section to the report:
```markdown
## Cross-App Consistency
| Component | Web | Mobile | Match? |
|-----------|-----|--------|--------|
| Button primary | teal, 12px radius | teal, 12px radius | ✅ |
| Card spacing | 24px padding | 20px padding | ❌ Mismatch |
```

### 2. Docker / Container Development

```bash
# Check if app runs in Docker
ls Dockerfile docker-compose.yml docker-compose.yaml 2>/dev/null
docker ps 2>/dev/null | grep -v CONTAINER | head -5
```

If Docker detected:
- Check if ports are mapped: `docker ps --format '{{.Ports}}'`
- Try both `localhost` and `host.docker.internal`
- For Maestro/Playwright: the app must be accessible from the HOST, not inside the container
- If app is only in container: "App runs in Docker. Maestro needs the app on the host. Either expose the port (`-p 3000:3000`) or run the app natively for testing."

### 3. Windows / WSL / Linux (No iOS Simulator)

```bash
# Detect OS
OS="unknown"
[[ "$OSTYPE" == "darwin"* ]] && OS="macos"
[[ "$OSTYPE" == "linux"* ]] && OS="linux"
[[ "$OSTYPE" == "msys" || "$OSTYPE" == "cygwin" ]] && OS="windows"
grep -q Microsoft /proc/version 2>/dev/null && OS="wsl"
```

| OS | iOS Simulator | Android Emulator | Web (Playwright) |
|----|--------------|-----------------|-------------------|
| macOS | ✅ | ✅ | ✅ |
| Linux | ❌ | ✅ | ✅ |
| Windows | ❌ | ✅ | ✅ |
| WSL | ❌ | ❌ (complex) | ✅ |

If no iOS simulator:
> "You're on [Linux/Windows/WSL]. iOS simulator isn't available.
> Options:
> 1. Test Android via emulator (if installed)
> 2. Test web via Playwright (if web version exists)
> 3. Skip mobile capture — generate manual device-test checklist instead
> 
> Which approach?"

Never assume macOS. Never assume iOS simulator exists.

### 4. Linter / Formatter Pre-commit Hooks

```bash
# Check for pre-commit hooks
ls .husky/ .git/hooks/pre-commit 2>/dev/null
grep -q "lint-staged\|prettier\|eslint" package.json 2>/dev/null
cat .husky/pre-commit 2>/dev/null | head -5
```

If formatters exist:
- After writing any code, run the formatter BEFORE committing: `npx prettier --write [file]` or `npx eslint --fix [file]`
- For pipeline commits (manifest.md, reports), these are markdown — formatters usually don't touch them
- If a pre-commit hook REJECTS a pipeline commit, tell the user: "Pre-commit hook rejected the commit. You may need to add `.claude/pipeline/` to your lint-staged ignore."
- Never use `--no-verify` to skip hooks unless the user explicitly asks

### 5. Protected Branches

```bash
# Check current branch
BRANCH=$(git branch --show-current)
# Check if branch has protection (can't detect remotely, but can check common patterns)
[[ "$BRANCH" == "main" || "$BRANCH" == "master" || "$BRANCH" == "production" ]] && echo "PROTECTED_BRANCH"
```

If on a protected branch:
> "You're on `main` which may have branch protection. Pipeline commits might fail.
> Options:
> 1. Create a feature branch first: `git checkout -b feature/[slug]`
> 2. Skip pipeline commits (just run skills, don't track in git)
> 3. Continue anyway (if branch isn't actually protected)"

For pipeline commits, prefer a feature branch. Merge pipeline artifacts with the feature code.

### 6. Production Database Safety

```bash
# Check which database the app is pointing to
grep -i "DATABASE_URL\|SUPABASE_URL\|MONGO_URI\|FIREBASE_PROJECT" .env .env.local 2>/dev/null
# Check for production indicators in the URL
grep -iE "prod|production|live|\.com" .env 2>/dev/null | grep -iE "DATABASE\|SUPABASE\|MONGO\|FIREBASE"
```

**CRITICAL GUARD:** Before ANY skill that writes data (`/data-seed`, `/db-migrate`, `/qa-test` destructive tests):

> "⚠️ Your database URL appears to be PRODUCTION:
> `https://pqnrqlrapypsqbfajyrq.supabase.co`
> 
> Data-modifying operations could affect real users.
> Options:
> 1. Switch to a dev/staging database first
> 2. Proceed with READ-ONLY operations (audits only, no writes)
> 3. I understand the risk, proceed anyway
> 
> I will NOT write to a production database without explicit confirmation."

For `/data-seed`: ALWAYS ask, even if the URL doesn't look like production.
For `/db-migrate`: Show the migration SQL and require explicit "run it" before executing.
For `/qa-test` destructive tests: Skip destructive scenarios on production.

### 7. Context Window Management

For large projects, the combined manifest + reports + screenshots can exceed context limits.

**Mitigation strategies:**
- **Summarize, don't dump:** When passing context between skills, pass a 50-line summary, not the full 500-line report
- **Lazy loading:** Skills read only the sections of manifest.md they need, not the whole file
- **Truncate history:** Only keep the last 3 feature pipelines in manifest. Archive older ones to `history/`
- **Screenshot references:** Store screenshot file paths, don't try to read 100 screenshots in one turn
- **Chunk large reports:** If a QA report has 50 test results, pass the summary ("38 passed, 12 failed") and the failures only

**Hard limits:**
- Manifest.md should stay under 200 lines
- Each skill report should stay under 300 lines
- Feature directory should be archived when the feature ships
- Never read more than 20 screenshots in one skill invocation

### 8. Concurrent Users / Parallel Runs

```bash
# Check for lock file
ls .claude/pipeline/.lock 2>/dev/null
```

Before starting a pipeline:
- Create a lock file: `.claude/pipeline/.lock` with timestamp + user
- If lock exists and is < 1 hour old: "Another pipeline run is in progress (started [time]). Wait or force unlock?"
- If lock exists and is > 1 hour old: stale lock, auto-remove
- Remove lock when pipeline completes or is cancelled

For skills running in parallel (design-audit + qa-test + security-audit):
- Each writes to its OWN report file — no conflicts
- Only the orchestrator writes to manifest.md — no parallel writes
- Use atomic file writes (write to .tmp, then rename)

### 9. Non-English Codebases

```bash
# Detect primary language of comments/strings
# Sample first 100 lines with user-facing text
grep -rn "label=\|title=\|text=" src/ --include="*.tsx" --include="*.vue" | head -20
```

If non-ASCII text detected:
- `/content-copy`: note the primary language, don't flag non-English strings as "inconsistent"
- `/qa-test`: edge case data should include the app's language characters, not just ASCII
- `/design-audit`: text overflow checks are MORE important (CJK characters have different widths)
- Don't assume English. Ask: "I detected [Japanese/Chinese/etc] strings. Is this the primary language?"

### 10. Rate Limiting on External APIs

When running skills in parallel, they may all hit the same APIs:

**Mitigation:**
- Stagger parallel skill starts by 5 seconds
- If any API returns 429 (rate limit), back off and retry after the `Retry-After` header value
- For Google Places API: cap at 10 requests per skill invocation
- For Supabase: respect the connection pool limit
- For Anthropic API (Scout/itinerary): only one skill should call AI endpoints at a time

**For autonomous/CI mode:**
- Add explicit delays between API-heavy operations
- Set a global rate limit: max 30 external API calls per pipeline run
- Log all API calls for cost tracking

### 11. Git Submodules / Nested Repos

```bash
# Check for submodules
ls .gitmodules 2>/dev/null
# Check for nested .git directories
find . -name ".git" -not -path "./.git" -type d 2>/dev/null | head -5
```

If submodules detected:
- Only scan the main repository, not submodules
- Note submodules in the manifest: "Submodules detected: [list]. These are scanned separately."
- Don't count submodule files in coverage metrics
- Don't modify files in submodules

If nested repos detected (not submodules):
- Ask user which is the main project
- Scope all skills to the selected root

### 12. Offline / No Internet

```bash
# Quick connectivity check
curl -s --max-time 3 https://api.github.com > /dev/null 2>&1 && echo "ONLINE" || echo "OFFLINE"
```

If offline:
- Skills that need internet: `/data-seed` (Google Places), `/security-audit` (npm audit), `/analytics-audit` (web search), `/app-store` (keyword research)
- Skills that work offline: `/design-audit` (screenshots are local), `/qa-test` (Maestro is local), `/perf-audit` (bundle analysis is local), `/db-migrate` (SQL generation is local)
- Mark internet-dependent operations as SKIPPED, not FAILED
- Tell user: "No internet detected. Running offline-capable skills only. These will be skipped: [list]"

### Edge Case Summary

Run ALL checks at pipeline start. Report status:

```
Edge Case Checks:
  Monorepo:         ✅ Single app (not monorepo)
  Docker:           ✅ Not Docker (native dev)
  OS:               ✅ macOS — iOS simulator available
  Pre-commit hooks: ⚠️ Husky + Prettier detected — will format before commits
  Branch:           ✅ On feature/photo-gallery (not protected)
  Database:         ⚠️ Looks like production URL — will ask before writes
  Context size:     ✅ Project is small/medium
  Concurrent:       ✅ No lock file — single user
  Language:         ✅ English
  Rate limits:      ✅ Will stagger parallel API calls
  Submodules:       ✅ None
  Internet:         ✅ Online
```

## Pipeline Execution

### Phase 1: Initialize

```bash
# Create feature directory
mkdir -p .claude/pipeline/features/[slug]

# Create initial manifest
cat > .claude/pipeline/manifest.md << 'EOF'
# Pipeline Manifest
## Current Feature
name: [feature name]
slug: [slug]
branch: [current git branch]
started: [timestamp]
status: initializing
EOF

# Git commit
git add .claude/pipeline/
git commit -m "pipeline: initialize [feature-name]"
```

### Phase 2: Execute Skills in Order

For each skill in the pipeline:

1. **Update manifest** — mark skill as IN PROGRESS
2. **Invoke the skill** — use the Skill tool or describe the task
3. **Skill reads manifest** — gets context from previous skills
4. **Skill executes** — runs its full workflow
5. **Skill writes report** — saves to feature directory
6. **Update manifest** — mark skill complete, add context for next
7. **Git commit** — `pipeline: [skill-name] complete for [feature]`

### Phase 3: Human Gates

Stop for human approval at these points:

| Gate | When | What user reviews |
|------|------|-------------------|
| **Spec approval** | After product-spec | "Is this what you want to build?" |
| **Implementation review** | After feature-dev | "Does the code look right?" |
| **Quality gate** | After design-audit + qa-test | "Are the remaining issues acceptable?" |
| **Ship decision** | After all audits pass | "Ready to deploy?" |

Between gates, skills run without stopping (unless they find a CRITICAL issue).

### Phase 4: Completion

When all skills complete and user approves:

```markdown
## Pipeline Complete
Feature: photo-gallery
Duration: 3h 20m
Skills run: 8
Issues found: 12
Issues fixed: 11
Issues deferred: 1 (nice-to-have, tracked in backlog)
Deployed: staging → production
```

Archive the feature:
```bash
# Move completed feature to history
git add .claude/pipeline/
git commit -m "pipeline: [feature-name] shipped"
```

## Skill Coordination

### How Skills Pass Context

Each skill writes specific context that downstream skills consume:

```
/product-spec writes:
  - Acceptance criteria → qa-test uses for test scenarios
  - User flows → qa-test maps to test flows
  - Data model changes → db-migrate generates migrations

/feature-dev writes:
  - Files changed → design-audit focuses on those screens
  - New API endpoints → api-spec validates them
  - New mutations → qa-test generates tests for them

/design-audit writes:
  - Screenshot paths → qa-test can reference visual state
  - Fixed issues → qa-test verifies fixes work functionally
  - Dark mode status → perf-audit checks dark mode rendering

/qa-test writes:
  - Pass/fail per flow → deploy decides go/no-go
  - Edge cases found → security-audit focuses on those paths
  - Performance timings → perf-audit has baseline numbers

/security-audit writes:
  - Vulnerabilities → must be fixed before deploy
  - RLS policy status → deploy includes migration if needed

/perf-audit writes:
  - Bundle size delta → deploy monitors for regression
  - Slow endpoints → api-spec flags for optimization
```

### Parallel Execution

Some skills can run simultaneously:

```
Sequential (must wait):
  product-spec → feature-dev → [parallel block] → deploy

Parallel block (can run together):
  ┌─ design-audit
  ├─ qa-test
  ├─ security-audit
  ├─ perf-audit
  └─ analytics-audit

These 5 read the same context (files changed, features added)
and write independent reports. Run them in parallel.
```

Use the Agent tool to launch parallel skills as background agents.

## Pipeline Templates

### Template: New Feature (full pipeline)
```
1. /product-spec → spec.md
   GATE: user approves spec
2. /feature-dev → implement code
   GATE: user reviews code
3. PARALLEL:
   - /design-audit → design-audit.md
   - /qa-test → qa-report.md
   - /security-audit → security-report.md
   - /perf-audit → perf-report.md
   - /analytics-audit → analytics-report.md
   GATE: user reviews all reports
4. Fix any CRITICAL/MAJOR issues from reports
5. /deploy → staging
6. /qa-test → regression on staging
   GATE: user approves production deploy
7. /deploy → production
```

### Template: Bug Fix (fast track)
```
1. /incident → diagnose root cause (if not already known)
2. /feature-dev → implement fix
3. /qa-test → test the fix + regression
   GATE: user approves
4. /deploy → hotfix to production
```

### Template: Release (quality gate)
```
1. PARALLEL:
   - /design-audit → full app
   - /qa-test → full regression
   - /security-audit → full scan
   - /perf-audit → full profiling
2. GATE: user reviews all reports
3. Fix any blockers
4. /app-store → prepare submission (if mobile)
5. /deploy → production
```

### Template: Audit Only (no deploy)
```
1. PARALLEL:
   - /design-audit
   - /qa-test
   - /security-audit
   - /perf-audit
   - /analytics-audit
2. Generate combined report
3. Prioritize findings
```

## Status Commands

The user can ask for status at any time:

"What's the status?" → Read manifest.md, summarize progress
"What's blocking?" → Read manifest blockers section
"Skip QA" → Mark qa-test as skipped with reason, continue pipeline
"Go back to spec" → Reset pipeline to spec phase, preserve existing reports
"Cancel" → Archive current state, no deploy

## Error Handling

If a skill fails or finds CRITICAL issues:

1. **CRITICAL from security-audit** → STOP pipeline. Cannot deploy with critical vulnerabilities.
2. **CRITICAL from qa-test** → STOP pipeline. Core flow broken.
3. **CRITICAL from design-audit** → Continue but flag. Visual issues rarely block deploy.
4. **Skill crashes** → Log error, ask user: "Design audit failed to complete. Skip it or retry?"
5. **API unavailable** → Mark dependent tests as SKIPPED, continue with non-API tests.

## Session Resumption

If the user leaves and comes back:

1. Read `.claude/pipeline/manifest.md`
2. "I see you were working on photo-gallery. Last completed: design-audit. Next up: qa-test. Continue?"
3. Resume from where they left off — all context is in files.

## Metrics (optional)

Track across features over time:

```markdown
# Pipeline Metrics — 2026 Q2

## Throughput
Features shipped: 12
Avg cycle time: 4.2 hours
Fastest: 45 min (bug fix)
Slowest: 8 hours (new feature with API changes)

## Quality
Avg issues found per feature: 8.3
Issues caught before production: 97%
Post-deploy incidents: 1

## By Skill
| Skill | Avg Issues Found | Avg Time |
|-------|-----------------|----------|
| design-audit | 4.2 | 12 min |
| qa-test | 2.1 | 18 min |
| security-audit | 1.8 | 8 min |
| perf-audit | 0.4 | 6 min |
```

## Anti-Patterns

- DON'T skip the spec phase for non-trivial features — it saves time downstream
- DON'T run deploy without at least qa-test passing
- DON'T run all skills for a typo fix — match pipeline to task size
- DON'T ignore CRITICAL findings from any skill
- DON'T modify manifest.md manually — let the orchestrator manage it
- DON'T run parallel skills that depend on each other's output
