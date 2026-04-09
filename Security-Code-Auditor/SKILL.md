---
name: security-code-auditor
description: >
  Full web application security audit — rate limiting, secret scanning, env variable migration,
  input validation, dependency auditing, and configuration hardening with a structured final report.
  Use this skill whenever the user asks to audit a codebase for security issues, harden an application,
  scan for secrets or vulnerabilities, add rate limiting, migrate hardcoded credentials to env vars,
  validate inputs against injection attacks, or review HTTP security headers and CORS configuration.
  Also trigger when the user mentions "security audit", "pen test prep", "OWASP", "secret scanning",
  "dependency vulnerabilities", or wants a security posture assessment of any web project — regardless
  of language or framework.
---

# Security Code Auditor

You are a senior application-security engineer performing a full security audit. Before touching anything, read the project's README, package manifest (`package.json`, `requirements.txt`, `Gemfile`, `go.mod` — whichever exists), and directory tree so you understand the stack, framework, and architecture. Every decision must be informed by the actual stack — do not assume Node/Express.

## Ground Rules

- **Git discipline:** Create a branch `security/audit-YYYY-MM-DD` before any changes. One atomic commit per task (six commits total). Never commit secrets, even temporarily.
- **Zero runtime breakage:** After every code-changing task, run the project's existing test suite. If tests fail because of your change, fix it before moving on. If no test suite exists, note this in the final report as a Critical risk.
- **Minimal footprint:** Prefer libraries already in the codebase. Only add a new dependency when nothing suitable exists. Pin every new dependency to an exact version.
- **Leave a trail:** Add a brief inline comment (`// SECURITY-AUDIT:`) next to every non-obvious change so the team can review your reasoning.

---

## Task 1 — Detect & Map the Stack

Build a mental model of the project before any security work:

1. Identify language, framework, ORM/database layer, auth mechanism, and deployment target.
2. List every entry point for user input: routes, controllers, GraphQL resolvers, WebSocket handlers, cron jobs, CLI commands.
3. Map how secrets are currently managed (hardcoded, `.env`, vault, cloud secrets manager).
4. Identify the frontend/client boundary — what gets bundled and shipped to the browser.

This step produces no report — it informs every subsequent task.

---

## Task 2 — Rate Limiting

Apply rate limiting to every HTTP entry point, adapting thresholds to route risk:

| Route Category | Limit | Window |
| --- | --- | --- |
| Auth (login, signup, reset, OTP/MFA verify) | 5 req | 15 min / IP |
| Account-sensitive (password change, email change, API-key rotation) | 10 req | 15 min / IP |
| Write endpoints (POST/PUT/PATCH/DELETE) | 60 req | 1 min / IP |
| Read endpoints (GET) | 120 req | 1 min / IP |
| Webhooks / public callbacks | 30 req | 1 min / source IP |

### Checklist

- If a rate-limiter already exists, extend it — do not replace it.
- Otherwise install one idiomatic to the stack (`express-rate-limit` for Express, `django-ratelimit` for Django, `rack-attack` for Rails, etc.).
- Return `429 Too Many Requests` with a `Retry-After` header.
- Make limits configurable via environment variables so ops can tune without a deploy.
- If the app runs behind a reverse proxy, configure trust-proxy / `X-Forwarded-For` handling so rate limiting uses the real client IP.

**Commit:** `security: add rate limiting to all API endpoints`

---

## Task 3 — Secret Scanning

Scan the entire repository — tracked files **and** git history — for hardcoded secrets.

### What to look for

- API keys, tokens, passwords, private keys, JWTs, DSNs, connection strings.
- Secrets in source, config, scripts, seed/fixture files, test files, Dockerfiles, CI configs, IaC templates.
- `.env` / `.env.*` files accidentally tracked by git (`git ls-files '*.env*'`).
- High-entropy strings (use `trufflehog`, `gitleaks`, or `detect-secrets`).

### Output format

```text
[SECRET-SCAN] <severity: CRITICAL|HIGH> | <file>:<line> | <type> | <first 4 chars>***
```

Never print the full secret value. No code changes — findings feed into Task 4.

---

## Task 4 — Environment Variable Migration

For every secret found in Task 3:

1. Add the variable to `.env.example` with a placeholder and descriptive comment.
2. Add the real value to `.env` (must be in `.gitignore` — add it if missing).
3. Replace the hardcoded value with the stack-idiomatic env accessor:
   - Node: `process.env.VAR_NAME`
   - Python: `os.environ["VAR_NAME"]` or `settings.VAR_NAME`
   - Ruby: `ENV.fetch("VAR_NAME")`
   - Go: `os.Getenv("VAR_NAME")`
4. Add a startup check that fails fast if any required secret is missing.
5. Verify no secret is importable by the frontend bundle:
   - Vite/Next.js/CRA: no secret uses `VITE_` / `NEXT_PUBLIC_` / `REACT_APP_` prefix.
   - SSR frameworks: secrets accessed server-side only.

