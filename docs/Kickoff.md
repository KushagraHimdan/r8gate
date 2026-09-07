# R8Gate — Project Kickoff Brief

**Purpose of this file:** paste this into a new chat, along with `memory.md` (once it has entries) and `docs/PRD.md`, `docs/SRS.md`, `docs/architecture.md`, `docs/design.md`, `docs/phases.md` if deeper detail is needed on any step, to give a new conversation full context on what R8Gate is and exactly where we've verified things are working.

---

We are building **R8Gate** — a distributed rate limiter and API gateway. Tech stack: **React** (dashboard frontend, Vite + Tailwind), **Node.js + Express** (backend — both middleware and gateway), **Redis** (rate-limit counters, atomic operations), **MongoDB** (client/tier/config storage), **Docker + Docker Compose** (local run), **Jest + autocannon/k6** (unit + concurrency/load testing).

R8Gate ships in two forms: an importable **Express middleware** (`app.use(r8gate(config))`) and a standalone **gateway** service that proxies requests to a backend target after applying rate-limit checks. Two algorithms are in scope: **fixed window** and **token bucket**, both Redis-backed with atomic operations so limits stay correct even across multiple gateway instances — that concurrency correctness is the core engineering claim of the project, not a side detail.

This is a fresh build — no existing code yet. Five planning documents (PRD, SRS, architecture, design, phases) were fully written before any code, so **decisions are already made**; we're not re-deriving scope or architecture as we go, only building what those docs already specify. Full detail on any step lives in `docs/phases.md`; this brief lists the condensed task sequence.

This time, going step by step, running and verifying each step myself before moving to the next. For every step: tell me exactly which file the code goes in, explain what the code does in plain language, and don't move to the next step until I confirm the current one is working.

**Phase 0: Setup**
1. Monorepo folder structure created (`packages/r8gate-core`, `apps/gateway`, `apps/demo-backend`, `apps/dashboard`, `scripts/`, `docs/`)
2. Root workspace config (npm/yarn/pnpm workspaces)
3. Git repo + `.gitignore` initialized
4. `docker-compose.yml` — Redis + MongoDB services
5. `.env.example` with expected variables
6. ESLint + Prettier + Jest configured
7. Planning docs moved into `docs/`, stub `README.md` written

**Phase 1: Core Rate Limiting Engine** ⚠ the heart of the project — don't rush this
8. Redis client connection module
9. Fixed window algorithm (Redis `INCR`+`EXPIRE` or Lua script)
10. Token bucket algorithm (Redis Lua script — atomic refill + deduct)
11. Shared algorithm interface so both are swappable via config
12. Unit tests (boundary behavior, fresh-client state, zero-quota denial)
13. **Concurrency test** — 200 parallel requests against a 100 limit, assert exactly 100 allowed — the single most important test in the project

**Phase 2: Config & Client Resolution Layer**
14. Mongoose schemas: `Client`, `Tier`, `RouteOverride`
15. Schema-level validation
16. `scripts/seed-clients.js` — demo clients across at least two tiers
17. Config Resolver — in-memory cache, `resolvePolicy(clientId, route)`, route-override > tier > default precedence
18. Periodic cache refresh + graceful MongoDB-outage degradation
19. Fail-fast startup if MongoDB unreachable at boot

**Phase 3: Express Middleware Integration**
20. `r8gate(config)` middleware factory
21. Client identity resolution (API key → JWT claim → IP fallback)
22. Full request evaluation flow wired end-to-end
23. Rate-limit response headers (`X-RateLimit-*`, `Retry-After`)
24. Unknown-API-key handling (default tier or reject mode)
25. Fail-open/fail-closed Redis-unreachable behavior
26. Global error handler
27. Integration tests against a minimal test Express app

**Phase 4: Standalone Gateway Mode**
28. `apps/gateway` standalone Express app using the same core middleware
29. Gateway config loader (`r8gate.config.json`)
30. Proxy layer — forward allowed requests to one backend target
31. Backend-unreachable handling (`502`/`504`, no crash)
32. `apps/demo-backend` — trivial API to proxy to during dev/demo
33. `/r8gate/health` and `/r8gate/ready` endpoints

**Phase 5: Logging & Metrics Feed**
34. Structured JSON logging per request (correlation ID, masked client ID, outcome)
35. In-memory rolling metrics aggregator
36. `/r8gate/metrics` endpoint
37. (stretch) WebSocket live push feed

**Phase 6: Dashboard (Frontend)**
38. `apps/dashboard` — Vite + React + Tailwind, design tokens from `docs/design.md`
39. Static layout with mock data (stat cards, chart, table)
40. Wire up real data via polling/WebSocket
41. Interactions: sortable table, search/filter, time-range toggle, inline row expansion
42. Loading / error / empty states
43. Responsive + accessibility pass

**Phase 7: Load Testing & Correctness Validation**
44. `scripts/load-test.js` (autocannon/k6)
45. Multi-instance concurrency correctness test (2+ gateway instances, shared Redis)
46. Latency test (p95 < 10ms target)
47. Throughput test (≥500 req/s target)
48. Document results honestly in `docs/load-test-results.md`

**Phase 8: Security Review & Hardening**
49. Manually verify each SEC requirement (no plaintext keys in logs, no Redis-key injection, hop-by-hop headers stripped, stricter default-tier limits, no key leakage via dashboard)
50. `npm audit` across workspaces
51. `docs/security-checklist.md` documenting each check

**Phase 9: Documentation, Polish & Deployment Packaging**
52. Finalize `docker-compose.yml` for the full stack, proper Dockerfiles
53. Real `README.md` — architecture diagram, quick-start, algorithm trade-offs, known limitations, threat model paragraph
54. Cleanup pass (dead code, stray logs, unused deps)
55. Verify "5-minute clone-to-running" claim on a clean checkout
56. Tag `v1.0-mvp`

I'm comfortable with core MERN (React/Node/Express/MongoDB) but newer to: **Redis and atomic/Lua operations, distributed-systems concurrency concepts (race conditions, idempotency), and reverse-proxy mechanics**. Please explain those simply as we hit them, not just write the code.

## How `memory.md` works

`memory.md` is a running log of **what has actually been built and verified**, kept separate from this brief and from `docs/phases.md`. This brief and the planning docs describe the plan; `memory.md` records what's actually done. Append entries as work completes and is confirmed — don't rewrite history; if something logged as done later breaks, add a new entry referencing the old one and explaining why, rather than silently overwriting it. At the start of any new chat, read `memory.md` first to know exactly where we left off.

## Current status

We are currently on **Phase 0, Task 1**.