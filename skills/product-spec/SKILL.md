---
name: product-spec
description: Product specification pipeline — generates PRDs from user descriptions, writes user stories with acceptance criteria, defines scope (v1 vs deferred), maps data model and API requirements, estimates effort, and pushes back on over-scoped features. Use when the user says "spec this", "write a PRD", "plan this feature", "what would it take to build X", or wants to define a feature before building it.
---

# Product Spec Skill

End-to-end product specification: Interview → Research → Scope → Spec → Review.

## When to Use

- User says "spec this feature", "write a PRD", "plan this feature"
- User describes a feature they want to build
- User asks "what would it take to build X?"
- Before starting development on anything non-trivial
- User wants to break a big idea into shippable chunks

## Pipeline Context Protocol

### On Start

1. Read `.claude/pipeline/manifest.md` if it exists — check if this spec is part of a larger pipeline
2. Read the codebase's CLAUDE.md — understand existing features, tech stack, design system
3. Scan `src/features/` — understand what feature modules already exist
4. Check `docs/FEATURES.md` if it exists — understand the feature catalog

### On Complete

Write spec to `.claude/pipeline/features/[slug]/spec.md`:

```markdown
# Product Spec: [Feature Name]

## Overview
[1-2 sentences]

## User Stories
[numbered list with acceptance criteria]

## Scope
### v1 (ship first)
### Deferred (later)

## Data Model
[tables, fields, relationships]

## API Requirements
[endpoints, edge functions]

## UI Description
[screen-by-screen text wireframes]

## Risks
[what could go wrong]

## Effort Estimate
[small/medium/large with rationale]

## Success Metrics
[how to measure if it works]
```

Update `.claude/pipeline/manifest.md` — mark product-spec complete, add context for next skill.

## Phase 0A: User Interview

### Required Questions

**1. What's the feature?**
"Describe what you want in your own words. What does the user do? What happens?"
→ Raw input. Don't over-structure yet.

**2. Why build this?**
"What problem does this solve? Who asked for it? What's the business case?"
→ Determines priority and success metrics.

**3. Who's it for?**
"All users, or a specific segment? (new users, power users, premium only)"
→ Affects scope and gating.

**4. What exists today?**
"Is this net new, or improving something that already exists?"
→ Determines whether this is additive or a refactor.

**5. Must-haves vs nice-to-haves:**
"If you could only ship one thing, what's the core?"
→ Forces prioritization early.

**6. Design vision:**
"Any apps that do something similar? What should this feel like?"
→ Sets the design benchmark.

**7. Timeline:**
"When do you need this? (this week, this sprint, this quarter)"
→ Scopes the v1 appropriately.

### Scope Guard — Push Back on Over-Scoping

After hearing the feature description, assess complexity. If the feature is actually 2+ features:

**STOP and say:**
"This is actually 3 features, not 1:
1. [Feature A] — the core interaction
2. [Feature B] — the social/sharing layer
3. [Feature C] — the analytics/reporting

I'd ship [Feature A] first. It delivers the core value in ~[estimate]. Want me to spec just that?"

Common over-scope patterns:
- "Build X and also let users share it" → X is one feature, sharing is another
- "Add a dashboard with charts and exports and alerts" → that's 3 features
- "Let users create, edit, delete, share, collaborate, and comment" → v1 is create+edit, v2 is share+collaborate

### Store Preferences

Save to `.claude/pipeline/preferences.md`:
```
## Product Spec Preferences
detail_level: [concise | detailed]
include_competitive_analysis: [yes | no]
include_effort_estimates: [yes | no]
```

## Phase 1: Codebase Research

Before writing the spec, understand what already exists.

### 1a. Existing Feature Scan

```bash
# What feature modules exist?
ls -d src/features/*/

# What screens exist?
find app/ -name "*.tsx" -not -name "_*" -not -name "*.test.*" | sort

# What types are defined?
find src/ -name "*Types.ts" -o -name "*types.ts" | sort

# What hooks exist?
find src/ -name "use*.ts" -not -name "*.test.*" | sort
```

### 1b. Related Code

