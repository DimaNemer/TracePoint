# TracePoint — Product Requirements Document

**Version:** 1.0  
**Status:** MVP Requirements Approved  
**Date:** November 17, 2026  
**Product:** TracePoint  
**Document owner:** Project developer  
**Related documents:** `overview.md`, `user-stories.md`, `architecture.md`, `database-design.md`, `api-specification.md`, `ui-ux.md`, `test-plan.md`

---

## 1. Purpose

This Product Requirements Document (PRD) defines the functional requirements, non-functional requirements, business rules, constraints, and acceptance criteria for TracePoint's Minimum Viable Product (MVP).

It establishes what the product must accomplish and provides a shared reference for implementation, testing, and release decisions.

Technical implementation details that do not affect product behavior will be documented separately in architecture and design documents.

## 2. Product Summary

TracePoint is a web-based API monitoring platform designed for developers and students who need to monitor the availability and performance of their deployed APIs.

Users can create monitors for HTTP endpoints, configure expected responses, and review monitoring results through a personal dashboard.

TracePoint performs scheduled checks on the server, independently of whether a user has the website open.

The system records HTTP responses, response times, timeouts, network errors, and monitoring history. It also identifies incidents based on repeated unhealthy results and records their recovery.

The MVP prioritizes reliable monitoring, secure execution, understandable reporting, and manageable infrastructure usage.

## 3. Problem Statement

Developers often need a simple way to determine whether their deployed APIs remain available and respond as expected.

Manually testing endpoints does not provide continuous visibility, and a website that only checks an API while a browser is open cannot provide dependable background monitoring.

TracePoint addresses these problems by offering scheduled API checks, historical results, uptime reporting, and incident tracking.

## 4. Product Goals

### G-001 — Continuous Monitoring

Execute scheduled API checks independently of user browser sessions.

### G-002 — Visibility

Provide understandable information about API health, response times, and monitoring history.

### G-003 — Incident Awareness

Detect repeated unhealthy results and record confirmed incidents and recoveries.

### G-004 — Security

Protect user data and prevent monitoring functionality from being used to access restricted network resources.

### G-005 — Reliability

Handle timeouts, network failures, worker interruptions, and duplicate processing predictably.

### G-006 — Manageable MVP Scope

Deliver a functional, deployed monitoring platform without introducing unnecessary operational complexity.

## 5. Target Users

### Primary Users

- Junior and experienced developers monitoring deployed APIs.
- Students testing personal or academic projects.
- Developers managing a small number of application endpoints.

### User Needs

Users need to:

- Sign in securely.
- Register endpoints for monitoring.
- Configure how an endpoint should be checked.
- Review the latest monitoring status.
- Inspect historical check results.
- Understand uptime and monitoring coverage.
- Identify incidents and recoveries.
- Pause, resume, edit, or delete monitors.

## 6. MVP Scope

### 6.1 Included Features

- Google and GitHub OAuth authentication.
- Personal monitoring dashboard.
- Monitor creation, editing, viewing, pausing, resuming, and deletion.
- Support for GET, HEAD, POST, PUT, PATCH, and DELETE.
- Permitted custom request headers.
- Optional JSON request bodies for POST, PUT, and PATCH.
- Configurable expected HTTP status codes.
- Automated five-minute monitoring intervals.
- HTTP response and latency recording.
- Timeout and network-error handling.
- Check-based uptime calculations.
- Monitoring coverage calculations.
- Incident detection and recovery tracking.
- Historical monitoring results and incident records.
- Permanent monitor deletion.
- Automatic expiration of detailed check results after 90 days.
- Per-user monitor limits and global execution concurrency limits.
- Server-side security and ownership enforcement.

### 6.2 Excluded Features

The following features are outside the MVP:

- Email, SMS, or push notifications.
- Team accounts and shared workspaces.
- Public status pages.
- Custom monitoring intervals.
- Advanced monitoring schedules.
- Real-time streaming dashboards.
- Custom alert thresholds.
- Multi-region monitoring.
- Advanced request-body formats.
- Built-in authentication presets for monitored APIs.
- Automatic retries of monitored HTTP requests.
- Automatic historical catch-up checks.
- Archived monitors and restore functionality.
- Traditional email/password registration.
- Automatic linking of Google and GitHub accounts.
- Long-term aggregated uptime reporting beyond retained detailed history.

These features may be considered for future releases.

