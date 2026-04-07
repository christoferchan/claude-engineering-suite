---
name: content-copy
description: Content and copy audit pipeline — scans all user-facing strings, checks consistency and tone, enforces brand voice, generates App Store copy, onboarding copy, error messages, empty states, SEO metadata, privacy/ToS templates, and flags localization issues. Use when the user asks to audit copy, write App Store descriptions, improve error messages, check brand consistency, or prepare strings for localization.
---

# Content Copy Skill

End-to-end copy audit and generation pipeline: Interview → Scan → Audit → Generate → Deliver.


## Update Check

Before starting, check if the skill suite has updates available:

```bash
if [ -d ~/.claude/skills/.git ]; then
  git -C ~/.claude/skills fetch --quiet 2>/dev/null
  LOCAL=$(git -C ~/.claude/skills rev-parse HEAD 2>/dev/null)
  REMOTE=$(git -C ~/.claude/skills rev-parse origin/master 2>/dev/null)
  [ "$LOCAL" != "$REMOTE" ] && [ -n "$REMOTE" ] && echo "Update available: cd ~/.claude/skills && git pull"
fi
```

If update available, notify the user once (don't block execution):
> "Skill suite update available. Run: cd ~/.claude/skills && git pull"


## When to Use

- User says "audit my copy", "check the strings", "write App Store description"
- User wants consistent tone across the app
- User wants to improve error messages, empty states, or onboarding text
- User wants to prepare for localization (i18n readiness)
- User wants SEO copy for web pages
- Before a release to verify all user-facing text is polished

## Pipeline Context Protocol

### At Start — Read Pipeline State

```bash
# Check if running inside a pipeline
if [ -f .claude/pipeline/manifest.md ]; then
  cat .claude/pipeline/manifest.md
fi

# Check for feature context
FEATURE_SLUG=$(grep "^slug:" .claude/pipeline/manifest.md 2>/dev/null | awk '{print $2}')
if [ -n "$FEATURE_SLUG" ] && [ -d ".claude/pipeline/features/$FEATURE_SLUG" ]; then
  ls .claude/pipeline/features/$FEATURE_SLUG/
  # Read previous skill reports for context
  cat .claude/pipeline/features/$FEATURE_SLUG/*.md 2>/dev/null
fi
```

### At End — Write Output

Write the copy audit report and any generated content to the pipeline feature directory:

```bash
# If running inside a pipeline
mkdir -p .claude/pipeline/features/$FEATURE_SLUG
# Write the report
cat > .claude/pipeline/features/$FEATURE_SLUG/copy-report.md << 'EOF'
[report content]
EOF

# Update manifest
# Mark content-copy as complete, add context for next skill
```

If NOT in a pipeline, write to `.claude/pipeline/features/copy-audit/copy-report.md` as a standalone run.

## Phase 0A: User Interview

Ask these questions before starting. Wait for answers.

### Required Questions

**1. Brand voice:**
"How would you describe your app's voice? Pick 2-3 adjectives."
Examples: friendly + confident, minimal + professional, playful + warm, editorial + calm
→ This anchors every copy decision.

**2. Target audience:**
"Who's your primary user? Age range, context, familiarity with your app."
→ Determines vocabulary level, cultural references, formality.

**3. Copy priorities — rank these 1-5:**
- Consistency (same terminology everywhere)
- Tone (voice matches brand personality)
- Clarity (zero confusion, zero jargon)
- Brevity (as few words as possible)
- Inspiration (copy that motivates action, not just informs)

→ Determines which issues get flagged as CRITICAL vs MINOR.

**4. Known pain points:**
"Any specific copy that bothers you? Screens where the text feels off?"
→ Focus the audit on what the user already knows is wrong.

**5. Off-limits:**
"Any copy I should NOT change? (e.g., specific taglines, legal text)"
→ Prevents wasting time on text the user has already approved.

**6. Scope:**
- Full app audit (every string, every screen)
- Specific area (onboarding, errors, empty states, etc.)
- Generation only (App Store copy, privacy policy, etc.)
- Localization prep only (flag hardcoded strings)

**7. Implementation preference:**
After the audit, do you want me to:
- Just report issues (no code changes)
- Report + suggest rewrites (you pick which to apply)
- Report + apply approved rewrites
- Full autopilot (fix everything, ask only on subjective choices)

### Optional Questions

**8. Competitor apps:**
"Any apps whose copy you admire? (e.g., Airbnb's warmth, Duolingo's playfulness)"
→ Reference for tone calibration.

**9. Existing style guide:**
"Do you have a writing style guide, brand book, or tone document?"
→ Use as source of truth if it exists.

**10. Localization plans:**
"Planning to support other languages? If so, which ones first?"
→ Affects string extraction recommendations.

### Store Preferences

Save to persistent memory so future runs don't re-ask:

```
memory/feedback_content_copy_preferences.md
```

On subsequent runs: "I have your copy preferences from last time — [summary]. Still accurate, or want to update?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after scan for user review of terminology map
- STOP after audit for user review of findings
- STOP after generation for user approval
- Only apply changes the user explicitly approves

### Autonomous Mode (invoked by subagent or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** If no stored preferences, use safe defaults:
  - Voice: friendly + clear
  - Priority: clarity > consistency > brevity
  - Scope: full app
  - Implementation: report + suggest only (NO auto-apply)
  - Off-limits: none

- **Phase 1-2: Scan and audit run fully — no stop.**

- **Phase 3: Generate — ONLY auto-apply if ALL of these are true:**
  1. User's stored preference is "full autopilot"
  2. The change is a clear inconsistency (e.g., "Log in" vs "Sign in" — pick one)
  3. The change is not subjective (not rewriting a tagline)
  4. The app still builds after the change

  Otherwise: generate a report file and notify the user to review.

## Phase 1: Scan All User-Facing Strings

### 1.1 Read Project Context

```bash
# Read CLAUDE.md for brand rules, naming conventions, design philosophy
cat CLAUDE.md

# Read existing copy/content files
find src/ -name "strings.*" -o -name "messages.*" -o -name "constants.*" -o -name "copy.*" | head -20
```

### 1.2 Extract All User-Facing Strings

Scan every file for strings that users see:

```bash
# Button labels, placeholders, titles
grep -rn 'label=\|title=\|placeholder=\|accessibilityLabel=' src/ app/ --include="*.tsx" --include="*.ts" | grep -v node_modules | grep -v test

# Toast messages
grep -rn 'showToast\|Toast\.\|toast(' src/ app/ --include="*.tsx" --include="*.ts" | grep -v node_modules

# Error messages
grep -rn 'message:\|errorMessage\|Error(' src/ app/ --include="*.tsx" --include="*.ts" | grep -v node_modules

# Empty state text
grep -rn 'emptyText\|emptyMessage\|StateCard\|empty-state' src/ app/ --include="*.tsx" --include="*.ts" | grep -v node_modules

# Alert dialogs
grep -rn 'Alert\.alert\|confirm(' src/ app/ --include="*.tsx" --include="*.ts" | grep -v node_modules

# Hardcoded strings in JSX (quoted text inside tags)
grep -rn '>[A-Z][^<]*<' src/ app/ --include="*.tsx" | grep -v node_modules | grep -v test | head -100

# Web-specific: meta descriptions, OG titles, page titles
grep -rn 'title:\|description:\|og:' web/src/ --include="*.tsx" --include="*.ts" 2>/dev/null
```

### 1.3 Build Terminology Map

Create a map of all terms used for the same concept:

```
AUTHENTICATION
  "Sign in" — found in: [files]
  "Log in" — found in: [files]
  "Login" — found in: [files]

SAVING
  "Save" — found in: [files]
  "Bookmark" — found in: [files]
  "Add to saved" — found in: [files]

NAVIGATION
  "Back" — found in: [files]
  "Go back" — found in: [files]
  "Return" — found in: [files]

ERRORS
  "Something went wrong" — found in: [files]
  "An error occurred" — found in: [files]
  "Failed to load" — found in: [files]
```

## Phase 2: Audit

### 2.1 Consistency Check

For each concept in the terminology map, flag inconsistencies:

| Concept | Preferred Term | Variants Found | Files to Fix | Severity |
|---------|---------------|----------------|--------------|----------|
| Auth action | "Sign in" | "Log in" (3), "Login" (1) | auth.tsx, nav.tsx, settings.tsx | MAJOR |
| Save action | "Save" | "Bookmark" (2) | spot-detail.tsx, list.tsx | MAJOR |

Rules:
- Pick the most common variant as preferred (or the one matching brand voice)
- Flag any screen where the same action uses different words
- Flag noun/verb mismatches ("Login" as verb vs "Log in")

### 2.2 Tone Consistency Check

Analyze the tone of every string category:

| Category | Expected Tone | Actual Tone | Examples | Severity |
|----------|--------------|-------------|----------|----------|
| Empty states | Inspiring, action-oriented | Flat, informational | "No saved spots" → should be "Your saved spots will appear here" | MAJOR |
| Error messages | Helpful, human | Technical, cold | "Network error" → should be "Couldn't connect — check your WiFi" | MAJOR |
| Button labels | Active, clear | Passive, vague | "Submit" → should be "Share review" | MINOR |
| Onboarding | Warm, exciting | Generic | "Set up your profile" → "Make it yours" | MINOR |

### 2.3 Brand Voice Enforcement

Read CLAUDE.md (or user-provided brand rules) and check:

- **Product name usage**: Is the app name used correctly everywhere? (e.g., "Spots" = app, "Scout" = AI feature)
- **Prohibited terms**: Any terms the user has banned?
- **Voice consistency**: Does the copy match the stated brand adjectives?
- **Capitalization**: Title Case vs Sentence case — is it consistent?

### 2.4 Microcopy Audit

Check every micro-interaction:

- **Button labels**: Are they specific? ("Save" is okay, "Submit" is vague — prefer "Share review", "Create trip")
- **Placeholder text**: Is it helpful? ("Search..." is lazy — "Search cities, spots, or vibes" is better)
- **Toast messages**: Do they confirm the action? ("Saved!" vs "Spot saved to your list")
- **Loading states**: Do they set expectations? ("Loading..." vs "Finding spots near you...")
- **Confirmation dialogs**: Is the consequence clear? ("Are you sure?" is bad — "Delete this trip? This can't be undone." is good)

### 2.5 Localization Readiness

Flag strings that would break localization:

```
HARDCODED STRINGS (not in a strings file)
  - "Welcome back, {name}" in HomeScreen.tsx line 42 — should be in strings/en.ts
  - "3 spots" in ListCard.tsx line 18 — pluralization won't work in other languages

STRING CONCATENATION (breaks translation)
  - `"You have " + count + " trips"` — should use interpolation with plural rules

DATE/NUMBER FORMATTING
  - `new Date().toLocaleDateString()` — needs locale parameter
  - `price.toFixed(2)` — decimal separator varies by locale

HARDCODED UNITS
  - "5 miles" — should support km conversion

RTL CONCERNS
  - Hardcoded padding-left/margin-left that should flip for RTL languages
```

## Phase 3: Generate Content

Based on the audit and user's scope, generate any of the following:

### 3.1 Copy Style Guide

```markdown
# [App Name] Copy Style Guide

## Voice
[2-3 adjectives from interview]: [description of what that sounds like]

## Terminology
| Concept | Use This | Not This |
|---------|----------|----------|
| Authentication | Sign in | Log in, Login |
| Saving content | Save | Bookmark, Favorite |
| ...

## Capitalization
- Screen titles: Title Case
- Button labels: Sentence case
- Section headers: Sentence case
- ...

## Patterns
- Empty states: Always include a CTA. Inspire action, don't state absence.
- Error messages: [What happened] + [What to do]. Never show technical details.
- Toast messages: Past tense confirmation. "Spot saved" not "Saving spot..."
- Placeholders: Describe what to type, with examples.

## Brand Rules
- [App name] rules from CLAUDE.md
- Prohibited terms
- Required disclaimers
```

### 3.2 App Store Copy

```markdown
# App Store Listing

## Name (30 chars max)
[App Name]

## Subtitle (30 chars max)
[Compelling subtitle]

## Description (4000 chars max)
[Full description with keyword optimization]

## Keywords (100 chars max, comma-separated)
[keyword1, keyword2, ...]

## What's New (release notes)
[Version-specific release notes]

## Promotional Text (170 chars, can be updated without review)
[Current promotional message]
```

Research competitor keywords:
- Search the App Store category for competing apps
- Analyze their subtitles and descriptions for keyword patterns
- Suggest keywords with estimated search volume (high/medium/low)

### 3.3 Onboarding Copy

For each onboarding screen, generate:
- Headline (max 6 words, action-oriented)
- Subtext (max 2 lines, benefit-focused)
- CTA button label

### 3.4 Error Messages

For every error state in the app, generate:
- User-friendly message (what happened + what to do)
- Recovery action (button label)
- Never show: stack traces, error codes, technical jargon

Template:
```
[Emoji optional] [What happened in plain English]
[What the user can do about it]
[CTA: Try again / Go back / Contact support]
```

### 3.5 Empty State Copy

For every empty state, generate:
- Headline (inspiring, not just "No data")
- Subtext (explain the benefit of filling this state)
- CTA (action to fill the empty state)

Bad: "No saved spots"
Good: "Your saved spots will appear here — start exploring to find your favorites"

### 3.6 Privacy Policy & Terms of Service

Generate template-based documents from the app's actual data usage:

```bash
# Detect what data the app collects
grep -rn "requestPermission\|getUserLocation\|getContacts\|camera\|photo" src/ --include="*.ts" --include="*.tsx"
grep -rn "analytics\|tracking\|PostHog\|Sentry\|firebase" src/ --include="*.ts" --include="*.tsx"
grep -rn "signIn\|signUp\|createUser\|email\|password" src/ --include="*.ts" --include="*.tsx"
```

Generate:
- Privacy policy covering: data collected, how it's used, third parties, retention, user rights
- Terms of service covering: acceptable use, IP, liability, termination
- Both as HTML/Markdown for web hosting at /privacy and /terms

**Important**: Flag that these are TEMPLATES and must be reviewed by a lawyer before publishing.

### 3.7 SEO Copy (Web)

For each web page, generate:
- `<title>` tag (60 chars max)
- `<meta name="description">` (155 chars max)
- OG title and description
- Structured data suggestions (JSON-LD)

### 3.8 Help Documentation

Generate help content for common user questions:
- FAQ entries (question + answer format)
- Feature explanations (what it does + how to use it)
- Troubleshooting guides (symptom + fix)

## Phase 4: Deliver

### Output Report Format

```markdown
# Copy Audit Report — [date]

## User Preferences
Voice: [adjectives]
Priority: [ranked list]
Scope: [what was audited]

## Terminology Map
[from Phase 1.3]

## Findings

### CRITICAL (blocks release)
| # | Issue | Category | File:Line | Current Copy | Suggested Copy |
|---|-------|----------|-----------|-------------|----------------|
| 1 | Brand name wrong | Brand | HomeScreen.tsx:15 | "Welcome to Scout" | "Welcome to Spots" |

### MAJOR (should fix)
| # | Issue | Category | File:Line | Current Copy | Suggested Copy |
|---|-------|----------|-----------|-------------|----------------|

### MINOR (nice to have)
| # | Issue | Category | File:Line | Current Copy | Suggested Copy |
|---|-------|----------|-----------|-------------|----------------|

## Consistency Fixes
[Table of all terminology fixes with file paths]

## Generated Content
[Links to generated files: style guide, App Store copy, etc.]

## Localization Readiness
- Hardcoded strings: [count]
- String concatenation issues: [count]
- Date/number formatting issues: [count]
- Recommendation: [ready / needs work / not ready]

## Recommended Next Steps
1. [prioritized action items]
```

### String Change Manifest

For automated application, generate a structured list:

```json
{
  "changes": [
    {
      "file": "src/features/auth/LoginScreen.tsx",
      "line": 42,
      "current": "Log in",
      "replacement": "Sign in",
      "category": "consistency",
      "severity": "MAJOR"
    }
  ]
}
```

## Severity Definitions

- **CRITICAL**: Brand name wrong, legal text missing, offensive/inappropriate copy, accessibility label missing on critical element
- **MAJOR**: Terminology inconsistency across screens, tone mismatch with brand voice, unhelpful error message on common error, empty state that doesn't inspire action
- **MINOR**: Slightly verbose copy, placeholder could be more helpful, capitalization inconsistency, toast could be more specific

## Anti-Patterns

- DON'T rewrite copy that the user marked as off-limits
- DON'T use marketing jargon in UI copy (no "leverage", "utilize", "synergy")
- DON'T make empty states depressing ("Nothing here yet..." with no CTA)
- DON'T use technical language in error messages ("Error 500", "null reference")
- DON'T assume American English — ask about locale preferences
- DON'T change legal text without flagging it needs lawyer review
- DON'T generate privacy policy without scanning actual data collection in code
- DON'T skip the brand voice check — it's the most common source of copy debt
