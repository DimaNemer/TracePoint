# TracePoint — System Architecture

**Version:** 0.1 (working architecture)
**Date:** 2026-11-17
**Status:** Partially approved; further technical decisions pending
**Related:** PRD v1.0, user-stories.md, ADR-001, ADR-002

## 1. Purpose

TracePoint is a full-stack API monitoring application. Users register public HTTPS endpoints, and server-side checks run every five minutes independently of the browser. This document records approved architectural choices and explicitly separates them from decisions still pending.

## 2. Approved architecture

- **Web application:** Next.js + TypeScript handles OAuth authentication, monitor management, protected APIs, dashboards, and presentation of monitoring data.
- **Monitoring process:** A **separate, persistent Node.js worker** handles due checks, outbound HTTP execution, results, and incident processing. It runs even when no browser is open. **Accepted: ADR-001.**
- **Persistence and scheduling coordination:** MongoDB stores users, monitors, checks, incidents, and scheduling state. Scheduled work uses durable slot identities and atomic claims. **Accepted: ADR-002.**
- **External targets:** Only permitted public HTTPS endpoints may be checked. Requests must be protected against SSRF and DNS-based bypasses; redirects are disabled.
- **Code organization:** Web app and worker may share one repository and shared TypeScript modules, but run as distinct processes. Repository structure is not yet fixed.

### Conceptual flow

```text
User / Browser
      |
      v
Next.js web application (auth, CRUD, dashboard)
      |
      v
MongoDB <--------------------> Persistent Node.js monitoring worker
 (monitors, slots, results,       | schedule + claim due slots
  incidents, users)               | validate destination
                                 | execute HTTP with timeout
                                 | record outcome and incidents
                                 v
                         External public HTTPS APIs
```

The diagram represents shared persistent state, not a direct dependency of the worker on the browser or on a live web request.

## 3. Component responsibilities

| Component | Responsibilities | Explicit non-responsibilities |
|---|---|---|
| Next.js | OAuth via Google/GitHub, secure sessions, authorization, monitor CRUD, configuration validation, dashboard, history and incident views | Does not perform ongoing monitoring in a browser tab or depend on page traffic for scheduling |
| MongoDB | Durable monitor/configuration data, scheduled slot identities and claims, check outcomes, incidents, user ownership | Does not itself send HTTP checks |
| Node.js worker | Identify due slots, claim atomically, enforce timeout and concurrency, dispatch safe outbound requests, persist results, process incident transitions, reconcile interrupted work | Does not blindly replay uncertain outbound requests |
| External endpoints | Respond to permitted monitoring requests | Not controlled by TracePoint |

## 4. Scheduling and execution — approved principles

- Cadence: one scheduled opportunity every **five minutes** for each active monitor.
- A slot has a stable identity: **monitor ID + scheduled UTC time**, enforced uniquely in storage.
- The worker claims due work with an atomic conditional operation; only a successful claimant may proceed.
- Conceptual states: `PENDING`, `CLAIMED`, `RUNNING`, `COMPLETED`, `MISSED`, `INTERRUPTED`. These are **conceptual**, not yet a final database schema.
- Never replay missed historical checks. Never blindly retry an outbound request whose execution is uncertain, especially POST/PUT/PATCH/DELETE.
- A completed outcome is stored at most once per slot; incident transitions must be idempotent.
- A 10-second total request timeout applies. Record HTTP status separately from evaluated health. Default healthy statuses are 2xx, with configurable exact expected statuses.
- Two consecutive completed unhealthy checks open an incident; one completed healthy check resolves it. Missed or uncertain slots are not automatically unhealthy checks.
- Check-based uptime uses completed outcomes with health classifications; coverage compares executed to expected slots, excluding paused periods. Treatment of uncertain dispatch in coverage remains to be finalized.

## 5. Deployment implications

**Vercel can host the Next.js web application.** A conventional persistent Node.js worker cannot simply run forever inside Vercel's request-based/serverless execution model. The worker needs a host that supports long-running processes (for example, a suitable Railway or Render service), or the architecture would have to change to scheduled functions.

