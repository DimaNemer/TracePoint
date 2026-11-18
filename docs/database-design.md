# TracePoint — Database Design

**Version:** 1.0 (design baseline)  
**Status:** Approved conceptual decisions DB-001–DB-013; implementation details marked open  
**Database:** MongoDB Atlas with Mongoose  
**Related documents:** `docs/prd.md`, `docs/architecture.md`, `docs/api-specification.md`, ADR-001–ADR-003

## 1. Purpose and scope

TracePoint monitors user-configured public HTTPS API endpoints through a persistent Node.js worker. MongoDB stores users, monitor configuration and operational state, scheduled execution slots, definitive check results, and incident history. The web application and worker share the database but have distinct responsibilities.

This document specifies the **conceptual data model**, constraints, indexes, lifecycle invariants, and remaining implementation questions. Code examples are illustrative, not final Mongoose models. Do not interpret an open implementation item as approved behavior.

## 2. Approved design decisions

| ID | Decision |
|---|---|
| DB-001 | Separate `scheduledChecks` (execution slots) from `checkResults` (observed outcomes). |
| DB-002 | Retain detailed scheduled checks and results for 90 days; retain incidents until their monitor is deleted. |
| DB-003 | Keep configuration in `monitors`; encrypt sensitive request headers and JSON bodies at rest. |
| DB-004 | Store `nextCheckAt` scheduling cursor in `monitors`. |
| DB-005 | Cache latest completed health summary in `monitors`. |
| DB-006 | Store incident-processing cursor, failure count, and active incident reference in `monitors`. |
| DB-007 | Monitor lifecycle states: `ACTIVE`, `PAUSED`, `DELETING`. |
| DB-008 | Version meaningful monitor configuration changes and use controlled execution snapshots. |
| DB-009 | Scheduled slots have execution states, atomic claims, claim tokens, and lease metadata. |
| DB-010 | Store structured diagnostic results; do not capture full response bodies or unrestricted headers. |
| DB-011 | Store detailed incident lifecycle and references to trigger/recovery results. |
| DB-012 | Close incidents without verified recovery as `CLOSED_UNVERIFIED` when paused or meaningfully reconfigured. |
| DB-013 | Enforce 10 monitors per user with an atomic `users.monitorCount` reservation and transaction. |

Related approved product rules: five-minute checks; 10-second total request timeout; configurable expected statuses (default 2xx); no automatic request retries; incidents open after two consecutive unhealthy completed checks and resolve after one healthy completed check; missed/uncertain checks do not count as healthy or unhealthy; 10 monitors per user including paused; five concurrent HTTP checks globally under a strictly single-active-worker deployment.

## 3. Collections and relationships

```text
users (1) ── (many) monitors
monitors (1) ── (many) scheduledChecks
monitors (1) ── (many) checkResults
monitors (1) ── (many) incidents
scheduledChecks (1) ── (0..1) checkResults
monitors.activeIncidentId ── (0..1) open incident
incidents.triggerResultId / recoveryResultId ── optional links to checkResults
```

Each scheduled slot is identified by `(monitorId, scheduledAt)`. Results share this identity. Incident records must remain readable after result TTL expiry; result IDs are not guaranteed to resolve forever. MongoDB does not enforce relational foreign keys: application-level ownership and deletion consistency are required.

## 4. `users`

Conceptual fields:

| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | Primary key. |
| `monitorCount` | integer | Nonnegative, maximum 10; defaults to 0 for new users. |
| Auth-related fields | per auth library | Google/GitHub OAuth only in MVP; exact account/session collections depend on chosen adapter. |
| `createdAt`, `updatedAt` | Date | Standard timestamps. |

**Atomic creation:** In one MongoDB transaction, conditionally increment `monitorCount` when below 10, insert monitor, and commit. Handle write conflicts and retries by rechecking the limit. A failed transaction consumes no slot. A paused or `DELETING` monitor counts until final deletion. Counter reconciliation and idempotent create requests need implementation safeguards.

