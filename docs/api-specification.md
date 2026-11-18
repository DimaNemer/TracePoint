# TracePoint — REST API Specification

**Status:** Implementation-ready MVP baseline  
**Version:** 1.0 · 2026-11-18  
**Stack:** Next.js App Router / TypeScript / Auth.js / MongoDB + Mongoose / separate persistent Node.js worker  
**Base path:** `/api`  
**Scope:** Authenticated web application only; no public developer API in MVP.

> **Decision provenance:** API-001 through API-004 and PRD D-001–D-010 / DB-001–DB-013 reflect prior design discussions. Specific payload shapes, numeric limits other than the approved 10 monitors / 5 concurrent checks / 5-minute interval / 10-second timeout / 90-day retention, and detailed endpoint behavior below are **implementation defaults** adopted to unblock coding. Adjust deliberately and update this file when implementation proves a better option.

## 1. System boundaries

- **Next.js web/API:** OAuth login, secure sessions, authenticated CRUD, read APIs, validation, aggregation, owner isolation.
- **Persistent worker:** independently schedules and executes external HTTP checks, handles claims, records outcomes, updates health/incident state. It must continue when all browsers close. Deploy separately from the Vercel web app (e.g., Railway/Render) with a single active worker for the MVP.
- **MongoDB Atlas:** shared persistence for `users`, `monitors`, `scheduledChecks`, `checkResults`, `incidents`, plus Auth.js adapter collections if used.
- Browser clients **never** call worker-only functionality or connect directly to MongoDB. No publicly exposed worker trigger endpoints in MVP.

## 2. API conventions

### 2.1 Auth and ownership (API-004 default)

Use Google and GitHub OAuth through Auth.js with secure, HttpOnly, SameSite cookies (Secure in production), session validation in every protected route handler, and server-side authorization. No email/password login and no automatic account linking solely by email. Session strategy and adapter must be chosen consistently during setup; use a stable internal user ID in all owner queries. For browser mutation endpoints, enforce origin/CSRF protection appropriate to Auth.js and the deployment; do not assume SameSite alone is sufficient. Never use client-supplied `userId` to authorize a resource.

- Missing/invalid/expired session → `401 AUTH_REQUIRED`.
- Other user's or nonexistent monitor → identical `404 MONITOR_NOT_FOUND` (avoid existence disclosure).
- All check/incident reads must first establish ownership of the parent monitor or use an owner-constrained aggregation.
- Rate-limit sensitive mutations and sign-in endpoints; log only sanitized audit metadata.
- Web and worker credentials are separate; no browser access to encryption keys.

### 2.2 Response envelopes (API-002)

Success: `{ "data": <resource | array | object> }`; list endpoints may additionally return `pagination`. Error: `{ "error": { "code": "STABLE_CODE", "message": "Safe human-readable message", "details": { ... } } }`. Omit `details` when not needed. No stack traces, decrypted headers/body, raw network errors, or database IDs/credentials in errors. `Content-Type: application/json`; dates ISO-8601 UTC; durations milliseconds; statuses uppercase enum strings. For delete, use `200` with `{ "data": { "deleted": true, "id": "..." } }` rather than a bodyless 204 to preserve the envelope.

### 2.3 Pagination and filtering (API-003)

`GET /api/monitors` returns all owned monitors (maximum 10, including paused/deleting); no pagination. Check history and incident lists use **opaque cursor pagination**, newest first, with `limit` default 20 and maximum 100 (implementation defaults). Example:

```json
{ "data": [ { "id": "..." } ], "pagination": { "nextCursor": "opaque-or-null", "hasMore": true } }
```

Use stable indexed sort (`scheduledAt DESC, _id DESC` for checks; `confirmedAt DESC, _id DESC` for incidents). Cursor encodes the last sort tuple plus a filter signature and is authenticated/signed or otherwise tamper-resistant; reject invalid/mismatched cursors with `400 INVALID_CURSOR`. Fetch `limit + 1` records to determine `hasMore`. Filter **before** paginating. Cap date ranges at 90 days for detailed checks. Validate all query values; never accept raw MongoDB operators or arbitrary sort keys.

