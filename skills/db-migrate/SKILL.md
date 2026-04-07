---
name: db-migrate
description: Database migration pipeline — generates migration SQL from schema changes, performs safety checks on destructive operations, generates rollback scripts, validates indexes and RLS policies, plans data backfills, and checks for table locks. Use when the user asks to create migrations, check schema changes, add columns/tables, or review migration safety.
---

# Database Migration Skill

End-to-end migration pipeline: Detect → Diff → Generate → Validate → Report → Apply.

## When to Use

- User says "create a migration", "add a column", "change the schema", "check migrations"
- When a new feature requires database schema changes
- Before running migrations in production to verify safety
- When reviewing existing migrations for issues
- When indexes, RLS policies, or constraints need updating

## Phase 0: Pipeline Context

At the start of every run:

1. **Read the pipeline manifest** (if it exists):
   ```
   .claude/pipeline/manifest.md
   ```
   This tells you what feature is being audited, what other skills have already run, and any shared context.

2. **When finished**, write your report to:
   ```
   .claude/pipeline/features/[slug]/db-migrate-report.md
   ```
   Where `[slug]` comes from the manifest. If no manifest exists (standalone run), write to:
   ```
   .claude/pipeline/db-migrate-report.md
   ```

## Phase 0A: User Preferences Interview

Before doing anything, ask these questions. Wait for answers.

### Required Questions

**1. Database and ORM:**
"What database setup are you using?"
- Supabase (PostgreSQL + RLS)
- PostgreSQL (raw SQL)
- Prisma (any DB)
- TypeORM / Drizzle / Knex
- Django ORM
- Rails ActiveRecord
- MongoDB (Mongoose)
- Other / let me detect

**2. Migration tool:**
"How do you manage migrations?"
- Supabase CLI (supabase/migrations/)
- Prisma Migrate
- Django makemigrations
- Rails db:migrate
- Flyway / Liquibase
- Raw SQL files
- Other / let me detect

**3. Environment:**
"Which environment is this migration for?"
- Local development only
- Staging (verify before production)
- Production (need maximum safety)

Determines how strict safety checks are. Production gets CRITICAL flags for anything that could cause downtime.

**4. Data volume:**
"Roughly how many rows in your largest tables?"
- Small (< 10K rows)
- Medium (10K - 1M rows)
- Large (1M - 100M rows)
- Very large (100M+ rows)

Determines whether to flag table locks and recommend batched operations.

**5. Downtime tolerance:**
"Can you take the app offline during migration?"
- Zero downtime required
- Brief downtime OK (< 5 minutes)
- Maintenance window available

Determines whether to use online DDL techniques.

**6. What changed:**
"What schema changes do you need?"
- Add new table(s)
- Add column(s) to existing table
- Modify column type or constraint
- Add/remove indexes
- Update RLS policies
- Drop table/column (destructive)
- Data backfill
- Let me detect from code changes

### Store Preferences

Save responses to memory for future runs:
```
memory/feedback_db_migrate_preferences.md
```

On subsequent runs, ask: "I have your migration preferences from last time — [summary]. Still accurate, or want to update anything?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after generating migration for user review
- STOP after safety checks for user approval
- NEVER auto-run migrations — always require explicit approval

### Autonomous Mode (--dangerously-skip-permissions or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** Fall back to safe defaults:
  - Environment: treat as production (maximum safety)
  - Downtime: zero tolerance
  - Implementation: generate + validate only (NEVER auto-run)

- **Phase 2: Generation runs fully — no stop.**

- **Phase 3: Apply — NEVER auto-apply migrations, even in autopilot.** Migrations are irreversible in production. Always:
  1. Generate the migration file
  2. Generate the rollback script
  3. Write the safety report
  4. Notify the user to review and apply manually

  The ONLY exception: local development with user's explicit stored preference for auto-apply.

## Platform Detection

### Step 1: Detect Database and ORM

