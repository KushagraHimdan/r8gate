# R8Gate — System Architecture Document

**Version:** 1.0
**Status:** Draft
**Companion documents:** PRD.md, SRS.md
**Owner:** [Your Name]

This document describes the practical, MVP-scoped architecture of R8Gate — deliberately avoiding patterns (microservices, multi-region, service mesh, etc.) that aren't justified at this project's scale. Every choice below is sized to what a solo developer can build, run, and explain in 2-4 weeks.

---

## 1. Recommended Tech Stack

| Layer | Choice | Why |
|---|---|---|
| **Frontend** | React (Vite) | Minimal live dashboard — allow/deny counts, per-client breakdown. Vite keeps setup light; no need for Next.js/SSR here since it's an internal-facing demo dashboard, not a public site. |
| **Backend** | Node.js + Express | Matches MERN, and Express's middleware model is a natural fit for a rate limiter — the core engine literally *is* Express middleware first. |
| **Rate-limit state store** | Redis | Atomic operations (`INCR`, Lua scripts), TTL-based expiry, sub-millisecond reads/writes — the correct tool for ephemeral, high-frequency counters. |
| **Config/persistent data store** | MongoDB | Client/API key records, tier definitions, route overrides — data that's written rarely and read often, a good fit for MongoDB's document model and the "M" in MERN. |
| **Containerization** | Docker + Docker Compose | Single-command local spin-up of Node app + Redis + MongoDB. |
| **Testing** | Jest (unit/integration), autocannon or k6 (load/concurrency testing) | Jest is standard for Node; autocannon is lightweight and sufficient for the concurrency correctness tests this project actually needs. |
| **Process/dev tooling** | dotenv, nodemon (dev only) | Standard, no need for anything heavier. |

**What's deliberately excluded:** Kubernetes, message brokers (Kafka/RabbitMQ), microservices split, API gateway frameworks (Kong/Envoy) as a base to build on top of, Redis Cluster/Sentinel, multi-region deployment. All of these solve problems R8Gate doesn't have yet at MVP scale — introducing them now would be over-engineering, and diminish rather than strengthen the "I understand what I built" interview story.

---

## 2. System Components

R8Gate is a **modular monolith** — one deployable Node.js application, internally organized into clearly separated modules, rather than split into microservices. This is the right call at this scale: it keeps deployment and debugging simple while the internal separation still demonstrates clean architecture.

```
┌─────────────────────────────────────────────────────────────┐
│                        R8Gate Process                        │
│                                                                │
│  ┌──────────────┐   ┌──────────────────┐   ┌──────────────┐ │
│  │   Express      │   │  Rate Limiting    │   │   Config /    │ │
│  │   HTTP Layer   │──▶│  Engine           │──▶│   Client      │ │
│  │  (routes/mw)   │   │  (algorithms)     │   │   Resolver    │ │
│  └──────────────┘   └──────────────────┘   └──────────────┘ │
│         │                     │                      │        │
│         │                     ▼                      ▼        │
│         │            ┌──────────────┐      ┌──────────────┐  │
│         │            │  Redis Client │      │ MongoDB Client│  │
│         │            └──────────────┘      └──────────────┘  │
│         ▼                                                     │
│  ┌──────────────┐                                             │
│  │  Proxy Layer  │  (gateway mode only)                       │
│  │  (http-proxy) │──────▶  Backend Target(s)                  │
│  └──────────────┘                                             │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────┐                                             │
│  │  Logger /     │──────▶  stdout / log file (JSON)           │
│  │  Metrics Feed │──────▶  Dashboard (via REST poll/WS)        │
│  └──────────────┘                                             │
└─────────────────────────────────────────────────────────────┘
         ▲
         │
   ┌──────────┐        ┌──────────────┐
   │  Clients  │        │  React        │
   │ (external)│        │  Dashboard    │
   └──────────┘        └──────────────┘
```

### Component responsibilities

