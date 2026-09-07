# R8Gate — Software Requirements Specification (SRS)

**Version:** 1.0
**Status:** Draft
**Companion document to:** PRD.md
**Owner:** [Your Name]

Requirement IDs are used throughout so each requirement can be traced to a test case later. Format: `[CATEGORY-###]`.

---

## 1. Introduction

### 1.1 Purpose
This SRS defines the functional and non-functional requirements for R8Gate, a distributed rate limiting and API gateway service built on the MERN stack with Redis as the rate-limit state store. It expands on the PRD with implementation-level, testable requirements.

### 1.2 Scope
R8Gate enforces per-client request limits on protected APIs, operating either as Express middleware embedded in an existing app, or as a standalone gateway proxying to backend services. This document covers both modes as scoped in the PRD's MVP.

### 1.3 Definitions
| Term | Meaning |
|---|---|
| **Client** | An entity making requests to a protected API, identified by an API key (primary) or IP address (fallback) |
| **Window** | A configured time interval (e.g., 60 seconds) over which a request limit applies |
| **Limit policy** | The combination of algorithm + max requests + window duration assigned to a client or tier |
| **Tier** | A named group of clients sharing the same limit policy (e.g., "free", "pro") |
| **Protected route** | An API endpoint that has rate limiting applied |
| **Gateway mode** | R8Gate running as a standalone reverse proxy in front of a backend target |
| **Middleware mode** | R8Gate embedded directly into an existing Express app via `app.use()` |
| **Fail-open** | Behavior where, if Redis is unreachable, requests are allowed through by default |
| **Fail-closed** | Behavior where, if Redis is unreachable, requests are rejected by default |

---

## 2. Overall Description

R8Gate consists of three logical components:
1. **Rate Limiting Engine** — algorithm implementations + Redis interaction layer.
2. **Express Integration Layer** — middleware (embeddable) and gateway (standalone proxy) modes, both built on the same engine.
3. **Admin/Config Layer** — client/tier/limit configuration (MongoDB-backed) and a minimal React dashboard for observability.

---

## 3. User Roles & Permissions

R8Gate has two categories of "users": **API clients** (the entities being rate limited) and **operators** (the people running/configuring R8Gate). There is no end-user-facing login in the MVP.

| Role | Description | Permissions |
|---|---|---|
| **API Client** | Any external caller sending requests to a protected route, identified via API key or IP | Can make requests; cannot view or modify configuration; receives only their own rate-limit headers/responses |
| **Operator (Admin)** | The engineer running/deploying R8Gate | Full access to configuration (limits, tiers, routes, targets) via config file/env for MVP; full read access to the dashboard and logs |

**[ROLE-001]** The system shall distinguish API clients from operators; API clients shall never have access to configuration data belonging to other clients.

**[ROLE-002]** For MVP, operator configuration access is managed via server-side config files/environment variables, not an authenticated web UI — see Section 9 (Out of Scope carryover from PRD) for deferred admin-API auth.

**[ROLE-003]** The dashboard (read-only, MVP) shall not require login for local/demo use but shall not be exposed on a public network without operator-added auth (documented as a deployment responsibility, not enforced by R8Gate itself in MVP).

---

## 4. Functional Requirements

### 4.1 Client Identification

**[FUNC-001]** The system shall identify a client by, in order of precedence: (1) API key provided in the `X-API-Key` header, (2) authenticated user ID if present in a decoded JWT (if the host app provides one), (3) client IP address as a fallback.

**[FUNC-002]** If no API key or user ID is present, the system shall rate-limit by IP address using the limit policy assigned to the "default"/unauthenticated tier.

**[FUNC-003]** The system shall treat each unique client identifier as an independent rate-limit bucket — one client's usage shall never affect another client's remaining quota.

### 4.2 Limit Policy Configuration

**[FUNC-004]** The system shall support defining a limit policy consisting of: algorithm type (`fixed-window` | `token-bucket`), max requests, window duration (seconds), and (for token bucket) refill rate.

