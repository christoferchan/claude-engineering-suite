---
name: security-audit
description: Security audit pipeline — scans for exposed secrets, checks auth/authorization, validates input sanitization, audits dependencies for CVEs, verifies CORS/HTTPS/rate-limiting, and reports findings by OWASP Top 10 category. Use when the user asks to audit security, check for vulnerabilities, review auth, or harden their app before release.
---

# Security Audit Skill

End-to-end security audit pipeline: Detect → Scan → Analyze → Report → Fix.

## When to Use

- User says "audit security", "check for vulnerabilities", "review auth", "harden the app"
- Before a release to verify no secrets are exposed and auth is solid
- After adding new API endpoints or auth flows
- When integrating a new third-party service
- Periodic security hygiene check

## Phase 0: Pipeline Context

At the start of every run:

1. **Read the pipeline manifest** (if it exists):
   ```
   .claude/pipeline/manifest.md
   ```
   This tells you what feature is being audited, what other skills have already run, and any shared context.

2. **When finished**, write your report to:
   ```
   .claude/pipeline/features/[slug]/security-audit-report.md
   ```
   Where `[slug]` comes from the manifest. If no manifest exists (standalone run), write to:
   ```
   .claude/pipeline/security-audit-report.md
   ```

## Phase 0A: User Preferences Interview

Before doing anything, ask these questions. Wait for answers.

### Required Questions

**1. Threat model:**
"What's your biggest security concern right now?"
- Exposed secrets/API keys
- Auth bypass / broken access control
- Injection attacks (SQL, XSS, command)
- Dependency vulnerabilities
- Data privacy / PII leakage
- All of the above

Prioritizes which scans run first and which findings get CRITICAL severity.

**2. Environment:**
"Which environments should I audit?"
- Source code only (no running services)
- Source + local dev environment
- Source + staging/production config (provide URLs)

Determines whether to check live CORS headers, HTTPS redirects, etc.

**3. Sensitive data types:**
"What sensitive data does your app handle?"
- Passwords / auth tokens
- Payment / financial data
- Health / medical data
- Location data
- PII (names, emails, phone numbers)
- User-generated content

Determines which compliance standards to check against (HIPAA, PCI-DSS, GDPR).

**4. Known concerns:**
"Any specific areas you're worried about? (e.g., a new endpoint, a recently added auth flow)"

Audit these first with extra depth.

**5. Scope:**
- Full app (every file, every endpoint)
- Backend only (API routes, edge functions, DB)
- Frontend only (client-side secrets, XSS)
- Specific area (user names which)

**6. Implementation preference:**
After the audit, do you want me to:
- Just report findings (no code changes)
- Report + propose fixes with options
- Report + fix critical/high severity automatically
- Full autopilot (fix everything, ask only on ambiguous cases)

### Store Preferences

Save responses to memory for future runs:
```
memory/feedback_security_audit_preferences.md
```

On subsequent runs, ask: "I have your security preferences from last time — [summary]. Still accurate, or want to update anything?"

## Execution Modes

### Interactive Mode (default)
- Ask all Phase 0A interview questions
- STOP after analysis for user review
- STOP after proposals for user approval
- Only fix what the user explicitly approves

### Autonomous Mode (--dangerously-skip-permissions or user says "full autopilot")
- **Phase 0A: Skip interview IF preferences exist in memory.** Fall back to safe defaults:
  - Threat model: all categories
  - Scope: full app
  - Implementation: report + propose only (NO auto-fix)
  - Severity threshold for auto-fix: CRITICAL only

- **Phase 2: Scan runs fully — no stop.** Produces the report.

- **Phase 3: Fix — ONLY auto-fix if ALL of these are true:**
  1. User's stored preference is "full autopilot"
  2. The fix is severity CRITICAL
  3. The fix is clearly correct (e.g., removing an exposed secret, adding a missing auth check)
  4. The fix passes typecheck and tests after applying
  5. The fix does not change application behavior (only hardens it)

  Otherwise: generate a report file and notify the user.

## Platform Detection

Auto-detect the tech stack and security-relevant configuration.

### Step 1: Detect Framework and Backend

```bash
# Backend framework
ls package.json 2>/dev/null && echo "NODE_PROJECT"
grep -q '"express"\|"fastify"\|"koa"\|"hapi"' package.json 2>/dev/null && echo "EXPRESS_LIKE"
grep -q '"next"' package.json 2>/dev/null && echo "NEXTJS"
grep -q '"@supabase/supabase-js"' package.json 2>/dev/null && echo "SUPABASE"
ls requirements.txt setup.py pyproject.toml 2>/dev/null && echo "PYTHON"
grep -q "django" requirements.txt 2>/dev/null && echo "DJANGO"
grep -q "flask\|fastapi" requirements.txt 2>/dev/null && echo "FLASK_OR_FASTAPI"
ls Gemfile 2>/dev/null && echo "RUBY"
grep -q "rails" Gemfile 2>/dev/null && echo "RAILS"
ls go.mod 2>/dev/null && echo "GO"
ls Cargo.toml 2>/dev/null && echo "RUST"
```