---

## 7. Functional Requirements

### FR-001 — Authentication and User Accounts

**Priority:** Must Have

TracePoint shall authenticate users through supported external identity providers.

**Requirements**

- **FR-001.1:** Users shall be able to sign in using Google OAuth.
- **FR-001.2:** Users shall be able to sign in using GitHub OAuth.
- **FR-001.3:** Successful first-time authentication shall create a TracePoint user account.
- **FR-001.4:** Returning users shall be recognized through their linked provider identity.
- **FR-001.5:** The system shall maintain secure authenticated sessions.
- **FR-001.6:** Users shall be able to sign out.
- **FR-001.7:** Protected monitoring resources shall require authentication.
- **FR-001.8:** Users shall only access monitoring resources belonging to their TracePoint account.
- **FR-001.9:** Authentication failures shall produce understandable, non-sensitive error messages.
- **FR-001.10:** Accounts shall not be automatically merged solely because provider email addresses match.

**Acceptance Criteria**

- A user can sign in using Google or GitHub.
- First-time sign-in creates an account.
- Repeated sign-in through the same provider identity retrieves the existing account.
- Unauthenticated requests to protected resources are rejected.
- One user cannot access another user's monitors, check results, or incidents.
- Sign-out terminates the authenticated application session.
- Authentication errors do not expose credentials or tokens.

### FR-002 — Monitor Management

**Priority:** Must Have

Authenticated users shall be able to manage their own API monitors.

**Requirements**

- **FR-002.1:** Users shall be able to create a monitor with a name, HTTPS endpoint URL, HTTP method, and expected status configuration.
- **FR-002.2:** Users shall be able to configure permitted request headers.
- **FR-002.3:** Users shall be able to configure an optional JSON body for POST, PUT, and PATCH requests.
- **FR-002.4:** GET, HEAD, and DELETE shall not support configured request bodies in the MVP.
- **FR-002.5:** Users shall be able to view their monitor list and individual monitor details.
- **FR-002.6:** Users shall be able to edit monitor configuration.
- **FR-002.7:** Users shall be able to pause active monitors.
- **FR-002.8:** Users shall be able to resume paused monitors.
- **FR-002.9:** Paused monitors shall not generate new scheduled monitoring requests.
- **FR-002.10:** Pausing shall preserve existing monitoring history and incidents.
- **FR-002.11:** Users shall be able to configure one or more exact expected HTTP status codes or use the default 2xx range.
- **FR-002.12:** The system shall validate monitor configuration before accepting it.
- **FR-002.13:** Users shall be able to permanently delete monitors they own.
- **FR-002.14:** Permanent deletion shall remove the monitor and its associated check results and incidents.
- **FR-002.15:** Permanent deletion shall require explicit user confirmation.
- **FR-002.16:** Deleted monitors shall no longer appear in the dashboard or monitor list.
- **FR-002.17:** Deleted monitors shall not generate future scheduled checks.
- **FR-002.18:** Background processing shall not recreate monitoring records after deletion is finalized.
- **FR-002.19:** Monitor management operations shall enforce ownership server-side.
- **FR-002.20:** Each user shall be limited to 10 monitors.
- **FR-002.21:** Active and paused monitors shall both count toward this limit.
- **FR-002.22:** Monitor creation beyond the limit shall be rejected.
- **FR-002.23:** The monitor limit shall be enforced server-side.
- **FR-002.24:** Concurrent creation requests shall not allow a user to exceed the limit.

**Acceptance Criteria**

- A user can create and configure a valid monitor.
- Invalid configurations are rejected with understandable errors.
- A user can edit, pause, and resume a monitor.
- Pausing preserves existing history.
- A user with 10 monitors cannot create an eleventh.
- Deleting a monitor frees capacity after deletion completes.
- Deletion removes associated results and incidents.
- A cancelled deletion confirmation changes nothing.
- Another user's monitor cannot be modified or deleted.

### FR-003 — Automated Monitoring and Scheduling

**Priority:** Must Have

TracePoint shall execute monitoring checks on a server-side schedule.

**Requirements**

