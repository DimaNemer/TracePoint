# ADR-002 — MongoDB-Backed Scheduling State and Atomic Work Claims

**Status:** Accepted  
**Date:** 2026-11-17  
**Project:** TracePoint  
**Related:** ADR-001, PRD v1.0 (D-002, D-004, D-005, D-006, D-008, D-010), US-011, US-014, US-015, US-028

## 1. Context

TracePoint uses a persistent Node.js background worker (ADR-001) to check active API monitors at five-minute intervals. The platform must keep monitoring when browsers are closed, skip missed historical checks, avoid unsafe HTTP retries, and report monitoring coverage honestly. The MVP allows up to five concurrent checks globally and supports potentially state-changing HTTP methods.

An in-memory timer alone cannot reliably identify historical gaps or reconcile interrupted work after a worker restart. Overlapping workers or deployments can also attempt to process the same scheduled check.

## 2. Decision

Use **MongoDB-backed scheduling state** with **atomic claims** to coordinate monitoring execution. Each scheduled opportunity has a stable identity consisting of a monitor identifier and a scheduled UTC timestamp. Enforce a unique constraint on this pair.

The worker shall:

1. Determine eligible scheduled slots for active monitors using the fixed five-minute cadence.
2. Persist or derive sufficient slot records to distinguish expected, executed, missed, and uncertain work within the supported reporting window.
3. Claim an eligible slot atomically, so competing workers cannot both successfully claim it.
4. Mark whether execution has begun before initiating an outbound request, subject to crash-safe ordering.
5. Record completed outcomes idempotently and update incident state consistently.
6. Expire abandoned claims and reconcile unfinished slots **without automatically repeating uncertain outbound HTTP requests**.
7. Skip slots that can no longer start within the permitted scheduling window; do not replay historical requests.
8. Respect monitor pause, deletion, and global concurrency constraints.

The exact schema, scheduling window, claim/lease duration, worker coordination strategy, and retention mechanism will be defined in the database and detailed architecture designs.

## 3. Conceptual Execution States

| State | Meaning | Recovery treatment |
|---|---|---|
| `PENDING` | Slot is eligible but not claimed | May be claimed while still eligible; otherwise missed |
| `CLAIMED` | Worker has reserved the slot; no confirmed outbound start | Reconcile claim expiration; only reassign when non-execution can be established safely |
| `RUNNING` | Outbound request may have started | Do not automatically resend after crash |
| `COMPLETED` | A final outcome was durably recorded | Never execute or persist again for that slot |
| `MISSED` | Slot expired without a request being started | Record gap; never backfill |
| `INTERRUPTED` | Attempt may have started but no reliable final result exists | Preserve uncertainty; never blindly retry |

These are conceptual states, not yet a fixed MongoDB schema. `COMPLETED` may represent healthy or unhealthy HTTP outcomes, timeouts, or network errors. `MISSED` and `INTERRUPTED` must not be automatically classified as unhealthy API responses.

## 4. Duplicate Prevention and Correctness

- Use a unique key on `(monitorId, scheduledAt)` or an equivalent stable slot identity.
- Claim eligible work with an atomic conditional update; only the successful claimant may proceed.
- Use claim ownership/fencing information so an expired claimant cannot overwrite a newer decision.
- Persist a maximum of one authoritative check outcome per slot.
- Make incident processing idempotent and protect against out-of-order result application.
- Check monitor ownership/state and deletion coordination before dispatch and before final persistence.
- Treat MongoDB atomicity as protection for database state, **not** a guarantee of exactly-once effects on external APIs.

## 5. Failure and Recovery Scenarios

### Worker unavailable before scheduled time

The slot remains unexecuted. Once its eligibility window expires, record it as `MISSED` or derive an equivalent missing-slot fact. Do not execute it later.

### Worker crashes after claiming but before dispatch

Reconcile the expired claim. A slot may be considered safe to release only when the design can reliably establish that the outbound request did not start. Otherwise preserve an uncertain state rather than risk duplicate dispatch.

### Worker crashes during or after HTTP dispatch