- **Express HTTP Layer** — entry point for all requests; wires up middleware order (identify client → resolve policy → evaluate limit → forward/proxy/deny).
- **Rate Limiting Engine** — pure-logic module implementing fixed-window and token-bucket algorithms; takes a client ID + policy, talks to Redis, returns an allow/deny decision plus header values. This module has no HTTP knowledge — it's unit-testable in isolation, which matters for confidence in the concurrency-critical code.
- **Config / Client Resolver** — resolves an incoming request's client identity and applicable limit policy, reading from an in-memory cache populated from MongoDB at startup (see Section 6, Data Flow).
- **Proxy Layer** — only active in gateway mode; forwards allowed requests to the configured backend target using a lightweight HTTP proxy library (`http-proxy-middleware` or Node's built-in `http` module).
- **Logger / Metrics Feed** — structured JSON logging plus an in-memory rolling counter feeding the dashboard endpoint.

---

## 3. API Design

R8Gate exposes two categories of API: the **protected API surface** (what clients interact with, in middleware or gateway mode) and R8Gate's own **operational API** (health, metrics, dashboard feed).

### 3.1 How R8Gate is used — two modes

**Middleware mode** (embedded in an existing Express app):
```js
const express = require('express');
const { r8gate } = require('r8gate');

const app = express();

app.use(r8gate({
  redisUrl: process.env.REDIS_URL,
  mongoUrl: process.env.MONGO_URL,
  defaultPolicy: { algorithm: 'token-bucket', maxRequests: 100, windowSeconds: 60, refillRatePerSecond: 1.67 },
  routeOverrides: {
    '/login': { algorithm: 'fixed-window', maxRequests: 5, windowSeconds: 60 }
  }
}));

app.get('/api/data', (req, res) => res.json({ ok: true }));
app.listen(3000);
```
Any request that fails the rate-limit check is short-circuited by the middleware with a `429` before it ever reaches `/api/data`'s handler.

**Gateway mode** (standalone proxy in front of a separate backend):
```bash
# via config file (r8gate.config.json)
{
  "mode": "gateway",
  "backendTarget": "http://localhost:4000",
  "redisUrl": "redis://localhost:6379",
  "mongoUrl": "mongodb://localhost:27017/r8gate",
  "defaultPolicy": { "algorithm": "fixed-window", "maxRequests": 1000, "windowSeconds": 60 }
}
```
```bash
npx r8gate start --config r8gate.config.json
# R8Gate now listens on :8080 and proxies allowed traffic to http://localhost:4000
```

### 3.2 Client-facing behavior (both modes)

| Scenario | Response |
|---|---|
| Request allowed | Forwarded/proxied normally; response includes `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` |
| Request denied | `429 Too Many Requests` + `Retry-After` + rate-limit headers; body: `{ "error": "Too Many Requests", "retryAfter": 12 }` |
| Unknown API key (reject mode on) | `401 Unauthorized` |
| Redis unreachable (fail-closed) | `503 Service Unavailable` + `X-RateLimit-Status: degraded` |
| Backend target unreachable (gateway mode) | `502 Bad Gateway` |

### 3.3 R8Gate's own operational endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/r8gate/health` | GET | Liveness — process is up |
| `/r8gate/ready` | GET | Readiness — Redis + MongoDB connections OK |
| `/r8gate/metrics` | GET | Aggregate allow/deny counts (JSON), consumed by the dashboard |
| `/r8gate/metrics/stream` | WS (optional) | Live push feed for the dashboard, if implemented over polling |

These are namespaced under `/r8gate/` specifically so they can never collide with a proxied backend's own routes in gateway mode.

---

## 4. Data Flow

### 4.1 Request evaluation flow (the core loop)

```
1. Request arrives → Express HTTP Layer
2. Extract client identity (API key header → JWT claim → IP, in that precedence)
3. Config/Client Resolver: look up client's tier + route override (from in-memory cache)
4. Rate Limiting Engine: run configured algorithm
     a. Fixed window  → Redis INCR + EXPIRE on windowed key
     b. Token bucket  → Redis EVAL (Lua script): compute refill, check/deduct token, atomically
5. Decision: allow or deny
     - Allow → attach headers → forward to next middleware (mw mode) or Proxy Layer (gateway mode) → response returned
     - Deny  → attach headers + Retry-After → respond 429 immediately, short-circuit
6. Logger records outcome (async, non-blocking) → stdout/file + in-memory metrics counter
7. Dashboard polls/subscribes to /r8gate/metrics → renders live counts
```

### 4.2 Configuration load flow (startup + periodic refresh)

```
1. On startup: connect to MongoDB → load all client/tier/route-override records → build in-memory cache
2. If MongoDB unreachable at startup → fail fast, refuse to start (ERR-004 in SRS)
3. Periodically (e.g., every 30s) or on-demand: refresh in-memory cache from MongoDB
     - keeps hot-path request evaluation fast (no MongoDB round-trip per request)
     - if MongoDB becomes unreachable after startup, keep serving from last-known-good cache (ERR-005)
```

This separation matters: **Redis is on the hot path** (every request), **MongoDB is not** (only read at startup/refresh). This is a deliberate design decision worth stating explicitly in an interview — it's why the performance budget in the SRS (`10ms p95`) is realistic.

---

## 5. Storage Design

### 5.1 Redis (ephemeral, hot path)

| Key pattern | Type | TTL | Purpose |
|---|---|---|---|
| `rl:fw:{clientId}:{routeId}:{windowStart}` | String (counter) | = window duration | Fixed window count |
| `rl:tb:{clientId}:{routeId}` | Hash `{tokens, lastRefill}` | Long TTL (e.g., 24h, for cleanup of inactive clients) | Token bucket state |

### 5.2 MongoDB (persistent, cold path)

Collections: `clients`, `tiers`, `routeOverrides` — as specified in SRS Section 5.1. Read-heavy, write-rarely (config changes are infrequent operator actions, not per-request writes).

### 5.3 Why this split, explicitly

A common mistake is putting rate-limit counters in a general-purpose database — it's the wrong tool (too slow for atomic per-request increments at scale) and the wrong access pattern (ephemeral, high-churn data doesn't belong in durable storage). Redis for counters + MongoDB for config is the practical, correctly-scoped split for this project.