### Step 2: Detect Auth Provider

```bash
grep -rq "firebase" src/ 2>/dev/null && echo "FIREBASE_AUTH"
grep -rq "supabase.*auth\|onAuthStateChange" src/ 2>/dev/null && echo "SUPABASE_AUTH"
grep -rq "next-auth\|NextAuth\|@auth/" src/ 2>/dev/null && echo "NEXTAUTH"
grep -rq "passport\|jsonwebtoken\|jwt" src/ 2>/dev/null && echo "JWT_AUTH"
```

### Step 3: Check Security-Relevant Files

```bash
# Environment files
ls .env .env.local .env.production .env.staging 2>/dev/null
cat .gitignore 2>/dev/null | grep -i "env\|secret\|key\|credential"

# Auth config
ls supabase/config.toml auth.config.* next-auth.* 2>/dev/null

# CORS config
grep -rn "cors\|Access-Control" src/ --include="*.ts" --include="*.js" | head -20

# Security headers
grep -rn "helmet\|csp\|Content-Security-Policy\|X-Frame-Options" src/ --include="*.ts" --include="*.js"
```

Present findings and note any missing security infrastructure.

## Phase 1: Secret Scanning

### 1.1 Hardcoded Secrets in Source

Scan for patterns that indicate exposed secrets:

```bash
# AWS keys
grep -rn "AKIA[0-9A-Z]{16}" src/ --include="*.ts" --include="*.tsx" --include="*.js"

# OpenAI keys
grep -rn 'sk-[a-zA-Z0-9]{48}' src/ --include="*.ts" --include="*.tsx"

# Stripe keys
grep -rn "sk_live_\|pk_live_" src/ --include="*.ts" --include="*.tsx"

# GitHub tokens
grep -rn "ghp_[a-zA-Z0-9]{36}" src/ --include="*.ts" --include="*.tsx"

# Slack tokens
grep -rn "xoxb-\|xoxp-" src/ --include="*.ts" --include="*.tsx"

# Generic password assignments (exclude types/mocks/schemas)
grep -rn "password\s*=\s*['\"]" src/ --include="*.ts" --include="*.tsx" | grep -vi "placeholder\|example\|test\|mock\|schema\|type\|interface"

# Hardcoded secrets (exclude env var references)
grep -rn "secret\s*[:=]\s*['\"][^'\"]*['\"]" src/ --include="*.ts" --include="*.tsx" | grep -vi "process\.env\|import\.meta"

# Bearer tokens in source
grep -rn "Bearer [a-zA-Z0-9._-]{20,}" src/ --include="*.ts" --include="*.tsx"

# Supabase keys not from env vars
grep -rn "supabase.*url\|supabase.*key\|supabase.*secret" src/ --include="*.ts" --include="*.tsx" | grep -vi "process\.env\|import\.meta\|NEXT_PUBLIC\|EXPO_PUBLIC"
```

### 1.2 Git History Secrets

```bash
# Check if secrets were ever committed (even if removed)
git log --all --diff-filter=A -- "*.env" "*.pem" "*.key" "*credentials*" "*secret*"
git log --all -p -S "sk-" --follow -- "*.ts" "*.tsx" "*.js" | head -100
```

### 1.3 Environment File Audit

```bash
# Are .env files tracked by git? (CRITICAL if so)
git ls-files --cached .env .env.local .env.production .env.staging 2>/dev/null

# Check .env has all vars needed (compare to .env.example if it exists)
diff <(grep -oP '^[A-Z_]+' .env.example 2>/dev/null | sort) <(grep -oP '^[A-Z_]+' .env 2>/dev/null | sort)
```

## Phase 2: Authentication Audit

### 2.1 Token and Session Handling

```bash
# Token storage — are tokens stored securely?
grep -rn "localStorage\|sessionStorage" src/ --include="*.ts" --include="*.tsx" | grep -i "token\|jwt\|session\|auth"

# AsyncStorage for mobile (insecure for tokens)
grep -rn "AsyncStorage.*token\|AsyncStorage.*jwt\|AsyncStorage.*session" src/ --include="*.ts" --include="*.tsx"

# SecureStore (Expo) — correct way for mobile
grep -rn "SecureStore\|expo-secure-store" src/ --include="*.ts" --include="*.tsx"

# Token expiry handling
grep -rn "tokenExpir\|refreshToken\|token.*refresh\|isExpired\|expiresAt" src/ --include="*.ts" --include="*.tsx"

# Session timeout
grep -rn "session.*timeout\|session.*expir\|idle.*timeout\|inactivity" src/ --include="*.ts" --include="*.tsx"
```

