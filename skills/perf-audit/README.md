# /perf-audit

Performance audit pipeline — analyzes bundle size, render performance, memory leaks, image optimization, font loading, lazy loading opportunities, list virtualization, and Core Web Vitals. Use when the user asks to optimize performance, check bundle size, find memory leaks, or improve load times.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/perf-audit
curl -o ~/.claude/skills/perf-audit/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/perf-audit/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/perf-audit
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
