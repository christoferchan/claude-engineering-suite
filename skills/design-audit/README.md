# /design-audit

Visual design audit pipeline — sets up Maestro screenshot capture, analyzes every screen for UI/UX issues, proposes fixes with design options, generates Figma mockups, and implements approved changes. Use when the user asks to audit their app, review UI quality, capture screenshots, or improve design consistency.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/design-audit
curl -o ~/.claude/skills/design-audit/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/design-audit/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/design-audit
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
