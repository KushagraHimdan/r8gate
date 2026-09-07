# R8Gate — Product Requirements Document

**Version:** 1.0
**Status:** Draft — MVP Scoping
**Owner:** [Your Name]

---

## 1. Overview

R8Gate is a distributed rate limiting and API gateway service. It sits in front of backend APIs and controls how many requests any given client can make within a defined time window, protecting downstream services from traffic spikes, abuse, and cascading failures.

R8Gate is built on the MERN stack, with Redis as the distributed state store for rate-limit counters. It ships first as an Express middleware library, then extends into a standalone gateway service that proxies requests to one or more backend targets.

---

## 2. Problem Statement

Backend services have finite capacity — database connections, CPU, memory, and often a hard budget ceiling on metered third-party calls. Without a controlling layer in front of them, a few common failure modes recur across nearly every production system:

- A single client (buggy retry loop, scraper, or bad actor) consumes disproportionate capacity, degrading service for every other client — the "noisy neighbor" problem.
- Traffic spikes (legitimate or malicious) overwhelm a service faster than it can scale, causing outages.
- Brute-force attacks (login endpoints, OTP verification) go unthrottled.
- Metered downstream costs (SMS, email, LLM APIs) run uncontrolled during a bug or attack, creating direct financial exposure.

Most teams solve this either with framework-level middleware (limited to a single instance, breaks under horizontal scaling) or by paying for a managed gateway (Kong, AWS API Gateway) without understanding what's actually happening underneath. There is a gap for a lightweight, self-hosted, horizontally-scalable rate limiter that teams can run, understand, and extend themselves.

---

## 3. Target Users

| User | Need |
|---|---|
| **Backend/platform engineers** at a startup or small team | A self-hosted rate limiter they can drop in front of internal or public APIs without paying for a managed gateway |
| **API providers** exposing tiered access (free/pro/enterprise) | Per-client configurable limits tied to a plan or API key |
| **Engineers building on Node/Express** | Middleware that integrates natively into an existing Express app, not a heavyweight external service |
| **(Secondary) You, the builder** | A resume-grade artifact demonstrating distributed systems fundamentals: concurrency correctness, atomicity, algorithm trade-offs |

---

## 4. Goals

1. Prevent any single client from exceeding a configured request rate against a protected API.
2. Provide correct behavior under concurrent, multi-instance load — no client should be able to exceed their limit by hitting different gateway instances simultaneously.
3. Support multiple rate-limiting algorithms with clearly documented trade-offs.
4. Be usable in two forms: an importable Express middleware, and a standalone proxying gateway.
5. Be simple enough to run locally via Docker Compose in under five minutes.

### Non-Goals (for MVP)
- Full API management platform (no billing, no developer portal, no analytics UI beyond basics).
- Support for frameworks other than Express/Node.
- Multi-region/geo-distributed rate limiting.
- Machine-learning-based anomaly detection (rules and algorithms only).

---

## 5. Core Features

