# /qa-test

End-to-end functional QA pipeline — maps user flows, generates test scenarios (happy path, edge cases, destructive, permissions), executes via Maestro/Playwright, reports pass/fail with failure screenshots and suggested fixes. Use when the user asks to test their app, run QA, check if flows work, or verify functionality before release.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/qa-test
curl -o ~/.claude/skills/qa-test/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/qa-test/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/qa-test
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