### 2.4 Error status mapping

| HTTP | Stable error code | Example |
|---|---|---|
| 400 | `VALIDATION_ERROR`, `INVALID_CURSOR` | Bad URL, JSON, filter or cursor |
| 401 | `AUTH_REQUIRED` | Session missing or expired |
| 403 | `FORBIDDEN` | Valid user, disallowed global operation (not used to reveal foreign monitor) |
| 404 | `MONITOR_NOT_FOUND`, `INCIDENT_NOT_FOUND` | Absent or non-owned resource |
| 409 | `MONITOR_LIMIT_REACHED`, `STATE_CONFLICT`, `CONFIG_VERSION_CONFLICT` | Limit, invalid transition, stale edit |
| 413 | `PAYLOAD_TOO_LARGE` | Excessive headers/body |
| 429 | `RATE_LIMITED` | Request rate exceeded; add `Retry-After` when applicable |
| 500 | `INTERNAL_ERROR` | Unexpected failure; sanitized response |
| 503 | `SERVICE_UNAVAILABLE` | Temporary dependency outage |

**Important:** These HTTP statuses describe **TracePoint API operations**, not the HTTP statuses observed from a monitored endpoint. Observed status is stored separately in `checkResults.httpStatus`.

## 3. Domain contracts

### 3.1 Monitor (safe public projection)

```json
{
  "id": "65fa...",
  "name": "Products API",
  "url": "https://api.example.com/products",
  "method": "GET",
  "lifecycleStatus": "ACTIVE",
  "configVersion": 1,
  "expectedStatuses": { "mode": "ANY_2XX", "codes": [] },
  "intervalSeconds": 300,
  "timeoutMs": 10000,
  "hasCustomHeaders": false,
  "hasRequestBody": false,
  "latestHealth": "HEALTHY",
  "lastHttpStatus": 200,
  "lastResponseTimeMs": 145,
  "lastCompletedAt": "2026-11-18T10:20:01.000Z",
  "nextCheckAt": "2026-11-18T10:25:00.000Z",
  "createdAt": "2026-11-18T09:00:00.000Z",
  "updatedAt": "2026-11-18T09:00:00.000Z"
}
```

`lifecycleStatus`: `ACTIVE | PAUSED | DELETING` (DELETING is transient/read-only). `latestHealth`: `HEALTHY | UNHEALTHY | null` (**last known**, not proof of current availability). Monitoring freshness (`CURRENT | DELAYED | NO_DATA | PAUSED`) is **derived**, not the same as health. `lastHttpStatus` can be null for timeout/network error. Sensitive configuration is never returned: not even masked ciphertext; expose only presence flags and safe nonsecret configuration. In the edit UI, replacing sensitive fields is an explicit action; omission means **retain existing** on PATCH, `null` means **clear** when supported. Avoid sending back original secret values.

### 3.2 Check result

```json
{
  "id": "...", "monitorId": "...", "scheduledAt": "2026-11-18T10:20:00.000Z",
  "configVersion": 1, "outcome": "HTTP_RESPONSE", "health": "HEALTHY",
  "httpStatus": 200, "responseTimeMs": 145, "errorCode": null,
  "startedAt": "2026-11-18T10:20:00.500Z", "completedAt": "2026-11-18T10:20:00.645Z"
}
```

`outcome`: `HTTP_RESPONSE | TIMEOUT | NETWORK_ERROR`. `health`: `HEALTHY | UNHEALTHY`. TIMEOUT and NETWORK_ERROR always unhealthy and have `httpStatus: null`. `responseTimeMs` is actual elapsed time for a completed attempt; may be null if execution never started. `errorCode` is a safe enum such as `DNS_FAILURE`, `CONNECTION_REFUSED`, `TLS_ERROR`, `TIMEOUT`, `NETWORK_ERROR` (never raw exception details). A scheduled slot marked MISSED/INTERRUPTED without a reliable completed result is **not** fabricated as an unhealthy check.

### 3.3 Incident