**Atomic final deletion:** Only after deletion is safely coordinated, remove the monitor and decrement the counter **once** in the finalization transaction. Prevent double decrement on retries and recover unfinished deletion after crashes. Do not decrement on the initial delete click.

## 5. `monitors`

Conceptual fields:

| Field | Type | Purpose |
|---|---|---|
| `_id` | ObjectId | Monitor identifier. |
| `ownerId` | ObjectId | Owning user; all APIs enforce ownership. |
| `name` | string | Display label. |
| `url` | string | Public HTTPS target; URL may itself contain sensitive query values and requires safe handling. |
| `method` | enum | GET, HEAD, POST, PUT, PATCH, DELETE. |
| `expectedStatuses` | number[] | Default expected 2xx; exact-status overrides supported. |
| `encryptedHeaders` | encrypted payload/null | Authenticated encryption, including nonce/tag/version metadata. |
| `encryptedBody` | encrypted payload/null | Optional JSON body for POST/PUT/PATCH only. |
| `status` | enum | `ACTIVE`, `PAUSED`, `DELETING`. |
| `nextCheckAt` | Date/null | Scheduling cursor; never proof of execution. |
| `configVersion` | integer | Increment for meaningful monitoring configuration changes. |
| `lastHealth` | enum/null | `HEALTHY`, `UNHEALTHY`, or null. |
| `lastCheckedAt` | Date/null | Completion time of latest completed result. |
| `lastScheduledAt` | Date/null | Scheduled-slot timestamp of latest completed result. |
| `lastHttpStatus` | number/null | Actual HTTP status when received. |
| `lastResponseTimeMs` | number/null | Measured duration. |
| `lastErrorType` | enum/null | e.g., `TIMEOUT`, `NETWORK_ERROR`. |
| `consecutiveFailures` | integer | Ordered unhealthy completed results in current incident-processing epoch. |
| `lastProcessedSlotAt` | Date/null | Last slot applied to incident-processing state. |
| `activeIncidentId` | ObjectId/null | Current open incident reference. |
| `createdAt`, `updatedAt` | Date | Standard timestamps. |

**Lifecycle:** Create as `ACTIVE`; `ACTIVE` ↔ `PAUSED`; either may transition to `DELETING`; `DELETING` is terminal and followed by physical removal. Pause prevents *new* scheduling, preserves history, excludes paused periods from expected coverage, closes any open incident as `CLOSED_UNVERIFIED`, and resets the failure sequence. Resume establishes a fresh future eligible schedule and failure sequence, without replaying paused slots. Exact handling of already-running checks remains open.

**Configuration edits:** Increment `configVersion` on meaningful changes. A running check retains its verified execution snapshot; historical results record the version actually executed. Do not mix old-version failures with the new failure sequence. Close any open incident as `CLOSED_UNVERIFIED` (`CONFIG_CHANGED`) and reset failure count. Pending/claimed checks must not silently execute with the wrong version. The durable-versus-memory snapshot strategy remains open.

**Health summary:** Update only for a completed result with a strictly newer `scheduledAt`; coordinate with result persistence. `MISSED` and `INTERRUPTED` do not overwrite last known health. UI must distinguish last known health from monitoring freshness. An older result may be saved to history but must not overwrite a newer summary or corrupt incident state.

**Proposed indexes:**

```ts
monitorSchema.index({ status: 1, nextCheckAt: 1 });
monitorSchema.index({ ownerId: 1 });
```

## 6. `scheduledChecks`

One document represents one scheduled slot, regardless of whether a definitive check result is produced.

