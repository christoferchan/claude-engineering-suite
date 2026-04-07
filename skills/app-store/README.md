# /app-store

App Store submission pipeline — generates screenshots via Maestro, formats for required device sizes, writes App Store description/subtitle/keywords, generates privacy policy, prepares review notes, runs compliance checklists (GDPR, COPPA, encryption), and handles ASO optimization. Supports iOS App Store, Google Play, and PWA. Use when the user asks to prepare for App Store submission, generate screenshots, write store listings, or check compliance.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/app-store
curl -o ~/.claude/skills/app-store/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/app-store/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/app-store
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
