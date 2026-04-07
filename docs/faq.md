# FAQ

## Does this work with [my framework]?

**React Native, Expo, Next.js, React, Vue, Nuxt, Svelte, Angular** — yes, fully supported. Skills auto-detect your stack.

**Flutter** — partially. Design audit works (Maestro supports Flutter). QA test works. Security audit scans Dart files. Skills that assume JavaScript/TypeScript tooling (perf-audit, analytics-audit) will skip or fall back to generic checks.

**Rails, Django, Laravel, Express, FastAPI, Go, Rust** — backend skills work well (security-audit, db-migrate, api-spec, deploy). Frontend-focused skills (design-audit) need a web frontend to capture.

**Static sites (Hugo, Jekyll, Astro)** — deploy and content-copy work. Most audit skills have limited value since there's less dynamic behavior to test.

If your framework isn't listed, skills will still attempt detection and fall back to generic behavior. They won't crash — they'll just tell you what they can and can't do.

## Will it modify my code without asking?

No. By default, every skill stops and asks before making changes. You see what it wants to do, you approve or reject.

The only exception is "full autopilot" mode, which you have to explicitly request. Even then, skills only auto-fix issues that are clearly safe (e.g., removing an exposed secret, fixing a contrast value). Anything ambiguous still prompts you.

Claude Code itself also has a permission system — every file write and shell command needs your approval unless you've configured it otherwise.

## Can I use it offline?

No. The skills are instructions that Claude Code follows, and Claude Code requires an internet connection to communicate with Claude's API. The skills themselves are local files, but executing them requires API access.

## Does it work on Windows/Linux?

**macOS** — fully supported. All skills work, including Maestro (iOS simulator).

**Linux** — most skills work. Maestro works for Android emulator testing. iOS-specific features (simulator, Xcode) are unavailable.

**Windows** — most skills work via WSL2. Maestro works for Android. iOS testing is not available. Some bash commands in skills may need adjustment for Windows paths.

## How much disk space does it need?

The skill files themselves are tiny — under 1MB for the entire suite. They're just markdown files.

Pipeline artifacts (reports, screenshots) accumulate over time in `.claude/pipeline/`. A typical feature pipeline generates 50-200KB of reports. Screenshots from design audits can be 5-20MB per run. Use `/ship clean` periodically to archive old pipeline data.

## Can two people use `/ship` on the same repo?

Yes, but not on the same feature simultaneously. The `.lock` file prevents concurrent pipeline runs on the same feature. If person A is running a pipeline on `photo-uploads`, person B will see a "Pipeline is LOCKED" message.

Different features work fine in parallel — each gets its own directory under `.claude/pipeline/features/`.

For teams, the recommended workflow:
1. Each person works on their own feature branch
2. Pipeline state is git-committed, so pulling brings in the latest reports
3. Use `/ship status` to see what's active

## Does it cost anything?

**The skills are free.** MIT license, no subscription, no usage tracking.

**Claude Code usage costs money.** Skills run through Claude's API, which charges per token. A full pipeline run (spec through deploy) typically uses 50-150k tokens across all skills. Check [Anthropic's pricing](https://www.anthropic.com/pricing) for current rates.

**Maestro is free.** Open-source mobile testing framework.

**Everything else is your existing infrastructure.** Supabase, Vercel, AWS — whatever you already use and pay for.

## Can I use my own skills with `/ship`?

Yes. Drop a directory with a `SKILL.md` into `~/.claude/skills/` and `/ship` discovers it automatically on the next run. See [Create a Skill](create-a-skill.md) for the full guide.

You can also override built-in skills by replacing their `SKILL.md` with your own version.

## What's the difference between `/ship` and running skills individually?

**`/ship`** is the orchestrator. It:
- Picks which skills to run based on your task
- Runs them in the right order
- Passes context between skills (spec feeds into dev, dev feeds into audit, etc.)
- Manages pipeline state (resume, cancel, history)
- Git commits after each phase

**Running skills individually** (e.g., `/design-audit`, `/qa-test`) skips the orchestrator. The skill runs standalone, writes its report, and you're done. Useful when you only need one specific check.

Both work. `/ship` is better for features and releases. Individual skills are better for quick one-off checks.

## Is my code sent anywhere?

Your code is sent to Claude's API for analysis — that's how the skills work. Claude reads your files, runs commands, and produces reports. This is the same as using Claude Code normally.

No code is sent to any other service. The skills don't phone home, collect telemetry, or transmit data anywhere besides the Claude API.

Review [Anthropic's data policy](https://www.anthropic.com/privacy) for how Claude handles your data. Key point: API inputs are not used to train models.
