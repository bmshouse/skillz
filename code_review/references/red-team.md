# Red Team Review Checklist

This is not a checklist review — it is adversarial analysis. Apply after completing all
other lenses, when the diff is > 200 lines or when significant security or reliability
findings were identified. Think like an attacker, a chaos engineer, and a hostile QA tester
simultaneously.

---

## 1. Attack the Happy Path

Assume the system is healthy. Now stress it:

- **10x load**: what breaks under normal load × 10? Look for shared state, connection pools,
  unbounded queues, synchronous blocking in hot paths.
- **Concurrent requests to the same resource**: two users updating the same record, the same
  background job running twice, a retry arriving before the first attempt completes.
- **Slow dependencies**: what happens when the database takes 5s per query? Does the caller
  have a timeout? Does it propagate back correctly, or does it hang the request thread?
- **Garbage response from external service**: malformed JSON, empty body, 200 with error in
  payload, truncated stream. Does the code handle any of these, or does it panic/crash?

## 2. Hunt Silent Failures

- **Swallowed exceptions**: catch-all handlers that log and continue. Ask: what invariant is
  now violated in the system state that the caller doesn't know about?
- **Partial completion**: 3 of 5 items processed, then crash. Is there cleanup? Is the state
  recoverable? Will the operation succeed on retry, or will it double-apply the first 3?
- **Inconsistent state on failure**: a write that modifies multiple records — what happens if
  it succeeds for record A but fails for record B? Is this inside a transaction?
- **Background jobs with no alerting**: jobs that fail silently. After N failures, does anyone
  know? Is there a dead-letter queue? Is there an alert?

## 3. Exploit Trust Assumptions

- **Frontend-only validation**: anything validated client-side but not server-side. An attacker
  bypasses the frontend entirely.
- **Internal API without authentication**: "only our code calls this" is an assumption that
  breaks in SSRF attacks, misconfigured proxies, and network compromises. Check internal routes.
- **Config assumed present**: what happens if an expected environment variable is missing or
  empty at startup? Does the service start with a broken default, or crash hard with a clear error?
- **Path/URL from user input**: any file path or URL constructed using user-controlled values
  that reaches the filesystem or an outbound HTTP call without sanitization.

## 4. Break the Edge Cases

- **Maximum input size**: what happens with the largest possible value? (Largest string, largest
  file, largest array.) Is there a size limit? Is it enforced before or after parsing?
- **Zero/empty/null**: what happens with no items, empty string, null where an object is expected?
  Follow every null check: is it consistent, or does it protect some paths but not others?
- **First run ever**: the system has never run before. No seed data, no previous records. Does
  anything assume pre-existing state?
- **Double submission**: user clicks the button twice in 100ms. Does the server create two records?
  Is the endpoint idempotent? Is there a uniqueness constraint?

## 5. Find What the Other Lenses Missed

Review findings from all other lenses, then look specifically for:

- **Cross-category issues**: a performance issue that's also a security issue (e.g., an
  unbounded query triggerable by any authenticated user = DoS vector).
- **Integration boundary issues**: where two systems meet. Does system A assume system B has
  already validated the data? Does system B assume it's only called by system A?
- **Deployment-specific issues**: issues that only manifest in production (specific config,
  traffic volume, DB size, multi-instance deployment) but not in the test environment.
- **The gap between lenses**: security found X and reliability found Y — is there a third issue
  at their intersection that each lens missed independently?
