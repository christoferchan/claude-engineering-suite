# /security-audit

Security audit pipeline — scans for exposed secrets, checks auth/authorization, validates input sanitization, audits dependencies for CVEs, verifies CORS/HTTPS/rate-limiting, and reports findings by OWASP Top 10 category. Use when the user asks to audit security, check for vulnerabilities, review auth, or harden their app before release.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/security-audit
curl -o ~/.claude/skills/security-audit/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/security-audit/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/security-audit
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
