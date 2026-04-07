# /incident

Incident response pipeline — reads error tracking (Sentry, Bugsnag, LogRocket), traces errors to source code, performs root cause analysis, suggests fixes with confidence levels, assesses severity, and generates post-mortems. Use when the user says "something broke", "check Sentry", "we have an incident", "debug this error", or wants to investigate production issues.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/incident
curl -o ~/.claude/skills/incident/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/incident/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/incident
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
