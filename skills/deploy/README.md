# /deploy

Deployment pipeline — detects platform (Vercel, Netlify, AWS, Railway, Fly.io, Heroku, Supabase, EAS), runs pre-deploy checks, executes deploy, verifies post-deploy health, and handles rollback. Use when the user says "deploy", "push to production", "ship to staging", or wants to release code to any environment.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/deploy
curl -o ~/.claude/skills/deploy/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/deploy/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/deploy
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