```json
{
  "id": "...", "monitorId": "...", "status": "OPEN",
  "firstFailureAt": "2026-11-18T10:00:01.000Z",
  "confirmedAt": "2026-11-18T10:05:01.000Z",
  "resolvedAt": null, "closedAt": null, "closureReason": null,
  "triggerResultId": "...", "recoveryResultId": null,
  "createdAt": "2026-11-18T10:05:01.000Z"
}
```

`status`: `OPEN | RESOLVED | CLOSED_UNVERIFIED`. Only one OPEN incident per monitor. Two consecutive **unhealthy completed** checks open an incident; one **healthy completed** check resolves it. Pausing or meaningful configuration change closes OPEN incident as CLOSED_UNVERIFIED with reason `MONITOR_PAUSED | CONFIG_CHANGED`, not as recovery. Event timestamps derive from relevant result `completedAt`. Terminal incidents are never reopened. Incident data remains meaningful after 90-day result TTL expiry.

## 4. Endpoint catalog

All endpoints below require a valid session unless explicitly indicated. Route paths use Next.js `[id]` segments. Return resource-specific `data` projections and structured errors.

| Method | Path | Purpose | Success |
|---|---|---|---|
| GET | `/api/monitors` | Owned monitors; optional `search`, `lifecycle`, `health`, `freshness` | 200 list |
| POST | `/api/monitors` | Create monitor | 201 monitor |
| GET | `/api/monitors/:id` | One owned monitor | 200 monitor |
| PATCH | `/api/monitors/:id` | Partial configuration edit | 200 monitor |
| DELETE | `/api/monitors/:id` | Confirmed permanent deletion | 200 `{deleted,id}` |
| POST | `/api/monitors/:id/pause` | Pause, idempotent effect | 200 monitor |
| POST | `/api/monitors/:id/resume` | Resume, idempotent effect | 200 monitor |
| GET | `/api/monitors/:id/checks` | Check history | 200 paginated |
| GET | `/api/monitors/:id/metrics` | Check-based uptime, coverage, latency | 200 metrics |
| GET | `/api/monitors/:id/incidents` | Monitor incident history | 200 paginated |
| GET | `/api/incidents` | All owned incidents | 200 paginated |
| GET | `/api/incidents/:id` | One owned incident | 200 incident |
| GET | `/api/me` | Minimal current user profile | 200 profile |
| GET/POST | `/api/auth/*` | Auth.js-owned OAuth/session routes | Auth.js semantics |

**Not in MVP:** manual check trigger, arbitrary custom intervals, retries, redirects, alert delivery, manual incident resolution/acknowledgment, public API keys, billing, team/workspaces, webhooks, worker control routes. Dashboard can compose existing monitor, metrics, and incident endpoints; add a read-only summary endpoint only if measured performance requires it.

## 5. Monitor creation — `POST /api/monitors`

### Request body

```json
{
  "name": "Products API",
  "url": "https://api.example.com/products",
  "method": "GET",
  "expectedStatuses": { "mode": "ANY_2XX", "codes": [] },
  "headers": [{ "name": "X-API-Key", "value": "secret-value" }],
  "jsonBody": null
}
```

`name`, `url`, `method` required. `expectedStatuses`, `headers`, `jsonBody` optional. Defaults: `expectedStatuses={mode:'ANY_2XX',codes:[]}`, `headers=[]`, `jsonBody=null`. `method`: GET, HEAD, POST, PUT, PATCH, DELETE. JSON body allowed only for POST/PUT/PATCH; not for GET/HEAD/DELETE. JSON body is a JSON value (object/array/etc.) serialized safely, not arbitrary executable text; UI validates JSON before submission. For HEAD, no response body is expected. For state-changing methods, show warning before creation; do not silently send user-supplied unsafe requests.

### Validation defaults (adjust with tests)

