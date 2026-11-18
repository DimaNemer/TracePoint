# ADR-001: Persistent Node.js Monitoring Worker

- **Status:** Accepted
- **Date:** 2026-11-17
- **Project:** TracePoint
- **Related requirements:** FR-003, FR-004, FR-006; D-002, D-004, D-005, D-006, D-008, D-010

## Context

TracePoint must monitor user-configured public HTTPS APIs every five minutes, even when no user has the website open. Checks have a ten-second total timeout, are never automatically retried, and must support safe recovery from worker interruption. Missed historical slots are not replayed. The deployment must cap monitoring requests at five globally concurrent checks and avoid duplicate persisted results and incident transitions.

The Next.js web application's request lifecycle is not an appropriate dependency for recurring background monitoring.

## Decision

Run monitoring in a **persistent, dedicated Node.js background-worker process**, separately from the Next.js web application. Both components use shared application persistence in MongoDB. Keep the components in the same repository where practical, with clear module boundaries and separate process entry points.

For the initial MVP, prefer one active worker process. Do not scale to multiple active workers until shared scheduling ownership and global concurrency controls are designed and verified. A single worker's local concurrency limiter is sufficient only while exactly one worker can execute requests.

The worker is responsible for:

1. Discovering eligible scheduled slots for active monitors.
2. Claiming work safely and enforcing a maximum of five in-flight monitoring HTTP requests globally.
3. Validating destination safety at execution time, including DNS and outbound connection restrictions.
4. Executing allowed HTTP methods with a ten-second total timeout, disabled redirects, and no automatic retries.
5. Persisting check outcomes and applying incident transitions consistently.
6. Reconciling interrupted execution state without replaying historical requests.
7. Producing logs and operational signals for missed slots, errors, and recovery.

The web application is responsible for OAuth authentication, session management, authorization, monitor management, and presenting monitoring history and statistics. It must not rely on open browser sessions to trigger checks.

## High-Level Data Flow

1. A signed-in user configures a monitor through the Next.js application.
2. The web application validates and stores the monitor in MongoDB.
3. The worker identifies an eligible five-minute scheduled slot and reserves execution capacity.
4. The worker validates the target and sends the HTTP request.
5. The worker stores the outcome and updates incident state safely.
6. The dashboard reads authorized monitoring data from MongoDB.

## Alternatives Considered

### Scheduled serverless functions

**Benefits:** Less always-on infrastructure; hosting platform invokes scheduled work.

**Reasons not selected:** Invocation duration and overlap constraints, platform-specific scheduling guarantees, and coordination complexity for this project's workload. It remains a viable future alternative if deployment constraints change.

### Managed job queue and workers

**Benefits:** Mature queueing and workload distribution; useful for higher scale.

**Reasons not selected:** Adds infrastructure and operational complexity to the MVP. Queue retries would need explicit safeguards to prevent repeating state-changing HTTP requests.

## Consequences

### Benefits

- Background monitoring is independent of the browser and web request lifecycle.
- Scheduling, HTTP execution, and incident processing have explicit ownership.
- The architecture demonstrates backend execution, failure handling, and operational reasoning.
- One active worker makes initial concurrency control straightforward.

### Costs and Risks

- Requires hosting for a continuously running process, restart behavior, and operational monitoring.
- Worker outages can create missed slots; a persistent process does not guarantee perfect coverage.
- Overlapping deployments or multiple workers can violate concurrency and deduplication guarantees without shared coordination.
- Database failures and process crashes can leave uncertain execution outcomes.
- The monitoring engine handles untrusted destinations and must enforce strong SSRF defenses.

## Failure and Recovery Requirements

- **Browser closes:** Worker continues independently.
- **Web service outage:** Worker continues if its own dependencies remain available.
- **Worker stops before starting a slot:** Mark or derive the slot as missed; do not backfill.
- **Worker stops during an HTTP request:** Reconcile execution state without blindly retrying the request; the remote side effect may be unknown.
- **Database outage:** Do not claim results were persisted. Handle incomplete states and recovery without replaying uncertain requests.
- **Duplicate claim or worker overlap:** Do not persist duplicate results or apply incident transitions twice.
- **Monitor deletion during execution:** Coordinate state changes so that finalized deletion cannot be followed by recreated check results or incidents.
- **Capacity exhaustion:** Start only checks still eligible under the scheduling window; expired slots count as missed.

## Hosting and Operational Requirements

- Deploy the Next.js app and the worker as distinct processes or services.
- Provide a restart policy and observable worker logs/health signals.
- Keep OAuth credentials and monitoring secrets in server-side configuration.
- Ensure MongoDB connectivity and least-privilege access where practical.
- Plan for overlapping deploys before adding worker replicas.
- Test scheduling, shutdown, restart, timeouts, and database unavailability in the target hosting environment.

## Follow-Up Architecture Decisions

1. **ADR-002:** Scheduled-slot ownership, persistence, claim/lease policy, and recovery rules.
2. Execution-window duration and how delayed work becomes missed.
3. Atomic result persistence and incident transitions.
4. Safe monitor deletion and in-flight execution coordination.
5. Destination validation, DNS pinning/connection controls, and header-secret protection.
6. Hosting provider and deployment topology.

## Verification Criteria

- Checks continue with the browser closed.
- One worker never exceeds five concurrent outbound monitoring requests.
- A ten-second request timeout is enforced.
- Worker restart does not replay missed historical slots.
- Interrupted requests with uncertain outcomes are not automatically retried.
- Duplicate scheduling attempts cannot produce duplicate persisted results or incidents.
- Finalized monitor deletion cannot be undone by in-flight work.

## Decision Review Trigger

Revisit this ADR if hosting constraints prevent reliable long-running processes, throughput testing shows one worker is insufficient, or requirements demand multiple workers or queue-based distribution.