A plausible deployment is:

- **Vercel:** Next.js web app (candidate, not selected)
- **Railway or Render:** long-running Node.js worker (candidate, not selected)
- **MongoDB Atlas:** managed database (candidate, not selected)

These services can use the same Git repository. Each process needs its own deployment configuration and server-side environment variables. Costs, service availability, process restart behavior, network restrictions, and actual provider selection must be verified before implementation.

Worker hosting must provide automatic restart, logs, and a reliable approach to deployment overlap. **A single-worker deployment is not yet approved as the global concurrency strategy.** Atomic slot claims prevent two workers from claiming the *same* slot, but do not limit aggregate concurrency across workers.

## 6. Reliability and failure scenarios

| Scenario | Required behavior |
|---|---|
| Browser closes | Worker continues independently |
| Web app unavailable | Worker may continue if database and worker hosting remain healthy |
| Worker unavailable before a slot | Expired slot is missed; no historical HTTP replay |
| Worker crashes before request dispatch | Reconcile claim; reassign only if non-dispatch can be established safely |
| Worker crashes during/after dispatch | Preserve uncertain/interrupted state; no automatic resend |
| Worker crashes after persisting outcome | Deduplicate finalization and incident transitions |
| MongoDB unavailable | Do not claim success or dispatch work without reliable claim ownership; reconcile later |
| Multiple worker instances overlap | Unique slots and atomic claims prevent duplicate claims; separate global concurrency enforcement still required |
| Monitor paused | No new scheduled dispatch; preserve history; exclude paused periods from expected slots |
| Monitor deleted | Stop future dispatch, delete associated records, and prevent in-flight work from recreating them |
| External API hangs | Enforce 10-second total timeout and record unhealthy timeout outcome |

## 7. Security and data rules

- OAuth via Google and GitHub; do not auto-merge accounts solely because emails match.
- Server-side authorization for every user-owned monitor, check result, and incident.
- Permitted HTTP methods: GET, HEAD, POST, PUT, PATCH, DELETE; optional JSON bodies only for POST/PUT/PATCH.
- Only public HTTPS targets; protect against loopback/private IPs, DNS rebinding and unsafe redirects. Exact validation/pinning design is pending.
- Do not expose sensitive request headers, credentials, or bodies in logs/history.
- Maximum 10 monitors per user, including paused; creation cap must be enforced atomically.
- Maximum five globally concurrent outbound HTTP checks, **mechanism pending ADR-003**.
- Detailed check results retained for 90 days; incidents retained until monitor deletion. Cleanup strategy and indexes remain to be designed.

## 8. Open architecture decisions (not yet approved)

1. **ADR-003:** Global five-request concurrency control: strictly single active worker with local limiter vs MongoDB-backed global permits vs another distributed coordinator.
2. Precise scheduling eligibility window, claim lease/fencing protocol, and crash reconciliation.
3. Schema/indexes for monitors, slots, results, incidents, users, and deletion coordination.
4. Incident state behavior during pause/resume and out-of-order result handling.
5. Coverage treatment of uncertain HTTP dispatch and exact reporting windows.
6. Secure HTTP client implementation, destination validation, DNS pinning, allowed headers, and payload size limits.
7. Hosting provider selection, restart/overlap guarantees, secrets, logging, and deployment configuration.
8. OAuth account linking design, authorization implementation, and auth library configuration.

## 9. Architecture decision register

| ADR | Decision | Status |
|---|---|---|
| ADR-001 | Persistent Node.js background worker | **Accepted** |
| ADR-002 | MongoDB-backed scheduling and atomic work claims | **Accepted** |
| ADR-003 | Global concurrency mechanism | **Pending** |

## 10. Next step

Approve ADR-003 before committing to a concurrency implementation. Then design the detailed execution flow and MongoDB schema. Update this document as decisions are accepted; do not treat candidate services or conceptual states as final implementation choices.
