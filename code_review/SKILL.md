---
name: code-review
description: >
  Comprehensive, multi-dimensional code review skill grounded in best practices from
  "Looks Good to Me" (Braganza, 2025), the OWASP Code Review Guide v2, Learning
  Domain-Driven Design (Khononov), High Performance SRE (Mishra), Software Security
  for Developers (Saikali & Spilca), Supply Chain Software Security (Syed), and
  Identity Security for Software Development (Walsh et al.). Use this skill whenever
  the user asks to review code, critique a pull request, audit a module, check an
  implementation for correctness or security, or give feedback on code quality. Also
  trigger on "review this," "what's wrong with this code," "is this secure," "does
  this follow best practices," "check my PR," or "what do you think of these changes."
  The skill goes well beyond style — it checks for architectural soundness, domain
  model integrity, operational reliability, supply chain risks, identity/access
  vulnerabilities, and security flaws that automated linters miss.
---

<!--
  Methodology sourced from:
  - Braganza, A. "Looks Good to Me: Constructive Code Reviews." Manning, 2025.
  - OWASP Code Review Guide v2.
  - Khononov, V. "Learning Domain-Driven Design." O'Reilly, 2021.
  - Junker, A. "Mastering Domain-Driven Design." 2025.
  - Mishra, A. "High Performance SRE." 2024.
  - Saikali, A. & Spilca, L. "Software Security for Developers." Manning, 2025.
  - Syed, A. "Supply Chain Software Security." 2024.
  - Walsh, J., Ailon, U. & Barker, M. "Identity Security for Software Development." O'Reilly, 2025.
-->

# Code Review Skill

A good code review is not a style guide check — it is an act of professional responsibility.
Your job is to catch the things that escape linters and unit tests: design violations that will
haunt the codebase for years, reliability gaps that will wake someone up at 3 AM, and security
flaws that will become breach headlines. It is also an act of communication — comments should
be objective, specific, and kind.

---

## How to Approach the Review

1. **Read for intent first.** Before looking for problems, understand what the code is trying to
   do. A misunderstood patch produces shallow feedback.

2. **Prioritize by severity.** Lead with blockers, then architectural concerns, then style. Don't
   bury critical findings under minor comments.

3. **Be specific.** Point to the line or pattern. Explain *why* it matters, not just that it's
   wrong. If there's a better approach, show it.

4. **Distinguish blocking from advisory.** Mark every finding clearly:
   - 🔴 **Blocker** — must be fixed before merge (use `needs change:` or `needs rework:`)
   - 🟡 **Should fix** — significant concern, strong recommendation (use `align:`)
   - 🔵 **Consider** — worth thinking about, non-blocking (use `level up:`)

5. **Focus on the code, not the author.** Use "we" not "you." Ask, don't command.
   See `references/comment-patterns.md` for the full comment-writing framework.

---

## Step 1 — Gather Context

Before reviewing a line of code, understand what you're looking at.

**From the PR / change itself:**
- What is the stated purpose? (feature, fix, chore, breaking change, docs)
- What is the acceptance criteria or linked ticket?
- Are test steps or before/after comparisons provided?

**From the repository:**
- Explore the codebase to identify existing conventions: naming patterns, file organization,
  error handling style, logging approach, test structure, style guides, linter configs.
- Look at 2–3 files similar to those being changed to calibrate what "fits" here.

**Repo convention rule:** Changes should follow existing patterns *unless* the existing pattern
is outdated, violates idiomatic language conventions, has a known security risk, or a clearly
superior modern approach exists. When recommending a departure, say so explicitly and explain why.

---

## Step 2 — Review Lenses

Work through each lens. Not all apply to every PR — use judgment.

---

### Lens 1 — Domain Model Integrity
*Applies when: the code models a business domain — entities, aggregates, services, repositories, events.*

Read `references/domain-model.md` for the full checklist. Key signals to watch for inline:

- **Ubiquitous language** — Names should reflect domain concepts, not infrastructure.
  `userData` instead of `Customer` signals the model hasn't captured the domain.