- **FR-003.1:** Active monitors shall be scheduled at five-minute intervals.
- **FR-003.2:** Monitoring shall continue independently of browser sessions.
- **FR-003.3:** Paused monitors shall not be scheduled for new checks.
- **FR-003.4:** Each execution shall be associated with its monitor and scheduled execution slot.
- **FR-003.5:** The system shall record an outcome for each executed monitoring attempt when an outcome can be established.
- **FR-003.6:** The monitoring engine shall distinguish successful HTTP responses, unexpected HTTP responses, timeouts, and network errors.
- **FR-003.7:** Monitoring requests shall have a fixed 10-second total timeout.
- **FR-003.8:** Monitoring requests shall not be automatically retried.
- **FR-003.9:** Request execution shall follow the approved endpoint and request-security restrictions.
- **FR-003.10:** The system shall record the actual execution time and associate it with the intended scheduled slot.
- **FR-003.11:** The system shall identify scheduled checks missed because monitoring infrastructure was unavailable or unable to execute them.
- **FR-003.12:** Missed historical checks shall not be replayed.
- **FR-003.13:** After recovery, monitoring shall resume from the current or next eligible scheduled interval.
- **FR-003.14:** Missed checks shall be distinguished from unhealthy endpoint results.
- **FR-003.15:** Scheduling state shall support identification of duplicate and incomplete execution attempts.
- **FR-003.16:** Worker recovery shall not blindly repeat requests with uncertain outcomes.
- **FR-003.17:** Duplicate processing shall not create duplicate persisted results or inconsistent incident transitions.
- **FR-003.18:** Worker interruptions shall not conceal missing scheduled executions from coverage reporting.
- **FR-003.19:** The monitoring engine shall allow no more than five concurrent HTTP checks globally.
- **FR-003.20:** Additional eligible checks may wait for available execution capacity within their permitted scheduling window.
- **FR-003.21:** Global concurrency limits shall apply across all worker instances.
- **FR-003.22:** Worker coordination shall prevent overlapping execution from exceeding the configured limit.
- **FR-003.23:** Checks that cannot begin within the permitted scheduling window shall follow the missed-check policy rather than being replayed indefinitely.

**Acceptance Criteria**

- Checks continue when the user's browser is closed.
- A scheduled request that exceeds 10 seconds is classified as timed out.
- An unavailable worker produces identifiable missed slots.
- Missed checks are not replayed after recovery.
- An interrupted POST request is not blindly repeated.
- Duplicate processing does not create duplicate results.
- No more than five monitoring requests execute simultaneously across the deployment.
- Paused and deleted monitors do not receive new scheduled checks.

### FR-004 — HTTP Request Execution and Health Evaluation

**Priority:** Must Have

TracePoint shall execute supported HTTP requests and evaluate their results against monitor configuration.

**Requirements**

- **FR-004.1:** The system shall support GET requests.
- **FR-004.2:** The system shall support HEAD requests.
- **FR-004.3:** The system shall support POST requests.
- **FR-004.4:** The system shall support PUT requests.
- **FR-004.5:** The system shall support PATCH requests.
- **FR-004.6:** The system shall support DELETE requests.
- **FR-004.7:** Requests shall target permitted public HTTPS endpoints only.
- **FR-004.8:** Request headers shall be validated against a permitted-header policy.
- **FR-004.9:** POST, PUT, and PATCH may include a validated JSON request body.
- **FR-004.10:** Request headers and bodies shall be subject to configured size limits.
- **FR-004.11:** HTTP redirects shall not be followed automatically in the MVP.
- **FR-004.12:** The system shall warn users that repeatedly executing state-changing methods may produce remote side effects.
- **FR-004.13:** By default, HTTP statuses from 200 through 299 shall be classified as healthy.
- **FR-004.14:** Users shall be able to replace the default expected-status range with one or more exact expected HTTP status codes.
- **FR-004.15:** A received HTTP status shall be classified as healthy only when it matches the monitor's expected-status configuration.
- **FR-004.16:** Actual HTTP status codes shall be stored independently of the resulting health classification.
- **FR-004.17:** Timeouts and network errors shall always be classified as unhealthy, regardless of expected-status configuration.
- **FR-004.18:** The system shall measure and record elapsed request time for executed checks.
- **FR-004.19:** The system shall attempt to cancel requests when the configured timeout is reached.
- **FR-004.20:** The system shall not automatically retry monitoring requests.
- **FR-004.21:** Server-side request execution shall enforce protections against server-side request forgery (SSRF), including restricted destination addresses and DNS-related bypasses.
- **FR-004.22:** Sensitive request configuration shall not be exposed in monitoring history, logs, or unauthorized responses.

