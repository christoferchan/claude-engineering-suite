# /api-spec

API specification pipeline — auto-generates OpenAPI 3.0 specs from code, validates request/response contracts between client and server, tests endpoint behavior, checks auth/rate-limiting, detects breaking changes, and generates documentation. Use when the user asks to document APIs, generate specs, validate contracts, or check for breaking changes.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/api-spec
curl -o ~/.claude/skills/api-spec/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/api-spec/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/api-spec
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