```bash
# Search for anything related to the feature
grep -rn "[feature keyword]" src/ --include="*.ts" --include="*.tsx" | head -30

# Check if there's a partial implementation
grep -rn "[feature keyword]" app/ --include="*.tsx" | head -20

# Check for TODO/FIXME related to this feature
grep -rn "TODO\|FIXME\|HACK" src/ --include="*.ts" --include="*.tsx" | grep -i "[feature keyword]"
```

### 1c. Data Model

```bash
# Check existing database schema
ls supabase/migrations/*.sql 2>/dev/null | sort
grep -rn "CREATE TABLE\|ALTER TABLE" supabase/migrations/ | head -30

# Check TypeScript types
grep -rn "interface\|type " src/features/*/types.ts | head -30

# Check existing API calls
grep -rn "\.from(\|\.rpc(" src/ --include="*.ts" | head -20
```

### 1d. Design System

```bash
# Check available components
ls src/components/ui/ src/components/cards/ 2>/dev/null

# Check theme tokens
grep -rn "export" src/constants/theme.ts | head -30
```

Present findings: "Here's what already exists that's relevant to this feature: [summary]. We can reuse [X, Y, Z]."

## Phase 2: Competitive Analysis

If the user wants competitive analysis (or the feature is complex enough to warrant it):

### Research Approach

1. **Identify benchmark apps** that have this feature
2. **Document how they implement it**: UI patterns, user flow, edge cases handled
3. **Note what works and what doesn't** in each implementation
4. **Recommend the best approach** for this app's context

```markdown
## Competitive Analysis: [Feature]

### [App 1]
- How they do it: [description]
- What works: [strengths]
- What doesn't: [weaknesses]
- Screenshot/reference: [URL or description]

### [App 2]
...

### Recommended Approach
Based on [App X]'s approach, adapted for our context: [description]
```

## Phase 3: Scope Definition

### v1 Scope (Ship First)

Define the minimum version that delivers core value:

- **Include**: The core interaction, basic happy path, essential UI
- **Exclude**: Sharing, analytics, admin tools, edge case handling beyond basics
- **Data model**: Only the tables/fields needed for v1
- **UI**: Only the screens needed for the core flow

### Deferred (Later)

Explicitly list what's NOT in v1:

```markdown
### Deferred to v2
- [ ] Social sharing — users can share [X] with friends
- [ ] Analytics dashboard — track [X] metrics
- [ ] Bulk operations — do [X] for multiple items
- [ ] Advanced filters — filter by [criteria]

### Deferred Indefinitely
- [ ] [Feature that sounds cool but doesn't serve the core use case]
```

### Scope Decision Framework

Ask for each sub-feature:
1. Can users get value without this? → If yes, defer
2. Is this blocking another feature? → If yes, include
3. Does this require new infrastructure? → If yes, carefully consider
4. Can this be added later without breaking changes? → If yes, defer

## Phase 4: Write the Spec

### User Stories

Format each as:

```markdown
### US-1: [Title]
**As a** [user type]
**I want to** [action]
**So that** [benefit]

**Acceptance Criteria:**
- [ ] [Specific, testable criterion]
- [ ] [Another criterion]
- [ ] [Edge case handling]

**Notes:** [any context, constraints, or design considerations]
```

### Data Model

```markdown
## Data Model

### New Tables

#### [table_name]
| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| id | uuid | no | gen_random_uuid() | PK |
| user_id | uuid | no | — | FK → profiles.id |
| ... | ... | ... | ... | ... |
| created_at | timestamptz | no | now() | — |

**RLS Policies:**
- SELECT: users can read their own + public items
- INSERT: authenticated users only
- UPDATE: owner only
- DELETE: owner only

### Modified Tables

#### [existing_table]
- ADD COLUMN [name] [type] — [reason]

### Indexes
- [table]([column]) — speeds up [query pattern]
```

### API Requirements

```markdown
## API Requirements

### New Endpoints

#### [method] /api/[path]
- **Auth:** required
- **Input:** { field: type, ... }
- **Output:** { field: type, ... }
- **Errors:** 401 (unauthorized), 404 (not found), 422 (validation)
- **Notes:** [rate limiting, caching, etc.]

### Edge Functions (Supabase)

#### [function-name]
- **Purpose:** [what it does]
- **Input:** { ... }
- **Output:** { ... }
- **Dependencies:** [external APIs, AI calls, etc.]
```