### 5.1 Rate Limiting Engine
- Configurable per-client (API key / IP / user ID) request limits.
- Multiple algorithm implementations: fixed window, sliding window counter, token bucket.
- Redis-backed, atomic operations — safe under concurrent access across multiple gateway instances.
- Returns standard `429 Too Many Requests` with `Retry-After` and rate-limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`).

### 5.2 Express Middleware
- Drop-in middleware (`app.use(r8gate(config))`) for any existing Express app.
- Per-route and per-client override support (e.g., stricter limits on `/login`, looser on `/health`).

### 5.3 Standalone Gateway Mode
- Express app that receives all incoming traffic and proxies to one or more configured backend targets after applying rate-limit checks.
- Basic routing config (path → target service mapping).

### 5.4 Configuration
- Config file (JSON/YAML) or environment variables defining: algorithm choice, limits per tier/client, protected routes, backend targets (gateway mode).

### 5.5 Observability
- Structured request logging (client ID, route, allowed/denied, timestamp, correlation ID).
- Minimal `/health` and `/metrics`-style endpoint exposing current limiter stats.

### 5.6 Admin/Demo Dashboard (MVP-lite)
- Simple frontend (the "R" in MERN) showing live request/denial counts per client — primarily a demo artifact, not a full admin console.

---

## 6. MVP Scope

The MVP is intentionally narrow: **prove the core rate-limiting engine works correctly under concurrency, and make it usable as both middleware and a minimal gateway.**

**In scope for MVP:**
- Fixed window and token bucket algorithms (sliding window counter as a stretch goal within MVP if time allows).
- Redis-backed counters with atomic Lua-scripted operations for token bucket.
- Express middleware package, usable standalone.
- Minimal gateway mode: single backend target, path-based routing not required (proxy everything to one target).
- Per-client (API key based) limit configuration via a config file.
- 429 responses with correct headers.
- Basic structured logging to console/file.
- Docker Compose setup (Node app + Redis) for local run.
- A minimal React dashboard showing live allow/deny counts (polling or simple WebSocket push).

**Explicitly deferred (see Out of Scope):** multi-target routing, per-route dashboards, auth/RBAC on the admin API, sliding window log, leaky bucket, distributed config management, alerting.

---

## 7. User Stories

1. **As a backend engineer**, I want to wrap my Express API with R8Gate middleware so that no client can exceed 100 requests/minute without writing my own limiter.
2. **As an API provider**, I want to configure different limits per API key (e.g., free tier vs. pro tier) so that I can enforce my pricing plan's usage limits.
3. **As a client of a protected API**, I want to receive a clear `429` response with a `Retry-After` header so that I know exactly when I can retry.
4. **As an engineer running multiple instances of my API behind a load balancer**, I want rate limits enforced correctly across all instances so that clients can't bypass limits by hitting different servers.
5. **As a developer evaluating R8Gate**, I want to run it locally via Docker Compose in under five minutes so that I can try it before integrating it.
6. **As a team lead**, I want a simple dashboard showing which clients are being rate-limited so that I can spot abuse or misbehaving integrations.
7. **As a backend engineer choosing an algorithm**, I want documentation explaining the trade-offs between fixed window and token bucket so that I can pick the right one for my use case.

---

## 8. Success Metrics

Since this is a portfolio project rather than a live product, "success" is measured by correctness, completeness, and demonstrability rather than production usage metrics:

| Metric | Target |
|---|---|
| Correctness under concurrency | Zero over-limit requests allowed through in a concurrent load test (e.g., 50 parallel clients hammering a 100 req/min limit) |
| Latency overhead | Rate-limit check adds < 10ms p95 latency per request |
| Multi-instance correctness | Running 2+ gateway instances behind a load balancer produces the same aggregate limit enforcement as a single instance |
| Setup time | A new developer can clone the repo and have it running locally via `docker-compose up` in under 5 minutes |
| Test coverage | Core rate-limiting logic (algorithms) covered by unit tests at >85% |
| Demonstrability | A scripted load test + dashboard clearly shows requests being allowed/denied in real time — usable as an interview demo |

---

## 9. Assumptions

- The builder has working knowledge of Node.js/Express and is learning Redis as part of this project.
- Single-region deployment is sufficient; no cross-region latency/consistency concerns for MVP.
- Clients identify themselves via a provided API key (assumed already authenticated) rather than R8Gate handling its own auth/login flow.
- Redis is available as a managed or self-hosted single instance for MVP (Redis Cluster/HA is out of scope).
- Backend "targets" proxied in gateway mode are simple HTTP services; no need to support gRPC, WebSockets, or GraphQL passthrough for MVP.

---

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Race conditions in counter increment logic | Rate limits silently bypassed under load — undermines the entire premise of the project | Use Redis atomic operations (`INCR`, Lua scripts) exclusively; validate with concurrency-focused load tests before calling any algorithm "done" |
| Redis becomes a single point of failure | Gateway fails closed or open depending on error handling — either blocks all traffic or allows unlimited traffic if Redis is down | Explicitly decide and document fail-open vs fail-closed behavior; add basic Redis connection error handling |
| Scope creep (adding leaky bucket, multi-target routing, auth, alerting) | MVP never finishes; timeline slips past 2-4 weeks | Hold firm to the "Out of Scope" list below; treat deferred items as v2 backlog, not MVP requirements |
| Clock skew / timing edge cases in window-based algorithms | Off-by-a-few-requests inaccuracies near window boundaries | Document known limitations honestly rather than over-engineering a fix; token bucket largely sidesteps this |
| Underestimating Redis learning curve | Time spent learning Redis (new to the builder) eats into build time | Timebox a short dedicated Redis-learning phase before implementation begins; keep MVP algorithms to those with well-documented Redis patterns |

---

## 11. Out of Scope (MVP)

- Sliding window log and leaky bucket algorithms (documented as future work; token bucket + fixed window cover the core learning goals)
- Multi-target / path-based routing in gateway mode (single backend target only)
- Authentication/RBAC on the admin dashboard or config API
- Distributed configuration management (config is file/env based, not dynamically updatable via API)
- Alerting/notification system for abuse patterns
- Horizontal auto-scaling of the gateway itself
- Support for non-Express frameworks or non-Node languages
- Geo-distributed / multi-region rate limiting
- Billing or plan-management UI (limits are configured directly, not derived from a billing system)
- Circuit breaker / bulkhead patterns for backend target failures (may be a natural v2 extension, not MVP)

---

## 12. Acceptance Criteria

The MVP is considered complete when all of the following are true:

1. `npm install`-able middleware package exists and can be added to any Express app with a single `app.use()` call.
2. At minimum, fixed window and token bucket algorithms are implemented, unit tested, and selectable via config.
3. Rate-limit state is stored in Redis using atomic operations; a documented concurrency test demonstrates no over-limit requests are allowed when multiple gateway instances run against the same Redis instance.
4. Denied requests return `429` with `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers.
5. A standalone gateway mode exists that proxies requests to at least one configured backend target, applying the same rate-limiting engine.
6. Per-client (API key) limit configuration is supported and demonstrably enforced differently for at least two distinct clients/tiers.
7. The entire system (Node app + Redis) runs via a single `docker-compose up` command.
8. A minimal React dashboard displays live allow/deny request counts per client.
9. A load test script and results (showing correctness under concurrent load) are included in the repo.
10. README includes an architecture diagram, setup instructions, and a written explanation of the algorithm trade-offs implemented.

---

## 13. Tech Stack Summary

- **Frontend:** React (minimal dashboard)
- **Backend:** Node.js + Express (middleware + gateway)
- **Data store:** Redis (rate-limit state)
- **Database (if needed for config/client management):** MongoDB (client/API key + tier configuration storage — the "M" in MERN)
- **Containerization:** Docker + Docker Compose
- **Testing:** Jest (unit tests), a load-testing tool (e.g., autocannon or k6) for concurrency validation