```bash
# Supabase
ls supabase/ supabase/config.toml 2>/dev/null && echo "SUPABASE_PROJECT"
ls supabase/migrations/ 2>/dev/null && echo "SUPABASE_MIGRATIONS"
find supabase/migrations/ -name "*.sql" 2>/dev/null | wc -l

# Prisma
ls prisma/schema.prisma 2>/dev/null && echo "PRISMA"
grep -q '"prisma"' package.json 2>/dev/null && echo "PRISMA_PKG"

# Django
ls manage.py 2>/dev/null && echo "DJANGO"
find . -name "models.py" -path "*/models.py" 2>/dev/null | head -5

# Rails
ls db/migrate/ 2>/dev/null && echo "RAILS_MIGRATIONS"
ls db/schema.rb 2>/dev/null && echo "RAILS_SCHEMA"

# TypeORM
grep -q '"typeorm"' package.json 2>/dev/null && echo "TYPEORM"

# Drizzle
grep -q '"drizzle-orm"\|"drizzle-kit"' package.json 2>/dev/null && echo "DRIZZLE"

# Knex
grep -q '"knex"' package.json 2>/dev/null && echo "KNEX"

# Raw SQL
find . -name "*.sql" -path "*/migrations/*" 2>/dev/null | head -10
```

### Step 2: Check Current Schema State

```bash
# Supabase: list existing migrations
ls -la supabase/migrations/ 2>/dev/null

# Prisma: current schema
cat prisma/schema.prisma 2>/dev/null | head -100

# Django: show migrations status
python manage.py showmigrations 2>/dev/null | head -30

# Rails: schema
cat db/schema.rb 2>/dev/null | head -100

# Check for pending migrations
grep -rn "TODO\|FIXME\|PENDING" supabase/migrations/ prisma/ db/migrate/ 2>/dev/null
```

### Step 3: Detect What Changed

```bash
# Git diff for schema-related files
git diff HEAD --name-only -- "supabase/migrations/" "prisma/" "db/migrate/" "*/models.py" "*/models/" 2>/dev/null

# New type definitions that might need schema changes
git diff HEAD --name-only -- "src/**/*types*" "src/**/*model*" "src/**/*schema*" 2>/dev/null

# Code referencing columns/tables that might not exist yet
grep -rn "\.from(\|\.select(\|\.insert(\|\.update(" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -30
```

## Phase 1: Schema Diff Analysis

### 1.1 Compare Code Types to Database Schema

```bash
# TypeScript types that map to DB tables
grep -rn "type.*Row\|interface.*Row\|type.*Insert\|type.*Update" src/ --include="*.ts" | head -30

# Supabase generated types
ls src/types/supabase.ts src/lib/database.types.ts 2>/dev/null
# Compare against migration history

# Prisma: diff model vs DB
npx prisma db diff --from-schema-datasource prisma/schema.prisma --to-schema-datamodel prisma/schema.prisma 2>/dev/null
```

### 1.2 Identify Required Changes

From the diff, categorize changes:
- **New tables**: CREATE TABLE needed
- **New columns**: ALTER TABLE ADD COLUMN needed
- **Modified columns**: ALTER TABLE ALTER COLUMN needed (potentially destructive)
- **New indexes**: CREATE INDEX needed
- **New constraints**: ALTER TABLE ADD CONSTRAINT needed
- **Dropped objects**: DROP needed (destructive — flag immediately)
- **RLS policies**: CREATE POLICY / ALTER POLICY needed

## Phase 2: Migration Generation

### 2.1 Supabase Migration

Generate a timestamped migration file:

```bash
# Create migration file
# Format: supabase/migrations/YYYYMMDDHHMMSS_description.sql
```

Template:
```sql
-- Migration: [description]
-- Generated: [timestamp]
-- Safety: [safe / requires-review / destructive]

BEGIN;

-- Forward migration
[SQL statements]

COMMIT;
```

### 2.2 Prisma Migration

```bash
npx prisma migrate dev --name [description] --create-only 2>/dev/null
# Review generated SQL before applying
```

### 2.3 Django Migration

```bash
python manage.py makemigrations --name [description] 2>/dev/null
# Review generated migration file
```

### 2.4 Rails Migration

```bash
rails generate migration [Description] 2>/dev/null
# Review generated migration file
```

### 2.5 Rollback Script

For every migration, generate a corresponding rollback:

```sql
-- Rollback: [description]
-- Reverses: [migration filename]

BEGIN;

-- Reverse migration
[inverse SQL statements]

COMMIT;
```

