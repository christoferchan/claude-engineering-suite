# Pipeline Context Protocol

All skills in the engineering suite communicate through files in `.claude/pipeline/`.

## How It Works

1. `/ship` creates a feature directory and manifest
2. Each skill reads the manifest to understand context
3. Each skill writes its report to the feature directory
4. `/ship` updates the manifest between skills
5. Everything is git-committed for full history

## Directory Structure

```
.claude/pipeline/
├── manifest.md                    # Current state (the orchestrator's brain)
├── preferences.md                 # User preferences across all skills
├── features/
│   ├── [feature-slug]/
│   │   ├── spec.md               # From /product-spec
│   │   ├── design-audit.md       # From /design-audit
│   │   ├── qa-report.md          # From /qa-test
│   │   ├── security-report.md    # From /security-audit
│   │   ├── perf-report.md        # From /perf-audit
│   │   ├── analytics-report.md   # From /analytics-audit
│   │   ├── deploy-log.md         # From /deploy
│   │   └── screenshots/          # From /design-audit
│   └── ...
└── history/
    └── [date]-session.md          # Daily summaries
```

## Rules for Skills

1. **Read manifest.md at start** — know what's happening
2. **Read previous reports** — understand what other skills found
3. **Write your report** to the feature directory
4. **Don't modify other skills' reports** — append-only
5. **Update manifest.md** — mark your step complete
6. **Git commit** when done

## Manifest Format

See `/ship` skill documentation for the full manifest schema.
