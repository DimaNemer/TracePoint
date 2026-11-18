# TracePoint — User Stories

**Version:** 1.0  
**Status:** Draft for Implementation Planning  
**Date:** November 17, 2026  
**Source:** Product Requirements Document v1.0  
**Project:** TracePoint

---

## 1. Purpose

This document translates TracePoint's approved MVP requirements into user-focused stories and testable acceptance criteria.

Stories describe the behavior users need. Technical implementation tasks will be derived from these stories during architecture and development planning.

### User Story Format

**As a** [user or system operator],  
**I want** [capability],  
**So that** [benefit].

### Priority Definitions

- **P0 — Critical:** Required for secure, correct core operation.
- **P1 — High:** Required for the complete MVP experience.
- **P2 — Medium:** Useful for usability or operational improvements within the approved MVP.

All stories below are part of the approved MVP unless explicitly identified as a future enhancement.

---

## 2. EPIC-01 — Authentication and Accounts

### US-001 — Sign In with Google

**Priority:** P0  
**Related PRD:** FR-001

**Story**

As a developer, I want to sign in using my Google account so that I can access TracePoint without creating another password.

**Acceptance Criteria**

- A user can initiate Google sign-in.
- Successful authentication creates an account for a first-time user.
- Returning users are recognized through their linked Google identity.
- Authentication failures display an understandable error.
- A successful sign-in opens the user's personal workspace.

### US-002 — Sign In with GitHub

**Priority:** P0  
**Related PRD:** FR-001

**Story**

As a developer, I want to sign in using GitHub so that I can access TracePoint using an account I already use for development.

**Acceptance Criteria**

- A user can initiate GitHub sign-in.
- First-time authentication creates an account.
- Returning users are recognized through their linked GitHub identity.
- Authentication failures are handled without exposing sensitive information.

### US-003 — Maintain and End Sessions

**Priority:** P0  
**Related PRD:** FR-001

**Story**

As an authenticated user, I want my session to remain secure and be able to sign out so that my monitoring workspace is protected.

**Acceptance Criteria**

- Authenticated sessions are maintained securely.
- Protected pages and APIs reject unauthenticated access.
- Signing out ends the application session.
- Sensitive authentication credentials are not exposed to the browser unnecessarily.

### US-004 — Access Only My Workspace

**Priority:** P0  
**Related PRD:** FR-001, NFR-001

**Story**

As a user, I want to access only my monitors and monitoring data so that other users cannot view or modify my information.

**Acceptance Criteria**

- Monitor lists contain only monitors owned by the authenticated user.
- Requests for another user's monitor are denied.
- The same restriction applies to check results and incidents.
- Ownership checks are enforced on the server.

---

## 3. EPIC-02 — Monitor Management

### US-005 — Create an API Monitor

**Priority:** P0  
**Related PRD:** FR-002, FR-004

**Story**

As a developer, I want to register an API endpoint so that TracePoint can monitor it automatically.

**Acceptance Criteria**

- The user can enter a monitor name and public HTTPS URL.
- The user can choose GET, HEAD, POST, PUT, PATCH, or DELETE.
- The user can configure expected HTTP statuses.
- The default expected status range is 200–299.
- Valid configurations can be saved.
- Invalid configurations display understandable errors.
- The monitor becomes eligible for scheduled checks.

### US-006 — Configure Request Headers and JSON Body

**Priority:** P1  
**Related PRD:** FR-002, FR-004

**Story**

As a developer, I want to configure permitted request headers and an optional JSON body so that I can monitor endpoints requiring specific request formats.

**Acceptance Criteria**

- The user can configure headers permitted by TracePoint's security policy.
- POST, PUT, and PATCH support optional JSON bodies.
- GET, HEAD, and DELETE do not support configured bodies.
- Invalid JSON is rejected.
- Headers and bodies exceeding configured limits are rejected.
- Sensitive configuration is not exposed in monitoring history.
- The interface warns about possible side effects of state-changing methods.

### US-007 — View and Edit Monitors

**Priority:** P1  
**Related PRD:** FR-002

**Story**

As a user, I want to view and update my monitor configurations so that I can manage changes to my APIs.

**Acceptance Criteria**

- The user can view their monitors.
- The user can open monitor details.
- The user can edit supported configuration fields.
- Changes are validated before saving.
- Changes affect future eligible monitoring checks.
- Users cannot edit monitors they do not own.

### US-008 — Pause and Resume Monitoring

**Priority:** P1  
**Related PRD:** FR-002

**Story**

As a user, I want to pause and resume a monitor so that I can temporarily stop checks without losing its history.

**Acceptance Criteria**

