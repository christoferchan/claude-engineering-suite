# Changelog

All notable changes to the Claude Engineering Suite.

## [1.1.0] — 2026-04-07

### Fixed
- **Invocation docs**: clarified `use ship` → then plain text (not `/ship audit-only`)
- **Template execution**: templates now explicitly run ALL steps, not just the first skill
- **Audit-only**: now launches all quality skills in parallel as intended

### Added
- **Version check**: `/ship` notifies when updates are available
- **6 documentation files**: examples, create-a-skill guide, troubleshooting, architecture, contributing, FAQ
- **Known limitations section**: honest about Claude Code platform constraints
- **Per-skill READMEs**: standalone install instructions for each skill

## [1.0.0] — 2026-04-07

### Added
- **16 skills**: ship, design-audit, qa-test, security-audit, perf-audit, analytics-audit, api-spec, db-migrate, deploy, incident, product-spec, content-copy, infra-setup, data-seed, app-store, device-test
- **Pipeline context protocol**: skills communicate via `.claude/pipeline/`
- **Orchestrator (`/ship`)**: coordinates skills with templates, gates, and tracking
- **12 edge case guards**: monorepo, Docker, Windows/WSL, linters, protected branches, prod DB safety, context overflow, concurrent runs, non-English, rate limits, submodules, offline
- **10 recovery guards**: crash resume, session timeout, permission denied, agent failure, Maestro hang, disk full, git conflicts, MCP disconnect, multi-day resume, context compression
- **Cross-instance resume**: pipeline state in git, any instance can pick up
- **Session logging**: structured logs with decisions, context, and blockers
- **Custom templates**: create, edit, delete pipeline templates
- **Third-party skill support**: auto-discover any skill in `~/.claude/skills/`
- **Adaptive pipeline**: only runs installed skills, skips missing ones
- **Monorepo support**: single app, all apps, or changed-only modes
- **Project bootstrap**: auto-creates CLAUDE.md, pipeline dir, missing infrastructure