**[FUNC-005]** The system shall support assigning limit policies at the **tier** level (e.g., "free" = 100 req/min) and support mapping individual API keys to a tier.

**[FUNC-006]** The system shall support a **per-route override**, allowing a specific route (e.g., `/login`) to enforce a stricter limit policy than the client's default tier policy.

**[FUNC-007]** If no explicit policy is configured for a client or route, the system shall apply a documented default policy rather than failing or allowing unlimited requests.

### 4.3 Rate Limiting Algorithms

**[FUNC-008]** The system shall implement the **fixed window** algorithm: count requests per client within a discrete time window (e.g., per calendar minute); reset the counter at each window boundary.

**[FUNC-009]** The system shall implement the **token bucket** algorithm: each client has a bucket with a maximum token capacity; tokens refill at a configured rate; each request consumes one token; requests are denied when the bucket is empty.

**[FUNC-010]** All counter read-modify-write operations (increment, token deduction/refill) shall be executed as atomic Redis operations (native atomic commands or Lua scripts) such that concurrent requests for the same client cannot produce an incorrect count.

**[FUNC-011]** The algorithm used per client/route shall be configurable and shall not require a code change to switch between fixed-window and token-bucket for a given policy.

### 4.4 Request Evaluation Flow

**[FUNC-012]** On each incoming request to a protected route, the system shall: (1) resolve client identity, (2) resolve applicable limit policy, (3) evaluate the policy against current Redis state, (4) allow or deny the request, (5) attach rate-limit headers to the response, (6) log the outcome.

**[FUNC-013]** If the request is allowed, the system shall forward it to the next middleware (middleware mode) or proxy it to the configured backend target (gateway mode).

**[FUNC-014]** If the request is denied, the system shall short-circuit the request and respond directly with `429 Too Many Requests` without forwarding/proxying it.

### 4.5 Response Headers

**[FUNC-015]** Every response to a protected route (allowed or denied) shall include: `X-RateLimit-Limit` (max requests for the current window/bucket), `X-RateLimit-Remaining` (requests/tokens left), `X-RateLimit-Reset` (Unix timestamp when the window resets or bucket next refills).

**[FUNC-016]** A denied (`429`) response shall additionally include a `Retry-After` header expressed in seconds until the client may retry successfully.

### 4.6 Gateway Mode (Proxying)

**[FUNC-017]** In gateway mode, the system shall accept incoming HTTP requests and, if allowed by rate-limit evaluation, forward them to a single configured backend target (MVP scope — see PRD Out of Scope for multi-target routing) preserving method, headers (except hop-by-hop headers), and body.

**[FUNC-018]** The system shall return the backend target's response (status, headers, body) unmodified to the original client, except for the addition of R8Gate's own rate-limit headers.

**[FUNC-019]** If the backend target is unreachable or times out, the system shall return `502 Bad Gateway` or `504 Gateway Timeout` as appropriate, and shall log the failure.

### 4.7 Logging

**[FUNC-020]** The system shall log, for every evaluated request: timestamp, client identifier, route, algorithm used, allow/deny outcome, and a correlation ID unique to that request.

**[FUNC-021]** Logs shall be structured (JSON) to support later ingestion by log-analysis tools, even though log aggregation itself is out of scope for MVP.

### 4.8 Dashboard

**[FUNC-022]** The system shall expose a read endpoint (or WebSocket feed) providing aggregate allow/deny counts per client over a recent time window.

**[FUNC-023]** The React dashboard shall display, at minimum: total requests, allowed count, denied count, and a per-client breakdown, updating without requiring a manual page reload.

---

## 5. Data Requirements

### 5.1 Data Entities