The external endpoint may have received or processed the request even if no result was saved. Mark or retain an `INTERRUPTED`/uncertain outcome after recovery. Never automatically replay the request.

### Worker crashes after persisting the outcome

A retry of internal finalization must not create a second result or repeat incident transitions. Reconciliation uses the already persisted outcome.

### MongoDB becomes unavailable

Do not pretend results were stored. Avoid dispatching new work when claim ownership or durable recording cannot be established. On recovery, reconcile missed and uncertain slots without historical HTTP replay.

### Overlapping worker instances

Atomic claims prevent competing claims for the same slot. A **separate globally coordinated capacity mechanism** is still required if more than one worker can run; atomic slot claims alone do not enforce the five-request global concurrency cap.

### Monitor paused or deleted

Prevent new dispatch for paused/deleted monitors. Deletion must coordinate with in-flight work so finalized deletion cannot be followed by newly persisted check results or incidents.

## 6. Monitoring Metrics Implications

- **Uptime:** Healthy completed checks / completed checks with recorded health outcomes. Missed and uncertain outcomes do not count as healthy or unhealthy API results.
- **Coverage:** Executed scheduled slots / expected scheduled slots, excluding intentionally paused intervals and counting each slot once.
- A worker crash can leave dispatch uncertain. The detailed metrics design must explicitly define how to represent this uncertainty without falsely counting a successful execution or masking a monitoring gap.
- The data model must preserve enough slot or scheduling information to calculate coverage over the supported reporting window, including after restarts.

## 7. Alternatives Considered

### A. In-memory scheduling

**Rejected.** Simpler initially, but loses important execution state on restart, complicates missed-slot reporting, and provides weak coordination across workers.

### B. MongoDB-backed scheduling state

**Accepted.** Uses an existing dependency, supports atomic claims, persistent execution state, deduplication, and recovery with manageable MVP infrastructure.

### C. External scheduler or job queue

**Deferred.** Can offer richer scheduling and coordination but introduces additional infrastructure and queue retry semantics that must be carefully constrained for state-changing requests.

## 8. Consequences and Trade-offs

**Benefits**

- Durable execution state survives process restarts.
- Atomic claims reduce duplicate scheduling.
- Missing and interrupted work can be represented explicitly.
- Avoids an additional queue service in the MVP.
- Provides a foundation for monitoring coverage and incident consistency.

**Costs and limitations**

- More database writes, indexes, and scheduling-state management.
- Claim expiration, fencing, and worker crashes require careful testing.
- External HTTP effects cannot be guaranteed exactly once.
- Atomic slot claims do not by themselves enforce global concurrency.
- Retention of slot/coverage information must be coordinated with the 90-day check-result policy.

## 9. Implementation Boundaries / Open Design Decisions

The following are **not yet approved implementation choices**:

1. Exact collections and fields for monitors, slots, attempts, results, and incidents.
2. Slot eligibility/grace window and scheduling alignment.
3. Claim lease length, heartbeat, expiration, and fencing strategy.
4. Global five-check concurrency coordination, including overlapping deployments.
5. Transaction boundaries for result persistence and incident transitions.
6. Handling out-of-order completions for incident streaks.
7. Distinguishing confirmed dispatch from uncertain dispatch for coverage.
8. Storage and retention of expected-slot information and pause/resume intervals.
9. Safe coordination of permanent deletion with in-flight execution.

## 10. Verification Criteria

- Two competing workers cannot both successfully claim the same eligible slot.
- A unique slot identity prevents duplicate authoritative results.
- Restart after missed intervals does not issue catch-up HTTP requests.
- Restart after uncertain POST dispatch does not resend POST.
- A completed slot remains completed after worker restart.
- Missing/uncertain slots do not independently trigger incidents.
- A stale worker cannot overwrite a finalized slot outcome.
- Paused or deleted monitors do not receive newly dispatched work.
- Global concurrency remains at most five under the separately selected coordination design.
- Coverage remains explainable across interruptions and scheduling gaps.

## 11. Follow-up

**ADR-003:** Choose the mechanism for globally enforcing five concurrent HTTP checks, including worker overlap, crash recovery, and lease/capacity release.
