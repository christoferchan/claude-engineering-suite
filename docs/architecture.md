# Architecture

How the pieces fit together.

## Pipeline Flow

```
User invokes /ship "add photo uploads"
         │
         ▼
┌─────────────────┐
│   /ship (orchestrator)   │
│                          │
│  1. Detect intent        │
│  2. Pick skills          │
│  3. Create manifest      │
│  4. Create feature dir   │
└──────────┬───────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
 Interview    Platform
 questions    detection
     │           │
     └─────┬─────┘
           │
           ▼   ┌──────────────────────┐
   ┌───────────┤  .claude/pipeline/   │
   │           │  manifest.md         │
   │           └──────────────────────┘
   │                    ▲
   │                    │ (read/write)
   ▼                    │
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Skill 1  │───▶│ Skill 2  │───▶│ Skill 3  │───▶ ...
│ (spec)   │    │ (dev)    │    │ (audit)  │
└──────────┘    └──────────┘    └──────────┘
   │ write         │ write         │ write
   ▼               ▼               ▼
┌──────────────────────────────────────────┐
│  .claude/pipeline/features/[slug]/       │
│  ├── spec.md                             │
│  ├── design-audit.md                     │
│  ├── qa-report.md                        │
│  └── ...                                 │
└──────────────────────────────────────────┘
           │
           ▼
   /ship updates manifest
   /ship git commits
   /ship calls next skill
           │
           ▼
   ┌───────────────┐
   │ Human approval │  (at key gates)
   │ gate           │
   └───────┬───────┘
           │
           ▼
       Next phase
```

## Communication: `.claude/pipeline/`

Skills don't call each other directly. They communicate through files.

```
.claude/pipeline/
├── manifest.md              # The orchestrator's brain — current state
├── preferences.md           # User preferences (persisted across runs)
├── skill-registry.md        # Discovered skills and capabilities
├── .lock                    # Prevents concurrent pipeline runs
├── features/
│   └── [feature-slug]/
│       ├── spec.md          # /product-spec output
│       ├── design-audit.md  # /design-audit output
│       ├── qa-report.md     # /qa-test output
│       ├── security-report.md
│       ├── perf-report.md
│       ├── analytics-report.md
│       ├── deploy-log.md
│       ├── changelog.md
│       └── screenshots/     # Captured by /design-audit and /qa-test
└── history/
    ├── 2026-04-03-session.md
    ├── 2026-04-04-session.md
    └── 2026-04-05-session.md
```

### Why files, not function calls?

- Skills run in separate Claude Code sessions — they can't share memory
- Files are inspectable — you can read any report at any time
- Files are git-committable — full history of every pipeline run
- Files survive crashes — resume from where you left off
- Any tool can produce a file — not locked to a specific runtime

## Manifest Lifecycle

```
Created            Updated              Updated           Archived
  │                  │                    │                  │
  ▼                  ▼                    ▼                  ▼
/ship start    Each skill marks     /ship updates      /ship cancel
               its step done        context for        or /ship clean
               in manifest          next skill         moves to history/
```

### Manifest states

| State | Meaning |
|-------|---------|
| `pending` | Pipeline created, no skills have run |
| `[skill-name]` | That skill is currently executing |
| `blocked` | Waiting for human approval |
| `completed` | All steps done |
| `cancelled` | User cancelled, archived |

### What's in the manifest

```markdown
# Pipeline Manifest

## Current Feature
name: user-photo-uploads
slug: user-photo-uploads
branch: feature/user-photo-uploads
started: 2026-04-05T10:00:00Z
status: qa-testing

## Pipeline Progress
- [x] product-spec — completed 2026-04-05T10:15:00Z
- [x] feature-dev — completed 2026-04-05T11:00:00Z — 4 files changed
- [ ] design-audit — IN PROGRESS
- [ ] qa-test — pending
- [ ] security-audit — pending

## Context for Next Skill
- Feature: photo uploads on spot detail
- Files changed: PhotoUploadButton.tsx, useUploadPhoto.ts, SpotPhotoGallery.tsx
- Dependencies added: expo-image-manipulator

## Decision Log
- 10:05 — User chose grid layout over carousel
- 10:12 — Max upload size: 15MB (compress before upload)
```

## Skill Execution Order

### Sequential (default)

Skills run one at a time. Each skill reads the previous skill's output.

```
spec → dev → design-audit → qa-test → security → perf → analytics → deploy
```

This is the safest mode. Each skill has full context from everything before it.

### Parallel (where safe)

Some skills have no dependencies on each other and can run simultaneously:

```
                    ┌── security-audit ──┐
dev → design-audit ─┤                    ├── deploy
                    └── perf-audit ──────┘
```

`/ship` determines parallelism from skill capability tags:
- `reads: [spec, code]` — skill needs spec and source code
- `reads: [spec, code, design-audit]` — skill depends on design audit output

If two skills read the same inputs and neither reads the other's output, they can run in parallel.

### When skills skip

If a skill isn't installed, `/ship` skips it. If a skill isn't relevant (e.g., `/app-store` for a web-only project), `/ship` skips it. The pipeline adapts.

## Context Passing Chain

Each skill adds context that later skills use:

```
/product-spec
  writes: spec.md (requirements, acceptance criteria, scope)
    │
    ▼ read by
/feature-dev
  writes: code changes + updates manifest with file list
    │
    ▼ read by
/design-audit
  reads: spec (what should it look like?) + file list (where to look)
  writes: design-audit.md (visual issues found)
    │
    ▼ read by
/qa-test
  reads: spec (what to test) + file list (what changed) + design issues (verify fixes)
  writes: qa-report.md (functional test results)
    │
    ▼ read by
/security-audit
  reads: file list (what to scan) + spec (security requirements)
  writes: security-report.md
    │
    ▼ read by
/deploy
  reads: all reports (any blockers?) + spec (deploy requirements)
  writes: deploy-log.md
```

Each skill benefits from the cumulative context. The security audit knows which files changed (from dev) and what the feature does (from spec). The deploy step knows if there are open issues from any audit.

## Directory Structure

```
claude-engineering-suite/
├── README.md                    # Project overview, install, skill table
├── CONTRIBUTING.md              # How to contribute
├── LICENSE                      # MIT
├── docs/
│   ├── context-protocol.md      # How skills communicate
│   ├── architecture.md          # This file
│   ├── examples.md              # Real output examples
│   ├── create-a-skill.md        # Build your own skill
│   ├── troubleshooting.md       # Common issues
│   └── faq.md                   # Frequently asked questions
└── skills/
    ├── ship/                    # Orchestrator
    │   ├── SKILL.md
    │   └── README.md
    ├── design-audit/            # Visual audit
    │   ├── SKILL.md
    │   └── README.md
    ├── qa-test/                 # Functional testing
    │   ├── SKILL.md
    │   └── README.md
    ├── security-audit/          # Security scanning
    │   ├── SKILL.md
    │   └── README.md
    ├── perf-audit/              # Performance
    ├── analytics-audit/         # Event tracking
    ├── api-spec/                # API contracts
    ├── db-migrate/              # Database migrations
    ├── deploy/                  # Deployment
    ├── product-spec/            # PRDs and specs
    ├── content-copy/            # Copy and content
    ├── incident/                # Error diagnosis
    ├── infra-setup/             # Infrastructure
    ├── data-seed/               # Content seeding
    ├── app-store/               # App Store prep
    └── device-test/             # Device testing checklists
```

Each skill directory contains:
- `SKILL.md` — the skill definition (frontmatter + instructions). This is what Claude reads.
- `README.md` — human-readable description for GitHub. Not used by the system.
