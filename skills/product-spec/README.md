# /product-spec

Product specification pipeline — generates PRDs from user descriptions, writes user stories with acceptance criteria, defines scope (v1 vs deferred), maps data model and API requirements, estimates effort, and pushes back on over-scoped features. Use when the user says "spec this", "write a PRD", "plan this feature", "what would it take to build X", or wants to define a feature before building it.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/product-spec
curl -o ~/.claude/skills/product-spec/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/product-spec/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/product-spec
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