- An active monitor can be paused.
- Paused monitors do not receive new scheduled checks.
- Pausing preserves check results and incident history.
- A paused monitor can be resumed.
- Monitoring resumes according to the normal scheduling policy.
- Paused periods are excluded from expected monitoring slots.

### US-009 — Permanently Delete a Monitor

**Priority:** P1  
**Related PRD:** FR-002, FR-007

**Story**

As a user, I want to permanently delete a monitor so that I can remove endpoints I no longer need.

**Acceptance Criteria**

- Deletion requires explicit confirmation.
- Cancelling the confirmation leaves the monitor unchanged.
- Confirmed deletion removes the monitor, check results, and incidents.
- Future checks are prevented.
- In-flight workers cannot recreate records after finalized deletion.
- Users cannot delete monitors they do not own.

### US-010 — Respect the Monitor Limit

**Priority:** P0  
**Related PRD:** FR-002, D-010

**Story**

As a user, I want TracePoint to communicate monitor limits so that I understand how many endpoints I can register.

**Acceptance Criteria**

- Each user can own at most 10 monitors.
- Active and paused monitors count toward the limit.
- Creating an eleventh monitor is rejected.
- The restriction is enforced server-side.
- Concurrent creation requests cannot bypass the limit.
- Deleting a monitor frees a slot after deletion completes.

---

## 4. EPIC-03 — Automated Monitoring

### US-011 — Run Scheduled Checks Automatically

**Priority:** P0  
**Related PRD:** FR-003

**Story**

As a developer, I want TracePoint to check my APIs every five minutes so that I can monitor them without keeping the website open.

**Acceptance Criteria**

- Active monitors are scheduled at five-minute intervals.
- Checks execute on the server.
- Closing the browser does not stop monitoring.
- Paused and deleted monitors do not receive new scheduled checks.
- Each executed check is associated with its scheduled slot.

### US-012 — Evaluate HTTP Responses

**Priority:** P0  
**Related PRD:** FR-004, D-003

**Story**

As a developer, I want TracePoint to compare actual HTTP responses with expected statuses so that I can identify unexpected API behavior.

**Acceptance Criteria**

- Default HTTP statuses 200–299 are healthy.
- Users can configure one or more exact expected statuses.
- Unexpected HTTP statuses are unhealthy.
- Actual HTTP status and evaluated health are stored separately.
- A configured 404 response can be classified as healthy.
- Timeout and network-error outcomes remain unhealthy.

### US-013 — Handle Timeouts and Network Errors

**Priority:** P0  
**Related PRD:** FR-003, FR-004, D-002

**Story**

As a developer, I want TracePoint to record timeouts and connection failures so that I can distinguish them from normal HTTP responses.

**Acceptance Criteria**

- Monitoring attempts have a 10-second total timeout.
- Requests are cancelled when possible after timeout.
- Timeouts are classified as unhealthy.
- Network errors are classified as unhealthy.
- Error outcomes are recorded distinctly from HTTP status responses.
- Requests are not automatically retried.

### US-014 — Recover from Monitoring Interruptions

**Priority:** P0  
**Related PRD:** FR-003, D-008

**Story**

As a developer, I want TracePoint to recover safely from monitoring interruptions so that monitoring can continue without misleading or unsafe repeated checks.

**Acceptance Criteria**

- Missed historical checks are not replayed.
- Missing slots remain identifiable.
- Current or future eligible checks resume after recovery.
- Requests with uncertain outcomes are not blindly repeated.
- Duplicate processing does not create duplicate results.
- Worker interruptions are reflected in monitoring coverage.

### US-015 — Enforce Global Monitoring Concurrency

**Priority:** P0  
**Related PRD:** FR-003, D-010

**Story**

As a system operator, I want to limit concurrent monitoring requests so that the platform remains within its intended resource capacity.

**Acceptance Criteria**

- No more than five HTTP monitoring requests execute simultaneously.
- The limit applies across all worker instances.
- Additional eligible checks wait for capacity within their permitted scheduling window.
- Expired scheduling slots are handled as missed rather than replayed indefinitely.
- Worker interruptions do not permanently consume execution capacity.

---

## 5. EPIC-04 — Dashboard and Statistics

### US-016 — View Monitoring Dashboard

**Priority:** P1  
**Related PRD:** FR-005

**Story**

As a developer, I want a dashboard showing my monitors so that I can understand their current status quickly.

**Acceptance Criteria**

- The dashboard shows only the authenticated user's monitors.
- Each monitor displays its latest available status.
- Healthy, unhealthy, paused, and not-yet-checked states are distinguishable.
- The dashboard provides access to individual monitor details.
- Empty states are handled clearly.

