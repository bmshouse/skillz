# Security Code Review Checklist

Based on the OWASP Code Review Guide v2. Apply this checklist to any code that handles
user input, authentication, sessions, data storage, external services, or configuration.

---

## Before You Start: Security Context Questions

Answer these before diving into security-specific checks:

- What data does this application handle? How sensitive is it?
- Who are the users? Are there different privilege levels?
- Where does data enter the system (entry points), and where is it consumed (sinks)?
- What would happen if this data were compromised, modified, or leaked?

Use **source-to-sink analysis**: trace user input from where it enters (HTTP request, form, API)
to where it terminates (database query, HTML output, shell command, file system). Every step in
that path must be validated or encoded appropriately.

---

## Threat Modeling Quick Reference (STRIDE)

Use STRIDE to think about what an attacker might do with this code:

| Threat | Description | Ask yourself |
|--------|-------------|--------------|
| **Spoofing** | Assuming another user's identity | Can input be used to impersonate another user? |
| **Tampering** | Modifying data (cookies, headers, params) | Can client-side values be manipulated to change behavior? |
| **Repudiation** | Disputing transactions due to missing audit trail | Is there sufficient logging of sensitive operations? |
| **Information Disclosure** | Exposing private data | Does error output reveal internals? Are logs safe? |
| **Denial of Service** | Exhausting resources | Are there unprotected expensive operations (queries, uploads)? |
| **Elevation of Privilege** | Gaining higher access than authorized | Can a normal user reach admin functions? |

---

## A1 — Injection

Injection vulnerabilities occur when untrusted data is sent to an interpreter as part of a command or query.

**SQL Injection — what to look for:**
- [ ] No raw string concatenation of user input into SQL queries
- [ ] Parameterized queries / prepared statements used in all database calls
- [ ] Even ORM solutions (Hibernate, Entity Framework, ActiveRecord) can be vulnerable when using raw/dynamic queries — check for those
- [ ] Input validated for type, length, format, and range *before* use

**Command Injection — what to look for:**
- [ ] User input never passed directly to `Runtime.exec()`, `subprocess`, `os.system()`, shell commands
- [ ] Avoid shell=True (Python) or equivalent where user data is involved

**Log Injection — what to look for:**
- [ ] User input sanitized before being written to logs (newlines, CRLF characters can forge log entries)

**Language-specific red flags to search for:**
- Java: `createStatement`, `executeQuery` with string concat; `Runtime.exec`, `getRuntime`
- .NET: `SqlCommand` with string concat; `Process.Start` with user input
- PHP: `mysql_query`, `exec`, `eval` with user input; `$_GET`/`$_POST` in queries
- Python: `cursor.execute` with `%` formatting; `subprocess` with `shell=True`
- Go: `db.Query` / `db.Exec` with `fmt.Sprintf` or `+` string concat; `exec.Command` with user-controlled args; `os/exec` with shell interpolation; `text/template` used where `html/template` is needed
- Rust: `Command::new("sh").arg("-c").arg(user_input)`; raw SQL via `format!()` instead of parameterized queries (e.g., `sqlx::query!` macro or bound parameters); `unsafe` blocks that handle user-controlled memory or pointers

---

## A2 — Broken Authentication & Session Management

**Authentication — what to look for:**
- [ ] Login pages served over TLS only (not HTTP with HTTPS form action)
- [ ] Error messages do not reveal whether the username exists ("Invalid credentials" not "Username not found")
- [ ] No logging of submitted passwords (even wrong ones)
- [ ] Minimum password length enforced (10+ characters recommended)
- [ ] Account lockout or rate limiting after failed login attempts
- [ ] Passwords stored using modern hashing: bcrypt, Argon2, or scrypt — never MD5, SHA1, or unsalted SHA256
- [ ] Multi-factor authentication implemented for sensitive operations

**Go-specific auth red flags:**
- `crypto/md5` or `crypto/sha1` used for password hashing — use `golang.org/x/crypto/bcrypt` or `golang.org/x/crypto/argon2` instead
- JWT libraries that accept `alg: none` (check for `WithoutClaimsValidation` or permissive algorithm lists)
- Custom token generation using `math/rand` — must use `crypto/rand`

**Rust-specific auth red flags:**
- `md5` or `sha1` crates used for password storage — use `bcrypt`, `argon2`, or `rust-argon2` crates
- `rand::thread_rng()` used for token/session ID generation — use `rand::rngs::OsRng` or the `getrandom` crate for cryptographic randomness
- JWT libraries configured to skip signature verification (`dangerous_insecure_decode`)