**Acceptance Criteria**

- A 200 response is healthy under the default configuration.
- A 500 response is unhealthy under the default configuration.
- A 404 response can be healthy when explicitly configured as expected.
- A network error remains unhealthy even when a custom status is configured.
- The actual HTTP status is preserved.
- A request exceeding the timeout is recorded as a timeout outcome.
- Private network destinations and prohibited redirect behavior are blocked.
- State-changing requests are not automatically retried.
- Invalid headers or request bodies are rejected.

### FR-005 — Dashboard and Monitoring Statistics

**Priority:** Must Have

TracePoint shall provide a personal dashboard showing monitoring status and historical performance.

**Requirements**

- **FR-005.1:** Users shall see their own monitors.
- **FR-005.2:** Each monitor shall display its latest available health state.
- **FR-005.3:** The dashboard shall distinguish healthy, unhealthy, paused, and not-yet-checked conditions.
- **FR-005.4:** Users shall be able to view recent HTTP response information.
- **FR-005.5:** Users shall be able to view response-time history.
- **FR-005.6:** Users shall be able to view incident information.
- **FR-005.7:** The system shall calculate check-based uptime as healthy executed checks divided by all executed checks with a recorded health outcome, multiplied by 100.
- **FR-005.8:** The system shall calculate monitoring coverage as executed scheduled checks divided by expected scheduled checks, multiplied by 100.
- **FR-005.9:** Missed checks shall be excluded from the uptime denominator.
- **FR-005.10:** Missed checks shall reduce monitoring coverage.
- **FR-005.11:** Intentionally paused periods shall be excluded from expected scheduled checks.
- **FR-005.12:** Each scheduled slot shall be counted at most once for coverage purposes.
- **FR-005.13:** A late execution shall be attributed to its original scheduled slot when it remains eligible under the scheduling policy.
- **FR-005.14:** Metrics with no valid denominator shall display N/A rather than an invented percentage.
- **FR-005.15:** The dashboard shall distinguish check-based uptime from exact time-based service availability.
- **FR-005.16:** Historical statistics shall use a clearly defined reporting window within retained data.
- **FR-005.17:** Expired check results shall not be included in historical calculations.

**Acceptance Criteria**

Given six expected scheduled checks, with four healthy outcomes, one unhealthy outcome, and one missed check:

- Check-based uptime equals 80%.
- Monitoring coverage is approximately 83.3%.
- The missed check is not classified as an API failure.
- The dashboard does not claim exact time-based availability.
- A monitor with no executed checks displays N/A for uptime.

### FR-006 — Incident Detection and Recovery

**Priority:** Must Have

TracePoint shall detect and track incidents using completed monitoring outcomes.

**Requirements**

- **FR-006.1:** Two consecutive unhealthy completed checks shall confirm and open an incident.
- **FR-006.2:** A single unhealthy completed check shall not immediately open an incident.
- **FR-006.3:** One healthy completed check shall resolve an active incident.
- **FR-006.4:** A healthy check shall reset the consecutive failure count.
- **FR-006.5:** Timeouts and network errors shall count as unhealthy outcomes.
- **FR-006.6:** Missed scheduled checks shall not independently increase failure counts or trigger incident transitions.
- **FR-006.7:** The system shall not create multiple simultaneously active incidents for the same monitor.
- **FR-006.8:** Incident records shall preserve the time of the first observed failure, the confirmation time, and the recovery time when available.
- **FR-006.9:** Repeated unhealthy results during an active incident shall not create duplicate incidents.
- **FR-006.10:** Incident transitions shall be processed consistently and protected against duplicate execution.
- **FR-006.11:** Incident history shall be accessible to the monitor owner.
- **FR-006.12:** Incident records shall preserve essential timestamps, state, and failure information independently of detailed check-result retention.

**Acceptance Criteria**

- First unhealthy check: no incident yet.
- Second consecutive unhealthy check: incident opens.
- Additional unhealthy checks: the same incident remains open.
- First healthy check after an incident: incident resolves.
- A healthy check between failures resets the failure sequence.
- A missed check does not count as an additional failure.
- Duplicate result processing does not open or resolve an incident twice.

### FR-007 — Monitoring History and Data Retention

**Priority:** Must Have

TracePoint shall store and display historical monitoring results according to the approved retention policy.

**Requirements**