### US-017 — View Response-Time History

**Priority:** P1  
**Related PRD:** FR-005, FR-007

**Story**

As a developer, I want to view historical response times so that I can identify changes in API performance.

**Acceptance Criteria**

- Executed checks record available elapsed-time measurements.
- Response-time history can be viewed for an owned monitor.
- Missing checks are not presented as successful response-time measurements.
- Historical views use retained check results.
- The available reporting period is communicated clearly.

### US-018 — View Check-Based Uptime

**Priority:** P1  
**Related PRD:** FR-005, D-005

**Story**

As a developer, I want to see my API's check-based uptime so that I can understand how often completed checks were healthy.

**Acceptance Criteria**

- Uptime is calculated as healthy completed checks divided by completed checks with health outcomes, multiplied by 100.
- Unhealthy HTTP responses, timeouts, and network errors affect uptime.
- Missed scheduled checks are excluded from the uptime denominator.
- Uptime is displayed as N/A when there are no eligible completed checks.
- The interface does not represent check-based uptime as exact time-based availability.

### US-019 — View Monitoring Coverage

**Priority:** P1  
**Related PRD:** FR-005, D-005, D-008

**Story**

As a developer, I want to see monitoring coverage so that I know whether TracePoint performed the checks it was expected to perform.

**Acceptance Criteria**

- Coverage is calculated as executed scheduled slots divided by expected scheduled slots, multiplied by 100.
- Missed slots reduce coverage.
- Paused periods are excluded from expected slots.
- Each scheduled slot is counted at most once.
- Coverage is N/A when there are no expected slots.
- Historical coverage uses a clearly defined reporting window.

---

## 6. EPIC-05 — Incident Management

### US-020 — Detect Confirmed Incidents

**Priority:** P0  
**Related PRD:** FR-006, D-004

**Story**

As a developer, I want TracePoint to detect repeated unhealthy checks so that isolated failures do not immediately create incidents.

**Acceptance Criteria**

- One unhealthy completed check does not open an incident.
- Two consecutive unhealthy completed checks open an incident.
- Timeouts and network errors count as unhealthy outcomes.
- A healthy completed check resets the failure sequence.
- Missed checks do not independently increment failure counts.
- A monitor cannot have multiple active incidents simultaneously.

### US-021 — Detect Incident Recovery

**Priority:** P0  
**Related PRD:** FR-006, D-004

**Story**

As a developer, I want incidents to resolve when my API recovers so that I can see when service health returns.

**Acceptance Criteria**

- One healthy completed check resolves an active incident.
- Recovery time is recorded.
- Further unhealthy checks during an existing incident do not create duplicates.
- Duplicate result processing does not cause duplicate incident transitions.

### US-022 — View Incident History

**Priority:** P1  
**Related PRD:** FR-006, FR-007

**Story**

As a developer, I want to review past incidents so that I can understand when failures occurred and when recovery was observed.

**Acceptance Criteria**

- Incident records show their state.
- The first observed failure time is preserved.
- The incident confirmation time is preserved.
- The recovery time is preserved when available.
- Incident history remains accessible after related check results expire.
- Only the monitor owner can access incident history.

---

## 7. EPIC-06 — History and Retention

### US-023 — Review Historical Check Results

**Priority:** P1  
**Related PRD:** FR-007

**Story**

As a developer, I want to review previous monitoring checks so that I can investigate API behavior over time.

**Acceptance Criteria**

- Results are associated with the correct monitor.
- Available HTTP status, response time, health, and error information are displayed.
- Results are accessible only to the monitor owner.
- Missed slots are distinguishable from completed unhealthy checks.
- Expired check results are excluded from historical queries.

### US-024 — Automatically Expire Old Check Results

**Priority:** P1  
**Related PRD:** FR-007, D-009

**Story**

As a system operator, I want old detailed check results to expire automatically so that database storage remains manageable.

**Acceptance Criteria**

- Detailed check results are retained for 90 days.
- Results older than 90 days are excluded from application queries.
- Expired results are removed through automated cleanup.
- Incident records are not removed by normal check-result expiration.
- Historical metrics do not include expired results.

### US-025 — Preserve Incident History

**Priority:** P1  
**Related PRD:** FR-006, FR-007, D-009

**Story**

As a developer, I want incident history to remain available beyond the detailed check-result retention period so that I can review important past failures.

**Acceptance Criteria**

- Incident records remain available while their monitor exists.
- Incident records preserve essential details independently of expired check results.
- Permanent monitor deletion removes the associated incident history.
- Users cannot access incident history belonging to another user.

