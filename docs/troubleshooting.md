# Troubleshooting

Common issues and how to fix them.

## Skill not showing in `/ship skills`

**Symptom:** You installed a skill but `/ship skills` doesn't list it.

**Causes:**
1. Wrong directory — skill must be at `~/.claude/skills/[name]/SKILL.md`, not nested deeper
2. Missing `SKILL.md` — the file must be named exactly `SKILL.md` (not `skill.md` or `Skill.md`)
3. Missing frontmatter — the file needs `---` delimited YAML with `name` and `description`
4. Name mismatch — the `name` in frontmatter should match the directory name

**Fix:**
```bash
# Verify the file exists in the right place
ls ~/.claude/skills/your-skill/SKILL.md

# Check frontmatter
head -5 ~/.claude/skills/your-skill/SKILL.md
# Should show:
# ---
# name: your-skill
# description: ...
# ---
```

After fixing, start a new Claude Code session. Skills are discovered on session start.

## Maestro not working

**Symptom:** Design audit or QA tests fail with Maestro errors.

### Java not installed
```
Error: JAVA_HOME is not set
```
Maestro requires Java 11+.
```bash
# Check
java -version

# Install (macOS)
brew install openjdk@17
```

### Simulator not booted
```
Error: No booted simulator found
```
```bash
# List available simulators
xcrun simctl list devices available

# Boot one
xcrun simctl boot "iPhone 16 Pro"

# Verify
xcrun simctl list devices booted
```

### Metro not running
Maestro needs the app running in the simulator, which needs Metro.
```bash
# Start Metro
npx expo start

# Or with cache clear
npx expo start --clear
```

### Maestro not installed
```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
export PATH="$HOME/.maestro/bin:$PATH"
maestro --version
```

## Pipeline stuck / locked

**Symptom:** `/ship status` shows "Pipeline is LOCKED" but nothing is running.

**Cause:** A previous session crashed or was closed without cleaning up.

**Fix:**
```bash
# Check the lock file
cat .claude/pipeline/.lock

# If the listed session is definitely not running, remove it
rm .claude/pipeline/.lock

# Or use the built-in command
/ship clean
```

`/ship cancel --force` also removes stale locks.

## Screenshots all identical

**Symptom:** Design audit captures screenshots but they all show the same screen.

**Cause:** Maestro scroll/navigation commands aren't reaching the app. Common reasons:
- The app has a modal blocking navigation
- Maestro is tapping outside the scrollable area
- The app crashed silently and is showing an error boundary

**Fix:**
1. Check the simulator manually — is the app actually running and responsive?
2. Run a single Maestro flow to verify: `maestro test maestro/flows/shared/auto-login.yaml`
3. If the app crashed, restart Metro: `npx expo start --clear`
4. If a modal is blocking, add a dismiss step before navigation in the Maestro flow

## Skill finds 0 issues

**Symptom:** A skill runs but reports nothing found.

**Causes:**
1. **Wrong directory (monorepo)** — the skill is scanning the repo root but your code is in `apps/web/` or `packages/app/`
2. **Glob patterns miss your files** — skill looks for `src/**/*.tsx` but your code is in `lib/` or `components/`
3. **Different language/framework** — the skill's detection patterns don't match your stack

**Fix:**
- When the skill asks about scope during the interview, specify the exact directory
- Check which paths the skill is scanning (visible in the execution output)
- For monorepos, `cd` into the specific app directory before running

## Permission denied errors

**Symptom:** Skill tries to run a command but Claude Code asks for permission and blocks.

**Cause:** Claude Code requires explicit user approval for file writes, shell commands, and other operations.

**Fix:**
- Approve the tool calls when prompted — review what the skill wants to do and click approve
- For long pipeline runs, consider approving tool categories rather than individual calls
- If a skill is in autonomous mode, it still needs tool permissions from the Claude Code harness

## "Could not read manifest"

**Symptom:** A skill says it can't find `manifest.md`.

**Cause:** The pipeline hasn't been initialized. This happens when you run a skill directly without `/ship`.

**Fix:**
- If running standalone: this is normal. The skill should fall back to standalone mode and write its report to `.claude/pipeline/` root
- If running through `/ship`: run `/ship status` to check if a pipeline is active. If not, start one with `/ship "your task"`
- If the manifest was deleted: `/ship clean` then start fresh

## Fixes not showing in screenshots

**Symptom:** You fixed an issue, re-ran the design audit, but screenshots show the old version.

**Cause:** Metro (React Native) or the dev server (web) is serving cached code.

**Fix:**
```bash
# React Native — clear Metro cache
npx expo start --clear

# Next.js — clear .next cache
rm -rf .next && npm run dev

# Vite — clear cache
rm -rf node_modules/.vite && npm run dev
```

After restarting, wait for the app to fully reload before re-running the audit.

## Rate limit errors

**Symptom:** Skills fail mid-execution with API rate limit errors.

**Cause:** Running multiple skills in parallel or running skills that make many Claude API calls in quick succession.

**Fix:**
- Run skills sequentially (this is the default for `/ship` pipelines)
- If running skills manually in parallel, stagger them by a few minutes
- The `/ship` orchestrator already handles this — it runs one skill at a time

## Git push fails

**Symptom:** `/deploy` or pipeline commit fails with git push error.

### Protected branch
```
Error: protected branch hook declined
```
You're trying to push to `main` or `master` directly.

**Fix:** Work on a feature branch. `/ship` creates branches automatically.
```bash
git checkout -b feature/your-feature
```

### No remote tracking
```
Error: no upstream branch
```
```bash
git push -u origin your-branch-name
```

### Merge conflicts
```
Error: rejected — non-fast-forward
```
```bash
git pull --rebase origin main
# Resolve conflicts if any
git push
```