**Session Management — what to look for:**
- [ ] Session IDs are cryptographically random and at least 128 bits
- [ ] Session IDs contain no user or session data (must be opaque)
- [ ] Session IDs stored in cookies — never in URLs
- [ ] Cookies marked `Secure` (HTTPS only) and `HttpOnly` (no JavaScript access)
- [ ] Session IDs rolled (regenerated) on login, logout, and privilege change
- [ ] Session inactivity timeout implemented
- [ ] Sessions invalidated server-side on logout (not just cookie deletion)

---

## A3 — Cross-Site Scripting (XSS)

XSS occurs when untrusted data is rendered in a browser without proper encoding.

**What to look for:**
- [ ] No unencoded user input inserted into HTML (`innerHTML`, template literals with user data, etc.)
- [ ] Context-appropriate encoding applied: HTML entity encoding for HTML context, JavaScript escaping for JS context
- [ ] Safe DOM APIs used: `textContent`, `createTextNode`, `setAttribute` (second param only) — not `innerHTML`
- [ ] `eval()` never used with user input; `JSON.parse()` used instead
- [ ] jQuery: `.text()` used instead of `.html()` for untrusted data
- [ ] Content-Security-Policy header set to restrict script sources
- [ ] Framework auto-escaping not disabled (e.g., `validateRequest="false"` in ASP.NET is dangerous)

**Go-specific XSS red flags:**
- `text/template` used to render HTML — always use `html/template` for any output sent to a browser; it provides context-aware auto-escaping
- `template.HTML(userInput)` or `template.JS(userInput)` — these mark strings as safe and bypass escaping; only use with fully trusted, sanitized content
- `http.ResponseWriter.Write([]byte(userInput))` without encoding — use `html.EscapeString()` before writing user data directly to a response

**Rust-specific XSS red flags:**
- String interpolation into HTML responses without escaping (e.g., `format!("<p>{}</p>", user_input)` in an Actix/Axum handler)
- Templating libraries (Tera, Askama, Handlebars-rust) with auto-escaping disabled (`{{ value | safe }}` or `{{{ value }}}` in Handlebars)
- `askama` templates using `|safe` filter on user-supplied data

**Types to be aware of:**
- **Reflected XSS**: Input echoed back in the response without encoding
- **Stored XSS**: Input saved to DB and later retrieved and rendered unencoded
- **DOM XSS**: Client-side JavaScript uses `document.write`, `innerHTML`, or `eval` with URL/cookie data

---

## A4 — Insecure Direct Object Reference (IDOR)

IDOR occurs when internal object identifiers (database IDs, file paths, record numbers) are
exposed to users who can manipulate them.

**What to look for:**
- [ ] Authorization checked at the *object* level (not just page/route level)
- [ ] A user cannot access another user's record by modifying an ID in the URL or request body
- [ ] Internal database IDs not exposed directly; use opaque tokens or indirect reference maps where appropriate
- [ ] Mass assignment vulnerabilities: framework doesn't bind hidden model properties from HTTP params
- [ ] Parameter names aren't guessable in ways that allow unauthorized actions (`/delete?id=X`, `/admin=true`)

---

## A5 — Security Misconfiguration

**Configuration files to review (web.config, web.xml, application.yml, .env, etc.):**
- [ ] `debug=true` / `DEBUG=True` not set in production config
- [ ] Default credentials not present; no test accounts left enabled
- [ ] Directory listing disabled on web servers
- [ ] Error handling configured to not expose stack traces to end users
- [ ] Security headers set (see table below)
- [ ] HTTPS enforced; no mixed content
- [ ] Unnecessary features, services, or endpoints disabled

**Required HTTP security headers:**

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Enforce HTTPS |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Prevent clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `X-XSS-Protection` | `1; mode=block` | Enable browser XSS filter (legacy browsers) |
| `Content-Security-Policy` | `default-src 'self'` (tuned to app) | Restrict resource origins |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer header leakage |

---

## A6 — Sensitive Data Exposure

**What to look for:**
- [ ] No passwords, API keys, tokens, or secrets hardcoded in source code
- [ ] No credentials committed to version control (check .env files, config files)
- [ ] Sensitive data encrypted in transit (TLS) and at rest
- [ ] PII and sensitive fields not logged
- [ ] Weak algorithms not used: MD5, DES, RC2, SHA1 for security purposes
- [ ] `Math.random()` / `System.Random` not used for security-sensitive randomness; use `SecureRandom`, `secrets`, `crypto.randomBytes` etc.
- [ ] Encryption keys not stored in code; managed via environment variables, secret managers, or vaults

**Go-specific sensitive data red flags:**
- `math/rand` used for any security purpose — must use `crypto/rand`
- `fmt.Println` / `log.Printf` logging struct fields that may contain passwords or tokens (use structured logging with field-level redaction)
- Hardcoded secrets in `config.go`, `constants.go`, or `init()` functions
- TLS configured with `InsecureSkipVerify: true` in `tls.Config` — only acceptable in tests, never in production code
- `http.DefaultTransport` used without custom TLS config where certificate validation matters