**Rollback rules:**
- CREATE TABLE → DROP TABLE (if no data expected)
- ADD COLUMN → DROP COLUMN (WARNING: data loss)
- CREATE INDEX → DROP INDEX
- CREATE POLICY → DROP POLICY
- ALTER COLUMN type → ALTER COLUMN back (may fail if data doesn't fit)
- DROP anything → CANNOT auto-rollback (flag as irreversible)

## Phase 3: Safety Checks

### 3.1 Destructive Operation Detection

Flag these with severity:

```bash
# CRITICAL: Data-destroying operations
grep -in "DROP TABLE\|DROP COLUMN\|TRUNCATE\|DELETE FROM" [migration_file]

# MAJOR: Type changes that may fail or lose data
grep -in "ALTER.*TYPE\|ALTER.*SET NOT NULL\|ALTER.*DROP DEFAULT" [migration_file]

# MAJOR: Unique constraints on existing data (may fail)
grep -in "ADD CONSTRAINT.*UNIQUE\|CREATE UNIQUE INDEX" [migration_file]
```

### 3.2 Table Lock Analysis

Operations that lock tables (dangerous on large tables):

| Operation | Lock Type | Safe on Large Tables? |
|-----------|-----------|----------------------|
| ADD COLUMN (nullable, no default) | ACCESS EXCLUSIVE (brief) | Yes |
| ADD COLUMN (with DEFAULT) | ACCESS EXCLUSIVE (PG < 11: rewrites table) | PG 11+: Yes |
| ADD COLUMN NOT NULL (no default) | FAILS | No — needs backfill first |
| ALTER COLUMN TYPE | ACCESS EXCLUSIVE (rewrites) | No — use shadow column |
| CREATE INDEX | SHARE lock (blocks writes) | Use CONCURRENTLY |
| DROP COLUMN | ACCESS EXCLUSIVE (brief) | Yes (marks as dropped) |
| ADD CONSTRAINT | SHARE lock | Use NOT VALID + VALIDATE |

For large tables, recommend:
```sql
-- Instead of: CREATE INDEX idx ON large_table(col);
-- Use: CREATE INDEX CONCURRENTLY idx ON large_table(col);

-- Instead of: ALTER TABLE ADD CONSTRAINT ... CHECK ...;
-- Use:
ALTER TABLE ADD CONSTRAINT ... CHECK ... NOT VALID;
ALTER TABLE VALIDATE CONSTRAINT ...;
```

### 3.3 Data Backfill Planning

If a NOT NULL column is added to a table with existing rows:

```sql
-- Step 1: Add column as nullable
ALTER TABLE [table] ADD COLUMN [col] [type];

-- Step 2: Backfill existing rows (batched for large tables)
UPDATE [table] SET [col] = [default_value]
WHERE [col] IS NULL;
-- For large tables, batch:
-- UPDATE ... WHERE id IN (SELECT id FROM ... WHERE col IS NULL LIMIT 10000);

-- Step 3: Set NOT NULL constraint
ALTER TABLE [table] ALTER COLUMN [col] SET NOT NULL;
```

### 3.4 Idempotency Check

Verify migrations are safe to re-run:

```sql
-- Good: idempotent
CREATE TABLE IF NOT EXISTS ...;
CREATE INDEX IF NOT EXISTS ...;
DO $$ BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'my_constraint') THEN
    ALTER TABLE ... ADD CONSTRAINT ...;
  END IF;
END $$;

-- Bad: will fail on re-run
CREATE TABLE ...;  -- without IF NOT EXISTS
ALTER TABLE ADD COLUMN ...;  -- without existence check
```

### 3.5 RLS Policy Validation (Supabase)

```bash
# Check: does every new table have RLS enabled?
grep -in "CREATE TABLE" [migration_file]
grep -in "ENABLE ROW LEVEL SECURITY\|RLS" [migration_file]
# If CREATE TABLE without ENABLE RLS → CRITICAL

# Check: do policies reference auth.uid()?
grep -in "CREATE POLICY" [migration_file]
# Verify policies are not overly permissive (e.g., USING (true))

# Check: existing tables that might need updated policies for new columns
grep -rn "CREATE POLICY\|USING\|WITH CHECK" supabase/migrations/ --include="*.sql" | tail -20
```

### 3.6 Index Coverage

```bash
# Foreign keys without indexes (slow JOINs and cascading deletes)
grep -in "REFERENCES\|FOREIGN KEY" supabase/migrations/ --include="*.sql" | head -20
# Cross-reference: does each FK column have an index?

# Frequently queried columns without indexes
grep -rn "\.eq(\|\.filter(\|WHERE\|\.match(" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -30
# Extract column names and check if they're indexed
```

## Phase 4: Report

### Severity Levels

- **CRITICAL**: Destructive operation without rollback, missing RLS on new table, NOT NULL without backfill on populated table, table lock on large table in zero-downtime environment
- **MAJOR**: Missing index on FK, non-idempotent migration, type change that may fail, missing rollback for additive change
- **MINOR**: Could use CONCURRENTLY, missing IF NOT EXISTS, unused index

### Report Format

```markdown
# Migration Report — [date]

## Summary
- **Database**: [detected DB/ORM]
- **Migration tool**: [detected tool]
- **Changes**: [count] tables, [count] columns, [count] indexes, [count] policies
- **CRITICAL**: [count] | **MAJOR**: [count] | **MINOR**: [count]

## Schema Changes
| Operation | Object | Table | Details | Destructive? |
|-----------|--------|-------|---------|-------------|

## Safety Analysis
| Check | Status | Details |
|-------|--------|---------|
| Destructive operations | [PASS/WARN/FAIL] | |
| Table locks | [PASS/WARN/FAIL] | |
| Data backfill needed | [PASS/WARN/FAIL] | |
| Idempotent | [PASS/WARN/FAIL] | |
| RLS policies | [PASS/WARN/FAIL] | |
| Index coverage | [PASS/WARN/FAIL] | |
| Rollback exists | [PASS/WARN/FAIL] | |

## Findings

### CRITICAL
| # | Issue | Table | Operation | Description |
|---|-------|-------|-----------|-------------|

### MAJOR
| # | Issue | Table | Operation | Description |
|---|-------|-------|-----------|-------------|

### MINOR
| # | Issue | Table | Operation | Description |
|---|-------|-------|-----------|-------------|

## Generated Files
- Migration: [path]
- Rollback: [path]

## Execution Plan
[Step-by-step instructions to safely apply the migration]

## Recommended Next Steps
[Prioritized list]
```

## Phase 5: Migration Proposals

For each required schema change, propose:
1. **What**: The SQL or ORM operation
2. **Why**: What feature/code needs this
3. **Safety**: Lock type, downtime risk, data impact
4. **Rollback**: How to reverse if something goes wrong
5. **Order**: Dependencies between migrations (e.g., create table before adding FK)

### Migration Rules
- ALWAYS generate a rollback script
- ALWAYS use IF NOT EXISTS / IF EXISTS for idempotency
- ALWAYS enable RLS on new Supabase tables
- ALWAYS add indexes on foreign key columns
- NEVER auto-apply to production
- NEVER drop columns without explicit user approval
- NEVER change column types without verifying existing data fits
- For large tables, ALWAYS use CONCURRENTLY for indexes
- For NOT NULL columns, ALWAYS do the 3-step add-backfill-constrain pattern
- Present the full migration for review before any application

## Pipeline Report Output

When done, write the structured report to the pipeline path (see Phase 0). The report must include:
- Timestamp
- Severity summary (CRITICAL/MAJOR/MINOR counts)
- Every schema change with safety analysis
- Generated migration file path
- Generated rollback file path
- Fix status (proposed / approved / applied / skipped)

## Anti-Patterns

- DON'T run migrations on production without explicit user confirmation
- DON'T generate DROP TABLE or DROP COLUMN without a rollback script
- DON'T add NOT NULL columns without a default value or backfill plan
- DON'T create indexes on large tables without CONCURRENTLY (locks the table)
- DON'T assume the migration tool — detect Prisma, Supabase, Rails, Django, etc.
- DON'T skip RLS policy updates when adding new tables (Supabase)
- DON'T generate a migration that depends on application code running first
- DON'T batch unrelated schema changes in one migration — one concern per migration
