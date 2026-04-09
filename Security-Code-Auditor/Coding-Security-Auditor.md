# Full Web App Security Audit

## Role & Context

You are a senior application-security engineer performing a full security audit of this codebase. Before touching anything, read the project's README, package manifest (package.json, requirements.txt, Gemfile, go.mod — whichever exists), and directory tree so you understand the stack, framework, and architecture. Every decision you make must be informed by the actual stack — do not assume Node/Express.

## Ground Rules

- **Git discipline:** Create a branch `security/audit-YYYY-MM-DD` before any changes. Make one atomic commit per task (six commits total). Never commit secrets, even temporarily.
- **Zero runtime breakage:** After every task that changes code, run the project's existing test suite. If tests fail because of your change, fix the issue before moving on. If no test suite exists, note this in the final report as a Critical risk.
- **Minimal footprint:** Prefer the library already in the codebase. Only add a new dependency when nothing suitable exists. Pin every new dependency to an exact version.
- **Leave a trail:** Add a brief inline comment (`// SECURITY-AUDIT:`) next to every non-obvious change so the team can review your reasoning.

---

## Task 1 — Detect & Map the Stack (Prerequisite)

Before any security work, build a mental model of the project:

1. Identify language, framework, ORM/database layer, auth mechanism, and deployment target.
2. List every entry point for user input: routes, controllers, GraphQL resolvers, WebSocket handlers, cron jobs, CLI commands.
3. Map how secrets are currently managed (hardcoded, .env, vault, cloud secrets manager).
4. Identify the frontend/client boundary — what gets bundled and shipped to the browser.

Do not write a report for this step. Use what you learn to make every subsequent task precise.

---

## Task 2 — Rate Limiting

Apply rate limiting to every HTTP entry point. Adapt thresholds to the route's risk profile:

| Route Category | Limit | Window |
|---|---|---|
| Auth (login, signup, reset, OTP/MFA verify) | 5 requests | 15 min per IP |
| Account-sensitive (password change, email change, API-key rotation) | 10 requests | 15 min per IP |
| Write endpoints (POST/PUT/PATCH/DELETE) | 60 requests | 1 min per IP |
| Read endpoints (GET) | 120 requests | 1 min per IP |
| Webhooks / public callbacks | 30 requests | 1 min per source IP |

### Execution checklist

- [ ] If a rate-limiter already exists, extend it — do not replace it.
- [ ] If not, install one idiomatic to the stack (e.g., `express-rate-limit` + `rate-limit-redis` for Express, `django-ratelimit` for Django, `rack-attack` for Rails).
- [ ] Return a `429 Too Many Requests` response with a `Retry-After` header.
- [ ] Make limits configurable via environment variables so ops can tune without a deploy.
- [ ] If the app runs behind a reverse proxy (Nginx, Cloudflare), configure trust-proxy / X-Forwarded-For handling so rate limiting uses the real client IP, not the proxy IP.

**Commit:** `security: add rate limiting to all API endpoints`

---

## Task 3 — Secret Scanning

Scan the entire repository — tracked files **and** git history — for hardcoded secrets.

### What to look for

- API keys, tokens, passwords, private keys, JWTs, DSNs, connection strings
- Secrets in source files, config files, scripts, seed/fixture files, test files, Dockerfiles, CI configs, and IaC templates
- `.env` / `.env.*` files accidentally tracked by git (`git ls-files '*.env*'`)
- High-entropy strings that look like secrets (use a scanner like `trufflehog`, `gitleaks`, or `detect-secrets` — pick one and run it)

### Output format

For every finding, log:

```
[SECRET-SCAN] <severity: CRITICAL|HIGH> | <file>:<line> | <type: api-key|password|token|...> | <redacted-preview: first 4 chars + ***>
```

Do **not** print the full secret value to stdout or to any file.

**Commit:** no code changes — findings feed into Task 4.

---

## Task 4 — Environment Variable Migration

For every secret found in Task 3 (plus any you discover along the way):

1. Add the variable to `.env.example` with a placeholder value and a descriptive comment.
2. Add the real value to `.env` (which must be in `.gitignore` — add it if missing).
3. Replace the hardcoded value in source with the stack-idiomatic env accessor:
   - Node: `process.env.VAR_NAME`
   - Python: `os.environ["VAR_NAME"]` or `settings.VAR_NAME` via pydantic / django-environ
   - Ruby: `ENV.fetch("VAR_NAME")`
   - Go: `os.Getenv("VAR_NAME")`
