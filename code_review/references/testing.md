# Testing Review Checklist

Apply during every review. Work through each category and flag gaps.

---

## 1. Missing Negative-Path Tests

- New code paths that handle errors, rejections, or invalid input with no corresponding test
- Guard clauses and early returns that are completely untested
- Error branches in try/catch, rescue, or error boundaries with no failure-path test
- Permission/auth checks asserted in code but never tested for the denied case
- `else` branches in conditional logic that handle the "bad" path with no test

## 2. Missing Edge-Case Coverage

- Boundary values: zero, negative, max-int, empty string, empty array, nil/null/undefined
- Single-element collections (off-by-one errors in loops)
- Unicode, emoji, and special characters in any user-facing input
- Maximum-length inputs (for fields with length constraints)
- Concurrent access patterns with no race-condition test
- First-run behavior (no pre-existing data in the system)

## 3. Test Isolation Violations

- Tests sharing mutable state (class variables, global singletons, DB records not cleaned up between tests)
- Order-dependent tests: pass when run in sequence, fail when randomized or run in isolation
- Tests that depend on the system clock, timezone, or locale without controlling for them
- Tests that make real network calls instead of using stubs or mocks
- Tests that depend on file system state left by a previous test

## 4. Flaky Test Patterns

- Timing-dependent assertions: `sleep`, `setTimeout`, `waitFor` with tight or hardcoded timeouts
- Assertions on ordering of inherently unordered results: hash keys, Set iteration, async resolution order
- Tests that depend on external services (APIs, cloud services) without fallback or mock
- Randomized test data without seed control — same test can produce different results on different runs
- Assertions that rely on log output order from concurrent operations

## 5. Missing Security Enforcement Tests

- Auth/authz checks in handlers with no test for the "unauthorized" / "forbidden" case
- Rate limiting logic with no test proving it actually blocks after the threshold
- Input sanitization or validation with no test for malicious, oversized, or boundary input
- CSRF/CORS configuration with no integration test verifying it rejects cross-origin requests
- Token expiration checks with no test for an expired token being rejected

## 6. Coverage Gaps

- New public methods or functions with zero test coverage
- Changed methods where existing tests only cover the old behavior, not the new branch
- Utility functions called from multiple places but only tested indirectly through callers
- New database queries with no test for the case where the query returns zero results