---

## 8. EPIC-07 — Security and Reliability

### US-026 — Prevent Unsafe Monitoring Destinations

**Priority:** P0  
**Related PRD:** FR-004, NFR-001

**Story**

As a system operator, I want TracePoint to reject unsafe monitoring destinations so that users cannot use the monitoring engine to access restricted network resources.

**Acceptance Criteria**

- Only permitted public HTTPS endpoints are accepted.
- Loopback and private network destinations are blocked.
- Restricted destination addresses are blocked.
- DNS-related bypasses are addressed during actual request execution.
- Automatic redirect following is disabled.
- Invalid destinations produce safe error messages.

### US-027 — Protect Sensitive Request Configuration

**Priority:** P0  
**Related PRD:** FR-004, NFR-001

**Story**

As a developer, I want sensitive request configuration to remain protected so that monitoring my APIs does not expose credentials.

**Acceptance Criteria**

- Only permitted request headers are accepted.
- Request headers and bodies are validated and size-limited.
- Sensitive configuration is not exposed in logs or historical results.
- Unauthorized users cannot retrieve another user's request configuration.
- Provider credentials and application secrets remain server-side.

### US-028 — Prevent Duplicate Processing

**Priority:** P0  
**Related PRD:** FR-003, FR-006, NFR-002

**Story**

As a system operator, I want duplicate scheduled processing to be handled safely so that monitoring history and incident state remain consistent.

**Acceptance Criteria**

- Duplicate processing does not create duplicate check results.
- Duplicate processing does not open multiple active incidents.
- Repeated recovery handling does not produce duplicate incident transitions.
- Incomplete execution attempts are distinguishable from completed outcomes.
- Unsafe HTTP request retries are not introduced.

### US-029 — Preserve Data Integrity During Deletion

**Priority:** P0  
**Related PRD:** FR-002, FR-007, NFR-004

**Story**

As a user, I want monitor deletion to complete consistently so that my deleted monitors and their data do not reappear.

**Acceptance Criteria**

- Deletion prevents new scheduled work for the monitor.
- Associated check results and incidents are removed.
- In-flight processing cannot recreate records after finalized deletion.
- The application does not report deletion success before required cleanup is complete.
- Deletion does not affect unrelated monitors.

### US-030 — Observe Monitoring Infrastructure Failures

**Priority:** P1  
**Related PRD:** NFR-007

**Story**

As a system operator, I want to identify worker failures and missed checks so that I can distinguish TracePoint infrastructure problems from failures of monitored APIs.

**Acceptance Criteria**

- Worker failures are logged.
- Recovery events are identifiable.
- Missed scheduled checks can be diagnosed.
- Monitoring infrastructure failures are distinguishable from endpoint failures.
- Logs do not expose sensitive configuration.

---

## 9. Story Summary

| Epic | Story IDs | Count |
|---|---|---:|
| Authentication and Accounts | US-001–US-004 | 4 |
| Monitor Management | US-005–US-010 | 6 |
| Automated Monitoring | US-011–US-015 | 5 |
| Dashboard and Statistics | US-016–US-019 | 4 |
| Incident Management | US-020–US-022 | 3 |
| History and Retention | US-023–US-025 | 3 |
| Security and Reliability | US-026–US-030 | 5 |
| **Total** | **US-001–US-030** | **30** |

## 10. Definition of Done

A user story is considered complete when:

1. Its approved acceptance criteria are satisfied.
2. Relevant validation and authorization checks are implemented.
3. Automated tests cover important successful and failure scenarios.
4. The implementation has been manually verified where appropriate.
5. Existing related functionality continues to work.
6. Errors and edge cases are handled.
7. Relevant documentation is updated.
8. The implementation is committed to the project repository.

A story involving deployed infrastructure must also be verified in the target environment before being considered production-ready.

## 11. Implementation Planning Notes

These stories define required behavior, not a mandatory coding order.

Implementation planning should consider dependencies such as:

- Authentication before protected monitor management.
- Monitor persistence before scheduled execution.
- Safe HTTP execution before enabling automated monitoring.
- Check-result storage before historical dashboards.
- Health evaluation before incident detection.
- Scheduling-slot tracking before accurate monitoring coverage.
- Worker coordination before multi-worker concurrency guarantees.
- Data relationships before deletion and retention mechanisms.

Technical tasks, database schemas, API routes, and architecture decisions will be derived from these dependencies.

## 12. Review Status

**Document status:** Draft for implementation planning.

**Requirements source:** TracePoint PRD v1.0.

**Next step:** Review story scope and dependencies, then design the system architecture before implementation.