- **Aggregate boundaries** — A single transaction should modify exactly one aggregate root.
  Cross-aggregate mutation without sagas or domain events is a consistency bug, not a style issue.
- **Value objects** — Must be immutable and compare by value, not reference.
- **Repositories** — Should load and save complete aggregates, never fragments.
- **Domain events** — Must be published reliably after commit (outbox pattern), not synchronously
  inside a transaction.
- **Bounded contexts** — Direct instantiation of classes from another bounded context without an
  anti-corruption layer is a boundary violation.

---

### Lens 2 — Reliability and Operability
*Applies when: the code runs in production, handles I/O, calls external services, or is part of a distributed system.*

Read `references/reliability.md` for the full checklist. Key signals to watch for inline:

- **Observability** — New code paths should emit structured logs, metrics (the four golden signals:
  latency, traffic, errors, saturation), and propagate distributed trace context.
- **Error handling** — Silent swallowing of exceptions is a 🔴 blocker. Every failure mode a
  caller might care about must be explicitly handled.
- **Timeouts and retries** — External calls without timeouts are latency time bombs. Retries must
  use exponential backoff with jitter to avoid cascading failures.
- **Resilience** — Flag single points of failure with no fallback. Ask: what happens when this
  dependency is unavailable?
- **Deployment safety** — Schema migrations must be backward-compatible. Ask: if this is deployed
  and immediately rolled back, does anything break?

---

### Lens 3 — Application Security
*Applies when: the code handles user input, calls a database, renders output, or manages authentication/sessions.*

See `references/security-checklist.md` for the full OWASP-based checklist with Go and Rust
language-specific patterns. Critical checks inline:

**Input validation and injection:**
- 🔴 Raw string concatenation into SQL queries is a blocker — use parameterized queries only.
- 🔴 User input passed to shell commands without strict allowlist validation is a blocker.
- 🔴 User-supplied data rendered to HTML must be encoded at the output layer; context matters
  (HTML body vs. attribute vs. JavaScript).
- All entry points validate type, length, format, and range before use.

**Authentication:**
- Passwords stored with bcrypt, scrypt, or Argon2 — never MD5, SHA-1, or unsalted SHA-256.
- Rate limiting on login endpoints (credential stuffing prevention).
- Prefer delegating to an established SSO/OIDC provider over building from scratch.

**Authorization:**
- Authorization checked on every protected *action*, not just at the entry point of a flow.
  Middleware guards don't protect against re-entrant or internal calls.
- 🔴 Endpoints using user-supplied IDs to look up resources without ownership verification are
  a blocker (broken object-level authorization / IDOR).

**Cryptography and secrets:**
- 🔴 Hardcoded secrets anywhere in source code or committed config files are a blocker.
- Standard library PRNG is not cryptographically secure — use the platform's CSPRNG for tokens,
  nonces, and salts.
- TLS enforced for all network communication, including internal service-to-service calls.

---

### Lens 4 — Supply Chain Integrity
*Applies when: the code adds dependencies, modifies build configuration, or touches CI/CD pipelines.*

**Dependency provenance:**
- New dependencies should come from reputable, actively maintained sources. Check for abandoned
  or recently transferred projects (the Event-Stream npm attack exploited a maintainer transfer).
- Verify transitive dependencies are tracked. A clean direct dep can have a compromised transitive one.
- An SBOM (Software Bill of Materials) should be updated for every release.
- **Go:** Run `govulncheck ./...`; ensure `go.sum` is committed.
- **Rust:** Run `cargo audit`; use `cargo deny` for policy enforcement; confirm `Cargo.lock` is
  committed for binaries.
- **Other ecosystems:** `npm audit`, `pip-audit`, `bundle audit`, `trivy`.

**Build pipeline security:**
- Security gates (SAST, dependency scanning, secrets detection) must run in CI before merge —
  not optionally or post-merge.
- 🔴 CI/CD pipeline config changes deserve the same scrutiny as application code — a compromised
  pipeline can inject malicious code into any artifact (SolarWinds SUNBURST was inserted this way).