| Field | Type | Purpose |
|---|---|---|
| `_id` | ObjectId | Slot identifier. |
| `monitorId` | ObjectId | Monitor reference. |
| `scheduledAt` | Date | Original scheduled time; unique with monitor ID. |
| `state` | enum | `PENDING`, `CLAIMED`, `RUNNING`, `COMPLETED`, `MISSED`, `INTERRUPTED`. |
| `claimedBy` | string/null | Worker identity. |
| `claimToken` | string/null | Unique claim/fencing token. |
| `claimedAt` | Date/null | Claim time. |
| `leaseExpiresAt` | Date/null | Lease expiry. |
| `startedAt` | Date/null | Execution-start marker. |
| `completedAt` | Date/null | Terminal transition time. |
| `configVersion` | integer/null | Version assigned to execution. |
| `createdAt` | Date | Record creation. |
| `expiresAt` | Date/null | TTL only after terminal state; retention policy 90 days. |

**Claim lifecycle:** Due slot → atomic claim (`CLAIMED`) → verify active monitor and valid configuration snapshot → mark `RUNNING` before dispatch → record definitive result and `COMPLETED` together. Expired pending slots become `MISSED`; expired leases require recovery classification. A crash after marking `RUNNING` but before actual dispatch may be indistinguishable from a request that was sent: classify uncertainty as `INTERRUPTED`, **not** automatically retry. A lease expiring is not proof that the old worker stopped. Fencing guards database writes, while strict singleton deployment and request cancellation/coordination address external dispatch races.

**Slot generation and cursor:** Advancing `monitors.nextCheckAt` and recording due slots must be coordinated. During worker downtime, expired slots are skipped, not replayed. Expected-slot accounting and handling slots never materialized during downtime require a precise algorithm (open).

**Proposed indexes:**

```ts
scheduledCheckSchema.index({ monitorId: 1, scheduledAt: 1 }, { unique: true });
scheduledCheckSchema.index({ state: 1, scheduledAt: 1 });
scheduledCheckSchema.index({ state: 1, leaseExpiresAt: 1 });
scheduledCheckSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

## 7. `checkResults`

One document per scheduled slot **with a definitive observed health outcome**. No result is required for missed or uncertain interrupted slots.

| Field | Type | Purpose |
|---|---|---|
| `_id` | ObjectId | Result identifier. |
| `monitorId` | ObjectId | Monitor reference. |
| `scheduledAt` | Date | Links to scheduled slot. |
| `configVersion` | integer | Configuration used. |
| `outcome` | enum | `HTTP_RESPONSE`, `TIMEOUT`, `NETWORK_ERROR`. |
| `health` | enum | `HEALTHY`, `UNHEALTHY`. |
| `httpStatus` | number/null | Actual response status, null for timeout/network error. |
| `responseTimeMs` | number | Measured elapsed duration. |
| `errorCode` | safe enum/string/null | Structured category, never raw exception or secrets. |
| `startedAt`, `completedAt` | Date | Execution timestamps. |
| `createdAt`, `expiresAt` | Date | Persistence and 90-day retention. |

`HTTP_RESPONSE` health is evaluated against the monitor's expected statuses; a non-2xx response can be healthy if explicitly expected. `TIMEOUT` and `NETWORK_ERROR` are unhealthy completed outcomes, with null HTTP status. The configured total timeout is 10 seconds; store actual measured duration. No full response body, unrestricted response headers, or raw credentials are retained.

**Metrics:**

- **Check-based uptime** = healthy completed checks / completed checks with health outcomes × 100; show N/A for zero denominator. Missed/interrupted slots are excluded from uptime.
- **Coverage** = executed scheduled slots / expected scheduled slots × 100, each eligible slot counted once; paused periods excluded; missed slots reduce coverage. Uncertain execution and unmaterialized downtime slots require an explicit accounting algorithm (open).
- **Latency**: primary HTTP response-time chart/average uses `HTTP_RESPONSE` results; retain measured elapsed duration for timeout/network-error diagnostics separately. This is a proposed reporting convention to confirm in dashboard design.

**Proposed indexes:**

```ts
checkResultSchema.index({ monitorId: 1, scheduledAt: 1 }, { unique: true });
checkResultSchema.index({ monitorId: 1, scheduledAt: -1 });
checkResultSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

