---
name: api-spec
description: API specification pipeline — auto-generates OpenAPI 3.0 specs from code, validates request/response contracts between client and server, tests endpoint behavior, checks auth/rate-limiting, detects breaking changes, and generates documentation. Use when the user asks to document APIs, generate specs, validate contracts, or check for breaking changes.
---

# API Spec Skill

End-to-end API specification pipeline: Detect → Extract → Validate → Document → Report.

## When to Use

- User says "document my API", "generate spec", "check API contracts", "find breaking changes"
- Before a release to verify API stability
- After modifying endpoints or data models
- When onboarding new developers who need API docs
- When client and server types are out of sync

## Phase 0: Pipeline Context

At the start of every run:

1. **Read the pipeline manifest** (if it exists):
   ```
   .claude/pipeline/manifest.md
   ```
   This tells you what feature is being audited, what other skills have already run, and any shared context.

2. **When finished**, write your report to:
   ```
   .claude/pipeline/features/[slug]/api-spec-report.md
   ```
   Where `[slug]` comes from the manifest. If no manifest exists (standalone run), write to:
   ```
   .claude/pipeline/api-spec-report.md
   ```

## Phase 0A: User Preferences Interview

Before doing anything, ask these questions. Wait for answers.

### Required Questions

**1. API surface:**
"What API endpoints exist in your project?"
- REST API routes (Express, Next.js API routes, Django, Rails, FastAPI)
- Supabase Edge Functions
- Supabase RPC functions
- GraphQL
- tRPC
- All of the above / let me detect

**2. Consumers:**
"Who calls these APIs?"
- Mobile app (React Native, Flutter, Swift, Kotlin)
- Web app (Next.js, React, Vue)
- Third-party integrations
- Internal services only
- Multiple consumers

Determines how strict contract validation should be.

**3. Versioning:**
"Do you version your API?"
- URL versioning (/v1/, /v2/)
- Header versioning
- No versioning
- Not sure

Determines whether to check for breaking changes between versions.

**4. Existing documentation:**
"Do you have any existing API docs?"
- OpenAPI/Swagger spec file
- Postman collection
- README with endpoints listed
- Nothing / auto-detect from code

**5. Scope:**
- Full API (every endpoint)
- Specific endpoints (user lists which)
- New/changed endpoints only
- Contract validation only (client vs server types)

**6. Implementation preference:**
After the audit, do you want me to:
- Just generate the spec (no code changes)
- Generate spec + validate contracts + report mismatches
- Generate spec + fix type mismatches automatically
- Full autopilot (generate, validate, fix, document)

### Store Preferences

Save responses to memory for future runs:
```
memory/feedback_api_spec_preferences.md
```

On subsequent runs, ask: "I have your API preferences from last time — [summary]. Still accurate, or want to update anything?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after spec generation for user review
- STOP after contract validation for user approval
- Only fix what the user explicitly approves

### Autonomous Mode (--dangerously-skip-permissions or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** Fall back to safe defaults:
  - API surface: auto-detect
  - Scope: full API
  - Implementation: generate spec + report only (NO auto-fix)

- **Phase 2: Analysis runs fully — no stop.**

- **Phase 3: Fix — ONLY auto-fix if ALL of these are true:**
  1. User's stored preference is "full autopilot"
  2. The fix is a type mismatch (not a behavioral change)
  3. The fix aligns client types with server types (not the reverse)
  4. The fix passes typecheck and tests after applying

  Otherwise: generate a report file and notify the user.

## Platform Detection

### Step 1: Detect API Framework

```bash
# Express / Fastify / Koa
grep -q '"express"\|"fastify"\|"koa"' package.json 2>/dev/null && echo "EXPRESS_LIKE"

# Next.js API routes
find app/api/ pages/api/ -name "*.ts" -o -name "*.tsx" 2>/dev/null | head -5 && echo "NEXTJS_API"

# Supabase Edge Functions
ls supabase/functions/ 2>/dev/null && echo "SUPABASE_FUNCTIONS"
find supabase/functions/ -name "index.ts" 2>/dev/null | head -10

# Supabase RPC
grep -rn "\.rpc(" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -10

# Django
ls manage.py 2>/dev/null && echo "DJANGO"
find . -name "views.py" -o -name "viewsets.py" -o -name "urls.py" 2>/dev/null | head -10

# Rails
ls config/routes.rb 2>/dev/null && echo "RAILS"
find app/controllers/ -name "*.rb" 2>/dev/null | head -10

# FastAPI
grep -q "fastapi" requirements.txt pyproject.toml 2>/dev/null && echo "FASTAPI"
grep -rn "@app\.get\|@app\.post\|@router\." . --include="*.py" 2>/dev/null | head -10

# Go
ls go.mod 2>/dev/null && echo "GO"
grep -rn "http\.HandleFunc\|gin\.Default\|echo\.New\|fiber\.New" . --include="*.go" 2>/dev/null | head -10

# GraphQL
grep -rq "graphql\|apollo\|typeDefs\|resolvers" package.json src/ 2>/dev/null && echo "GRAPHQL"

# tRPC
grep -rq "trpc\|@trpc" package.json src/ 2>/dev/null && echo "TRPC"
```