**Client / API Key (MongoDB)**
| Field | Type | Notes |
|---|---|---|
| `apiKey` | String | Unique, indexed |
| `clientName` | String | Human-readable label |
| `tier` | String | References a tier policy |
| `createdAt` | Date | |
| `active` | Boolean | Inactive keys are rejected at auth, not rate-limited |

**Tier / Limit Policy (MongoDB)**
| Field | Type | Notes |
|---|---|---|
| `tierName` | String | Unique |
| `algorithm` | Enum: `fixed-window`, `token-bucket` | |
| `maxRequests` | Number | |
| `windowSeconds` | Number | |
| `refillRatePerSecond` | Number | Token bucket only; null for fixed window |

**Route Override (MongoDB or config file)**
| Field | Type | Notes |
|---|---|---|
| `routePattern` | String | e.g., `/login` |
| `overridePolicy` | Reference or inline policy object | |

**Rate Limit Counter State (Redis — ephemeral, not MongoDB)**
- Fixed window: key pattern `ratelimit:{clientId}:{routeId}:{windowStart}` → integer count, TTL = window duration.
- Token bucket: key pattern `ratelimit:bucket:{clientId}:{routeId}` → hash of `{tokens, lastRefillTimestamp}`, no forced TTL (persists between requests) or a long TTL for cleanup of inactive clients.

**[DATA-001]** All rate-limit counter state shall live exclusively in Redis; MongoDB shall never store per-request counters, only configuration entities (clients, tiers, route overrides).

**[DATA-002]** Redis keys shall include a TTL sufficient to auto-expire stale/inactive client counters, preventing unbounded key growth.

**[DATA-003]** API keys shall be stored in MongoDB in hashed form (not plaintext) if the project's security posture requires it for the demo; at minimum this shall be documented as a known simplification if plaintext storage is used for MVP speed.

### 5.2 Validation Rules

**[VALID-001]** `apiKey` shall be a non-empty string, minimum 16 characters, unique across all client records.

**[VALID-002]** `maxRequests` shall be a positive integer greater than 0.

**[VALID-003]** `windowSeconds` shall be a positive integer greater than 0.

**[VALID-004]** `algorithm` shall be restricted to the enumerated supported values; any other value shall be rejected at config load time with a clear error, not silently defaulted.

**[VALID-005]** `refillRatePerSecond` shall be required and positive when `algorithm` is `token-bucket`, and shall be ignored (with a warning logged) if provided for `fixed-window`.

**[VALID-006]** Route patterns shall be validated as syntactically correct Express route strings at config load time; invalid patterns shall prevent server startup with a descriptive error rather than failing silently at request time.

---

## 6. Authentication & Authorization

**[AUTH-001]** R8Gate shall not implement its own user login system in MVP; it shall trust API keys as pre-issued client credentials, provisioned by the operator (via config or a seed script), not via self-service signup.

**[AUTH-002]** An unrecognized API key (not present in the configured client set) shall be treated as an unauthenticated client, subject to the default/lowest tier policy — not rejected outright, unless the operator explicitly configures "unknown keys rejected" mode.

**[AUTH-003]** If "unknown keys rejected" mode is enabled, an unrecognized API key shall receive `401 Unauthorized`, distinct from a `429` rate-limit denial, so clients can distinguish "you're not allowed at all" from "you're allowed but over quota."

**[AUTH-004]** The admin/config layer (in any future version exposing a write API) shall require a separate operator credential distinct from client API keys; for MVP, since config is file/env based, this requirement is satisfied by restricting filesystem/deployment access rather than an in-app auth check.

**[AUTH-005]** The dashboard read endpoint shall not expose any individual client's API key value, only client names/IDs and aggregate counts.

---

## 7. Business Rules

**[BIZ-001]** A client's remaining quota shall never go negative; once zero, further requests are denied until the window resets or tokens refill.

**[BIZ-002]** Denied requests shall not consume additional quota — only allowed/forwarded requests count against the limit (a denied request is not itself "charged").