The unique index also supports history queries in reverse order; assess whether the separate descending index is needed using query plans before adding it.

## 8. `incidents`

Incidents are confirmed by two consecutive unhealthy **completed** checks, and are resolved by the first subsequent healthy completed check. Missed/interrupted slots do not themselves advance the failure count or establish recovery.

| Field | Type | Purpose |
|---|---|---|
| `_id` | ObjectId | Incident identifier. |
| `monitorId` | ObjectId | Monitor reference. |
| `status` | enum | `OPEN`, `RESOLVED`, `CLOSED_UNVERIFIED`. |
| `firstFailureAt` | Date | Observation time of first unhealthy completed check. |
| `confirmedAt` | Date | Observation time of confirming second unhealthy check. |
| `resolvedAt` | Date/null | Healthy recovery observation time only. |
| `closedAt` | Date/null | Administrative closure time only. |
| `closureReason` | enum/null | `MONITOR_PAUSED` or `CONFIG_CHANGED`. |
| `firstFailureResultId` | ObjectId/null | Optional link to initial failure. |
| `triggerResultId` | ObjectId | Link to confirming result. |
| `recoveryResultId` | ObjectId/null | Link to recovery result. |
| `createdAt`, `updatedAt` | Date | Database timestamps. |

Use the relevant check result's `completedAt` as observed failure, confirmation, or recovery time. An `OPEN` incident transitions once to either `RESOLVED` (observed healthy check) or `CLOSED_UNVERIFIED` (pause/meaningful edit); closed states are terminal. A later incident is a new document. Pause/edit also resets pending consecutive-failure count to zero, even if no incident was open. Old-version results cannot change the new configuration's incident-processing epoch.

Incidents retain timestamps independently because linked check results may expire after 90 days. Incidents are removed only when their monitor is deleted.

**Proposed indexes:**

```ts
incidentSchema.index({ monitorId: 1, confirmedAt: -1 });
incidentSchema.index(
  { monitorId: 1 },
  { unique: true, partialFilterExpression: { status: 'OPEN' } }
);
```

## 9. Consistency and concurrency invariants

1. **Owner isolation:** All monitor and child-record access is scoped to the authenticated owner; worker operations use trusted internal access.
2. **One slot, at most one result:** Unique `(monitorId, scheduledAt)` constraints in both execution and result collections.
3. **At most one open incident:** Unique partial index plus idempotent state transitions.
4. **Ordered incident processing:** Apply definitive completed results in scheduled order; unresolved earlier slots must be finalized or accounted for before applying later outcomes. Track a processing cursor and protect against duplicate/late writes.
5. **Latest summary monotonicity:** Older scheduled slots cannot overwrite newer health summaries.
6. **Atomic multi-document updates:** Use MongoDB transactions for monitor creation/counter reservation and for coupled result, slot, monitor-state, and incident changes where necessary.
7. **No blind HTTP retries:** A database transaction or expired lease never proves whether an external request occurred.
8. **Safe deletion:** `DELETING` stops new work; finalize only when worker writes are fenced/drained and child cleanup is complete; decrement `monitorCount` exactly once.
9. **Singleton concurrency assumption:** One active persistent worker enforces five simultaneous HTTP requests locally; deployment must prevent overlap or architecture must add distributed coordination.
10. **Secret handling:** Authenticated encryption at rest; keys server-side only; no plaintext secrets in results, incidents, scheduling records, normal API responses, or logs.

## 10. Retention and query behavior

- `scheduledChecks` and `checkResults`: 90-day detailed retention. Set `expiresAt` only for terminal scheduled slots; use TTL indexes and explicit query cutoff filters because TTL removal is asynchronous.
- `incidents`: retained until monitor deletion, independent of result expiry.
- `monitors`: retained until permanent deletion.
- Historical dashboards only claim statistics over retained data; no lifetime aggregate storage is approved for MVP.