### 2.2 Auth Flow Security

```bash
# Password reset — check: rate limited? Token expires? One-time use?
grep -rn "resetPassword\|forgotPassword\|password.*reset" src/ --include="*.ts" --include="*.tsx"

# OAuth — check: state parameter? PKCE? Redirect URL validation?
grep -rn "OAuth\|oauth\|google.*auth\|apple.*auth" src/ --include="*.ts" --include="*.tsx"

# Magic link
grep -rn "magicLink\|magic.*link\|signInWithOtp\|passwordless" src/ --include="*.ts" --include="*.tsx"
```

### 2.3 Auth Guards

```bash
# Protected routes — do all sensitive routes check auth?
grep -rn "useSession\|useUser\|useAuth\|requireAuth\|isAuthenticated\|withAuth\|ProtectedRoute" src/ app/ --include="*.ts" --include="*.tsx"

# API route protection
grep -rn "getServerSession\|getSession\|verifyToken\|authenticate" src/ app/ pages/ --include="*.ts" --include="*.tsx" | grep -i "api\|route\|handler"

# Supabase RLS — are policies defined for all tables?
grep -rn "CREATE POLICY\|ALTER TABLE.*ENABLE ROW LEVEL SECURITY\|RLS" supabase/migrations/ --include="*.sql"
```

## Phase 3: Authorization Audit

### 3.1 Access Control

```bash
# Role checks
grep -rn "role\s*===\|isAdmin\|isModerator\|hasPermission\|canEdit\|canDelete\|authorize" src/ --include="*.ts" --include="*.tsx"

# IDOR — does the app verify ownership before data access?
grep -rn "params\.id\|params\.userId\|req\.params\.\|searchParams\." src/ --include="*.ts" --include="*.tsx" | head -40

# Supabase RLS — check policies reference auth.uid()
grep -rn "auth\.uid\(\)\|auth\.jwt\(\)" supabase/migrations/ --include="*.sql"
```

### 3.2 Data Exposure

```bash
# Select * — queries returning more fields than needed
grep -rn 'select(\s*['"'"'"]\\*['"'"'"])\|\.select()' src/ --include="*.ts" --include="*.tsx" | head -20

# Sensitive fields exposed to client
grep -rn "password\|ssn\|credit_card\|bank_account" src/ --include="*.ts" --include="*.tsx" | grep -i "type\|interface\|select\|return"
```

## Phase 4: Input Validation

### 4.1 SQL Injection