- **FR-007.1:** The system shall retain monitoring results associated with their parent monitor.
- **FR-007.2:** Each check result shall identify its execution time and monitoring outcome.
- **FR-007.3:** Results shall include the actual HTTP status when one was received.
- **FR-007.4:** Results shall preserve available elapsed-time measurements and error classifications.
- **FR-007.5:** Users shall be able to retrieve their own monitoring history.
- **FR-007.6:** Historical reporting shall support check-based uptime calculations.
- **FR-007.7:** Historical reporting shall support monitoring coverage calculations.
- **FR-007.8:** Missing execution slots shall be distinguishable from unhealthy endpoint responses.
- **FR-007.9:** Deleting a monitor shall remove its associated check results and incidents.
- **FR-007.10:** Finalized deletion shall not leave accessible orphaned monitoring history.
- **FR-007.11:** Detailed check results shall be retained for 90 days.
- **FR-007.12:** Results older than the retention period shall be excluded from historical queries and automatically removed from active storage.
- **FR-007.13:** Incident records shall remain available until the associated monitor is deleted.
- **FR-007.14:** Incident records shall remain meaningful after related check results expire.
- **FR-007.15:** Expired results shall not contribute to uptime or latency calculations.
- **FR-007.16:** The interface shall communicate the available reporting history.
- **FR-007.17:** Permanent monitor deletion shall remove associated records regardless of age.

**Acceptance Criteria**

- Recent check results remain available.
- Check results older than 90 days are excluded from queries and eligible for automated cleanup.
- Incident records older than 90 days remain available while their monitor exists.
- Expired results are excluded from historical statistics.
- Deleting a monitor removes all associated check results and incidents.
- Users cannot retrieve another user's historical records.

---

## 8. Non-Functional Requirements

### NFR-001 — Security

- Authentication shall use maintained OAuth integration mechanisms.
- OAuth credentials and application secrets shall be stored securely on the server.
- Sessions shall use secure handling practices.
- Authorization shall be enforced server-side.
- Monitoring execution shall protect against SSRF, including private addresses, loopback addresses, restricted destinations, and DNS-related bypasses.
- HTTPS shall be required for monitored endpoints.
- Redirect following shall be disabled in the MVP.
- Request headers and bodies shall be validated and size-limited.
- Sensitive credentials and headers shall not be exposed through logs, historical results, or unauthorized API responses.
- The application shall use least-privilege access where practical.

### NFR-002 — Reliability

- Monitoring shall run independently of browser sessions.
- Monitoring attempts shall use a fixed 10-second timeout.
- No automatic HTTP request retries shall occur.
- Missed historical checks shall not be replayed.
- Worker recovery shall distinguish missing, incomplete, and completed executions.
- Duplicate processing shall not corrupt monitoring results or incident state.
- Worker interruptions shall not silently conceal monitoring gaps.
- Deletion and worker execution shall be coordinated safely.

### NFR-003 — Performance and Capacity

- Each user shall be limited to 10 monitors.
- The deployment shall permit a maximum of five concurrent monitoring requests globally.
- Capacity limits shall be enforced server-side.
- Database queries shall use appropriate indexes.
- Historical results shall follow the 90-day retention policy.
- Scheduling delays and worker throughput shall be tested under expected MVP load.
- Capacity settings shall be configurable server-side for future adjustments.

### NFR-004 — Data Integrity

- Monitor ownership relationships shall remain consistent.
- Check results and incidents shall remain associated with valid monitors.
- Deletion shall not report success while required cleanup is incomplete.
- Concurrent creation requests shall not bypass monitor limits.
- Duplicate scheduled execution processing shall not create duplicate results or incident transitions.
- Incident records shall survive normal check-result expiration.
- Expired check results shall not be used in historical calculations.

### NFR-005 — Usability

- The interface shall be understandable to developers with basic API knowledge.
- Monitor creation and configuration shall provide clear validation feedback.
- Monitoring status and error conditions shall be distinguishable.
- Destructive actions shall require confirmation.
- Historical metrics shall be labeled clearly.
- The dashboard shall support common desktop and mobile screen sizes.

### NFR-006 — Maintainability

- The codebase shall use TypeScript.
- Responsibilities for authentication, monitoring, scheduling, persistence, and presentation shall be separated.
- Key technical decisions shall be documented.
- Environment-specific configuration shall not be hard-coded into application logic.
- Automated tests shall cover critical business rules and security boundaries.
- Deployment and local development instructions shall be documented.

