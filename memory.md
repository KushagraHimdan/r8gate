# R8Gate — Build Memory

Running log of what has actually been built and verified. Append-only — if something logged as done later breaks, add a new entry referencing this one rather than editing it away.

---

## Phase 0: Setup

- **Task 1 — Monorepo folder structure**: Created (`packages/r8gate-core`, `apps/gateway`, `apps/demo-backend`, `apps/dashboard`, `scripts/`, `docs/`). ✅ Verified.
- **Task 2 — Root workspace config**: Root `package.json` created with `"workspaces": ["packages/*", "apps/*"]`. ✅ Verified.
- **Task 3 — Git + .gitignore**: `git init` done. `.gitignore` excludes `node_modules/`, `.env`, `dist/`, `build/`, `coverage/`, `*.log`. ✅ Verified via `git status`.
- **Task 4 — docker-compose.yml (Redis + MongoDB)**: Redis (`redis:7-alpine`, port 6379) and MongoDB (`mongo:7`, port 27017) services defined, named volumes for persistence. Docker Desktop installed (had to enable WSL2/VirtualMachinePlatform Windows features first). ✅ Verified — both containers `Up` via `docker compose ps`.
- **Task 5 — .env.example**: Created with `REDIS_URL`, `MONGO_URI`, `PORT`, `NODE_ENV`. Copied to real `.env` (gitignored). ✅ Verified.
- **Task 6 — ESLint + Prettier + Jest**: Installed at root. Note: ESLint v10 was installed, which requires the new flat-config format (`eslint.config.js`), not the old `.eslintrc.json` — switched to flat config accordingly. Prettier (`.prettierrc`) and Jest (`jest.config.js`) configured. Root `package.json` scripts updated (`test`, `lint`). ✅ `npx eslint .` runs clean.

- **Task 7 — Docs moved into `docs/`, README written**: All 5 planning docs + `docs/kickoff.md` moved into `docs/`, correctly cased to match README links (`PRD.md`, `SRS.md`, `architecture.md`, `design.md`, `phases.md`). Root `README.md` in place. Fixed a case-sensitivity issue (Windows renamed files as no-ops on case-only changes — needed a two-step rename via a temp filename). Also caught and fixed an empty `.prettierrc` (0 bytes) missed earlier. ✅ Verified.

## Phase 0 — COMPLETE

All 7 tasks done and verified. First commit made and pushed to GitHub (`main` branch).

## Current status

Starting **Phase 1: Core Rate Limiting Engine** — the heart of the project. First task: Redis client connection module.