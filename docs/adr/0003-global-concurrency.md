# ADR-003 — Single Worker and Local Concurrency Limit

- **Status:** Accepted (subject to single-instance deployment verification)
- **Date:** 2026-11-17
- **Project:** TracePoint
- **Related:** PRD v1.0 D-008, D-010; ADR-001; ADR-002

## Context
TracePoint runs five-minute API checks through a persistent Node.js worker. The MVP must permit no more than five HTTP monitoring requests concurrently across the entire deployment. MongoDB-backed atomic slot claims prevent multiple workers from claiming the same slot, but do not alone limit concurrent requests across different slots.

## Decision
Deploy exactly **one active persistent Node.js monitoring worker** and enforce a **local concurrency limit of five HTTP monitoring requests** within that worker. This is an operational constraint, not merely a code convention. Hosting and rollout settings must prevent overlap between old and new worker instances. If a hosting provider cannot enforce one active instance, deployment is blocked until a distributed permit mechanism is designed.

Use MongoDB-backed scheduling state and atomic slot claims as specified in ADR-002. The local limiter applies to actual outbound monitoring attempts, not merely database operations. Each attempt uses the PRD's 10-second total timeout and is not automatically retried. Eligible due slots may wait for capacity only within a scheduling window that will be defined in detailed design; expired unstarted slots become missed. Do not backfill old slots.

## Component responsibilities
- **Next.js application:** user-facing authentication, monitor configuration, dashboard and authorized reads; never responsible for continuous scheduling.
- **MongoDB:** durable monitors, slot identity and claims, results, incidents and coordination state.
- **Single Node.js worker:** find eligible slots, claim atomically, run up to five HTTP checks, persist results, and trigger incident processing.
- **Hosting platform:** maintain and restart one worker, prohibit overlapping instances during rollouts, and provide operational logs/alerts.

## Alternatives considered
1. **MongoDB-backed distributed permits:** supports multiple workers and overlapping deployments, but requires correct leases, fencing, expiry and capacity accounting.
2. **Redis-backed distributed semaphore:** enables distributed concurrency, but adds a service and more failure modes.
3. **Unbounded worker concurrency:** rejected; violates PRD D-010.

## Consequences and limitations
- Simpler MVP implementation and easier debugging, but one worker is a single point of execution availability.
- During worker downtime, monitoring coverage can decrease; missed slots must not be replayed.
- Local concurrency cannot enforce a global limit if more than one worker instance runs. A deployment guarantee is mandatory.
- MongoDB claims do not guarantee exactly-once external HTTP side effects. Uncertain executions must not be blindly retried.
- A 10-second timeout does not imply a remote server has stopped processing a request; state-changing methods carry an explicit warning.
- The worker's scheduler and local limiter must not treat queued work as already executed.

## Failure scenarios
| Scenario | Required behavior |
| --- | --- |
| More than five due slots | Start at most five requests; defer others only while eligible |
| Worker crashes before request starts | Recover slot state without claiming a completed HTTP outcome |
| Worker crashes after request may have started | Mark outcome uncertain/interrupted; do not automatically resend |
| Worker restarts | Resume future eligible slots; skip missed historical slots |
| New deployment overlaps old worker | Deployment must prevent simultaneous active instances or be blocked |
| Database unavailable | Do not fabricate persisted results; recover without unsafe replay |
| Monitor deleted during check | Finalized deletion prevents recreated records |

## Verification criteria
- An integration test with more than five due monitors observes no more than five simultaneous outbound requests.
- A deployment check demonstrates exactly one active worker during normal operation, restart and rollout.
- A worker restart does not replay an uncertain POST/PUT/PATCH/DELETE request.
- Slots that cannot start before the scheduling cutoff become missed and reduce coverage as specified in the PRD.
- Logs distinguish endpoint failures from worker/scheduler failures without exposing secrets.

## Follow-up decisions
- Define the **scheduling window** and exact rules for slot creation, claiming, and interrupted states.
- Design persistence schemas and unique indexes in `docs/database-design.md`.
- Select hosting providers and verify their single-instance rollout behavior before deployment.
- If horizontal worker scaling becomes necessary, replace this ADR with a distributed capacity-control design before adding workers.
