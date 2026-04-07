# /infra-setup

Infrastructure setup and audit pipeline — detects existing infra (Supabase, Vercel, AWS, Firebase), audits secrets management, sets up monitoring and alerting, configures CI/CD, verifies environment parity, and estimates costs. Use when the user says "set up infrastructure", "check my env vars", "configure monitoring", "set up CI/CD", or wants to audit their infrastructure before launch.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/infra-setup
curl -o ~/.claude/skills/infra-setup/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/infra-setup/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/infra-setup
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
