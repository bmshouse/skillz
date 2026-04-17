# Reliability and Operability Reference

Based on "High Performance SRE" (Mishra, 2024).

Apply this checklist when the code runs in production, handles I/O, calls external services,
or is part of a distributed system. Skip for purely offline, batch, or dev-tooling code
with no production reliability requirements.

---

## Observability

Unobserved code paths are operational blind spots. You cannot alert on, debug, or capacity-plan
something you cannot measure.

### Metrics
- [ ] New code paths emit the **four golden signals** where relevant:
  - **Latency** — How long does this take? Track p50, p95, p99 — not just averages.
  - **Traffic** — How often is this called? (requests/sec, events/sec, jobs/hour)
  - **Errors** — What fraction of requests fail? Distinguish client errors (4xx) from server errors (5xx).
  - **Saturation** — How full is the system? (queue depth, connection pool utilization, memory pressure)
- [ ] The **USE method** applied to resources: Utilization, Saturation, and Errors for each
  CPU, memory, disk, and network resource the new code uses.
- [ ] The **RED method** applied to services: Rate, Errors, Duration per endpoint or operation.
- [ ] Hard-coded resource limits (connection pool sizes, queue depths, memory caps) have defined
  behavior when the limit is reached and are configurable without redeployment.

### Logs
- [ ] Logs are structured (JSON or equivalent), not free-form text strings that break log parsing.
- [ ] Log entries include enough context to trace a failure to its source without additional tooling:
  request ID, user ID (or equivalent), operation name, duration, outcome.
- [ ] Log levels are used correctly:
  - `ERROR` / `FATAL` — requires human attention now
  - `WARN` — degraded state, may require action
  - `INFO` — normal operational events worth recording
  - `DEBUG` — verbose detail for development/troubleshooting only (should not run in production at scale)
- [ ] Sensitive data (passwords, tokens, PII) is never logged at any level.
- [ ] Log injection is prevented — user input is sanitized before being written to logs.

### Distributed Tracing
- [ ] In microservices or async systems, distributed trace context (correlation IDs, trace headers
  such as W3C `traceparent` or B3) is propagated through every service call and message.
- [ ] New services and workers initialize and propagate trace context rather than starting fresh.
- [ ] Trace spans are named meaningfully to make traces readable in Jaeger, Zipkin, or equivalent.

---

## Error Handling

The way a system fails is as important as the way it succeeds.

- [ ] 🔴 Generic catch-all exception handlers that swallow errors and continue silently are a blocker.
  Every failure mode a caller might care about must be explicitly handled or explicitly re-raised.
- [ ] Errors are distinguishable from slow responses in logs and metrics. An error disguised as
  high latency is invisible to alerting.
- [ ] Partial failure states are handled — what does the system do if step 3 of 5 fails?
  Is there cleanup? Compensation? A retry-safe path?
- [ ] Error messages are actionable — they tell an operator what went wrong and ideally what to do.
  "Internal server error" with no context is not actionable.
- [ ] Panics (Go), unwrap failures (Rust), and unhandled exceptions are not used as flow control.
  Reserve them for truly unrecoverable programmer errors; recoverable errors should be returned/propagated.

---

## Timeouts and Retries

Network calls, database queries, and external service calls that have no timeout will eventually
hang — and the hung goroutine/thread will accumulate until the system exhausts its resources.

### Timeouts
- [ ] Every network call has an explicit timeout configured at the client level.
  - HTTP clients: connection timeout + read timeout (or overall request deadline)
  - Database clients: query timeout / statement timeout
  - Message consumers: processing deadline per message
- [ ] Timeout values are configurable (not hardcoded) and documented with justification.
- [ ] Timeouts are propagated through context (Go `context.WithTimeout`, deadline propagation
  in gRPC, etc.) rather than applied only at the outermost layer.

### Retries
- [ ] Retries use **exponential backoff with jitter**. Tight retry loops without delay contribute
  to thundering-herd / cascading failures under load.
- [ ] Retries are only applied to idempotent operations. A non-idempotent write retried on timeout
  may succeed twice.
- [ ] Maximum retry count and total retry budget are bounded — infinite retry loops are not acceptable.
- [ ] Retries are not applied to errors that won't improve with time (4xx client errors, auth failures).

---

## Resilience Patterns

### Circuit Breakers
- [ ] Calls to unreliable or slow external services are wrapped with a circuit breaker.
  An open circuit should stop sending requests to a degraded dependency rather than
  continuing to pile up and fail.
- [ ] Circuit state transitions (closed → open → half-open) are logged and emit metrics.

### Bulkheads
- [ ] High-volume or low-priority workloads are isolated from critical paths.
  A bulkhead (separate thread pool, connection pool, or queue) prevents one slow consumer
  from exhausting shared resources.

### Fallback Responses
- [ ] Non-critical features have graceful degradation paths. Ask: what does the user see if
  this service call fails? Is that acceptable?
- [ ] Fallback responses are not confused with success — they are logged and metered separately.

### Single Points of Failure
- [ ] New code paths with no fallback are flagged. Ask: what happens when this dependency
  is unavailable?
- [ ] Stateful components (caches, queues, databases) have replication or failover configured,
  or the failure mode is explicitly documented and accepted.

---

## Deployment Safety

Changes should be safe to deploy incrementally and safe to roll back.

- [ ] New behavior is deployable behind a **feature flag** where possible, enabling canary
  releases and quick rollback without a code change.
- [ ] **Schema migrations** are backward-compatible with the previous version of the service:
  - Add columns as nullable or with defaults before making them required
  - Don't rename columns without a multi-step migration
  - Don't drop columns until all versions that read them are retired
- [ ] Changes that require a specific **deployment order** across multiple services are
  flagged explicitly in the PR for the release plan.
- [ ] Ask: if this is deployed and then immediately rolled back, does anything break?
  (Data written in the new format that the old version can't read? A migration that can't be undone?)
- [ ] **Database transactions** are not held open across slow operations (HTTP calls, user input)
  — long-held locks degrade concurrent throughput and cause cascading slowdowns.

---

## Avoiding Toil

Toil is repetitive manual operational work that scales linearly with service growth and provides
no lasting value. Code review is a good time to prevent new toil from being introduced.

- [ ] Code that requires manual steps to operate (runbooks where a human performs the same
  steps every time) should be automated. If the runbook describes identical steps each time,
  those steps belong in code.
- [ ] Alerts generated by the new code are **actionable** (the operator knows what to do)
  and **rare** (they don't fire routinely). Code that predictably triggers pages without a
  clear remediation path creates alert fatigue.
- [ ] New scheduled jobs, cron tasks, or batch processes have:
  - Execution logging (started, finished, records processed, errors)
  - Alerting on failure or missed schedule
  - Idempotent behavior so accidental double-runs are safe
- [ ] Operational documentation is updated alongside the code change — not filed as a
  follow-up ticket that will be forgotten.

---

## Capacity and Performance

- [ ] New code paths that could be called at scale are analyzed for computational complexity.
  An O(n²) operation that runs once is fine; one that runs in a hot loop is not.
- [ ] Database queries have appropriate indexes. A query that scans a full table on every request
  will eventually cause an outage.
- [ ] Caching is used where appropriate, but cache invalidation strategy is explicit and correct.
  Stale cache entries that serve incorrect data are worse than no cache.
- [ ] Memory allocations in hot paths are minimized — profiling data or benchmarks support
  any performance-sensitive claims made in the PR.
