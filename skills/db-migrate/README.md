# /db-migrate

Database migration pipeline — generates migration SQL from schema changes, performs safety checks on destructive operations, generates rollback scripts, validates indexes and RLS policies, plans data backfills, and checks for table locks. Use when the user asks to create migrations, check schema changes, add columns/tables, or review migration safety.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/db-migrate
curl -o ~/.claude/skills/db-migrate/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/db-migrate/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/db-migrate
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
