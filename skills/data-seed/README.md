# /data-seed

Data seeding pipeline — detects app data model, generates realistic seed data using AI, hydrates with external APIs (Google Places, Unsplash), ensures content quality and referential integrity, supports bulk seeding with idempotent SQL/JSON fixtures. Use when the user says "seed data", "generate test data", "populate the database", "I need fake data", or wants realistic content for development or demos.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/data-seed
curl -o ~/.claude/skills/data-seed/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/data-seed/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/data-seed
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