**[BIZ-003]** Changing a client's tier shall apply to future requests only; it shall not retroactively reset or preserve counters from the previous policy (switching policies resets the client's counter state, documented as expected behavior, not a bug).

**[BIZ-004]** Route-level overrides shall take precedence over tier-level defaults when both apply to the same request.

**[BIZ-005]** In gateway mode, rate-limit evaluation shall always occur before proxying; a request shall never reach the backend target without having been evaluated.

---

## 8. Error Handling

**[ERR-001]** If Redis is unreachable at request time, the system shall apply a configured fail-open or fail-closed policy (operator-configurable; fail-closed is the documented safe default) and shall log the Redis failure distinctly from a normal rate-limit denial.

**[ERR-002]** If Redis is unreachable, the response shall include a distinct status/header (e.g., `X-RateLimit-Status: degraded`) so clients and operators can tell a fail-open pass-through apart from normal operation.

**[ERR-003]** If the backend target (gateway mode) returns a malformed or no response, the system shall return `502 Bad Gateway` and shall not crash or hang the gateway process.

**[ERR-004]** If MongoDB is unreachable at startup, the system shall fail to start with a clear error message rather than starting with no configuration loaded.

**[ERR-005]** If MongoDB becomes unreachable after startup (config already cached in memory), the system shall continue operating on last-known-good configuration and log a warning, rather than failing all requests.

**[ERR-006]** All unhandled exceptions in request processing shall be caught by a global error handler, logged with a correlation ID, and shall return `500 Internal Server Error` without leaking stack traces to the client response body.

**[ERR-007]** Configuration validation errors (Section 5.2) shall prevent server startup and print a human-readable description of exactly which field failed validation.

---

## 9. Edge Cases

**[EDGE-001]** Two concurrent requests from the same client arriving at the exact same millisecond shall both be evaluated correctly — neither shall be allowed to "slip through" due to a race condition (validated via concurrency load test, see PRD Success Metrics).

**[EDGE-002]** A request arriving exactly at a fixed-window boundary shall be counted against exactly one window, not double-counted or dropped.

**[EDGE-003]** A client with zero tokens whose bucket is refilling shall become eligible again the instant sufficient tokens accumulate, not only at the top of a fixed interval.

**[EDGE-004]** A client making zero requests for a long period shall not accumulate unbounded tokens beyond the bucket's configured maximum capacity.

**[EDGE-005]** A newly-added client with no prior Redis key shall be initialized with a full quota/bucket on their first request, not treated as already-exhausted.

**[EDGE-006]** If a client's tier is deleted from configuration while active, the system shall fall back to the default tier rather than throwing an unhandled error on the next request.

**[EDGE-007]** Extremely high-frequency requests (e.g., thousands per second from one client) shall not degrade Redis or the gateway's ability to serve other clients — validated under load test as a non-functional requirement (Section 11).

**[EDGE-008]** Malformed or missing `X-API-Key` header shall not crash the request pipeline; it shall be treated per [FUNC-002] as an IP-based default-tier client.

**[EDGE-009]** Clock changes on the server (e.g., NTP correction) shall not permanently break window calculations; fixed-window logic shall use Redis TTL-based expiry (server-relative) rather than depending solely on wall-clock comparisons in application code.

---

## 10. Security Requirements

**[SEC-001]** API keys shall never be logged in plaintext in application logs; logs shall reference clients by a non-reversible identifier or truncated/masked key.

**[SEC-002]** The system shall not be vulnerable to header injection via client-supplied identifiers (API key, IP) used in Redis key construction — all client-supplied values shall be sanitized/validated before being interpolated into Redis key strings.

**[SEC-003]** The gateway proxy shall strip hop-by-hop headers (`Connection`, `Keep-Alive`, `Transfer-Encoding`, etc.) and shall not forward the operator's internal configuration or Redis connection details in any response to a client.

**[SEC-004]** The system shall apply a stricter default rate limit to unauthenticated/unknown clients than to any recognized tier, to reduce the effectiveness of brute-force or scraping attempts against unauthenticated endpoints.