- Artifacts should be signed and signatures verified at deployment time.
- Flag any new outbound network call in a build step.

**AI-generated code:**
- Review AI-generated code with the same rigor as any third-party contribution. LLMs commonly
  produce plausible but incorrect cryptography, authentication flows, and input validation.
- Check AI-generated code for unexplained external calls, unusual imports, or overly broad permissions.

---

### Lens 5 — Identity and Access Control
*Applies when: the code handles authentication, authorization, tokens, sessions, service accounts, API keys, or credentials.*

**JWT and token handling:**
- 🔴 Accepting tokens with `alg: none` (no signature verification) is a critical vulnerability.
  Validate the algorithm field against an explicit allowlist.
- Tokens must have explicit `exp` claims. Tokens without expiry are permanent credentials.
- Verify expired and revoked tokens are rejected consistently, especially on logout.

**OAuth 2.0 / OIDC:**
- Redirect URIs must be explicitly allowlisted — an open redirect in an OAuth flow allows token theft.
- The `state` parameter must be cryptographically random, tied to the user's session, and validated
  on callback. Missing `state` enables CSRF against the OAuth flow.
- Public clients (SPAs, mobile apps) must implement PKCE.
- Request only the minimum scopes the application actually needs.

**Non-human identities (service accounts, API keys, machine tokens):**
- 🔴 API keys hardcoded in any file — including infrastructure-as-code, K8s manifests, CI/CD env
  vars defined inline, or Dockerfiles — are a blocker.
- Service accounts must follow least privilege. Admin-by-default service accounts are a privilege
  escalation waiting to happen.
- Service account tokens and API keys must have explicit expiration and rotation policies.
  Dynamic short-lived secrets (Vault, SPIFFE/SPIRE) are strongly preferred over static credentials.
- Kubernetes service account tokens mounted as readable files without projected volume restrictions
  give any compromised pod credential access to the cluster.

**Least privilege and authorization models:**
- Every service, user, and machine identity should have the minimum permissions needed.
- RBAC role boundaries must be enforced in code, not only in the IAM layer. Framework-level
  checks can be bypassed by internal calls.
- Privilege escalation must be rare, explicitly logged, time-limited, and require an approval chain.
- 🔴 Any code that reads permission levels from user-supplied input is a horizontal/vertical
  privilege escalation vector.

**Session management:**
- Sessions must have idle timeouts and absolute expiration.
- Session tokens must be invalidated server-side on logout — deleting the cookie client-side is
  not sufficient.
- Session IDs must be regenerated after login to prevent session fixation.
- In concurrent systems, session state updates must be atomic.

---

### Lens 6 — Code Quality
*Applies to all code changes.*

**Correctness and logic:**
- Does the code do what the PR says it does?
- Are there edge cases or failure modes the author seems to have missed?
- Are error handling patterns consistent with the rest of the codebase?
- Off-by-one errors, unhandled null/undefined/nil values, incorrect type assumptions?

**Readability and maintainability:**
- Can you understand this code on a cold read? Sections requiring re-reading need a comment or rename.
- Variable, function, and class names meaningful and consistent with repo naming conventions?
- DRY — is behavior duplicated where a shared abstraction already exists in the codebase?
- Functions doing one thing? No hidden side effects?

**Repo pattern adherence:**
- Follows the file structure, naming conventions, and patterns of the existing codebase?
- Uses libraries and utilities already in the project rather than reinventing them?
- Matches the existing error handling and logging style?
- If departing from existing patterns: is it justified by a meaningful improvement?

**Tests:**
- New behaviors covered by tests? Bug fixes accompanied by regression tests?
- Tests verify meaningful behavior, not just implementation details?
- Test names descriptive enough to understand what's being verified without reading the body?

**Documentation:**
- Public APIs and complex logic documented appropriately for this codebase?
- If a new feature: is user-facing documentation updated?

---

## Step 3 — Visual Validation (Web Applications)

If the change touches a web application accessible in a browser and browser tools are available:

