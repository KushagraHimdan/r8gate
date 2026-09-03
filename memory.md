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

## Current status

Phase 0 is essentially done — only **Task 7** (move planning docs into `docs/`, write stub README) remains before moving to Phase 1.