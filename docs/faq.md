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

---

## Known Limitations & Feature Requests

These are things the skill suite **cannot do** due to Claude Code platform constraints, not skill design choices.

### Skill invocation doesn't support arguments

```
❌  /ship audit-only          ← Claude Code ignores "audit-only"
✅  use ship → "audit-only"   ← load first, then type request
```

**Why:** Claude Code's Skill tool loads the SKILL.md as context but doesn't pass arguments from the slash command. All skills are invoked as `use [name]` or `/[name]`, then you communicate via normal messages.

**Workaround:** Type `use ship`, then type your command as a regular message.

**Feature request for Anthropic:** Allow `Skill(name, args)` to pass arguments so `use ship audit-only` works in one step.

### No interactive UI elements

```
❌  Dropdown menus, checkboxes, progress bars
✅  Numbered options the user types a number for
```

**Why:** Claude Code is text-only. No terminal UI widgets (like inquirer.js prompts).

**Workaround:** Skills use numbered options. Type `1`, `2`, or `3` to select.

### No real-time notifications

```
❌  "Security audit found a critical issue!" (push notification)
✅  Check status manually: "use ship" → "status"
```

**Why:** Skills can't send notifications. They run when invoked and produce output.

**Workaround:** For CI, generate a report file. For Slack/email alerts, use the `/schedule` skill to run audits on a cron and pipe results to a webhook.

### No persistent background processes

```
❌  "Watch for file changes and re-run design-audit"
✅  Run design-audit manually when needed, or schedule via cron
```

**Why:** Skills execute once per invocation. They can't watch for filesystem changes.

**Workaround:** Use the `/loop` skill to poll on an interval, or set up a GitHub Actions workflow that runs on PRs.

### Skills can't call other skills directly

```
❌  /design-audit internally calls /qa-test when it finds issues
✅  /ship orchestrates: runs design-audit, then qa-test, passing context via files
```

**Why:** Skills are independent. Only the orchestrator (`/ship`) chains them together by reading/writing to `.claude/pipeline/`.

**Workaround:** Use `/ship` for multi-skill workflows. Or just tell Claude: "run design-audit then qa-test" — it'll do both sequentially.

### Context window limits on large projects

```
❌  Read all 200 screenshots + all reports + full manifest in one turn
✅  Summarize reports, read screenshots in batches of 20
```

**Why:** Claude has a context window. Very large projects with many files can exceed it.

**Workaround:** Skills are designed to summarize, not dump. Manifest stays under 200 lines. Reports stay under 300 lines. Screenshots are referenced by path, not all read at once.

---

If Anthropic adds any of these capabilities, the skills will automatically benefit — the architecture is designed to take advantage of argument passing, notifications, and inter-skill communication when they become available.