### Step 2: Find Existing Specs

```bash
# OpenAPI / Swagger
find . -name "openapi.*" -o -name "swagger.*" -o -name "api-spec.*" 2>/dev/null | head -5

# Postman collections
find . -name "*.postman_collection.json" 2>/dev/null | head -5

# API type definitions
find src/ -name "*api*types*" -o -name "*api*schema*" -o -name "*dto*" 2>/dev/null | head -10
```

### Step 3: Check for API Client

```bash
# How does the client call the API?
grep -rn "fetch(\|axios\.\|ky\.\|got\.\|superagent\|ofetch" src/ --include="*.ts" --include="*.tsx" | grep -v test | grep -v node_modules | head -20

# Supabase client calls
grep -rn "supabase\.\(from\|rpc\|functions\)" src/ --include="*.ts" --include="*.tsx" | grep -v test | head -20

# Generated API clients
find src/ -name "*api*client*" -o -name "*generated*" -o -name "*openapi*" 2>/dev/null | head -5
```

## Phase 1: Endpoint Extraction

### 1.1 Extract All Endpoints

**Next.js API Routes:**
```bash
find app/api/ pages/api/ -name "*.ts" -o -name "*.tsx" 2>/dev/null | sort
# For each file, extract: HTTP method, path, request body type, response type
```

**Express/Fastify:**
```bash
grep -rn "app\.get\|app\.post\|app\.put\|app\.patch\|app\.delete\|router\.get\|router\.post\|router\.put\|router\.patch\|router\.delete" src/ --include="*.ts" --include="*.js" | grep -v test
```

**Supabase Edge Functions:**
```bash
find supabase/functions/ -name "index.ts" 2>/dev/null
# For each: extract the request/response handling from the serve() function
```

**Supabase RPC:**
```bash
grep -rn "\.rpc(" src/ --include="*.ts" --include="*.tsx" | grep -v test
# Extract function names and parameter types

# Check the actual SQL function definitions
grep -rn "CREATE.*FUNCTION\|CREATE OR REPLACE FUNCTION" supabase/migrations/ --include="*.sql"
```

**Django:**
```bash
# URL patterns
grep -rn "path(\|url(\|re_path(" . --include="*.py" | grep -v test | grep -v migration
# View classes/functions
grep -rn "class.*View\|class.*ViewSet\|def.*request" . --include="*.py" | grep -v test | head -30
```

**Rails:**
```bash
cat config/routes.rb 2>/dev/null
grep -rn "def.*create\|def.*update\|def.*destroy\|def.*index\|def.*show" app/controllers/ --include="*.rb"
```

**FastAPI:**
```bash
grep -rn "@app\.\|@router\." . --include="*.py" | grep -v test
# FastAPI auto-generates OpenAPI from Pydantic models
```

### 1.2 Extract Types

For each endpoint, extract:
- Request method (GET, POST, PUT, PATCH, DELETE)
- Path (with parameter placeholders)
- Path parameters and their types
- Query parameters and their types
- Request body shape and types
- Response body shape and types
- Error response format
- Auth requirement (public or authenticated)

### 1.3 Build Endpoint Inventory

```markdown
| Method | Path | Auth | Request Body | Response Body | File |
|--------|------|------|-------------|---------------|------|
```

## Phase 2: Contract Validation

### 2.1 Client-Server Type Matching

For each endpoint, compare:
- What the **server** expects (request type) vs what the **client** sends
- What the **server** returns (response type) vs what the **client** expects

```bash
# Find client-side types for API responses
grep -rn "type.*Response\|interface.*Response\|type.*Payload\|interface.*Payload" src/ --include="*.ts" | head -30

# Find server-side types
grep -rn "type.*Request\|interface.*Request\|type.*Body\|interface.*Body" supabase/ app/api/ pages/api/ --include="*.ts" | head -30
```

Flag mismatches:
- **CRITICAL**: Field exists on server but not on client (data loss)
- **CRITICAL**: Field type differs (string vs number)
- **MAJOR**: Field exists on client but not on server (will always be undefined)
- **MINOR**: Optional field on one side, required on the other

### 2.2 Endpoint Contract Testing

For endpoints where a dev server is running, send sample requests:

```bash
# Check if server is running
curl -s http://localhost:3000/api/health 2>/dev/null && echo "API_RUNNING"

# Test each endpoint with minimal valid request
# POST endpoints: send required fields, verify response shape
# GET endpoints: call with required params, verify response shape
```

### 2.3 Error Response Consistency

```bash
# Extract all error responses
grep -rn "res\.status\|NextResponse.*status\|new Response.*status\|json.*error\|json.*message" src/ app/api/ supabase/functions/ --include="*.ts" | head -30

# Check: do all errors follow the same shape?
# Expected: { error: string, code?: string, details?: any }
```

## Phase 3: Security Checks

### 3.1 Authentication on Protected Endpoints

```bash
# List endpoints and their auth status
grep -rn "auth\|session\|token\|bearer\|authorize" app/api/ supabase/functions/ --include="*.ts" | head -30

# Find unprotected mutation endpoints (POST/PUT/DELETE without auth)
# This is CRITICAL — mutations should almost always require auth
```