### NFR-007 — Observability

- Monitoring worker execution and failures shall be logged.
- Worker interruptions and recovery events shall be identifiable.
- Monitoring infrastructure failures shall be distinguishable from monitored API failures.
- Sensitive configuration shall be redacted from logs.
- Scheduling delays, missed checks, and execution failures shall be diagnosable.

---

## 9. Business Rules

| ID | Rule |
|---|---|
| BR-001 | Each monitor belongs to one TracePoint user. |
| BR-002 | Only the monitor owner may manage or access its data. |
| BR-003 | Only active monitors are eligible for scheduled checks. |
| BR-004 | The monitoring interval is fixed at five minutes for the MVP. |
| BR-005 | A monitoring request has a 10-second total timeout. |
| BR-006 | Monitoring requests are not automatically retried. |
| BR-007 | Monitoring continues independently of browser sessions. |
| BR-008 | Only permitted public HTTPS endpoints may be monitored. |
| BR-009 | Monitoring configuration must pass validation before execution. |
| BR-010 | Pausing a monitor preserves its existing history. |
| BR-011 | Expected HTTP status defaults to 200–299 unless overridden by exact status codes. |
| BR-012 | Actual HTTP status and evaluated health are separate values. |
| BR-013 | Two consecutive unhealthy completed checks open an incident. |
| BR-014 | One healthy completed check resolves an active incident. |
| BR-015 | A healthy check resets the consecutive failure count. |
| BR-016 | A monitor may have at most one active incident. |
| BR-017 | Missed checks do not independently cause incident transitions. |
| BR-018 | Check-based uptime uses healthy outcomes divided by executed checks with recorded health outcomes. |
| BR-019 | Monitoring coverage uses executed scheduled slots divided by expected scheduled slots. |
| BR-020 | Intentionally paused periods are excluded from expected slots. |
| BR-021 | Metrics with no valid denominator display N/A. |
| BR-022 | Permanent monitor deletion removes its configuration, check results, and incidents. |
| BR-023 | Permanent deletion requires explicit confirmation. |
| BR-024 | No new monitoring data may be persisted after deletion is finalized. |
| BR-025 | Only the monitor owner may request deletion. |
| BR-026 | Google and GitHub are the supported MVP authentication providers. |
| BR-027 | Monitoring resources belong to the TracePoint user identity, not directly to an OAuth provider. |
| BR-028 | Accounts are not automatically merged based solely on matching email addresses. |
| BR-029 | A missed check is not a healthy or unhealthy API result. |
| BR-030 | Missed historical monitoring requests are not replayed. |
| BR-031 | Worker recovery must not blindly retry requests with uncertain outcomes. |
| BR-032 | Duplicate processing must not create duplicate results or incident transitions. |
| BR-033 | Missed checks remain represented in monitoring coverage. |
| BR-034 | Detailed check results are retained for 90 days. |
| BR-035 | Incident records are retained until monitor deletion. |
| BR-036 | Historical metrics use retained data within the selected reporting window. |
| BR-037 | Expired results are excluded from queries and removed through automated cleanup. |
| BR-038 | A user may own up to 10 monitors, including paused monitors. |
| BR-039 | No more than five HTTP monitoring requests may execute concurrently across the deployment. |
| BR-040 | Capacity limits must remain effective under concurrent operations. |
| BR-041 | GET, HEAD, POST, PUT, PATCH, and DELETE are supported monitoring methods. |
| BR-042 | Only POST, PUT, and PATCH support optional JSON bodies in the MVP. |
| BR-043 | Custom headers must satisfy the permitted-header policy. |
| BR-044 | Automatic redirect following is disabled. |
| BR-045 | State-changing monitoring requests require a clear user warning. |

---

## 10. Primary User Workflows

### Workflow A — Sign In

1. User opens TracePoint.
2. User selects Google or GitHub.
3. Authentication provider verifies the user's identity.
4. TracePoint creates or retrieves the associated user account.
5. User is redirected to their personal dashboard.

### Workflow B — Create a Monitor

1. Authenticated user opens the monitor creation form.
2. User enters a monitor name and public HTTPS URL.
3. User selects the HTTP method.
4. User optionally configures permitted headers.
5. For POST, PUT, or PATCH, the user may provide a JSON body.
6. User selects expected HTTP statuses or accepts the default 2xx range.
7. TracePoint validates the configuration and monitor limit.
8. The monitor is saved and becomes eligible for scheduled checks.