- `name`: trimmed, 1–80 characters; reject control characters.
- `url`: absolute `https://` URL; no embedded credentials, fragments, local/private/reserved IPs, localhost or internal hostnames. Revalidate DNS **at execution time**, block redirects, and prevent DNS rebinding/SSRF by validating all resolved addresses and binding the validated destination during connection. Restrict ports to 443 in MVP (document exception only if needed); disallow IP literals by default. No request to private cloud metadata addresses. Reject URL length over 2,048 characters.
- `expectedStatuses`: `ANY_2XX` or `EXACT`; for EXACT require 1–20 unique integers 100–599. Treat 2xx as healthy only for ANY_2XX; EXACT uses membership. Timeouts and network errors unhealthy regardless.
- `headers`: up to 20 entries, unique case-insensitive names, valid HTTP token names, nonempty values up to 2,048 characters each, total header payload <= 8 KiB. Reject dangerous/hop-by-hop headers (`Host`, `Connection`, `Transfer-Encoding`, `Content-Length`, `Proxy-*`, etc.) and newline injection; controlled `Content-Type` for JSON body. Sensitive values encrypted at rest and never logged or returned.
- `jsonBody`: optional, max 16 KiB serialized; POST/PUT/PATCH only. Reject non-JSON or oversized payloads.
- Whole request body <= 32 KiB (implementation default); validate on server even if UI validates first.
- Fixed interval 300 seconds and total timeout 10,000 ms; clients cannot override them.
- `userId`, `lifecycleStatus`, `nextCheckAt`, `configVersion`, counters, incidents, worker state and IDs are server-managed and rejected if supplied.

### Create workflow

1. Validate session, rate limit, body and SSRF-safe URL policy.
2. In a MongoDB transaction atomically increment user's `monitorCount` only when `< 10`, and create monitor; roll back together on failure. Paused and DELETING monitors count until finalized deletion.
3. Encrypt headers/body with authenticated encryption before persistence; keys managed through protected environment/secret manager.
4. Initialize ACTIVE, configVersion=1, no last-known health, no active incident, next scheduled slot according to worker's fixed 5-minute alignment policy.
5. Return `201 {data: safeMonitor}`. Creation does **not** claim the endpoint has already been checked.

**Example:** `409 {"error":{"code":"MONITOR_LIMIT_REACHED","message":"You can create up to 10 monitors."}}`.

## 6. Read and update monitors

### `GET /api/monitors`

Optional `search` (case-insensitive name/URL substring, capped length 100), `lifecycle=ACTIVE|PAUSED|DELETING`, `health=HEALTHY|UNHEALTHY|NO_DATA`, `freshness=CURRENT|DELAYED|NO_DATA|PAUSED`. Combine filters with AND. No paging, no secrets. Do not accept arbitrary MongoDB regex/operators; escape regex input or use bounded indexed search.

### `GET /api/monitors/:id`

Validate ID, owner scope; return safe monitor projection. If `DELETING`, return read-only status while cleanup progresses or 404 once complete.

### `PATCH /api/monitors/:id`

Partial updates to `name`, `url`, `method`, `expectedStatuses`, `headers`, `jsonBody` only. No lifecycle changes. Require `If-Match` config version (e.g., `If-Match: "3"`) to avoid lost updates; missing precondition → `428 PRECONDITION_REQUIRED`, stale version → `409 CONFIG_VERSION_CONFLICT`. Merge only provided fields; validate the resulting full configuration and method/body compatibility. `headers` supplied replaces entire header set; `headers: []` clears it; omitted retains encrypted existing value. `jsonBody: null` clears; omitted retains. Never return existing sensitive values. A meaningful request/expectation edit increments configVersion, fences older in-flight results from incident state, resets consecutive failures, and closes OPEN incident as CLOSED_UNVERIFIED/CONFIG_CHANGED. A name-only edit does **not** reset health/incident state or increment execution config version; update timestamp normally. For a meaningful edit, clear or clearly label prior last-known health as belonging to an older config; new config starts with no current health. Coordinate changes transactionally with worker scheduling state.

## 7. Pause, resume, delete (API-001)

### `POST /api/monitors/:id/pause`

No request body. Owner only. ACTIVE→PAUSED; cancel/skip future scheduled work, fence already claimed/in-flight work from mutating new incident/summary state, close any OPEN incident as CLOSED_UNVERIFIED (`closureReason=MONITOR_PAUSED`), reset failure counter and active incident pointer. Retain history. Already PAUSED → `200` unchanged (idempotent effect). DELETING → `409 STATE_CONFLICT`. Existing last-known health may remain historical but freshness shows PAUSED.