### 3.2 Rate Limiting

```bash
grep -rn "rateLimit\|rate-limit\|throttle" app/api/ supabase/functions/ src/lib/ --include="*.ts" | head -10

# Public endpoints without rate limiting are MAJOR findings
```

## Phase 4: Breaking Change Detection

### 4.1 Compare Against Existing Spec

If an existing OpenAPI spec exists:
```bash
# Diff the generated spec against the existing one
# Flag: removed fields, changed types, removed endpoints, changed auth
```

### 4.2 Git-Based Change Detection

```bash
# What API files changed recently?
git diff HEAD~10 --name-only -- "app/api/" "supabase/functions/" "pages/api/" "src/**/api*" 2>/dev/null

# What type files changed?
git diff HEAD~10 --name-only -- "src/**/*types*" "src/**/*dto*" "src/**/*schema*" 2>/dev/null

# Detailed diff of API route changes
git diff HEAD~10 -- "app/api/" "supabase/functions/" 2>/dev/null | head -200
```

Flag breaking changes:
- **CRITICAL**: Endpoint removed
- **CRITICAL**: Required field removed from response
- **MAJOR**: Field type changed
- **MAJOR**: New required field in request body (existing clients will fail)
- **MINOR**: Optional field added (backward compatible)

## Phase 5: Pagination and Query Patterns

```bash
# Pagination implementation
grep -rn "limit\|offset\|page\|cursor\|after\|before\|hasMore\|nextPage\|totalCount" app/api/ supabase/functions/ --include="*.ts" | head -20

# List endpoints without pagination (MAJOR for large datasets)
grep -rn "\.select(" src/ --include="*.ts" | grep -v "\.limit\|\.range\|\.single" | head -20
```

## Phase 6: OpenAPI Spec Generation

Generate a complete OpenAPI 3.0 spec from the extracted endpoints:

```yaml
openapi: 3.0.3
info:
  title: [App Name] API
  version: 1.0.0
  description: Auto-generated from source code

servers:
  - url: http://localhost:3000
    description: Development
  - url: https://[production-url]
    description: Production

paths:
  /api/[path]:
    [method]:
      summary: [extracted from code comments or function name]
      security: [if auth required]
      parameters: [path + query params]
      requestBody: [if POST/PUT/PATCH]
      responses:
        200: [success shape]
        400: [validation error shape]
        401: [if auth required]
        404: [if applicable]
        500: [error shape]

components:
  schemas:
    [all extracted types]
  securitySchemes:
    [detected auth method]
```

## Phase 7: Report

### Severity Levels

- **CRITICAL**: Breaking change, missing auth on mutation, client-server type mismatch causing data loss
- **MAJOR**: Missing rate limiting, inconsistent error format, no pagination on list endpoint
- **MINOR**: Missing documentation, optional field mismatch, no request validation

### Report Format

```markdown
# API Spec Report — [date]

## Summary
- **Framework**: [detected stack]
- **Endpoints found**: [count]
- **Authenticated**: [count] / [total]
- **Rate limited**: [count] / [total]
- **CRITICAL**: [count] | **MAJOR**: [count] | **MINOR**: [count]

## Endpoint Inventory
| Method | Path | Auth | Rate Limited | Request Validated | Paginated |
|--------|------|------|-------------|-------------------|-----------|

## Contract Validation
| Endpoint | Client Type | Server Type | Status | Mismatches |
|----------|------------|-------------|--------|------------|

## Breaking Changes (since last spec)
| Change | Endpoint | Severity | Description |
|--------|----------|----------|-------------|

## Findings

### CRITICAL
| # | Issue | Endpoint | File | Description |
|---|-------|----------|------|-------------|

### MAJOR
| # | Issue | Endpoint | File | Description |
|---|-------|----------|------|-------------|

### MINOR
| # | Issue | Endpoint | File | Description |
|---|-------|----------|------|-------------|

## Generated OpenAPI Spec
[Path to the generated spec file]

## Proposed Fixes
[For each finding: exact file, current code, proposed fix]

## Recommended Next Steps
[Prioritized list]
```

## Phase 8: Fix Proposals

For each finding, propose a fix with:
1. **What**: Endpoint, file, current state
2. **Why**: What the issue causes (broken clients, security risk, etc.)
3. **Fix**: Specific code change
4. **Impact**: Which clients are affected
5. **Migration**: If breaking change, how to migrate clients

### Fix Rules
- NEVER remove endpoint auth to fix a contract mismatch
- NEVER change server response to match a wrong client type (fix the client instead)
- Prefer additive changes over modifications
- For breaking changes, propose a deprecation path rather than immediate removal
- After implementing, re-validate the contract

## Pipeline Report Output

When done, write the structured report to the pipeline path (see Phase 0). The report must include:
- Timestamp
- Severity summary (CRITICAL/MAJOR/MINOR counts)
- Complete endpoint inventory
- Contract validation results
- Every finding with file path
- Generated OpenAPI spec location
- Fix status (proposed / approved / applied / skipped)
