# Create a Skill

How to build a custom skill for the Claude Engineering Suite.

## Skill File Format

A skill is a single `SKILL.md` file inside a directory under `~/.claude/skills/`. The file uses YAML frontmatter followed by markdown instructions.

```
~/.claude/skills/
└── your-skill-name/
    ├── SKILL.md          # Required — the skill definition
    └── README.md         # Optional — for GitHub display
```

### Frontmatter

```yaml
---
name: your-skill-name
description: One sentence. This is what /ship reads to decide when to invoke your skill.
---
```

The `name` must match the directory name. The `description` is used by `/ship` for intent matching — make it specific about what triggers the skill.

## Required Sections

Every skill needs these sections in `SKILL.md`:

### 1. When to Use

List the trigger phrases and situations. Be explicit.

```markdown
## When to Use

- User says "run lighthouse", "check performance scores", "audit page speed"
- After deploying to staging to verify Core Web Vitals
- Before a release to catch performance regressions
```

### 2. User Interview (Phase 0A)

Questions to ask before running. This prevents wasted work.

```markdown
## Phase 0A: User Interview

**1. Target URLs:**
"Which pages should I audit? (e.g., homepage, /login, /dashboard)"

**2. Performance budget:**
"What scores are acceptable? (default: 90+ for all categories)"

**3. Scope:**
- Full site (every route)
- Critical pages only
- Specific URLs (user provides list)

**4. Implementation preference:**
After the audit, do you want me to:
- Just report scores
- Report + suggest fixes
- Report + fix automatically
```

### 3. Platform Detection

Auto-detect the user's stack so the skill works everywhere.

```markdown
## Platform Detection

```bash
ls next.config.* 2>/dev/null && echo "NEXTJS"
ls vite.config.* 2>/dev/null && echo "VITE"
ls angular.json 2>/dev/null && echo "ANGULAR"
curl -s http://localhost:3000 2>/dev/null && echo "DEV_SERVER_RUNNING"
```
```

### 4. Execution Steps

The core logic. Write it as instructions Claude will follow — bash commands to run, files to read, analysis to perform.

```markdown
## Phase 1: Run Audit

Run Lighthouse CI against each target URL:

```bash
npx lighthouse [url] --output=json --output-path=./lighthouse-report.json
```

Parse the JSON output and extract:
- Performance score
- First Contentful Paint
- Largest Contentful Paint
- Cumulative Layout Shift
- Total Blocking Time
```

### 5. Report Format

Define the exact output structure. This is what gets written to the pipeline.

```markdown
## Report Format

```markdown
# Lighthouse Report — [date]

## Summary
| Page | Performance | Accessibility | Best Practices | SEO |
|------|-------------|---------------|----------------|-----|

## Failing Metrics
| Page | Metric | Value | Budget | Delta |
|------|--------|-------|--------|-------|

## Recommendations
[Prioritized by impact]
```
```

### 6. Autonomous Mode

How the skill behaves when running unattended (inside a `/ship` pipeline).

```markdown
## Autonomous Mode

When invoked by /ship or with --dangerously-skip-permissions:
- Skip interview if preferences exist in memory
- Use default budget: 90+ for all categories
- Run full audit, write report
- Auto-fix only if: the fix is a config change (not a code rewrite) AND it won't change user-visible behavior
- Otherwise: write report and notify user
```

### 7. Anti-Patterns

What the skill should NOT do. Prevents common mistakes.

```markdown
## Anti-Patterns

- DON'T run Lighthouse against localhost with throttling disabled — scores will be misleadingly high
- DON'T flag third-party script issues as app bugs — note them separately
- DON'T suggest removing analytics scripts to improve performance without discussing tradeoffs
- DON'T re-run if nothing changed since last run — compare against cached report
```

## Pipeline Context Protocol

Skills communicate through `.claude/pipeline/`. Your skill must follow these rules:

### Reading Context

At the start of every run, read the manifest:

```markdown
## Phase 0: Pipeline Context

1. Read `.claude/pipeline/manifest.md` — know what feature is being built, what other skills found
2. Read previous reports in `.claude/pipeline/features/[slug]/` — understand context
3. If this is a standalone run (no manifest), proceed independently
```

### Writing Reports

When finished, write your report:

```markdown
## Pipeline Report Output

Write report to:
  .claude/pipeline/features/[slug]/lighthouse-report.md

Where [slug] comes from the manifest. If no manifest (standalone run):
  .claude/pipeline/lighthouse-report.md
```

### Updating Manifest

Mark your step complete in the manifest:

```markdown
- [x] lighthouse-ci — completed 2026-04-05T14:30:00Z — 3 pages, all passing
```

## How `/ship` Discovers Your Skill

1. On every run, `/ship` scans `~/.claude/skills/*/SKILL.md`
2. It reads the `name` and `description` from frontmatter
3. It adds the skill to its registry (`.claude/pipeline/skill-registry.md`)
4. When building a pipeline, it matches skill descriptions against the task
5. Your skill appears in `/ship skills` and can be invoked with `/ship run your-skill-name`