### UI Descriptions (Text Wireframes)

```markdown
## UI Screens

### Screen 1: [Name]
**Route:** app/[path].tsx
**Layout:**
- Header: [back button] [title] [action button]
- Body:
  - [Section 1]: [description of content and layout]
  - [Section 2]: [description]
- Footer: [primary CTA button]

**States:**
- Loading: skeleton screen
- Empty: [empty state message and CTA]
- Error: StateCard with error tone
- Loaded: [normal content]

**Interactions:**
- Tap [element] → [action/navigation]
- Swipe [element] → [action]
- Long press [element] → [action]

**Dark mode notes:** [any specific considerations]
```

### Risk Assessment

```markdown
## Risks

### Technical Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| [risk] | [H/M/L] | [H/M/L] | [how to avoid] |

### Product Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Users don't understand [X] | M | H | Add onboarding tooltip |

### Dependencies
- [External API] — needed for [feature]. Fallback: [alternative]
- [Another team/service] — needed for [feature]. Timeline: [estimate]
```

### Effort Estimate

```markdown
## Effort Estimate

**Overall: [SMALL | MEDIUM | LARGE]**

| Component | Estimate | Notes |
|-----------|----------|-------|
| Data model + migration | [S/M/L] | [X new tables, Y modified] |
| Backend (edge functions, API) | [S/M/L] | [X new endpoints] |
| Frontend (screens, components) | [S/M/L] | [X new screens, Y reused] |
| Tests | [S/M/L] | [X unit, Y integration] |
| Design audit | [S/M/L] | [new screens need review] |

**Size reference:**
- SMALL: < 2 hours, < 5 files changed, no new tables
- MEDIUM: 2-8 hours, 5-15 files, maybe 1 new table
- LARGE: 8+ hours, 15+ files, multiple new tables, new edge functions
```

### Success Metrics

```markdown
## Success Metrics

### Launch Metrics (first 2 weeks)
- [ ] [X]% of active users try the feature
- [ ] [Y]% completion rate of the core flow
- [ ] Zero CRITICAL bugs reported

### Growth Metrics (first 2 months)
- [ ] [X] monthly active users of the feature
- [ ] [Y]% retention — users come back to use it again
- [ ] Feature contributes to [Z] overall app metric

### Quality Metrics
- [ ] < [X]ms p95 response time
- [ ] < [Y]% error rate
- [ ] [Z]+ WCAG AA accessibility score
```

## Phase 5: Review & Approval

Present the spec to the user for review:

1. **Summary**: "Here's the spec for [feature]. It's [SMALL/MEDIUM/LARGE] — about [time estimate]."
2. **Key decisions**: "I scoped v1 to [X, Y, Z] and deferred [A, B, C]. The main risk is [risk]."
3. **Ask for approval**: "Does this match what you want? Anything to add, remove, or change?"

Wait for the user to approve before the pipeline moves to implementation.

If the user wants changes, iterate on the spec. Don't proceed to dev until the spec is approved.

## Autonomous Mode

When running autonomously (invoked by /ship or a subagent):

- **Skip interview** if the feature description is already in the manifest or provided by the orchestrator
- **Always do codebase research** — never spec blind
- **Use conservative scope** — v1 should be minimal, defer aggressively
- **Write spec.md** to the pipeline feature directory
- **Flag for human approval** — mark spec as DRAFT, do not let downstream skills proceed without explicit approval
- **If the feature description is too vague**, write what you can and list open questions in the spec

## Anti-Patterns

- DON'T write a spec without reading the codebase first — you'll miss existing implementations
- DON'T include everything in v1 — aggressive scoping is a feature, not a bug
- DON'T skip the "why" — user stories without business context lead to wrong implementations
- DON'T spec implementation details (specific React patterns, exact component names) — that's the dev's job
- DON'T write a 20-page spec for a small feature — match detail to complexity
- DON'T skip risks — every feature has at least one risk worth noting
- DON'T spec features that conflict with existing ones without noting the conflict
