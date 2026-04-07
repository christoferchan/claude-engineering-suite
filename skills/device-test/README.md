# /device-test

Device testing checklist generator — produces comprehensive manual QA checklists for real-device testing covering hardware (camera, GPS, haptics, biometrics), network conditions, OS integration (push notifications, deep links, share sheet), accessibility (VoiceOver, dynamic type), multi-device layouts, edge cases, and dark mode verification. Use when the user asks to test on real devices, prepare for QA, generate a test checklist, or verify hardware-dependent features.

## Install (standalone)

```bash
# Just this skill
mkdir -p ~/.claude/skills/device-test
curl -o ~/.claude/skills/device-test/SKILL.md https://raw.githubusercontent.com/christoferchan/claude-engineering-suite/master/skills/device-test/SKILL.md
```

Or install the full suite: `git clone https://github.com/christoferchan/claude-engineering-suite.git ~/.claude/skills`

## Usage

```
/device-test
```

## Part of [Claude Engineering Suite](https://github.com/christoferchan/claude-engineering-suite)

16 skills that coordinate as a complete engineering team. Use `/ship` to orchestrate them.