### `POST /api/monitors/:id/resume`

No request body. Owner only. PAUSED→ACTIVE, reset incident-processing state, establish next future 5-minute slot (no replay of pause-period slots), do not claim immediate health. Already ACTIVE → `200` unchanged. DELETING → `409 STATE_CONFLICT`. Avoid duplicate slot creation on retries.

### `DELETE /api/monitors/:id`

Owner only; permanent and irreversible; frontend requires explicit confirmation. Atomically mark DELETING/fence scheduling, safely remove associated scheduledChecks, checkResults, incidents and monitor, decrement `monitorCount` **exactly once** after successful finalization. Use recoverable cleanup with idempotent finalization; don't return `{deleted:true}` until deletion is complete. An in-flight worker must not recreate removed data or overwrite another lifecycle. On transient failure, return a safe retryable error and let cleanup resume; never falsely report complete deletion. Repeat DELETE after complete removal returns 404 (or choose 200 tombstone semantics only if implemented consistently).

## 8. Check history — `GET /api/monitors/:id/checks`

Query: `limit` (1–100, default 20), `cursor`, `health=HEALTHY|UNHEALTHY`, `outcome=HTTP_RESPONSE|TIMEOUT|NETWORK_ERROR`, `from`, `to` (ISO UTC; max 90-day range). Validate owner first. Return safe CheckResult projections, newest scheduledAt first, paginated. Include HTTP 500 as HTTP_RESPONSE/UNHEALTHY when unexpected. Do not invent status for timeout/network error. Historical results may be from older configVersion and should be labeled accordingly.

## 9. Metrics — `GET /api/monitors/:id/metrics`

Query: `window=24h|7d|30d|90d` (default 24h), optionally `to` for historical window end (implementation default). Use owner-scoped data within retained 90-day window. Return:

```json
{
  "data": {
    "window": "24h", "from": "...", "to": "...",
    "completedChecks": 280, "healthyChecks": 278,
    "uptimePercent": 99.29,
    "expectedSlots": 288, "executedSlots": 280,
    "coveragePercent": 97.22,
    "averageHttpResponseTimeMs": 148,
    "httpResponseSamples": 277,
    "lastKnownHealth": "HEALTHY",
    "lastCompletedAt": "..."
  }
}
```

- **Uptime** = healthy completed checks / all completed checks with health outcomes × 100. Exclude MISSED/INTERRUPTED slots; null when denominator 0. This is **check-based**, not time-based uptime.
- **Coverage** = executed scheduled slots / expected slots × 100. Count each scheduled slot once; late results attributed to original slot; missed/uncertain slots reduce coverage, not uptime. Exclude paused periods from expected slots. Null if no expected slots.
- **Average HTTP response time** includes all `HTTP_RESPONSE` outcomes (including unexpected 4xx/5xx), excludes TIMEOUT and NETWORK_ERROR; null when none.
- Round percentages to two decimals in API presentation, not before computing. Do not claim exact uptime when coverage is low.
- Charts can use a separate read-only `GET /api/monitors/:id/response-times?window=...` **only if** needed; default to returning a bounded `responseTimeSeries` array in metrics (e.g., <= 120 aggregated buckets) so the dashboard does not download raw thousands of checks. The exact bucket format is an implementation detail to settle when building charts.
- Monitoring freshness: compute from latest completed slot versus expected schedule and lifecycle; `DELAYED` indicates monitoring data is stale, **not** API failure. Define precise grace threshold in worker implementation and test it (suggest >2 intervals past due).

## 10. Incidents

### `GET /api/incidents`

Owned incidents across monitors. Query `status=OPEN|RESOLVED|CLOSED_UNVERIFIED`, `monitorId`, `from`, `to`, `limit`, `cursor`; default newest by confirmedAt. Include safe monitor name in presentation projection when available; do not leak foreign monitors.

### `GET /api/monitors/:id/incidents`

