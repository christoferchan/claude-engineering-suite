# Claude Engineering Suite

A complete engineering team for [Claude Code](https://claude.ai/code) — 16 skills that coordinate to ship features end-to-end.

```
/ship "add a photo gallery to spot detail"
    │
    ▼ /product-spec → writes PRD + acceptance criteria
    ▼ YOU APPROVE → "yes, build it"
    ▼ /feature-dev → implements the code
    ▼ /design-audit → captures screenshots, finds visual issues
    ▼ /qa-test → generates + runs functional tests
    ▼ /security-audit → scans for vulnerabilities
    ▼ /perf-audit → checks bundle size + render performance
    ▼ /analytics-audit → verifies tracking events
    ▼ YOU APPROVE → "ship it"
    ▼ /deploy → pushes to production
```

## Install

### Full suite (recommended)

```bash
git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills
```

### Cherry-pick individual skills

```bash
# Install only what you need — /ship adapts automatically
git clone https://github.com/christoferchan/design-audit-skill.git ~/.claude/skills/design-audit
git clone https://github.com/christoferchan/qa-test-skill.git ~/.claude/skills/qa-test
```

### Remove skills you don't need

```bash
# Full suite installed but don't need app-store prep?
rm -rf ~/.claude/skills/app-store
# /ship will skip it automatically
```

## The Skills

### Orchestrator

| Skill | What It Does |
|-------|-------------|
| `/ship` | Coordinates all other skills. Detects intent (new feature, bug fix, release), picks the right skills, manages context between them, tracks pipeline state. |

### Quality

| Skill | What It Does |
|-------|-------------|
| `/design-audit` | Captures every screen (Maestro/Playwright), analyzes against your design system, proposes fixes. 25+ point checklist. |
| `/qa-test` | Maps user flows, generates test scenarios (happy path, edge cases, destructive, permissions), executes via Maestro/Playwright. |
| `/security-audit` | OWASP Top 10 scanning, secret detection, auth/RLS audit, dependency vulnerabilities, input validation. |
| `/perf-audit` | Bundle size, render profiling, image optimization, memory leaks, Core Web Vitals, API response budgets. |
| `/analytics-audit` | Verifies tracking events fire correctly, maps flows to events, detects gaps, validates funnels. |

### Engineering

| Skill | What It Does |
|-------|-------------|
| `/api-spec` | Generates OpenAPI specs from code, validates contracts, tests endpoints, detects breaking changes. |
| `/db-migrate` | Safe migration generation, rollback scripts, destructive op detection, index analysis, RLS validation. |
| `/deploy` | Environment detection, pre-deploy checklists, deploy execution, rollback plans, post-deploy smoke tests. |

### Product & Ops

| Skill | What It Does |
|-------|-------------|
| `/product-spec` | Generates PRDs, user stories, competitive analysis, scope definition. Pushes back on over-scoping. |
| `/content-copy` | App copy audit, App Store descriptions, help docs, error messages, microcopy consistency. |
| `/incident` | Error diagnosis from logs/Sentry, root cause analysis, rollback guidance, post-mortem generation. |
| `/infra-setup` | Environment provisioning, secrets management, monitoring setup, CI/CD pipeline generation. |
| `/data-seed` | AI-generated content, Google Places photo hydration, geographic seeding, content quality checks. |
| `/app-store` | Screenshot generation, App Store metadata, compliance checklists, ASO keyword research. |
| `/device-test` | Manual testing checklists for real devices — hardware, network, accessibility, battery, edge cases. |

## How They Work Together

Skills communicate through `.claude/pipeline/` — a shared directory where each skill reads context from previous steps and writes its report. `/ship` orchestrates the flow and git-commits after each phase for full history.

```
.claude/pipeline/
├── manifest.md              # Pipeline state
├── features/
│   └── photo-gallery/
│       ├── spec.md          # From /product-spec
│       ├── design-audit.md  # From /design-audit
│       ├── qa-report.md     # From /qa-test
│       └── ...
└── history/
    └── 2026-04-07-session.md
```

See [Context Protocol](docs/context-protocol.md) for details.

## Adaptive Pipeline

`/ship` auto-detects which skills are installed and only runs those:

```
Full suite:     spec → dev → design → QA → security → perf → analytics → deploy
Just 2 skills:  dev → design → QA → done
Custom mix:     dev → design → security → deploy
```

No skill is required. Install what you need, `/ship` adapts.

## Platform Support

Every skill auto-detects your stack:

| Category | Supported |
|----------|-----------|
| **Mobile** | React Native, Expo, Flutter |
| **Web** | Next.js, React, Vue, Nuxt, Svelte, Angular |
| **Backend** | Supabase, Firebase, Prisma, Express, Django, Rails, Go, Laravel |
| **Deploy** | Vercel, Netlify, AWS, Railway, Fly.io, Heroku, EAS |
| **Testing** | Maestro (mobile), Playwright (web) |

## Quick Start

```bash
# Install
git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills

# Open your project
cd your-project
claude
```

Then in Claude Code:
```
# Load the orchestrator
you: use ship
claude: "What are we working on?"

# Tell it what you want (plain text, not slash commands)
you: audit-only
claude: [runs design-audit → qa-test → security-audit → perf-audit]

# Or run individual skills
you: use design-audit
you: use qa-test
you: use security-audit
```

> **Important:** Don't type `/ship audit-only` — that won't work. Type `use ship` first, then `audit-only` as a separate message. Skills are loaded first, then you talk to them.

### Common Mistakes

```
❌  /ship audit-only              ← doesn't work
❌  /ship fix the login bug       ← doesn't work

✅  use ship                      ← load skill first
    audit-only                    ← then type your request

✅  use ship
    fix the login bug

✅  use design-audit              ← or run skills directly
```

## Updating

The suite checks for updates automatically when you load `/ship`. If an update is available, you'll see:

```
⚠️ Skill suite update available (3 commits behind)
   Run: cd ~/.claude/skills && git pull
   Then: /reload-plugins
```

To update manually:
```bash
cd ~/.claude/skills
git pull
```

Then in Claude Code:
```
/reload-plugins
```

See [CHANGELOG.md](CHANGELOG.md) for what changed.

## Credits

Created by [Christofer Chan](https://github.com/christoferchan) at [Odd Hours](https://oddhours.ai).

Built while developing [Spots](https://usespots.com) — a travel discovery and trip-planning app.

## License

MIT — use it, modify it, share it.