---

## 6. Authentication & Authorization (architecture-level summary)

See SRS Section 6 for full requirements; architecturally:

- **Client auth** is a lightweight API-key lookup against the in-memory config cache — not a full auth system. R8Gate assumes clients are already-issued keys, not building signup/login.
- **Operator auth** for MVP is deployment-level (filesystem/env access to config), not an in-app permission system — appropriate for a single-operator MVP; flagged in SRS as a v2 area (admin API + auth) if extended.
- **No session state** anywhere in R8Gate — every request is evaluated independently, which keeps it horizontally scalable by construction (see Section 9).

---

## 7. Security (architecture-level summary)

- API keys never appear in logs in plaintext (masked/truncated) — logging module enforces this centrally so it can't be forgotten in individual log calls.
- All client-supplied values (API key, IP, route) are validated/sanitized before being interpolated into Redis key strings, preventing key-injection issues.
- Gateway proxy layer strips hop-by-hop headers before forwarding responses.
- Default configuration examples in the repo bind to `localhost`, not `0.0.0.0` — public exposure is an explicit operator decision, documented in the README, not a silent default.
- `npm audit` run as part of the pre-milestone checklist (not a full CI/CD security pipeline for MVP — that would be over-engineering for a solo portfolio project, but the manual check is cheap and worth doing).

---

## 8. Deployment

**Local development:**
```
docker-compose up
```
Spins up: R8Gate app container, Redis container, MongoDB container, all networked together via Docker Compose's default bridge network. This satisfies the PRD's "running in under 5 minutes" success metric.

**Suggested folder structure for the repo:**