### Workflow C — Execute a Scheduled Check

1. Scheduler identifies an eligible active monitor.
2. Execution capacity is reserved without exceeding the global limit.
3. TracePoint validates the request destination and configuration.
4. The monitoring request executes with a 10-second timeout.
5. The outcome and available response information are recorded.
6. Health is evaluated against expected statuses.
7. Incident state is updated when appropriate.
8. Dashboard history and metrics reflect the new result.

### Workflow D — Detect and Resolve an Incident

1. First unhealthy completed check is recorded.
2. Second consecutive unhealthy completed check confirms an incident.
3. TracePoint records incident details.
4. Further unhealthy checks leave the incident open.
5. One healthy completed check resolves the incident.
6. The incident remains available in history.

### Workflow E — Recover From a Worker Interruption

1. Monitoring infrastructure becomes unavailable.
2. Scheduled slots may be missed or left incomplete.
3. The worker becomes available again.
4. TracePoint reconciles scheduling state without blindly replaying HTTP requests.
5. Missed historical checks remain missing.
6. Eligible current or future checks resume.
7. Coverage reporting reflects the missed slots.

### Workflow F — Delete a Monitor

1. User selects a monitor they own.
2. User requests deletion.
3. TracePoint displays a permanent-deletion confirmation.
4. User confirms.
5. TracePoint coordinates deletion with monitoring execution.
6. Monitor configuration, check results, and incidents are removed.
7. Future checks are prevented.
8. The monitor disappears from the user's dashboard.

---

## 11. Data Requirements

TracePoint requires the following logical data entities. Physical schemas will be defined in `database-design.md`.

### 11.1 User

- Internal user identifier.
- Relevant profile information.
- OAuth provider identity relationships.
- Account creation and update timestamps.

### 11.2 Monitor

- Monitor identifier.
- Owner identifier.
- Name.
- Endpoint URL.
- HTTP method.
- Permitted request headers.
- Optional JSON body.
- Expected HTTP status configuration.
- Active or paused state.
- Scheduling information.
- Creation and update timestamps.

### 11.3 Check Result

- Check-result identifier.
- Monitor identifier.
- Scheduled execution identifier or slot.
- Scheduled and actual execution timestamps.
- HTTP status when available.
- Health classification.
- Response time when available.
- Timeout or network-error classification.
- Retention or expiration timestamp.

### 11.4 Incident

- Incident identifier.
- Monitor identifier.
- Incident state.
- First observed failure timestamp.
- Incident confirmation timestamp.
- Recovery timestamp when resolved.
- Relevant failure summary.

### 11.5 Scheduling and Execution State

The system must retain sufficient information to:

- Identify expected scheduled slots.
- Track executed and missed checks.
- Avoid duplicate result processing.
- Coordinate global concurrency.
- Recover from interrupted work.
- Prevent processing after monitor deletion.

The exact persistence model is an architecture decision.

---

## 12. Dependencies and Constraints

### 12.1 Technology Direction

The project is planned around:

- Next.js and TypeScript for the web application.
- MongoDB and Mongoose for application persistence.
- A maintained OAuth authentication solution.
- Server-side background scheduling and execution.
- A deployment environment capable of running scheduled monitoring tasks.

Final package choices, worker topology, and hosting configuration will be documented in `architecture.md`.

### 12.2 External Dependencies

- Google OAuth.
- GitHub OAuth.
- MongoDB hosting.
- Application and monitoring-worker hosting.
- External API endpoints configured by users.

### 12.3 Constraints

- Monitoring frequency is fixed at five minutes.
- Requests have a fixed 10-second timeout.
- Only public HTTPS endpoints are permitted.
- Redirect following is disabled.
- No automatic HTTP retries or historical replay.
- Detailed check results are retained for 90 days.
- Each user may create at most 10 monitors.
- Global monitoring concurrency is limited to five requests.
- The MVP is intended for individual users rather than teams.

---

## 13. MVP Acceptance and Release Criteria

TracePoint MVP is ready for release when the following conditions are satisfied.

### Authentication

- Google and GitHub sign-in work.
- Protected routes enforce authentication.
- User data isolation is verified.

### Monitor Management