4. Add a startup check that fails fast if any required secret is missing (don't let the app boot with `undefined` credentials).
5. Verify no secret is importable by the frontend bundle:
   - For Vite/Next.js/CRA: ensure no secret uses the `VITE_` / `NEXT_PUBLIC_` / `REACT_APP_` prefix.
   - For SSR frameworks: ensure secrets are only accessed server-side.

### Validation

- Run `grep -rn` for the old hardcoded value across the repo to confirm zero remaining occurrences.
- Confirm `.env` appears in `.gitignore`.

**Commit:** `security: migrate all hardcoded secrets to environment variables`

---

## Task 5 — Input Validation & Sanitisation

Audit every path where external input enters the system. For each, apply defence-in-depth:

### 5a — Schema Validation

- Every API endpoint must validate request body, query params, route params, and headers against a schema **before** any business logic runs.
- Use the validation library already in the project (Zod, Joi, Yup, Pydantic, Marshmallow, dry-validation, etc.). If none exists, add one.
- Enforce: type, required/optional, min/max length, allowed characters, enum constraints, numeric bounds.
- Reject with `400 Bad Request` and a safe error message (never echo raw input back in the error).

### 5b — Injection Prevention

- **SQL injection:** Confirm every database query uses parameterised queries or the ORM's built-in escaping. Search for raw string interpolation into SQL (`${}`, `%s`, `f"..."`, `+` concatenation). Fix every instance.
- **NoSQL injection:** If MongoDB/DynamoDB is in use, ensure query operators (`$gt`, `$ne`, etc.) cannot be injected via user input.
- **XSS:** Confirm all user-generated content is escaped before rendering in HTML. If a template engine is used, verify auto-escaping is enabled. If React/Vue, confirm no `dangerouslySetInnerHTML` / `v-html` with unsanitised input.
- **Command injection:** Search for `exec`, `spawn`, `system`, `os.popen`, backtick execution. If user input flows into any of these, refactor to use parameterised APIs or allowlists.
- **Path traversal:** Any file operation that accepts a user-supplied filename must resolve the path and confirm it stays within the allowed directory.

### 5c — File Uploads (if applicable)

- Validate MIME type (check magic bytes, not just the extension or Content-Type header).
- Enforce a max file size.
- Generate a random filename on the server — never use the client-supplied name for storage.
- Store uploads outside the webroot or in object storage.

**Commit:** `security: add input validation and sanitisation across all endpoints`

---

## Task 6 — Dependency & Configuration Audit

### 6a — Dependency vulnerabilities

- Run the stack's native audit tool: `npm audit`, `pip-audit`, `bundle audit`, `govulncheck`, `cargo audit`.
- For every finding with severity High or Critical: upgrade the package if a fix exists, or document why you can't (e.g., breaking change that needs manual migration).
- Remove unused dependencies.

### 6b — HTTP security headers

Verify or add a security-headers middleware. Required headers:

| Header | Required Value |
|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` |
| `Content-Security-Policy` | Strict policy — no `unsafe-inline` / `unsafe-eval` unless the framework requires it and you document why |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` (or `SAMEORIGIN` if the app uses iframes internally) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Deny unused browser features (`camera=(), microphone=(), geolocation=()`) |

### 6c — CORS

- If CORS is enabled, verify the allowed origin list is explicit — no wildcard `*` on authenticated endpoints.
- Verify `Access-Control-Allow-Credentials` is only set when the origin is allowlisted.

### 6d — Cookie & Token configuration

- Auth cookies: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict` if the app doesn't need cross-site requests).
- JWT / session token expiry: access tokens ≤ 15 minutes, refresh tokens ≤ 7 days. Flag anything longer.
- Verify tokens are invalidated on password change and logout.

### 6e — TLS & miscellaneous

- Confirm no HTTP-only endpoints exist in production config.
- Check for debug mode, verbose error output, or stack traces exposed in production responses.
- Verify logging does not write secrets or PII to log files.

**Commit:** `security: fix dependency vulnerabilities and harden configuration`

---

## Task 7 — Final Report

After all tasks are complete, generate a file called `SECURITY-AUDIT-REPORT.md` in the project root with this structure:

```markdown
# Security Audit Report
**Date:** YYYY-MM-DD
**Auditor:** Claude Code
**Codebase:** <repo name / path>
**Stack:** <detected language, framework, DB, deployment>

## Executive Summary
<3–5 sentences: overall posture before and after, highest-severity items>

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
- If a task is not applicable (e.g., no file uploads exist), state that explicitly and move on.
- If you are uncertain whether a change is safe, **flag it in the report instead of applying it.**
- After the final commit, run the full test suite one last time and report the result.
