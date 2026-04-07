# Contributing

## Submit a New Skill

### As a standalone repo

1. Create a repo with `SKILL.md` and `README.md`
2. Follow the [skill format guide](docs/create-a-skill.md)
3. Users install with: `git clone [your-repo] ~/.claude/skills/[name]`
4. Open an issue on this repo to get listed in the README

### As a PR to this repo

1. Fork `claude-engineering-suite`
2. Create `skills/your-skill-name/SKILL.md`
3. Add `skills/your-skill-name/README.md`
4. Add your skill to the skills table in `README.md`
5. Open a PR

## Skill Quality Checklist

Your skill **must** include:

- [ ] **Frontmatter** — `name` and `description` in YAML
- [ ] **When to Use** — clear trigger phrases and situations
- [ ] **User Interview** — questions asked before execution
- [ ] **Platform Detection** — auto-detect relevant stack/tools
- [ ] **Execution Steps** — the core logic with bash commands and analysis
- [ ] **Report Format** — structured output written to pipeline
- [ ] **Autonomous Mode** — how it behaves without user interaction
- [ ] **Anti-Patterns** — what it should NOT do
- [ ] **Pipeline Context** — reads manifest, writes report to feature dir

Your skill **should** include:

- [ ] Preference storage (so repeat runs skip the interview)
- [ ] Support for at least 2 frameworks/platforms
- [ ] Proposed fixes, not just problem reports
- [ ] Severity levels on findings

## Report Bugs

Open an issue with:
- Which skill has the bug
- What you expected
- What actually happened
- Your stack (framework, OS, Node version)
- Relevant output or error messages

## Propose Changes to Existing Skills

1. Open an issue describing the change
2. If it's small (typo, clarification), just open a PR
3. If it changes skill behavior, discuss in the issue first

## How Skills Get Added to Built-In Templates

The built-in templates (`full-feature`, `bug-fix`, `quick-release`, `audit-only`) are defined in the `/ship` skill. To suggest adding a skill to a template:

1. Open an issue explaining which template and why
2. The skill must be broadly useful (not niche to one framework)
3. It must be well-tested across multiple project types

## Code of Conduct

- Be constructive. "This doesn't work" is less useful than "This fails when X, here's the error."
- Be respectful. Disagreements about approach are fine. Personal attacks are not.
- Test your contributions. Don't submit a skill you haven't run at least once.
- Keep scope tight. One skill per PR. One bug per issue.

## License

By contributing, you agree your contribution is licensed under MIT, same as the project.