Same pagination/filtering scoped to one owned monitor. No mutation endpoints for incidents in MVP.

### `GET /api/incidents/:id`

Verify associated monitor ownership before returning incident. Expose first failure, confirmation, recovery or unverified closure, closure reason and relevant result references if still retained. If linked checkResult has expired, incident still renders using self-contained timestamps.

### Worker incident state rules

Process completed results in **scheduled order** with configVersion guard and idempotent cursor. Two consecutive unhealthy completed checks open one incident; one healthy completed check resolves it. Missed/uncertain slots do not count as failures or recovery. Repeated processing cannot create duplicate incidents. Unique partial index for one OPEN incident per monitor. On pause/config change close OPEN as CLOSED_UNVERIFIED with reason, never RESOLVED.

## 11. Worker scheduling/execution contract (internal, not HTTP)

- One persistent worker; at most **5 concurrent external HTTP requests globally**. This in-process limit is valid only while hosting guarantees one active worker, including deployments. Add distributed fencing/permits before scaling replicas.
- Each ACTIVE monitor is due every **5 minutes**; `monitors.nextCheckAt` supports due discovery. Create `scheduledChecks` unique `(monitorId,scheduledAt)`; states `PENDING`, `CLAIMED`, `RUNNING`, `COMPLETED`, `MISSED`, `INTERRUPTED`.
- Atomic claim/lease, claim owner and expiry; bounded scheduling window; missed slots skipped (no replay). Recover from worker crash without blindly retrying an external request that might have had side effects. Exactly-once **external execution** cannot be guaranteed; at-most-one persisted result per slot is enforced by unique indexes and guarded writes.
- Resolve/revalidate DNS for every execution; enforce HTTPS/public destination, disallow redirects and retries, cap headers/body and response size, cancel after **10-second total timeout**. Avoid sending credentials to unintended hosts. Use controlled User-Agent; redact request secrets and response bodies in logs.
- Persist a checkResult for each reliably completed attempt with outcome and safe error code. Ensure `scheduledChecks` completion/result persistence is coordinated transactionally or via a recoverable idempotent protocol. Worker only updates latest summary/incident state if configVersion and lifecycle still match and result is newer than processed cursor.
- Detailed scheduledChecks and checkResults retained **90 days**, with `expiresAt` TTL on terminal records plus application-level date filters. TTL is asynchronous. Incidents retained until monitor deletion; no lifetime aggregates in MVP.

## 12. Data model and indexes (summary)

- `users`: OAuth identity, `monitorCount` for atomic 10-monitor cap.
- `monitors`: `userId`, name, URL, method, encrypted headers/body, expectedStatuses, lifecycleStatus, configVersion, nextCheckAt, last-known summary, consecutiveFailures, activeIncidentId, lastProcessedCursor, timestamps.
- `scheduledChecks`: monitorId, scheduledAt, configVersion/snapshot reference, state, claim/lease metadata, started/completed timestamps, expiresAt; **unique `(monitorId,scheduledAt)`**, due/state indexes, TTL for terminal slots only.
- `checkResults`: monitorId, scheduledAt, configVersion, outcome, health, httpStatus?, responseTimeMs?, errorCode?, startedAt, completedAt, expiresAt; **unique `(monitorId,scheduledAt)`**, index `(monitorId,scheduledAt DESC)`, TTL.
- `incidents`: monitorId, status, firstFailureAt, confirmedAt, resolvedAt?, closedAt?, closureReason?, triggerResultId?, recoveryResultId?, timestamps; **partial unique OPEN incident per monitor**, `(monitorId,confirmedAt DESC)` index.
- Owner authorization via monitors.userId; don't assume checkResults has a trustworthy independent userId.

## 13. Frontend contract and UI states

