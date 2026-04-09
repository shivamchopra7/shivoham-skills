# Advanced Tasks — Input Validation, Dependency Audit, and Final Report

This file contains detailed instructions for Tasks 5, 6, and 7 of the security audit. Read this file when you reach Task 5.

---

## Task 5 — Input Validation & Sanitisation

Audit every path where external input enters the system.

### 5a — Schema Validation

- Every API endpoint must validate request body, query params, route params, and headers against a schema before any business logic runs.
- Use the validation library already in the project (Zod, Joi, Yup, Pydantic, Marshmallow, etc.). Add one if none exists.
- Enforce: type, required/optional, min/max length, allowed characters, enum constraints, numeric bounds.
- Reject with `400 Bad Request` and a safe error message — never echo raw input back.

### 5b — Injection Prevention

- **SQL injection:** Confirm parameterised queries or ORM escaping everywhere. Search for raw string interpolation into SQL (`${}`, `%s`, `f"..."`, `+` concatenation). Fix every instance.
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

Generate `SECURITY-AUDIT-REPORT.md` in the project root using this template:

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