No registration step needed. Drop the directory in `~/.claude/skills/` and it's live.

## Testing Your Skill

### Manual testing

```bash
# Open a test project
cd ~/my-test-project
claude

# Run your skill directly
/your-skill-name

# Run through the orchestrator
/ship run your-skill-name
```

### Verify these work

1. **Interview** — questions appear, answers are stored
2. **Detection** — correct stack detected in your test project
3. **Execution** — commands run, analysis produces results
4. **Report** — written to correct pipeline path with correct format
5. **Autonomous mode** — runs without prompts when preferences exist
6. **Anti-patterns** — none of the listed bad behaviors occur

### Test in a pipeline

```bash
/ship "add a contact form"
# Verify your skill runs at the right time
# Verify it reads context from previous skills
# Verify it writes report that later skills can read
```

## Sharing Your Skill

### Standalone repo

Create a repo with this structure:

```
your-skill-name/
├── SKILL.md
├── README.md
└── LICENSE
```

Users install with:

```bash
git clone https://github.com/you/your-skill-name.git ~/.claude/skills/your-skill-name
```

### PR to the suite

1. Fork `claude-engineering-suite`
2. Add your skill directory under `skills/`
3. Add it to the skills table in the root `README.md`
4. Open a PR with:
   - What the skill does
   - What stacks it supports
   - Example output

## Example: Building `/lighthouse-ci` From Scratch

Complete (abbreviated) skill file:

```markdown
---
name: lighthouse-ci
description: Lighthouse CI audit — runs Google Lighthouse against target URLs, reports Core Web Vitals scores, flags regressions, and suggests performance fixes. Use when the user asks to check page speed, audit performance, or verify Core Web Vitals.
---

# Lighthouse CI Skill

Performance auditing via Google Lighthouse.

## When to Use

- User says "check page speed", "audit performance", "run lighthouse"
- Before deploy to verify Core Web Vitals
- After adding heavy dependencies

## Phase 0: Pipeline Context

1. Read `.claude/pipeline/manifest.md` if it exists
2. Read previous reports (perf-audit may have related findings)
3. Write report to `.claude/pipeline/features/[slug]/lighthouse-report.md`

## Phase 0A: User Interview

**1. Target URLs:**
"Which pages should I audit?"

**2. Performance budget:**
"Minimum acceptable scores? (default: Performance 90, A11y 95, BP 95, SEO 90)"

**3. Comparison baseline:**
"Compare against production, last run, or just absolute scores?"

### Store Preferences
Save to `memory/feedback_lighthouse_preferences.md`

## Platform Detection

```bash
ls next.config.* 2>/dev/null && echo "NEXTJS"
ls vite.config.* 2>/dev/null && echo "VITE"
ls angular.json 2>/dev/null && echo "ANGULAR"
ls nuxt.config.* 2>/dev/null && echo "NUXT"
curl -s http://localhost:3000 2>/dev/null && echo "DEV_SERVER_3000"
curl -s http://localhost:5173 2>/dev/null && echo "DEV_SERVER_5173"
npx lighthouse --version 2>/dev/null && echo "LIGHTHOUSE_INSTALLED"
```

If Lighthouse is not installed:
```bash
npm install -g lighthouse
```

## Phase 1: Run Audits

For each target URL:

```bash
npx lighthouse [url] \
  --output=json \
  --output-path=./tmp-lighthouse.json \
  --chrome-flags="--headless --no-sandbox" \
  --preset=desktop
```

Parse JSON for scores and key metrics.

## Phase 2: Analyze Results

Compare scores against budget. Flag:
- Any category below budget → FAIL
- Any category within 5 points of budget → WARN
- Regression from baseline (if comparing) → REGRESSION

Identify top 3 opportunities from Lighthouse suggestions.

## Phase 3: Report

```markdown
# Lighthouse Report — [date]

## Summary
| Page | Perf | A11y | BP | SEO | Status |
|------|------|------|----|-----|--------|

## Failing Metrics
| Page | Metric | Value | Budget | Delta |
|------|--------|-------|--------|-------|

## Top Opportunities
1. [description] — estimated savings: [time]
2. ...

## Recommendations
[Prioritized list with file paths and suggested changes]
```

## Autonomous Mode

- Skip interview if preferences in memory
- Default budget: 90/95/95/90
- Run all detected pages
- Write report, no auto-fix (perf fixes are too risky to auto-apply)

## Anti-Patterns

- DON'T audit localhost without network throttling — unrealistic scores
- DON'T flag third-party scripts as app issues — separate section
- DON'T suggest removing analytics/error tracking for perf gains without tradeoff discussion
- DON'T run more than 3 times per URL — results stabilize, extra runs waste time
- DON'T compare desktop vs mobile scores — they use different configs
```

That's a complete, installable skill. Drop it in `~/.claude/skills/lighthouse-ci/SKILL.md` and `/ship` will discover it on the next run.