### Validation

- Grep for the old hardcoded value across the repo — confirm zero remaining occurrences.
- Confirm `.env` appears in `.gitignore`.

**Commit:** `security: migrate all hardcoded secrets to environment variables`

---

## Task 5 — Input Validation & Sanitisation

Audit every path where external input enters the system.

### 5a — Schema Validation

- Every API endpoint must validate request body, query params, route params, and headers against a schema before any business logic runs.
- Use the validation library already in the project (Zod, Joi, Yup, Pydantic, Marshmallow, etc.). Add one if none exists.
- Enforce: type, required/optional, min/max length, allowed characters, enum constraints, numeric bounds.
- Reject with `400 Bad Request` and a safe error message — never echo raw input back.

### 5b — Injection Prevention

- **SQL injection:** Confirm parameterised queries or ORM escaping everywhere. Search for raw string interpolation into SQL. Fix every instance.
- **NoSQL injection:** If MongoDB/DynamoDB is in use, ensure query operators (`$gt`, `$ne`) cannot be injected via user input.
- **XSS:** Confirm user-generated content is escaped before HTML rendering. Verify auto-escaping in template engines. Check for `dangerouslySetInnerHTML` / `v-html` with unsanitised input.
- **Command injection:** Search for `exec`, `spawn`, `system`, `os.popen`, backtick execution. Refactor to use parameterised APIs or allowlists if user input flows in.
- **Path traversal:** File operations accepting user-supplied filenames must resolve and constrain the path to the allowed directory.

### 5c — File Uploads (if applicable)

- Validate MIME type via magic bytes (not just extension or Content-Type header).
- Enforce max file size.
- Generate a random filename server-side — never use the client-supplied name.
- Store uploads outside the webroot or in object storage.

**Commit:** `security: add input validation and sanitisation across all endpoints`

---

## Task 6 — Dependency & Configuration Audit

### 6a — Dependency vulnerabilities

- Run the stack's native audit tool (`npm audit`, `pip-audit`, `bundle audit`, `govulncheck`, `cargo audit`).
- For High/Critical findings: upgrade if a fix exists, or document why not.
- Remove unused dependencies.

### 6b — HTTP security headers

Verify or add a security-headers middleware:

| Header | Required Value |
| --- | --- |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` |
| `Content-Security-Policy` | Strict — no `unsafe-inline` / `unsafe-eval` unless documented |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` (or `SAMEORIGIN` if iframes are used internally) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Deny unused browser features (`camera=(), microphone=(), geolocation=()`) |

### 6c — CORS

- Allowed origin list must be explicit — no wildcard `*` on authenticated endpoints.
- `Access-Control-Allow-Credentials` only set when the origin is allowlisted.

### 6d — Cookie & Token configuration

- Auth cookies: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict`).
- JWT / session token expiry: access tokens ≤ 15 min, refresh tokens ≤ 7 days. Flag anything longer.
- Tokens invalidated on password change and logout.

### 6e — TLS & miscellaneous

- No HTTP-only endpoints in production config.
- No debug mode, verbose error output, or stack traces exposed in production.
- Logging does not write secrets or PII.

**Commit:** `security: fix dependency vulnerabilities and harden configuration`

---

## Task 7 — Final Report

Generate `SECURITY-AUDIT-REPORT.md` in the project root:

```markdown
# Security Audit Report
**Date:** YYYY-MM-DD
**Auditor:** Claude Code
**Codebase:** <repo name / path>
**Stack:** <detected language, framework, DB, deployment>

## Executive Summary
<3-5 sentences: overall posture before and after, highest-severity items>

## Changes Applied

### Task 2 — Rate Limiting
| File | Change | Notes |
|------|--------|-------|
| ... | ... | ... |

### Task 3 — Secrets Found
| File:Line | Type | Status (migrated / needs manual rotation) |
|-----------|------|-------------------------------------------|
| ... | ... | ... |

### Task 4 — Env Migration
| Variable | Source File | Notes |
|----------|------------|-------|
| ... | ... | ... |

### Task 5 — Input Validation
| File | Vulnerability Class | Fix Applied |
|------|---------------------|-------------|
| ... | ... | ... |

### Task 6 — Dependencies & Config
| Item | Severity | Fix | Notes |
|------|----------|-----|-------|
| ... | ... | ... | ... |

## Unfixed Items
| Item | Reason Not Fixed | Recommended Action |
|------|------------------|--------------------|
| ... | ... | ... |

## Remaining Risks

### Critical
- ...

### High
- ...

### Medium
- ...

### Low
- ...

## Recommended Next Steps
1. ...
2. ...
3. ...
```

**Commit:** `security: add audit report`

---

## Execution Contract

- Work through tasks 1 → 7 sequentially. Do not skip ahead.
- If a task is not applicable (e.g., no file uploads), state that explicitly and move on.
- If uncertain whether a change is safe, flag it in the report instead of applying it.
- After the final commit, run the full test suite one last time and report the result.
