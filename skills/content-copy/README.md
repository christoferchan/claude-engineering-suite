# /content-copy

Content and copy audit pipeline — scans all user-facing strings, checks consistency and tone, enforces brand voice, generates App Store copy, onboarding copy, error messages, empty states, SEO metadata, privacy/ToS templates, and flags localization issues. Use when the user asks to audit copy, write App Store descriptions, improve error messages, check brand consistency, or prepare strings for localization.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/content-copy
curl -o ~/.claude/skills/content-copy/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/content-copy/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/content-copy
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