**Rust-specific sensitive data red flags:**
- `println!` / `dbg!` macros logging sensitive struct fields — implement a custom `Debug` or `Display` that redacts secrets
- `#[derive(Debug)]` on structs that contain passwords, tokens, or private keys — the auto-derived impl will expose them in logs/panics
- Secrets stored in `String` instead of a zeroing wrapper (e.g., `secrecy::Secret<String>` or `zeroize`) — heap memory is not wiped on drop by default
- `reqwest::ClientBuilder` with `.danger_accept_invalid_certs(true)` outside of test code
- Hardcoded secrets in `const` or `static` declarations

---

## A7 — Missing Function-Level Access Control

**What to look for:**
- [ ] Authorization checks on every function that performs a state change or accesses data
- [ ] Authorization not bypassable by manipulating parameters or roles in the request
- [ ] Admin/internal APIs protected — not just the UI routes
- [ ] Role checks consistent throughout the call chain (not just at the entry point)

---

## A8 — Cross-Site Request Forgery (CSRF)

**What to look for:**
- [ ] Anti-CSRF tokens included on all state-changing forms and AJAX requests
- [ ] Server verifies the CSRF token on every state-changing request
- [ ] `SameSite=Strict` or `SameSite=Lax` cookie attribute set
- [ ] Sensitive operations (transfers, profile changes, deletions) require re-authentication or out-of-band confirmation

---

## A9 — Using Components with Known Vulnerabilities

**What to look for:**
- [ ] Third-party dependencies are up to date
- [ ] Known CVEs in dependencies flagged (check `npm audit`, `pip-audit`, `bundle audit`, `trivy`, etc.)
- [ ] No unmaintained or abandoned libraries for security-critical functions
- [ ] Dependency versions pinned in lockfiles