## 11. Security considerations

- Accept public HTTPS targets only; validate DNS/IP destinations to prevent SSRF, block redirects, and apply outbound-network restrictions. Validate method, headers, JSON body, and size limits before execution.
- Treat configured headers and JSON bodies as sensitive and encrypt with authenticated encryption (for example, AES-256-GCM) with fresh nonces and versioned payload metadata. Keep encryption keys outside MongoDB and client bundles; plan for rotation.
- URL query strings can contain credentials: reject embedded userinfo and redact potentially sensitive URL data from logs and public error messages.
- Normal API responses use explicit safe projections. Mongoose `select: false` is helpful but not an authorization or serialization boundary.
- On edits, allow replacing/clearing secrets without returning existing plaintext values.
- Do not persist unrestricted raw error messages or response bodies.

## 12. Open implementation decisions and verification tasks

These are **not** approved as fully solved by DB-001–DB-013:

1. **Scheduler window and gap accounting:** Define the exact eligible execution window, how to materialize/count expected slots during downtime, and how `nextCheckAt` advances atomically without gaps or replay.
2. **Coverage under uncertainty:** Decide how `INTERRUPTED` and ambiguous dispatch affect numerator/denominator; distinguish confirmed executed from possibly executed slots.
3. **Snapshot durability:** Define when a configuration version is assigned, how a claimed slot retains a consistent encrypted snapshot without copying secrets into history, and what happens if a worker crashes or an edit occurs before dispatch.
4. **Pause and in-flight work:** Decide whether to cancel or let already-running checks finish, and how their results affect history, current health, and incidents after pause.
5. **Deletion barrier:** Choose concrete worker fencing/draining and recoverable deletion algorithm; ensure child records cannot reappear after cleanup and the user counter is decremented exactly once.
6. **Incident ordered processing:** Specify the transaction and cursor algorithm for late/out-of-order results, unresolved slots, and configuration epochs. Confirm `lastProcessedSlotAt` semantics across resets.
7. **Auth schema:** Finalize OAuth adapter, account linking rules, and any additional collections required by the chosen authentication library.
8. **Validation limits:** Set request-header allowlist, header/body size limits, error-code taxonomy, URL handling, and safe encrypted-payload format/key rotation.
9. **Indexes and performance:** Verify query plans and eliminate redundant indexes; consider operational recovery queries and status-based partial indexes only as needed.
10. **Counter repair/idempotency:** Define duplicate create-request handling and reconciliation if preexisting user data disagrees with `monitorCount`.
11. **Latency reporting convention:** Confirm whether the primary chart excludes timeout/network-error durations; these durations remain stored for diagnostics.
12. **Operational deployment:** Verify strict single-active-worker behavior across deploys, restarts, and failure recovery before relying on a local five-request concurrency limit.

## 13. Implementation checklist

- [ ] Implement Mongoose models with validation and explicit safe serializers.
- [ ] Create unique, lookup, and TTL indexes; test them against realistic queries.
- [ ] Implement transactional monitor creation and counter reservation.
- [ ] Implement encrypted request configuration and safe editing behavior.
- [ ] Implement atomic slot creation/claims, leases, fencing, and crash classification.
- [ ] Implement transactional result persistence and ordered incident updates.
- [ ] Implement pause, resume, configuration-change, and deletion state transitions.
- [ ] Test races: two simultaneous creates at nine monitors, duplicate slot claims, late results, worker crashes, pause/edit during execution, and deletion during an in-flight check.
- [ ] Verify uptime/coverage denominators, 90-day cutoff filters, and incident readability after result expiry.

---

**Design note:** This document intentionally separates approved product/database decisions from unresolved implementation mechanisms. It is a baseline for implementation planning, not a claim that all concurrency edge cases have already been solved.