1. Identify the URL (README, `package.json` scripts, Makefile, or ask the user).
2. Navigate to the affected pages/flows.
3. Compare what you see to what the PR claims — does it match?
4. Test the unhappy path: empty inputs, invalid values, missing data, unauthorized access.
5. Check the browser console for JavaScript errors, failed requests, and warnings.
6. Note any visual regressions: broken layout, missing styles, or UI elements.

Include specific observations (and screenshots if possible) in the review report.

---

## Step 4 — Write Your Comments

Structure every finding using a severity label followed by the **Triple-R pattern**:
**Request** (what to do) → **Rationale** (why, with references) → **Result** (what done looks like).

**Severity labels:**
- `needs change:` / 🔴 **Blocker** — correctness, security, data consistency, policy violation
- `needs rework:` — major structural problem; warrants offline discussion before proceeding
- `align:` / 🟡 **Should fix** — works but violates conventions or strong best practices
- `level up:` / 🔵 **Consider** — non-blocking improvement for a future PR
- `nitpick:` — purely subjective; never blocks a PR; use sparingly
- `praise:` — something done genuinely well; use it

**Tone rules:** Use "we" not "you." Ask, don't command. Back every suggestion with an objective
reason. Focus on the code, not the author.

See `references/comment-patterns.md` for the full framework: 5P process, Triple-R examples,
MMG Exchange for resolving disagreements, and MoSCoW / Conventional Comments alternatives.

---

## Step 5 — Produce the Review Report

```
## Code Review

### Summary
[2–3 sentences: overall assessment, most critical theme, and your verdict recommendation]

### 🔴 Blockers
[Findings that must be resolved before merge — security holes, data consistency bugs,
reliability risks. Use Triple-R format with file and line references.]

### 🟡 Should Fix
[Significant concerns that are strongly recommended but not strictly blocking.]

### 🔵 Consider
[Non-blocking suggestions worth thinking about in this or a future PR.]

### Visual Validation
[Observations from browser-based testing, if applicable.]

### What's Working Well
[At least one genuine praise: — something done well, a clever solution, good coverage, etc.]

### Verdict
[ ] Approved — no changes needed
[ ] Approved with suggestions — non-blocking feedback only
[ ] Changes requested — one or more blockers must be resolved before approval
```

### Rapid-Triage Checklist

Before writing up findings, run through this mentally:

**Domain** — Ubiquitous language used; single aggregate per transaction; value objects immutable;
repositories load complete aggregates; domain events via outbox; bounded context boundaries respected.

**Reliability** — New paths emit metrics/logs/traces; all failure modes handled; external calls
have timeouts and backoff retries; change is safely rollbackable; no new operational toil.

**Security** — All external inputs validated; no string concat into SQL/commands/HTML; auth uses
strong hashing or SSO; authorization checked per-action; TLS everywhere; no hardcoded secrets.

**Supply Chain** — New dependencies scanned for CVEs; SBOM updated; pipeline changes reviewed;
AI-generated code scrutinized; threat model updated for new trust boundaries.

**Identity** — JWT algorithm validated against allowlist; OAuth redirect URIs allowlisted with
state+PKCE; no hardcoded API keys or machine credentials; service accounts follow least privilege;
session tokens invalidated server-side on logout.

**Code Quality** — Logic correct and edge cases handled; code readable and consistent with repo
conventions; DRY; tests cover new behavior; docs updated.

---

## Reference Files

- `references/comment-patterns.md` — Full comment-writing guide: 5P process, Triple-R examples,
  Politeness Principles, MMG Exchange, MoSCoW and Conventional Comments systems
- `references/security-checklist.md` — Full OWASP security checklist with Go and Rust
  language-specific patterns (injection, auth, XSS, IDOR, CSRF, secrets, crypto, etc.)
- `references/domain-model.md` — Detailed DDD checklist: aggregates, value objects, repositories,
  domain events, bounded contexts, transactions
- `references/reliability.md` — Detailed SRE checklist: observability, error handling, resilience
  patterns, deployment safety, toil reduction
