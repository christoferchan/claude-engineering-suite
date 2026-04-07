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

### Step 1: Check project essentials

```bash
# Pipeline directory
ls .claude/pipeline/manifest.md 2>/dev/null || echo "NO_PIPELINE"

# Design system
find src/ -name "theme.*" -o -name "tokens.*" -o -name "tailwind.config.*" 2>/dev/null | head -1 || echo "NO_THEME"

# Project documentation
ls CLAUDE.md 2>/dev/null || echo "NO_CLAUDE_MD"

# Test infrastructure
ls jest.config.* vitest.config.* 2>/dev/null || echo "NO_TEST_CONFIG"

# CI/CD
ls .github/workflows/*.yml 2>/dev/null || echo "NO_CI"

# Git
ls .gitignore 2>/dev/null || echo "NO_GITIGNORE"
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
