# /ship

Pipeline orchestrator — coordinates all engineering skills (design-audit, qa-test, security-audit, perf-audit, deploy, etc.) to ship features end-to-end. Detects intent, picks the right skills in the right order, manages context between them, and tracks pipeline state. Use when the user says "ship this", "build this feature", "release", or wants the full pipeline.
Fast-track release — just QA + security + deploy
Complete pipeline for new features
Run all quality checks, no code changes

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/ship
curl -o ~/.claude/skills/ship/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/ship/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/ship
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