- Users can create, edit, pause, resume, and delete monitors.
- Request configuration is validated.
- Monitor limits cannot be bypassed by concurrent requests.
- Permanent deletion removes associated history safely.

### Monitoring Execution

- Scheduled checks run independently of browser sessions.
- All six supported HTTP methods are handled within approved restrictions.
- Timeout, HTTP error, and network-error outcomes are recorded correctly.
- SSRF restrictions are tested.
- Redirects and automatic retries are disabled.
- Global concurrency is enforced.

### Reliability

- Worker interruption does not trigger historical replay.
- Missed checks are distinguishable from API failures.
- Duplicate processing does not corrupt results or incidents.
- In-flight work cannot recreate records after finalized deletion.

### Dashboard and Incidents

- Dashboard displays monitor health and recent results.
- Uptime and coverage calculations are correct.
- Incident creation and recovery follow the approved thresholds.
- Historical results and incidents are accessible only to their owner.

### Retention

- Detailed results expire after 90 days.
- Expired data is excluded from queries.
- Incident history remains available until monitor deletion.

### Delivery

- The application is deployed.
- Core workflows have been tested.
- Known limitations are documented.
- Setup and deployment instructions are available.
- Relevant screenshots or demonstration evidence are included.

---

## 14. Decision Register

| ID | Decision | Status |
|---|---|---|
| D-001 | Six HTTP methods, permitted headers, optional JSON bodies for POST/PUT/PATCH, restricted HTTPS execution, no redirects or retries | Approved |
| D-002 | Fixed 10-second total timeout | Approved |
| D-003 | Default 2xx health expectation with configurable exact status codes | Approved |
| D-004 | Two consecutive unhealthy checks open an incident; one healthy check resolves it | Approved |
| D-005 | Check-based uptime plus monitoring coverage | Approved |
| D-006 | Permanent monitor deletion including results and incidents | Approved |
| D-007 | Google and GitHub OAuth; no traditional passwords | Approved |
| D-008 | Skip missed checks; controlled worker recovery without historical replay | Approved |
| D-009 | Retain detailed results for 90 days; retain incidents until monitor deletion | Approved |
| D-010 | Maximum 10 monitors per user and five concurrent checks globally | Approved |

All ten MVP product decisions are approved.

---

## 15. Open Architecture and Design Questions

The following questions remain for technical design. They do not change the approved product scope.

1. Which scheduling architecture will run five-minute checks reliably in the selected deployment environment?
2. How will scheduled execution slots be identified and persisted?
3. What scheduling window determines when a delayed check becomes missed?
4. How will worker leases, global concurrency, and crash recovery be coordinated?
5. How will monitor deletion prevent in-flight workers from recreating records?
6. How will monitor creation limits be enforced atomically?
7. How will OAuth identities and sessions be represented in MongoDB?
8. What exact request-header allowlist and body-size limits will be used?
9. How will sensitive header values be stored, protected, and redacted?
10. How will DNS resolution and outbound network connections enforce SSRF restrictions?
11. How will monitoring coverage be reconstructed accurately across pauses, deletions, and retention boundaries?
12. How will incomplete executions be represented without incorrectly treating them as completed API failures?
13. How will incident behavior during pause and resume be represented?
14. Which reporting windows will the dashboard offer?
15. How will database indexes and automated expiration be configured?
16. How will scheduling throughput and delays be measured during deployment testing?

These questions will be resolved through architecture decisions, database design, API specification, and testing.

---

## 16. Related Documentation

- `overview.md` — Product vision, motivation, and high-level scope.
- `user-stories.md` — User-focused requirements and acceptance scenarios.
- `architecture.md` — System components, execution flow, and infrastructure design.
- `database-design.md` — Data models, relationships, indexes, and retention.
- `api-specification.md` — Application API endpoints and request/response contracts.
- `ui-ux.md` — Screens, user flows, and interface behavior.
- `test-plan.md` — Test strategy, scenarios, and verification results.
- `adr/` — Architecture Decision Records for significant technical choices.

---

## 17. Review and Change Control

**Version 1.0:** Initial approved MVP requirements baseline.

Changes to approved product behavior shall be reflected in a new PRD revision and any affected supporting documents.

Architecture decisions that implement the approved behavior may be recorded separately without changing the PRD version unless they alter product requirements.

**Review outcome:** All ten MVP product decisions approved. Detailed technical design remains pending.