- **Add Monitor:** client-side convenience validation plus authoritative server validation; show inline field errors; disable duplicate submission; creation initially has NO_DATA until first result.
- **Monitors table:** lifecycle ACTIVE/PAUSED/DELETING, last-known health, freshness are separate. Search/filter from safe API. Show max 10 count including paused/deleting.
- **Details tabs:** Overview metrics/chart, Check History, Incidents, Configuration (masked secret presence only). Empty states for no results and paused monitor. Uptime/coverage display N/A for null.
- **Mutations:** pause/resume optimistic updates only if safely reversible; on conflict refetch; confirm destructive deletion. Editing sensitive fields should offer replace/clear/keep semantics.
- **Session expiry:** on 401 show sign-in path and avoid retry loops. 429/503 show retry guidance. Never expose internal errors.

## 14. Security checklist

- [ ] Auth.js OAuth providers configured, secure cookie and origin/CSRF protections verified.
- [ ] Every protected endpoint validates session; all resource queries owner-scoped.
- [ ] No account auto-linking solely on email.
- [ ] Server-side schema validation for path, query, and body; reject unexpected fields.
- [ ] HTTPS-only SSRF protection at creation **and execution**, DNS rebinding protection, redirect disabled, no private/reserved targets.
- [ ] Request credentials encrypted at rest with authenticated encryption; secrets never returned/logged.
- [ ] Timeouts, size caps, concurrency limit and rate limits enforced.
- [ ] Configuration-version fencing prevents stale checks from mutating current state.
- [ ] Atomic monitor limit and recoverable idempotent deletion verified.
- [ ] Indexes, TTL retention and ownership isolation tested.

## 15. Minimum API and worker tests

| Area | Essential scenarios |
|---|---|
| Auth | no session 401; own resource allowed; foreign resource indistinguishable 404; expired session |
| Create | valid GET; invalid HTTPS; SSRF/local IP; DNS rebinding; invalid method/status/header/body; 10-monitor cap under concurrent creates |
| Edit | name-only edit; config edit resets incidents safely; stale If-Match conflict; secret retain/replace/clear; forbidden lifecycle field |
| Lifecycle | pause/resume idempotence; paused slots excluded; in-flight result fenced; open incident closes unverified |
| Delete | complete cascade; retry after crash; exact-once counter decrement; worker cannot recreate records |
| Checks | HTTP 200/500; expected exact status; timeout; DNS/network failure; no redirects/retries; response-size limits |
| Scheduling | every 5m; unique slots; missed vs completed; crash/lease recovery; single worker max 5 concurrency |
| Metrics | 0 denominator null; low coverage; missed slots excluded from uptime; 500 included in HTTP latency; 90d boundary |
| Incidents | two consecutive failures; one recovery; dedup; stale config result; pause/config change unverified closure |
| Pagination | cursor stable across new inserts; filter mismatch rejected; ownership; empty page; limit cap |

## 16. Recommended implementation sequence

1. Initialize Next.js TypeScript/Tailwind repository, environment validation, lint/test tooling and docs.
2. Set up MongoDB/Mongoose models, indexes, connection reuse and Auth.js Google/GitHub sign-in.
3. Implement shared API envelopes, validation schemas, auth/ownership helper and safe error handler.
4. Implement monitor list/create/detail/edit with atomic user monitor count and encrypted request config.
5. Implement pause/resume/delete with transactional lifecycle and recoverable cleanup.
6. Build worker scheduler/claims, SSRF-safe HTTP execution, outcome persistence, max-5 concurrency and recovery.
7. Implement incident processor, check history, metrics, incident reads and cursor pagination.
8. Implement UI using Figma designs when available; start with functional minimal pages if needed.
9. Add integration/security tests, deployment, README, API documentation and demo evidence.

### Before coding: small implementation choices to make directly in code

Choose compatible versions of Next.js, Auth.js and MongoDB adapter, decide session strategy; select schema validator (e.g., Zod), test runner, encryption key versioning, cursor-signing format, worker lease duration and precise schedule/freshness thresholds. These are **implementation details**, not reasons to delay the MVP. Record changes here as they are made.

---

**Definition of done for API phase:** Every implemented endpoint matches the above success/error envelopes, validation, owner isolation and lifecycle rules; worker is independent of browser; metrics distinguish uptime from coverage; tests cover security and failure paths. The document is a baseline contract, not evidence that any endpoint has already been implemented.