```
r8gate/
├── docker-compose.yml
├── README.md
├── docs/
│   ├── PRD.md
│   ├── SRS.md
│   └── architecture.md
├── packages/
│   └── r8gate-core/                # the importable middleware/engine
│       ├── src/
│       │   ├── algorithms/
│       │   │   ├── fixedWindow.js
│       │   │   ├── tokenBucket.js
│       │   │   └── index.js
│       │   ├── redisClient.js
│       │   ├── configResolver.js
│       │   ├── middleware.js        # app.use(r8gate(config))
│       │   └── index.js
│       ├── test/
│       │   ├── fixedWindow.test.js
│       │   ├── tokenBucket.test.js
│       │   └── concurrency.test.js  # the correctness-under-load tests
│       └── package.json
├── apps/
│   ├── gateway/                     # standalone gateway mode app
│   │   ├── src/
│   │   │   ├── server.js
│   │   │   ├── proxy.js
│   │   │   ├── routes/
│   │   │   │   ├── health.js
│   │   │   │   └── metrics.js
│   │   │   └── config/
│   │   │       └── r8gate.config.json
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── demo-backend/                 # a trivial Express API to demo gateway mode against
│   │   ├── src/server.js
│   │   └── package.json
│   └── dashboard/                    # React app
│       ├── src/
│       │   ├── App.jsx
│       │   ├── components/
│       │   │   ├── LiveCounts.jsx
│       │   │   └── ClientBreakdown.jsx
│       │   └── api/metricsClient.js
│       ├── Dockerfile
│       └── package.json
├── scripts/
│   ├── seed-clients.js               # seeds MongoDB with demo clients/tiers
│   └── load-test.js                  # autocannon/k6 concurrency test runner
└── .env.example
```

This structure mirrors the two real deliverables from the PRD — a **publishable middleware package** (`packages/r8gate-core`) and a **standalone gateway app** (`apps/gateway`) — built on the same core engine, exactly as scoped.

---

## 9. Scalability (kept practical, not hypothetical)

R8Gate is stateless at the application layer — no in-process session or counter state that would break under multiple instances — because all counter state lives in Redis. This means the realistic, honest scalability story is:

- **Horizontal scaling of the gateway itself:** run N instances of the R8Gate app behind a load balancer (e.g., nginx or a cloud LB); since Redis is the single shared source of truth for counters, correctness holds across instances (this is exactly what SRS `[PERF-002]` validates).
- **Redis as the scaling ceiling:** a single Redis instance comfortably handles the throughput this project targets (500 req/s, per PRD). If this genuinely needed to scale further, the next real step would be Redis Cluster — explicitly noted as **out of scope**, not built, but it's the correct answer if asked "how would you scale this further" in an interview.
- **MongoDB is not on the scaling-critical path** — it's read at startup/refresh only, so it doesn't need special scaling treatment for this project's scope.

This is deliberately not over-built. The honest, defensible story is: "stateless app layer + Redis as shared state = horizontally scalable by construction, validated with a multi-instance load test," without pretending to have solved problems (geo-distribution, Redis Cluster failover, auto-scaling policies) that a 2-4 week solo project shouldn't attempt.

---

## 10. Monitoring

MVP-scoped, intentionally lightweight:

- **Structured JSON logs** (stdout) — every request evaluation logged with correlation ID, client, outcome (SRS `[FUNC-020]`/`[FUNC-021]`).
- **`/r8gate/health` and `/r8gate/ready`** — the two standard liveness/readiness endpoints, enough for any container orchestrator (even just Docker's own healthcheck) to know the app is alive and has working dependencies.
- **`/r8gate/metrics`** — in-memory aggregate counters (total/allowed/denied per client), exposed as JSON, consumed by the React dashboard. Not a full Prometheus/Grafana setup — that would be reasonable for a "v2 observability" extension, but isn't necessary to prove the core engineering claims of this project.

**What's excluded:** distributed tracing, alerting/paging, log aggregation infrastructure (ELK/Datadog). These are real production concerns, but adding them here would be building infrastructure-around-infrastructure for a portfolio project — better to note them explicitly as "what I'd add next" than to half-build them.

---

## 11. Summary: What Makes This Architecture Defensible

If asked "why did you build it this way" in an interview, the throughline is:
1. **Modular monolith, not microservices** — right-sized for the actual complexity and team size (one person).
2. **Redis for hot-path ephemeral state, MongoDB for cold-path config** — each store used for what it's actually good at.
3. **Stateless app layer** — scalability comes from the data layer being the single source of truth, not from complex orchestration.
4. **Explicitly documented exclusions** (Redis Cluster, service mesh, distributed tracing) — showing awareness of what a larger system would need, without building it prematurely.

This is the difference between "I added Redis because it's trendy" and "I chose Redis because atomic operations solve a specific correctness problem, and here's the load test that proves it" — the latter is what this whole document is scaffolding toward.