**Go:** Run `govulncheck ./...` (Go's official vulnerability scanner) and check `go.sum` is committed. Watch for indirect dependencies pulling in vulnerable transitive deps via `go mod graph`.

**Rust:** Run `cargo audit` (from the `cargo-audit` crate) against `Cargo.lock`. Check `cargo deny` for license and vulnerability policy enforcement. Ensure `Cargo.lock` is committed for binaries (it is often gitignored for libraries — confirm the project type).

---

## A10 — Unvalidated Redirects and Forwards

**What to look for:**
- [ ] User input never used directly as a redirect destination
- [ ] Whitelist of valid redirect targets enforced
- [ ] Open redirect attacks not possible by manipulating a `?redirect=` or `?next=` parameter

---

## Input Validation Summary

Apply to all user-controlled data:

- [ ] Validate type, length, format, and range server-side (client-side is never sufficient alone)
- [ ] Whitelist (known-good) approach preferred over blacklist (known-bad)
- [ ] Reject or sanitize binary data, escape sequences, comment characters, and path traversal patterns (`../`)
- [ ] Canonicalization handled before validation (e.g., URL-decoded values may bypass checks applied to encoded strings)
- [ ] Centralized validation logic applied consistently — not scattered ad hoc

---

## Logging & Error Handling Security

- [ ] Errors logged server-side; no stack traces or internal detail exposed to the client
- [ ] Logs contain enough to detect and investigate attacks, but no passwords, tokens, or PII
- [ ] Failed authentication attempts logged
- [ ] Log injection prevented (user input sanitized before logging)
- [ ] `finally` blocks used where resources must be released even on exception

**Go:** Errors are values — check that all returned `error` values are handled, not silently discarded with `_`. Returning raw `error.Error()` strings to API clients can leak internal paths, query structure, or stack details; wrap errors for external responses (`fmt.Errorf("operation failed")`, not the raw db error).

**Rust:** `unwrap()` and `expect()` will panic and may expose internal state in panic messages if not caught. In production handlers, propagate errors with `?` and map them to sanitized responses. Avoid `panic!` in library code. Use structured logging crates (`tracing`, `log`) with field-level control rather than `println!`.

---

## Go-Specific Security Patterns

### Concurrency & Race Conditions
- [ ] Shared mutable state protected by `sync.Mutex`, `sync.RWMutex`, or atomic operations — data races can cause security-relevant non-determinism (e.g., TOCTOU bugs in auth checks)
- [ ] Race detector run during testing: `go test -race ./...`
- [ ] `goroutine` leaks checked — leaked goroutines holding references to request context (including auth tokens) can cause unintended data access across requests

### HTTP & Network
- [ ] `http.ListenAndServe` not used in production without a custom `http.Server` with explicit `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` — the default has no timeouts, enabling slowloris-style DoS
- [ ] Request body size limited with `http.MaxBytesReader` to prevent memory exhaustion from large uploads
- [ ] `http.Redirect` uses a safe, validated destination — not directly from query params
- [ ] `net/http` CSRF middleware applied to all state-changing routes (e.g., `gorilla/csrf`)

### File & Path Handling
- [ ] `filepath.Join` used for path construction — but verify the result doesn't escape the intended base directory with `strings.HasPrefix(resolved, base)`
- [ ] `os.Open` / `ioutil.ReadFile` with user-supplied paths validated against a whitelist or restricted root
- [ ] Temporary files created with `os.CreateTemp` (not predictable names) and cleaned up after use

### Deserialization
- [ ] `encoding/gob` and `encoding/json` with interface types (`interface{}`) can deserialize unexpected types — validate the decoded structure before use
- [ ] `json.Unmarshal` into a struct with exported fields is generally safe, but check for unintended field binding if the struct is also used elsewhere with different trust levels

### Go Crypto Cheat Sheet
| Unsafe | Safe replacement |
|--------|-----------------|
| `crypto/md5`, `crypto/sha1` for passwords | `golang.org/x/crypto/bcrypt` or `argon2` |
| `math/rand` for tokens/IDs | `crypto/rand` |
| `tls.Config{InsecureSkipVerify: true}` | Proper cert validation; custom CA if needed |
| `http.DefaultClient` (no timeout) | `&http.Client{Timeout: 10 * time.Second}` |
| `text/template` for HTML output | `html/template` |

---

## Rust-Specific Security Patterns

### `unsafe` Code
- [ ] Every `unsafe` block has a comment explaining *why* it is sound — what invariants are being upheld manually
- [ ] `unsafe` blocks that process user-controlled data (lengths, offsets, pointers) are reviewed with extra scrutiny for buffer overflows, out-of-bounds access, and use-after-free
- [ ] FFI (`extern "C"`) boundaries validate all inputs before passing to C code; assume C functions can behave arbitrarily on bad input
- [ ] `transmute` is not used to bypass type safety on user-controlled values

### Memory Safety at Trust Boundaries
- [ ] Slice indexing with user-controlled indices uses `.get(index)` (returns `Option`) rather than direct `[index]` (panics on out-of-bounds)
- [ ] Integer arithmetic on user-supplied values uses checked (`checked_add`, `checked_mul`) or saturating arithmetic to prevent wrapping in debug-off (release) builds
- [ ] `from_utf8_unchecked` not used on user-supplied bytes — use `from_utf8()` and handle the error

### Secrets in Memory
- [ ] Structs containing passwords, private keys, or tokens use `zeroize::Zeroize` (or the `secrecy` crate's `Secret<T>`) so memory is wiped on drop
- [ ] `#[derive(Debug)]` removed or overridden on types that contain secrets — the auto-derived impl will print the value in logs and panic messages
- [ ] Secrets not cloned unnecessarily — each clone is another copy that needs zeroing

### Async & Concurrency
- [ ] `Arc<Mutex<T>>` held across `.await` points checked carefully — holding a lock across an await can deadlock or cause priority inversion
- [ ] Shared state in async handlers uses `tokio::sync::Mutex` (async-aware) rather than `std::sync::Mutex` where the lock may be held across await points
- [ ] TOCTOU (time-of-check/time-of-use) races in async code: auth checks and the operations they protect happen atomically or within the same locked scope

### Web Framework Patterns (Actix-web / Axum)
- [ ] Extractors validate and bound user input — `Path<T>`, `Query<T>`, `Json<T>` with `serde` validation (e.g., `validator` crate) rather than raw string parsing
- [ ] `State` shared between handlers contains no secrets in plaintext (use `Secret<T>` wrappers)
- [ ] CORS configured explicitly — `Any` origin (`*`) not used for credentialed endpoints
- [ ] Request body size limited at the framework level (Axum's `DefaultBodyLimit`, Actix's `web::JsonConfig::limit`)
- [ ] Error responses from `?` propagation use a custom error type that maps to sanitized HTTP responses — `anyhow::Error` should not be surfaced directly to clients

### Rust Crypto Cheat Sheet
| Unsafe | Safe replacement |
|--------|-----------------|
| `md5`, `sha1` crates for passwords | `bcrypt`, `argon2`, or `rust-argon2` crates |
| `rand::thread_rng()` for secrets | `rand::rngs::OsRng` or `getrandom` crate |
| `reqwest::ClientBuilder::danger_accept_invalid_certs(true)` | Proper cert validation; custom root CA if needed |
| `String` for secrets | `secrecy::Secret<String>` with `zeroize` |
| `#[derive(Debug)]` on secret structs | Manual `Debug` impl that redacts the value |
| Direct slice indexing `buf[user_idx]` | `buf.get(user_idx).ok_or(...)` |