```bash
# Raw queries with string interpolation
grep -rn 'query\s*(\s*`\|\.raw\s*(' src/ --include="*.ts" --include="*.tsx" | grep -v "test\|mock"

# Template literals in SQL
grep -rn 'sql`.*\$\{' src/ supabase/ --include="*.ts" --include="*.tsx" --include="*.sql"
```

### 4.2 XSS Vectors

```bash
# React: unsafe HTML insertion
grep -rn "dangerouslySetInner\|__html\|innerHTML" src/ --include="*.ts" --include="*.tsx"

# Vue: v-html directive
grep -rn "v-html" src/ --include="*.vue"

# URL-based injection
grep -rn "window\.location\s*=\|location\.href\s*=" src/ --include="*.ts" --include="*.tsx" | grep -v "test"
```

### 4.3 Validation Libraries

```bash
# Check if input validation exists
grep -rn "zod\|yup\|joi\|class-validator\|superstruct\|valibot" package.json src/ --include="*.ts" --include="*.tsx"

# API route input validation — check for parse/validate before use
grep -rn "body\.\|req\.body\|request\.json\|req\.query" src/ app/ --include="*.ts" --include="*.tsx" | head -30
```

## Phase 5: Infrastructure Security

### 5.1 CORS

```bash
grep -rn "cors\|Access-Control-Allow-Origin\|Access-Control-Allow" src/ supabase/ --include="*.ts" --include="*.js" --include="*.toml"
# Flag: origin: "*" is CRITICAL for non-public APIs
```

### 5.2 Security Headers

```bash
grep -rn "helmet\|Content-Security-Policy\|X-Frame-Options\|X-Content-Type-Options\|Strict-Transport-Security\|Referrer-Policy" src/ --include="*.ts" --include="*.js" --include="*.tsx"

# Next.js headers config
grep -rn "headers\(\)" next.config.* --include="*.ts" --include="*.js" --include="*.mjs"
```

### 5.3 Rate Limiting

```bash
grep -rn "rateLimit\|rate-limit\|throttle\|rateLimiter\|express-rate-limit\|bottleneck" src/ package.json --include="*.ts" --include="*.js"

# Supabase edge function rate limiting
grep -rn "rate\|limit\|throttle" supabase/functions/ --include="*.ts"
```

### 5.4 HTTPS Enforcement

```bash
# HTTP links that should be HTTPS
grep -rn "http://" src/ --include="*.ts" --include="*.tsx" | grep -v "localhost\|127\.0\.0\.1\|0\.0\.0\.0\|http://schemas\|http://www\.w3"
```

## Phase 6: Dependency Audit

```bash
# npm audit
npm audit --json 2>/dev/null | head -100

# Outdated packages with known CVEs
npm outdated --json 2>/dev/null | head -50

# Python
pip-audit 2>/dev/null || safety check 2>/dev/null

# Ruby
bundle audit check 2>/dev/null
```

## Phase 7: Error Handling Audit

```bash
# Stack traces exposed to users (should be DEV-only)
grep -rn "console\.error\|console\.log" src/ --include="*.ts" --include="*.tsx" | grep -v "__DEV__\|test\|mock" | head -30

# Error responses that might leak internal info
grep -rn "error\.message\|error\.stack\|err\.message\|err\.stack" src/ --include="*.ts" --include="*.tsx" | grep -i "res\.\|response\.\|json\.\|return" | head -20

# Catch blocks that swallow errors silently
grep -rn "catch\s*(\s*)\|catch\s*{\s*}" src/ --include="*.ts" --include="*.tsx"
```

## Phase 8: Report

### Severity Levels

- **CRITICAL**: Exploitable now, data breach risk (exposed secrets, no auth on endpoints, SQL injection)
- **MAJOR**: Significant risk requiring fix before release (missing RLS, no rate limiting, insecure token storage)
- **MINOR**: Best practice violation, low risk (missing security headers, http links, verbose error messages)

### Report Format

```markdown
# Security Audit Report — [date]

## Summary
- **Scanned**: [file count] files, [endpoint count] endpoints
- **Framework**: [detected stack]
- **CRITICAL**: [count] | **MAJOR**: [count] | **MINOR**: [count]

## OWASP Top 10 Coverage
| Category | Status | Findings |
|----------|--------|----------|
| A01: Broken Access Control | [PASS/WARN/FAIL] | [count] |
| A02: Cryptographic Failures | [PASS/WARN/FAIL] | [count] |
| A03: Injection | [PASS/WARN/FAIL] | [count] |
| A04: Insecure Design | [PASS/WARN/FAIL] | [count] |
| A05: Security Misconfiguration | [PASS/WARN/FAIL] | [count] |
| A06: Vulnerable Components | [PASS/WARN/FAIL] | [count] |
| A07: Auth Failures | [PASS/WARN/FAIL] | [count] |
| A08: Data Integrity Failures | [PASS/WARN/FAIL] | [count] |
| A09: Logging Failures | [PASS/WARN/FAIL] | [count] |
| A10: SSRF | [PASS/WARN/FAIL] | [count] |

## Findings

### CRITICAL
| # | Issue | OWASP | File | Line | Description |
|---|-------|-------|------|------|-------------|

### MAJOR
| # | Issue | OWASP | File | Line | Description |
|---|-------|-------|------|------|-------------|

### MINOR
| # | Issue | OWASP | File | Line | Description |
|---|-------|-------|------|------|-------------|

## Proposed Fixes
[For each finding: exact file, line, current code, proposed fix, risk of change]

## Recommended Next Steps
[Prioritized list]
```

## Phase 9: Fix Proposals

For each finding, propose a fix with:
1. **What**: Exact file, line number, current code
2. **Why**: What the vulnerability enables
3. **Fix**: The specific code change
4. **Risk**: Could this fix break anything?
5. **Alternatives**: If there are multiple valid approaches, present options

### Fix Rules
- NEVER remove auth checks, even if they seem redundant
- NEVER weaken security (e.g., changing from strict to permissive CORS)
- NEVER commit or log secrets, even in fix descriptions
- Present options for any fix that changes application behavior
- After implementing, re-run the relevant scan to verify the fix works

## Pipeline Report Output

When done, write the structured report to the pipeline path (see Phase 0). The report must include:
- Timestamp
- Severity summary (CRITICAL/MAJOR/MINOR counts)
- Every finding with file path and line number
- Fix status (proposed / approved / applied / skipped)
