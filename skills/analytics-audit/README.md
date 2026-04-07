# /analytics-audit

Analytics audit pipeline — scans all tracking calls, maps user flows to events, verifies event properties, checks for duplicates and gaps, validates funnels, audits PII compliance, and generates a tracking plan. Use when the user asks to audit analytics, check tracking coverage, review events, or create a tracking plan.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/analytics-audit
curl -o ~/.claude/skills/analytics-audit/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/analytics-audit/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/analytics-audit
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