**[SEC-005]** The dashboard and any admin-facing endpoint shall not be enabled by default on a public-facing network interface without the operator explicitly configuring exposure (documented in README, not silently defaulted to `0.0.0.0` in production config examples).

**[SEC-006]** Dependencies (npm packages) shall be kept free of known critical vulnerabilities as reported by `npm audit`, checked before each milestone/release.

---

## 11. Performance Requirements

**[PERF-001]** The rate-limit evaluation step (Redis round-trip + decision) shall add no more than 10ms at p95 latency per request, measured under a load test of at least 100 concurrent clients.

**[PERF-002]** The system shall correctly enforce aggregate limits when running as 2+ gateway instances against a shared Redis instance, with no more than 0% over-limit requests allowed through in a sustained concurrency test (see PRD Success Metrics).

**[PERF-003]** The system shall handle at least 500 requests/second in gateway mode on modest local hardware (development machine) without error responses attributable to the gateway itself (as opposed to the backend target's own capacity).

**[PERF-004]** Redis key expiry (TTL) shall ensure inactive client counters are automatically cleaned up, preventing unbounded memory growth in Redis over time.

---

## 12. Acceptance Criteria

Each functional area is considered complete when its requirements are demonstrably testable and passing:

1. All `[FUNC-###]` requirements have at least one corresponding automated test (unit or integration).
2. Concurrency correctness (`[EDGE-001]`, `[PERF-002]`) is validated by a documented load test with results included in the repo, showing zero over-limit requests allowed.
3. All response header requirements (`[FUNC-015]`, `[FUNC-016]`) are verifiable via integration tests inspecting actual HTTP response headers.
4. Fail-open/fail-closed behavior (`[ERR-001]`, `[ERR-002]`) is demonstrable by manually stopping Redis during a running test and observing the configured behavior.
5. Validation rules (`[VALID-001]` through `[VALID-006]`) each have a corresponding negative test case confirming invalid config is rejected with a clear error.
6. Security requirements `[SEC-001]` through `[SEC-004]` are manually verified via a documented security review checklist before considering MVP complete.
7. Performance requirements `[PERF-001]` through `[PERF-003]` are validated via a load-testing script (e.g., autocannon/k6) with results documented in the README.
8. All edge cases in Section 9 have a corresponding test case, even if some are manually verified rather than automated where automation isn't practical (documented which is which).

---

## 13. Traceability Note

This SRS should be read alongside PRD.md — the PRD's "Out of Scope" section (multi-target routing, admin API auth, sliding window log, leaky bucket, alerting) applies equally here; requirements for those features are intentionally not included in this document and should be added to a v2 SRS addendum if pursued later.

---

## 14. Suggested Additions (flagged for your review)

A few things I'd recommend considering adding, beyond what was asked, since they tend to come up quickly once implementation starts:

- **A glossary/README cross-reference** so anyone reading the SRS without the PRD open still understands terms like "tier" and "gateway mode" (drafted above in Section 1.3 — expand if needed).
- **A "Configuration Schema" appendix** — a literal JSON Schema or TypeScript interface for the config file, so validation rules in Section 5.2 have one canonical source of truth to test against.
- **A short "Known Limitations" section** (separate from Out of Scope) — e.g., fixed-window's boundary burst behavior is a known, accepted trade-off, not a bug — useful to state explicitly so it reads as an intentional design decision in an interview, not an oversight.
- **A minimal threat model paragraph** — even 3-4 sentences on "what R8Gate does and doesn't protect against" (e.g., it mitigates volumetric abuse; it does not replace WAF/DDoS protection at the network layer) — this kind of scoping statement is something senior engineers specifically look for and its absence is often noticed.

Happy to draft any of these into the document now if you'd like them included, or keep them as a follow-up once implementation surfaces which ones actually matter in practice.