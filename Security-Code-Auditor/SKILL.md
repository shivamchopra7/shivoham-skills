---
name: security-code-auditor
description: >
  Full web app security audit — rate limiting, secret scanning, env migration, input validation,
  dependency audit, and config hardening. Use when auditing code for vulnerabilities, OWASP issues,
  hardcoded secrets, missing rate limits, injection flaws, or insecure HTTP headers.
version: 1.0.0
author: Shivam Chopra
tags:
  - security
  - audit
  - owasp
  - vulnerabilities
  - secrets
  - rate-limiting
  - input-validation
  - hardening
requires:
  - git
---

# Security Code Auditor

A structured, seven-task security audit for any web application. Works across stacks — Node, Python, Ruby, Go, and beyond. Each task produces one atomic git commit, and the final output is a comprehensive `SECURITY-AUDIT-REPORT.md`.

## When to Use

- You need a full security posture assessment of a web project
- You want to find and migrate hardcoded secrets to environment variables
- You need rate limiting added across API endpoints
- You want to audit inputs for SQL injection, XSS, command injection, or path traversal
- You need HTTP security headers, CORS, and cookie configuration reviewed
- You want a dependency vulnerability scan and remediation pass
- You are preparing for a penetration test or compliance review

## Overview

The audit runs seven sequential tasks, each building on the last:

1. **Detect & Map the Stack** — understand the project before touching it
2. **Rate Limiting** — protect every HTTP entry point
3. **Secret Scanning** — find hardcoded credentials in files and git history
4. **Env Variable Migration** — move secrets to `.env` with startup validation
5. **Input Validation & Sanitisation** — schema validation and injection prevention
6. **Dependency & Configuration Audit** — fix vulnerable deps, harden headers/CORS/cookies
7. **Final Report** — structured `SECURITY-AUDIT-REPORT.md`

For detailed instructions on Tasks 5-7 (input validation, dependency audit, and report template), see `references/advanced-tasks.md`.

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

- [ ] If a rate-limiter already exists, extend it — do not replace it.
- [ ] Otherwise install one idiomatic to the stack (`express-rate-limit` for Express, `django-ratelimit` for Django, `rack-attack` for Rails, etc.).
- [ ] Return `429 Too Many Requests` with a `Retry-After` header.
- [ ] Make limits configurable via environment variables so ops can tune without a deploy.
- [ ] If the app runs behind a reverse proxy, configure trust-proxy / `X-Forwarded-For` handling so rate limiting uses the real client IP.

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

## Tasks 5-7

For the remaining tasks — Input Validation & Sanitisation, Dependency & Configuration Audit, and the Final Report template — see `references/advanced-tasks.md`.

---

## What NOT to Do

- Do NOT assume the stack is Node/Express — detect it first in Task 1.
- Do NOT replace an existing rate-limiter — extend it.
- Do NOT print full secret values in scan output — redact after 4 characters.
- Do NOT commit secrets to git, even temporarily in a "fix" commit.
- Do NOT add dependencies when an existing library can do the job.
- Do NOT skip running the test suite after code-changing tasks.
- Do NOT apply a change you are uncertain about — flag it in the report instead.

## Execution Contract

- Work through tasks 1 → 7 sequentially. Do not skip ahead.
- If a task is not applicable (e.g., no file uploads), state that explicitly and move on.
- If uncertain whether a change is safe, flag it in the report instead of applying it.
- After the final commit, run the full test suite one last time and report the result.

## Examples

**Example 1: User request**
Input: "Run a security audit on this Express app"
Output: Creates `security/audit-2026-04-09` branch, executes Tasks 1-7, produces 6 commits and `SECURITY-AUDIT-REPORT.md`.

**Example 2: Targeted request**
Input: "Scan this repo for hardcoded secrets and move them to env vars"
Output: Runs Tasks 1, 3, and 4 — scans for secrets, migrates them to `.env`, adds startup validation.

**Example 3: Rate limiting only**
Input: "Add rate limiting to all my API routes"
Output: Runs Tasks 1 and 2 — detects the stack, applies risk-tiered rate limits with configurable thresholds.
