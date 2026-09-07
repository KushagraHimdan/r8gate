# R8Gate — Development Plan (Phases)

**Version:** 1.0
**Status:** Draft
**Companion documents:** PRD.md, SRS.md, architecture.md, design.md
**Owner:** [Your Name]
**Timeframe:** 2-4 weeks, solo build

This document sequences everything defined in PRD/SRS/architecture/design into an executable roadmap. Each phase has a clear goal, concrete tasks, dependencies, a Definition of Done, and priority. Work through phases in order — later phases assume earlier ones are actually done, not just started.

---

## How to read this document

- **P0** = blocking, MVP cannot ship without it. **P1** = important, do if on schedule. **P2** = stretch, only if time remains.
- **Depends on** lists what must be complete (per its own Definition of Done) before a phase can start.
- **Definition of Done (DoD)** is the exit checklist — don't move to the next phase until every box is true, not just "mostly working."
- Every phase maps back to specific requirement IDs from SRS.md so nothing here is invented outside what was already specified.

---

## Phase 0 — Project Setup & Environment

**Goal:** A clean, working skeleton that runs, with all tooling in place, before any real logic is written.
**Priority:** P0
**Depends on:** Nothing (starting point)
**Estimated time:** 0.5–1 day

### Tasks
1. Initialize monorepo folder structure exactly as defined in `architecture.md` Section 8 (`packages/r8gate-core`, `apps/gateway`, `apps/demo-backend`, `apps/dashboard`, `scripts/`, `docs/`).
2. Set up root `package.json` workspaces (npm/yarn/pnpm workspaces) so `packages/r8gate-core` can be imported by `apps/gateway` locally without publishing.
3. Initialize Git repo, `.gitignore` (node_modules, `.env`, build artifacts), initial commit.
4. Set up `docker-compose.yml` with three services: `redis`, `mongo`, and a placeholder `app` service (real Dockerfile comes later, once there's something to containerize).
5. Create `.env.example` with all expected environment variables (`REDIS_URL`, `MONGO_URL`, `PORT`, etc.) per `architecture.md`.
6. Install and configure: ESLint + Prettier (consistent style from day one), Jest (root-level test runner config), `dotenv`.
7. Move PRD.md, SRS.md, architecture.md, design.md into `docs/`.
8. Write a stub `README.md` with project name, one-paragraph description, and a "Docs" section linking to `docs/`.

### Definition of Done
- [ ] `docker-compose up` starts Redis and MongoDB containers successfully with no errors.
- [ ] Folder structure matches architecture.md exactly (or documented deviations noted).
- [ ] `npm install` at root succeeds and resolves workspace packages.
- [ ] Lint/format runs clean on the (currently empty) codebase.
- [ ] All four docs are in `docs/` and linked from README.

---

## Phase 1 — Core Rate Limiting Engine (the heart of the project)

**Goal:** Fixed-window and token-bucket algorithms implemented, Redis-backed, atomic, and fully unit-tested — in isolation, with no HTTP/Express involved yet.
**Priority:** P0
**Depends on:** Phase 0 complete
**Maps to:** SRS `[FUNC-008]`–`[FUNC-011]`, `[EDGE-001]`–`[EDGE-005]`, `[EDGE-009]`, `[DATA-001]`, `[DATA-002]`
**Estimated time:** 3–4 days — this is the phase to *not* rush.

### Tasks
1. Set up Redis client connection module (`packages/r8gate-core/src/redisClient.js`) with connection error handling.
2. Implement **fixed window** algorithm:
   - Redis key pattern: `rl:fw:{clientId}:{routeId}:{windowStart}`
   - Use `INCR` + `EXPIRE` (or a single Lua script combining both atomically — preferred, avoids a race between the two calls)
   - Return `{ allowed, remaining, resetAt }`
3. Implement **token bucket** algorithm:
   - Redis key pattern: `rl:tb:{clientId}:{routeId}` (hash: `tokens`, `lastRefill`)
   - Implement as a **Lua script** (`EVAL`) that atomically: computes elapsed time, refills tokens (capped at max capacity), checks/deducts one token, returns result
   - Return `{ allowed, remaining, resetAt }`
4. Write a shared algorithm interface/contract (`packages/r8gate-core/src/algorithms/index.js`) so both algorithms expose the same function signature — this is what lets the config layer swap algorithms without touching call sites.
5. Unit tests (mocking Redis or using a local test Redis instance):
   - Fixed window: boundary behavior (`EDGE-002`), fresh client starts full (`EDGE-005`)
   - Token bucket: refill math correctness, cap at max capacity (`EDGE-004`), fresh client starts full (`EDGE-005`)
   - Both: zero-quota client is denied (`BIZ-001`)
6. **Concurrency test** (this is the single most important test in the whole project): fire N concurrent requests (e.g., 200) for one client against a limit of 100, assert exactly 100 are allowed and 100 are denied, no more, no less. This directly validates `EDGE-001` and is the proof-point for the entire "atomicity matters" thesis of the project.

### Definition of Done
- [ ] Both algorithms implemented, passing all unit tests.
- [ ] Concurrency test passes consistently across multiple runs (run it 5+ times to rule out flakiness, not just once).
- [ ] Test coverage on `algorithms/` ≥ 85% (per PRD success metric).
- [ ] No HTTP/Express code exists yet in this phase — the engine is proven correct as a standalone library first.
- [ ] A short `docs/algorithms.md` note (can be brief) documenting the trade-offs between the two, written once you've actually implemented and tested them, not before.

---

## Phase 2 — Config & Client Resolution Layer

**Goal:** MongoDB-backed client/tier/route-override configuration, loaded into an in-memory cache, resolvable per request.
**Priority:** P0
**Depends on:** Phase 0 complete (can run in parallel with Phase 1 if you want, but must be done before Phase 3)
**Maps to:** SRS `[FUNC-004]`–`[FUNC-007]`, `[VALID-001]`–`[VALID-006]`, `[DATA-001]` (config side), `[ERR-004]`, `[ERR-005]`, `[EDGE-006]`
**Estimated time:** 2 days

### Tasks
1. Define Mongoose schemas (or equivalent) for `Client`, `Tier`, `RouteOverride` exactly per `SRS.md` Section 5.1 field tables.
2. Implement schema-level validation matching `[VALID-001]`–`[VALID-006]` exactly (e.g., `maxRequests` positive integer, `algorithm` enum-restricted).
3. Write `scripts/seed-clients.js` — seeds a handful of demo clients across at least two tiers (e.g., "free" and "pro") for local testing/demo purposes.
4. Implement the **Config Resolver** module: loads all config from MongoDB at startup into an in-memory cache; exposes `resolvePolicy(clientId, route)` returning the effective policy (route override > tier default > global default, per `BIZ-004`).
5. Implement periodic cache refresh (e.g., every 30s) and graceful degradation if MongoDB becomes unreachable after startup (`ERR-005`) — keep serving from last-known-good cache, log a warning.
6. Implement fail-fast startup behavior if MongoDB is unreachable at boot (`ERR-004`).
7. Implement fallback-to-default-tier behavior if a client's assigned tier is deleted/missing (`EDGE-006`).
8. Unit tests: validation rejects bad config with clear errors; resolver correctly applies precedence order; cache refresh and degradation behavior (can mock MongoDB failures).

### Definition of Done
- [ ] Seed script populates MongoDB with demo data reproducibly.
- [ ] Resolver correctly returns route-override > tier > default precedence, tested explicitly.
- [ ] Invalid config (bad enum value, negative number) is rejected at load time with a human-readable error, not a silent default or crash.
- [ ] Simulated MongoDB outage after startup does not take the resolver down (test this manually or via a mocked failure).

---

## Phase 3 — Express Middleware Integration

**Goal:** Wire Phase 1 (engine) + Phase 2 (config) into real Express middleware — the first fully working, demoable slice of the project.
**Priority:** P0
**Depends on:** Phase 1 + Phase 2 both complete
**Maps to:** SRS `[FUNC-001]`–`[FUNC-003]`, `[FUNC-012]`–`[FUNC-016]`, `[AUTH-001]`–`[AUTH-003]`, `[ERR-001]`, `[ERR-002]`, `[ERR-006]`, `[ERR-007]`, `[EDGE-008]`, `[SEC-001]`, `[SEC-002]`
**Estimated time:** 2–3 days

### Tasks
1. Build the `r8gate(config)` middleware factory function (`packages/r8gate-core/src/middleware.js`) matching the usage example in `architecture.md` Section 3.1.
2. Implement client identity resolution precedence: API key header → JWT claim (if present) → IP fallback (`FUNC-001`, `FUNC-002`, `EDGE-008` for malformed/missing key).
3. Implement the full request evaluation flow per `architecture.md` Section 4.1: resolve identity → resolve policy → evaluate → attach headers → allow/deny.
4. Implement all required response headers (`FUNC-015`, `FUNC-016`): `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` on 429.
5. Implement unknown-API-key handling: default-tier fallback, or `401` if reject-mode is configured (`AUTH-002`, `AUTH-003`).
6. Implement fail-open/fail-closed Redis-unreachable behavior, configurable, fail-closed as documented default (`ERR-001`, `ERR-002`).
7. Implement global error handler catching unhandled exceptions, returning `500` without leaking stack traces (`ERR-006`).
8. Implement config validation at middleware initialization — bad config prevents startup with a clear message (`ERR-007`).
9. Security pass: ensure API keys are never logged in plaintext (`SEC-001`); sanitize client-supplied values before Redis key interpolation (`SEC-002`).
10. Build a minimal test Express app (can live temporarily in `packages/r8gate-core/test/fixtures/`) using the middleware, for integration testing.
11. Integration tests: allowed request passes through with correct headers; denied request short-circuits with 429; unknown key behavior both modes; Redis-down fail-open vs fail-closed behavior (can simulate by stopping the test Redis connection).

### Definition of Done
- [ ] Middleware is a working, importable module — `app.use(r8gate(config))` functions exactly as documented in architecture.md.
- [ ] All response headers present and correct on both allowed and denied responses.
- [ ] Integration tests cover: normal allow, normal deny, unknown key (both modes), Redis-down (both fail-open and fail-closed configurations).
- [ ] **Milestone demo point:** you can `curl` a locally protected test route repeatedly and watch it return `200`s then `429`s with correct headers. This is a legitimate demo-able artifact even before gateway mode or dashboard exist.

---

## Phase 4 — Standalone Gateway Mode

**Goal:** Extend the same core engine into a standalone reverse-proxy app, per the second deliverable defined in the PRD.
**Priority:** P0
**Depends on:** Phase 3 complete
**Maps to:** SRS `[FUNC-017]`–`[FUNC-019]`, `[BIZ-005]`, `[ERR-003]`, `[SEC-003]`
**Estimated time:** 2 days

### Tasks
1. Build `apps/gateway/src/server.js` — a standalone Express app that uses the same `r8gate-core` middleware.
2. Build the config loader for gateway mode (reads `r8gate.config.json` per `architecture.md` Section 3.1 example).
3. Implement the proxy layer (`apps/gateway/src/proxy.js`) using `http-proxy-middleware` or equivalent — forward allowed requests to the single configured backend target, preserving method/headers/body (`FUNC-017`).
4. Ensure rate-limit evaluation always happens before proxying — never proxy first and check after (`BIZ-005`).
5. Strip hop-by-hop headers before forwarding responses back to the client (`SEC-003`).
6. Handle backend-unreachable/timeout cases: return `502`/`504` appropriately, log the failure, don't crash the process (`ERR-003`, `FUNC-019`).
7. Build `apps/demo-backend/` — a trivial Express API (2–3 routes returning canned JSON) purely to have something realistic to proxy to during development and demos.
8. Build the `/r8gate/health` and `/r8gate/ready` endpoints (readiness checks Redis + MongoDB connectivity).
9. Integration tests: allowed request is correctly proxied and backend response returned unmodified (aside from added headers); denied request never reaches the backend (verify via a call counter on the demo backend); backend-down scenario returns 502 cleanly.

### Definition of Done
- [ ] `npx r8gate start --config r8gate.config.json` (or equivalent local script) starts the gateway and successfully proxies to `demo-backend`.
- [ ] A denied request demonstrably never reaches the backend (test asserts backend's request count didn't increment).
- [ ] Backend-down scenario handled gracefully with correct status code, no crash.
- [ ] Health/readiness endpoints return accurate status.

---

## Phase 5 — Logging & Metrics Feed

**Goal:** Structured logging and an in-memory metrics feed that the dashboard (Phase 6) will consume.
**Priority:** P0 (logging) / P1 (metrics feed polish)
**Depends on:** Phase 3 complete (needs a working middleware to log from); ideally Phase 4 too so gateway-mode requests are logged as well
**Maps to:** SRS `[FUNC-020]`–`[FUNC-023]`
**Estimated time:** 1–1.5 days

### Tasks
1. Implement structured JSON logging (correlation ID, client ID [masked], route, algorithm, outcome, timestamp) on every evaluated request (`FUNC-020`, `FUNC-021`).
2. Implement an in-memory rolling metrics aggregator: total/allowed/denied counts, both globally and per-client, over a configurable rolling window.
3. Build `/r8gate/metrics` REST endpoint exposing current aggregate state as JSON (`FUNC-022`).
4. (P1, if time allows) Build a WebSocket push feed as an alternative/addition to polling, per architecture.md Section 3.3 — otherwise polling every 3–5s from the dashboard is sufficient and simpler.
5. Unit/integration tests: metrics counters increment correctly on allow/deny events; `/r8gate/metrics` returns well-formed JSON matching the shape the dashboard will expect.

### Definition of Done
- [ ] Every request evaluation produces a structured log line.
- [ ] `/r8gate/metrics` returns accurate, correctly-shaped counts that match manually-verified request counts in a test.
- [ ] API keys never appear in plaintext in any log output (spot-check this explicitly).

---

## Phase 6 — Dashboard (Frontend)

**Goal:** The React dashboard exactly as specified in `design.md` — one screen, live data, all defined states handled.
**Priority:** P0 (core screen) / P1 (polish: animations, inline row expansion, time-range toggle) / P2 (WebSocket live push if not done in Phase 5)
**Depends on:** Phase 5 complete (needs `/r8gate/metrics` to consume)
**Maps to:** design.md in full; SRS `[FUNC-022]`, `[FUNC-023]`, `[SEC-005]`
**Estimated time:** 3–4 days

### Tasks (follow design.md Section 14's build order deliberately)
1. Set up `apps/dashboard` with Vite + React + Tailwind; configure the color/spacing/type tokens from `design.md` Sections 9–11 as Tailwind theme config.
2. Build static layout first with **mock data** — Section A (stat cards), Section B (chart placeholder), Section C (table) — matching the visual reference in `design.md` Section 13.
3. Wire up real data: polling `/r8gate/metrics` (3–5s interval) or WebSocket if built; replace mock data.
4. Implement all interaction behaviors from `design.md` Section 5: sortable table (default sort by Denied descending), search/filter (debounced), time-range toggle, inline row expansion.
5. Implement all three required states from `design.md` Section 7: loading (skeletons on initial load only), error (stale-data banner + retry), empty (explicit "no traffic yet" messaging, not a blank screen).
6. Implement responsive behavior per `design.md` Section 8 (desktop → tablet 2×2 → mobile stacked cards).
7. Accessibility pass per `design.md` Section 12: keyboard navigation, focus states, `aria-live` regions (throttled), semantic table/heading markup.
8. Ensure the dashboard does not expose raw API key values anywhere in the UI (`SEC-005`).

### Definition of Done
- [ ] Dashboard visually matches `design.md` Section 13's reference layout.
- [ ] All three states (loading/error/empty) manually verified by simulating each (stop the backend, seed zero data, throttle network).
- [ ] Live polling correctly reflects real allow/deny activity generated by hitting the gateway/middleware from Phase 3/4.
- [ ] Keyboard-only navigation works for search, sort, and row expansion (manually tested, tab through the whole page).
- [ ] Responsive check at all three breakpoints (browser devtools resize is sufficient).

---

## Phase 7 — Load Testing & Correctness Validation

**Goal:** Prove, with real numbers, the claims the whole project rests on: correctness under concurrency and acceptable latency overhead. This phase produces the artifacts you'll actually show in an interview.
**Priority:** P0
**Depends on:** Phase 4 complete (need a running gateway to load-test against); Phase 6 optional but nice to have running alongside for a live visual demo
**Maps to:** SRS `[PERF-001]`–`[PERF-004]`, PRD Success Metrics, `[EDGE-007]`
**Estimated time:** 1.5–2 days

### Tasks
1. Write `scripts/load-test.js` using autocannon or k6.
2. **Concurrency correctness test:** 50+ parallel simulated clients hammering a route with a known limit (e.g., 100 req/min) — assert zero over-limit requests allowed through. Run against **2+ gateway instances** sharing one Redis instance (per `PERF-002`) — this is the test that proves the "distributed" claim, not just single-instance correctness (already covered in Phase 1's unit test).
3. **Latency test:** measure p95 latency added by rate-limit evaluation under load — target < 10ms (`PERF-001`).
4. **Throughput test:** confirm the gateway sustains ≥ 500 req/s locally without gateway-attributable errors (`PERF-003`).
5. **Sustained high-frequency single-client test:** one client sending very high-frequency requests shouldn't degrade service to other clients (`EDGE-007`) — run one aggressive client alongside several normal ones, confirm normal clients' latency/success is unaffected.
6. Document results in `docs/load-test-results.md` — raw numbers, methodology, and how to reproduce them. This becomes primary interview material — be honest about the numbers, don't cherry-pick a lucky run.

### Definition of Done
- [ ] Multi-instance concurrency test shows 0% over-limit requests allowed, run at least 3 times to confirm consistency, not a one-off.
- [ ] Latency and throughput numbers meet or documented-honestly-fall-short-of PRD targets.
- [ ] Results are written down with methodology, not just a screenshot of one run.
- [ ] If any target is missed, the gap and likely cause are documented rather than hidden — this is more credible in an interview than suspiciously perfect numbers.

---

## Phase 8 — Security Review & Hardening Pass

**Goal:** Walk through SRS Section 10 deliberately, as a dedicated pass rather than hoping earlier phases covered it incidentally.
**Priority:** P0
**Depends on:** Phases 3, 4, 6 complete
**Maps to:** SRS `[SEC-001]`–`[SEC-006]`
**Estimated time:** 0.5–1 day

### Tasks
1. Manually verify `SEC-001` (no plaintext API keys in logs) by grepping actual log output.
2. Manually verify `SEC-002` (no key-injection into Redis keys) — attempt a crafted API key with special characters (`:`, newlines) and confirm it's sanitized/rejected, not silently accepted into a Redis key.
3. Manually verify `SEC-003` (hop-by-hop headers stripped in proxy responses) using a raw `curl -i` inspection.
4. Confirm `SEC-004` — unauthenticated/default-tier limit is stricter than any named tier, by checking config.
5. Confirm `SEC-005` — dashboard/metrics endpoint doesn't leak raw API key values; check the actual JSON payload of `/r8gate/metrics`.
6. Run `npm audit` across all workspaces, resolve or explicitly document any critical/high vulnerabilities found (`SEC-006`).
7. Write `docs/security-checklist.md` documenting each check and its result — this is a strong, concrete artifact for interviews ("here's my security review process," not just a claim).

### Definition of Done
- [ ] All six SEC requirements manually verified with documented evidence (log excerpt, curl output, etc.), not just "looks fine."
- [ ] `npm audit` run and results addressed or documented.
- [ ] `docs/security-checklist.md` committed to the repo.

---

## Phase 9 — Documentation, Polish & Deployment Packaging

**Goal:** Make the finished project easy for a stranger (or an interviewer) to clone, run, and understand in minutes.
**Priority:** P0
**Depends on:** All prior phases complete
**Maps to:** PRD Acceptance Criteria (all), SRS `[ERR-007]` (validated by a clean setup), architecture.md Section 8
**Estimated time:** 1–1.5 days

### Tasks
1. Finalize `docker-compose.yml` to spin up everything (`redis`, `mongo`, `gateway`, `demo-backend`, `dashboard`) with one command, using proper Dockerfiles (multi-stage builds for the Node apps to keep images small).
2. Write the real `README.md`: project description, architecture diagram (can reuse/simplify the one from `architecture.md`), quick-start (`docker-compose up`, seed script, open dashboard), explanation of the algorithm trade-offs (link to Phase 1's `docs/algorithms.md`), link to load test results and security checklist.
3. Add a short **"Known Limitations"** section to the README (per SRS Section 14's suggestion) — fixed-window boundary bursts, single-Redis-instance/no-cluster, single-backend-target-only, etc. — framed as intentional MVP scoping, referencing PRD's Out of Scope section.
4. Add a short **threat model paragraph** (per SRS Section 14's suggestion) — what R8Gate does and doesn't protect against.
5. Clean up: remove dead code, unused dependencies, console.logs left over from debugging; run linter/formatter one final time across the whole repo.
6. Verify the PRD's "under 5 minutes from clone to running" success metric by literally timing a fresh clone + setup on a clean checkout (or ask a friend to try it).
7. Tag a `v1.0-mvp` release/commit.

### Definition of Done
- [ ] A completely fresh clone of the repo, following only the README, results in a working local demo in under 5 minutes.
- [ ] README includes architecture diagram, setup steps, algorithm trade-off explanation, known limitations, and threat model paragraph.
- [ ] No stray debug code, TODOs without tracking, or unused dependencies remain.
- [ ] Repo tagged/released at a clear MVP milestone.

---

## Milestones Summary

| Milestone | Phases included | What you can demo at this point |
|---|---|---|
| **M1 — Engine proven correct** | 0, 1 | Unit + concurrency tests passing on the core algorithms, in isolation |
| **M2 — Middleware working end-to-end** | 2, 3 | `curl` demo: protected route returns 200s then 429s with correct headers |
| **M3 — Full gateway operational** | 4, 5 | Standalone gateway proxying to a real backend, with rate limiting and logging |
| **M4 — Full product demoable** | 6 | Live dashboard showing real traffic, all states handled |
| **M5 — Claims validated** | 7, 8 | Load test results and security checklist — the proof behind the pitch |
| **M6 — Ship-ready** | 9 | Clean repo, full docs, 5-minute setup — ready to link on a resume |

---

## Suggested Timeline (2-4 week range)

| Week | Focus |
|---|---|
| **Week 1** | Phases 0–3 (setup → engine → config → middleware). This is the highest-risk, most important week — don't move on until Phase 1's concurrency test is genuinely solid. |
| **Week 2** | Phases 4–6 (gateway mode → logging/metrics → dashboard). This is where the project becomes visually demoable. |
| **Week 3** | Phases 7–8 (load testing → security review), plus buffer for any Week 1–2 slippage — these phases are commonly underestimated. |
| **Week 4 (if used)** | Phase 9 (docs/polish/deployment) + stretch goals (sliding window algorithm, WebSocket live push, additional test coverage) if ahead of schedule. If running to exactly 2-3 weeks instead, compress by treating Phase 9 as a rolling task done incrementally throughout, not a dedicated final week. |

---

## Priority Recap (P0 vs P1 vs P2, project-wide)

**P0 — MVP cannot ship without these:** Phases 0, 1, 2, 3, 4, 5 (logging only, not WS), 6 (core screen only), 7, 8, 9.

**P1 — do if on schedule:** WebSocket live push (vs. polling), inline row expansion animation polish, sliding-window-counter algorithm as a third option.

**P2 — only if significantly ahead of schedule:** anything explicitly listed in PRD's "Out of Scope" section (multi-target routing, admin API auth, leaky bucket, alerting) — these are v2 backlog, not this project's finish line. Resist pulling them forward; a fully-polished MVP beats a half-finished superset every time, per the PRD's own guiding principle.

---

## Definition of Done — Whole Project

The project is complete when **all P0 items across all phases are checked off**, and specifically:
- Every PRD Acceptance Criterion (PRD.md Section 12) is demonstrably true.
- Every SRS requirement tagged in this document's phases has a passing test or documented manual verification.
- The README-driven 5-minute setup has been independently verified at least once.
- Load test and security review artifacts exist in the repo, not just claimed in conversation.

No phase should be considered "done" on the basis of code existing — it's done when its Definition of Done checklist is fully checked, tests pass, and (where specified) it's been manually verified, not assumed.