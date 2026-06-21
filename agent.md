# TalentMesh — Implementation Runbook (`agent.md`)

> **Purpose.** This is the master, agent-executable build plan for TalentMesh — the
> multi-agent, human-in-command recruiting platform specified in [`PRD.md`](PRD.md)
> and [`TalentMesh_Design.md`](TalentMesh_Design.md). It is organized into **14 phases**,
> each with **multiple tasks**, and a mandatory **review-and-fix gate** between every task
> and every phase. An agent (or engineer) executes it top-to-bottom; you do **not** advance
> to phase _N+1_ until phase _N_ is implemented, reviewed, bug-free, and tested green.
>
> **Golden rule:** _Before starting any phase, re-read [`PRD.md`](PRD.md) in full_ (and the
> design doc sections it points to) to re-anchor on the product before writing code.

---

## 0. How to use this document

1. Work **one phase at a time, top to bottom**. Within a phase, work **one task at a time**.
2. **Before each phase:** complete the **Phase Pre-flight** (re-read the PRD, confirm prerequisites, restate the phase goal in your own words).
3. **For each task:** implement → **self-review as the Expert Code Reviewer** (§2) → log findings in the phase's _Defect Log_ → fix all findings → write/run tests → confirm green.
4. **End of each phase:** run the **Consolidated Phase Review** (review every task together for integration bugs, contract drift, security, and PRD conformance) → fix → run the **Phase Exit Gate** (full test suite + acceptance checks). Only a **fully green gate** unlocks the next phase.
5. Keep a running **Defect Log** and **Decision Log** per phase (templates in §15).
6. Never silently skip a task, a review, or a failing test. If something is blocked, record it in the Risk Register (§16) and surface it.

**Conventions in this file**
- `T-<phase>.<n>` = a task. `R-<phase>.<n>` = the review of that task.
- "DoD" = Definition of Done (must all be true to close the item).
- 🔒 = security-relevant. ⚖️ = compliance-relevant (maps to a `COMP-*` rule). ♿ = accessibility-relevant.

---

## 1. Target architecture (recap — read alongside the PRD)

Three planes, exactly as in the design doc and PRD §13:

- **Frontend** — TypeScript / **Next.js (App Router, React Server Components)**: mission-control UI, Approvals Inbox, Candidate 360, Compliance/Audit, Analytics, candidate chat. Real-time via **SSE** (`EventSource`) for the agent feed and **WebSocket** for candidate chat.
- **Backend** — **Python / FastAPI** gateway → **LangGraph** durable graph orchestrating **Google ADK** agents (Sequential / Parallel / Loop / Custom) → **Claude Opus 4.8 / Sonnet 4.6 / Haiku 4.5**. **A2A** client/server to external vendor agents. **RAG** over **Chroma**.
- **Data & infra** — **PostgreSQL** (app state + LangGraph checkpointer), **Redis** (stream pub/sub), **Chroma** (vector DB), **LangSmith** (tracing/eval). Object storage for artifacts (JDs, reports, offer letters).
- **Runtime** — the entire system is **Dockerized**: every service (web, api, postgres, redis, chroma, mock-a2a) runs in a container, and **`docker compose up` is the single entry point** for the full stack in both dev (hot-reload) and prod (parity images). The frontend is built entirely on the **[shadcn/ui](https://ui.shadcn.com)** design system.

**The 11 agents** (codename · model · ADK type) — full table in PRD §16.2: Atlas (Opus, root), Scribe (Sonnet), Scout (Sonnet, Parallel), Sift (Haiku→Sonnet, Sequential), Gauge (Opus, Loop), Echo (Sonnet+Haiku), Sync (Haiku, A2A), Aegis (Opus, guardrail/Custom), Verify (Haiku+Sonnet, Parallel/A2A), Lens (Sonnet), Bridge (Haiku, A2A).

**Model IDs** (use a central `model_for(role)` resolver — never hard-code at call sites): Opus 4.8 `claude-opus-4-8`, Sonnet 4.6 `claude-sonnet-4-6`, Haiku 4.5 `claude-haiku-4-5-20251001`.

**Seed data** (already in this folder): `jobs.json` (100 reqs), `candidates.json` (1000), `leveling_comp.json` (ladders + comp bands), `company_knowledge.json` (300+ KB docs), `compliance_rules.json`, `interview_rubrics.json`, `external_agent_cards.json` (A2A vendor cards, locally runnable). RAG collections per PRD §7 / design §7: `job_knowledge`, `candidate_corpus`, `company_knowledge`, `compliance_kb`, `interview_kb`.

---

## 2. The Expert Code Reviewer protocol (applies to EVERY task and EVERY phase)

After implementing each task — and again, consolidated, at the end of each phase — switch hats and act as a **senior staff engineer doing an adversarial code review**. Your job is to **find bugs and issues, list them explicitly, fix them all, and re-test**. Do not rubber-stamp. Assume there _are_ defects until proven otherwise.

### 2.1 Review dimensions (check every one)
1. **Correctness** — does it do what the task/PRD says? Off-by-one, wrong branch, race conditions, async/await misuse, unhandled `None`/`undefined`, wrong threshold comparison (e.g. borderline = `threshold-0.10`), incorrect funnel state transitions.
2. **Contract conformance** — request/response shapes match the shared OpenAPI/Zod schema; SSE event names exactly `agent.thinking` / `agent.token` / `stage.changed` / `approval.required` / `task.completed`; DB columns match models.
3. **Human-decision boundary** ⚖️ — verify **no agent path can auto-advance, auto-reject, or auto-hire** (COMP-EUAI-02). Every consequential edge must hit a LangGraph `interrupt()` surfaced in the Approvals Inbox.
4. **Reasoning log & auditability** ⚖️ — every ranking/advancement/rejection appends to the append-only `reasoning_log` (COMP-EUAI-01); time-travel replay reconstructs it.
5. **PII segmentation** 🔒⚖️ — demographic data never reaches scoring/ranking agents (COMP-PII-01); confirm the architectural gate, not just a comment.
6. **Security** 🔒 — authN/authZ on every endpoint, RBAC enforced server-side, input validation, no secrets in code/logs, signed A2A card verification, least-privilege OAuth scopes, SQL/template injection, SSRF on A2A/MCP calls.
7. **Reliability** — durable execution survives restart (checkpointer), idempotency, retries/timeouts/backoff on A2A & external calls, graceful SSE/WS reconnect, no resource leaks.
8. **Performance** — N+1 queries, unbounded fan-out, blocking calls in async paths, missing pagination on the 1000-candidate / 100-req data, streaming back-pressure.
9. **Tests** — unit + integration cover happy path **and** the PRD edge cases (§11): borderline, below-threshold, vendor timeout, version negotiation, clearance filter, resume-after-crash.
10. **Code quality / 2026 best practices** (§4) — types are strict, no `any`/`# type: ignore` without justification, lint/format clean, naming matches surrounding code, no dead code, docstrings/JSDoc on public APIs, error handling is explicit.
11. **Accessibility** ♿ (frontend tasks) — keyboard nav, ARIA/live-region for the agent feed, color-contrast, focus management.
12. **Observability** — structured logs, traces to LangSmith/OTel, meaningful error messages.

### 2.2 Review output (write it down — do not keep it in your head)
For each task and each phase, produce a **Defect Log** entry list:

```
[<id>] <severity: blocker|major|minor> <dimension> — <file:line> — <issue>
   → Fix: <what you changed>
   → Verified: <test/command that now passes>
```

### 2.3 The fix-and-re-test loop
- Fix **every blocker and major**. Triage minors (fix now or log to backlog with rationale).
- After fixing, **re-run the relevant tests** and, for cross-cutting fixes, the whole suite.
- A task/phase is **not done** while any blocker or major is open.

### 2.4 (Optional, recommended) second-set-of-eyes
For high-risk phases (governance, A2A, auth), run an **independent reviewer pass** — e.g. spawn a fresh review with no memory of the implementation rationale, or use `/code-review` / `/security-review`. Reconcile its findings into the Defect Log.

---

## 3. Phase Pre-flight & Exit Gate (the gate between phases)

### 3.1 Phase Pre-flight (do before writing any code in a phase)
- [ ] **Re-read [`PRD.md`](PRD.md)** end-to-end; skim the design-doc sections this phase touches.
- [ ] Restate the phase **objective** and its **PRD acceptance criteria** (PRD §14) in your own words.
- [ ] Confirm **all prerequisites** from prior phases are green (run their tests).
- [ ] Confirm the **environment** is healthy (services up, keys present) — `make doctor`.
- [ ] Create/clear this phase's **Defect Log** and **Decision Log**.

### 3.2 Phase Exit Gate (all must pass to advance)
- [ ] Every task in the phase closed with **zero open blockers/majors**.
- [ ] **Consolidated Phase Review** completed; its findings fixed and re-tested.
- [ ] **Full test suite green**: `make test` (backend `pytest`, frontend `vitest` + `playwright`), **lint/format/type-check clean** (`ruff`, `mypy`, `eslint`, `tsc`).
- [ ] **PRD acceptance criteria** relevant to this phase verified against seed data.
- [ ] **Security & compliance checks** for the phase pass (no auto-decision path; reasoning-log present; PII gate; signed-card verification where applicable).
- [ ] **No regressions** in earlier phases (re-run their suites).
- [ ] Decision Log + README/docs updated. Commit on a phase branch with a clean message.

> If any box is unchecked, **stay in this phase**. Do not advance.

---

## 4. Engineering standards (2026 best practices)

**Monorepo & tooling**
- **pnpm workspaces + Turborepo** for the JS side; **uv** (or Poetry) for Python. One repo, clear package boundaries.
- **Conventional Commits**, trunk-based with short-lived phase branches, PR per phase, required CI checks.
- **pre-commit** hooks: ruff, mypy, eslint, prettier, type-check, secret-scan (gitleaks).

**Backend (Python 3.12+)**
- **FastAPI** (async) + **Pydantic v2** models; **pydantic-settings** for config (12-factor, env-driven, no secrets in code).
- **SQLAlchemy 2.0** (typed) + **Alembic** migrations; **asyncpg** driver.
- **LangGraph** with `PostgresSaver` checkpointer; **Google ADK** for agent types; **a2a-sdk** for vendor calls; **MCP** clients for LinkedIn/ATS/calendar connectors; **Anthropic SDK** for Claude; **Chroma** + **Voyage** (or local `bge-large`) for RAG.
- **ruff** (lint+format), **mypy --strict**, **pytest** + **pytest-asyncio** + **httpx** test client. **structlog** + **OpenTelemetry** + **LangSmith** for observability.
- Typed boundaries everywhere; **no bare `except`**, explicit error types; retries via **tenacity** with jittered backoff on external I/O.

**Frontend (TypeScript, strict)**
- **Next.js (App Router) + React Server Components**, **TypeScript `strict`**, **Tailwind CSS** + **[shadcn/ui](https://ui.shadcn.com) — the design system for every screen** (Radix-based accessible primitives; install components via the shadcn CLI; theme via CSS variables + `components.json`), **TanStack Query v5** (server state) + **Zustand** (client state), **Zod** validation, **dnd-kit** for the kanban.
- **Auth.js (NextAuth v5)** OAuth2/OIDC + RBAC; **native `EventSource`** for SSE, dedicated **WebSocket** hook for candidate chat.
- **Types generated from the backend OpenAPI** (`openapi-typescript` / `orval`) — one source of truth; never hand-maintain API types.
- **ESLint (flat config) + Prettier**; **Vitest + Testing Library** (unit), **Playwright** (e2e), **Storybook** (component states), **axe-core** (a11y) in CI.
- **WCAG 2.1 AA** target; live-region announcements for the agent activity feed.

**Cross-cutting**
- **Contract-first**: backend publishes OpenAPI; SSE event union is a shared typed enum.
- **Secrets** via env/secret manager only; **.env.example** documents every var.
- **Multi-tenancy** isolation considered from day one (tenant scoping on every query).
- **Definition of Done** for any code: typed, tested, linted, documented, reviewed, and traceable.

**Containerization (the whole app is Dockerized — Docker is the only host prerequisite)**

- Every service — `web`, `api`, `postgres`, `redis`, `chroma`, and the `mock-a2a` vendor agents — runs as a **Docker container**; **`docker compose up` is the single command** that brings up the entire stack. There is no "run it on the host" path.
- **Dev runs in containers too.** `make dev` starts `web` + `api` **inside containers** with the source **bind-mounted for hot reload** (Next.js Fast Refresh, `uvicorn --reload`). The host needs **only Docker + Docker Compose** — no local Node, Python, or uv.
- **Multi-stage Dockerfiles**: `web` (deps → build → slim runtime via Next.js `output: "standalone"`) and `api` (uv install → `python:3.12-slim` runtime). Both run as a **non-root** user with pinned base-image digests; `.dockerignore` keeps images lean.
- **Dev/prod parity**: the same Dockerfiles produce dev and prod images (compose `target:` / build args switch behaviour); the prod image is exactly what ships in Phase 13.
- A **`.devcontainer/`** (VS Code Dev Containers) provides a one-click, fully-provisioned in-container IDE that matches CI.
- Images are **scanned in CI** (trivy/grype); **secrets are injected at runtime via env**, never baked into a layer; healthchecks on every container.

---

## 5. Repository / folder structure (create in Phase 1, grow thereafter)

```
talentmesh/
├── agent.md                      # this runbook
├── PRD.md  TalentMesh_Design.md  # product + design source of truth
├── README.md  Makefile  turbo.json  pnpm-workspace.yaml  .pre-commit-config.yaml
├── .github/workflows/ci.yml      # lint, type-check, test, build (all packages)
├── .env.example                  # every env var documented
├── .devcontainer/devcontainer.json   # one-click in-container IDE (matches CI)
├── .dockerignore                 # lean images
├── docker-compose.yml            # FULL stack: postgres, redis, chroma, mock-a2a, api, web
├── docker-compose.override.yml   # dev: bind-mount source + hot reload (web & api)
│
├── apps/
│   └── web/                      # Next.js recruiter app (+ candidate chat surface)
│       ├── app/                  # App Router routes
│       │   ├── (auth)/login/
│       │   ├── (recruiter)/pipeline/  intake/  approvals/  sourcing/
│       │   │                 analytics/  compliance/  candidates/[id]/
│       │   ├── (candidate)/chat/      # public candidate surface
│       │   └── api/                   # route handlers / BFF proxy + SSE relay
│       ├── components/  (ui/ kanban/ approvals/ candidate360/ feed/ charts/)
│       ├── lib/         (api-client/ sse/ ws/ auth/ rbac/ zod-schemas/)
│       ├── stores/      # Zustand
│       ├── hooks/  styles/  tests/  e2e/  .storybook/
│       └── package.json  tsconfig.json  tailwind.config.ts  components.json  Dockerfile  .dockerignore
│
├── services/
│   └── api/                      # FastAPI + LangGraph + ADK backend
│       ├── talentmesh/
│       │   ├── main.py           # FastAPI app, routers, SSE/WS, lifespan
│       │   ├── config.py         # pydantic-settings
│       │   ├── api/              # routers: reqs, candidates, approvals,
│       │   │                     #          sourcing, analytics, compliance, chat, stream
│       │   ├── graph/            # LangGraph: state.py, build.py, nodes/, edges/, checkpointer.py
│       │   ├── agents/           # ADK agents: atlas, scribe, scout, sift, gauge,
│       │   │                     #             echo, sync, aegis, verify, lens, bridge
│       │   ├── models/           # model_for(role) resolver + prompts/
│       │   ├── rag/              # retriever port, chroma_store.py, embeddings.py, ingest.py
│       │   ├── a2a/              # client.py (signed-card verify, version negotiation), callbacks
│       │   ├── mcp/              # MCP tool clients (linkedin, ats, calendar)
│       │   ├── db/               # SQLAlchemy models, session, alembic/
│       │   ├── compliance/       # aegis gates: pii_gate.py, bias_scan.py, impact_ratio.py, reasoning_log.py
│       │   ├── streaming/        # redis pub/sub → SSE bridge, event types
│       │   └── seed/             # load jobs/candidates/etc. into DB + Chroma
│       ├── tests/                # pytest (unit + integration)
│       ├── pyproject.toml  alembic.ini  Dockerfile
│
├── packages/
│   └── shared/                   # generated OpenAPI types, SSE event enum, shared Zod
│
├── mock-a2a/                     # locally runnable mock vendor agents (from external_agent_cards.json)
│   ├── compose.yml  Dockerfile  registry/.well-known/agents
│   └── agents/ (calendar/ bgcheck/ reference/ hris/ itprov/ … signed cards, /healthz)
│
├── infra/                        # IaC, deploy manifests (Cloud Run / GKE / LangGraph Platform)
└── data/ -> (jobs.json … live at repo root or symlinked here for seeding)
```

---

# PHASES

> Each phase below: **Objective → Pre-flight (re-read PRD) → Tasks (with per-task review) → Consolidated Phase Review → Exit Gate**. The per-task review and the consolidated review both follow §2; the exit gate follows §3.2. They are not repeated verbatim each time — they are mandatory every time.

---

## Phase 0 — Prerequisites & Environment

**Objective.** A reproducible developer environment with every account, key, runtime, and CLI present, before any product code.

**Pre-flight.** Read PRD §11–§12 (tech constraints), design §11 (stack). Restate the runtime/version targets.

**Tasks**
- **T-0.1 — Docker-first runtime.** The **only required host tooling is Docker + Docker Compose** (and Git). All language runtimes — Node 20 LTS + **pnpm**, Python **3.12+** + **uv**, **Turborepo** — live **inside containers** and are pinned in the Dockerfiles / `.tool-versions` (consumed by the dev container), not installed on the host. Provide a **`.devcontainer/`** so the IDE runs in the same image as CI. Document versions and the "Docker is all you need" setup in `README.md`.
- **T-0.2 — Accounts & API keys** 🔒. Provision and store (in a secret manager / local `.env`, never committed): **Anthropic API key** (Opus/Sonnet/Haiku access), **Voyage** embeddings key (or choose local `bge-large`), **LangSmith** key, OAuth/OIDC app creds (Google Workspace / Okta / Entra) for Auth.js. Create `.env.example` listing every variable with comments.
- **T-0.3 — Cloud/deploy targets (decide & document).** Choose deploy target(s): Cloud Run / GKE / LangGraph Platform / Vertex Agent Engine. Record the decision; defer actual provisioning to Phase 13.
- **T-0.4 — Repo bootstrap.** `git init`, license, `.gitignore` (node, python, env, chroma data), `.editorconfig`, branch protection plan, Conventional Commits + commitlint.
- **T-0.5 — `make doctor`.** A script that verifies **Docker + Compose are installed and runnable**, the stack images build, and required env vars are present — printing a clear ✅/❌ table.

**Per-task review (§2)** focus: secret hygiene (no keys committed), version pinning, reproducibility on a clean machine.

**Consolidated Phase Review.** Fresh-clone simulation: from an empty dir, follow the README and confirm `make doctor` passes.

**Exit Gate.** `make doctor` green; `.env.example` complete; no secrets in git history (gitleaks clean).

---

## Phase 1 — Monorepo Scaffolding, Tooling & Shared Contracts

**Objective.** The empty-but-wired monorepo: web app, api service, shared package, lint/format/type-check/test/CI all runnable and green on a hello-world.

**Pre-flight.** Re-read PRD §6 (surface map), §12.8 (contract between TS↔Python). Re-read §4–§5 of this doc.

**Tasks**
- **T-1.1 — Workspace.** `pnpm-workspace.yaml`, `turbo.json`, root `Makefile` with `dev/test/lint/typecheck/build/doctor` targets.
- **T-1.2 — `apps/web` skeleton.** Next.js (App Router, TS strict), Tailwind, shadcn/ui init, ESLint flat + Prettier, Vitest + Testing Library + Playwright + Storybook, axe in CI. One placeholder route + one tested component.
- **T-1.3 — `services/api` skeleton.** FastAPI app with `config.py` (pydantic-settings), health endpoint, ruff + mypy strict + pytest wired, `Dockerfile`. One tested route.
- **T-1.4 — Shared contracts (`packages/shared`).** Define the **SSE event union** (`agent.thinking|agent.token|stage.changed|approval.required|task.completed`) and the **OpenAPI→TS type-gen** pipeline (`openapi-typescript`/`orval`). Wire `make generate-types`.
- **T-1.5 — CI.** GitHub Actions: install, lint, type-check, unit tests, build — for all packages. Required to pass.
- **T-1.6 — Pre-commit.** ruff/mypy/eslint/prettier/gitleaks hooks.
- **T-1.7 — Dockerized dev environment.** Multi-stage `Dockerfile` for `apps/web` and `services/api`; `docker-compose.yml` + `docker-compose.override.yml` wiring **hot-reload dev in containers** (bind-mounted source, dev build target); `.devcontainer/devcontainer.json`; `.dockerignore`. `make dev` brings up **web + api as containers** and proves Fast Refresh / `uvicorn --reload` work in-container. **No language runtime is required on the host** — prove it on a host with only Docker installed.

**Per-task review** focus: TS↔Python contract really is single-source (regenerate types and diff = no drift), strict configs actually strict, CI fails on a deliberately broken test (prove the gate works).

**Consolidated Phase Review.** Build the whole monorepo from clean **on a Docker-only host**; verify hot-reload dev (`make dev`) brings up web + api **as containers**; deliberately break a type and confirm CI red.

**Exit Gate.** `make build/test/lint/typecheck` all green; CI green; type-gen produces no diff on re-run.

---

## Phase 2 — Local Infrastructure & Configuration

**Objective.** One command (`docker compose up`) brings up the **entire dockerized stack** — Postgres, Redis, Chroma, the mock-A2A registry, **and the `api` + `web` containers** — locally, all wired together and health-gated.

**Pre-flight.** Re-read PRD §12.1–§12.4 (real-time, reliability, security), design §6 (architecture), §9 (A2A).

**Tasks**
- **T-2.1 — `docker-compose.yml`.** Services: `postgres`, `redis`, `chroma`, `mock-a2a` (registry + vendor agents), plus `api` and `web` profiles. Healthchecks on each; named volumes for Postgres & Chroma persistence.
- **T-2.2 — Config & secrets** 🔒. `config.py` reads DB/Redis/Chroma URLs, Anthropic/Voyage/LangSmith keys, OAuth creds, A2A registry URL (local vs prod), tenant id. Fail-fast on missing required vars.
- **T-2.3 — Connection health.** `/healthz` (liveness) and `/readyz` (checks Postgres, Redis, Chroma, A2A registry reachability). Wire into `make doctor`.
- **T-2.4 — Local A2A registry stub.** Stand up `mock-a2a/registry/.well-known/agents` returning the cards from `external_agent_cards.json` with their **local** URLs/ports. (Full mock skills land in Phase 7; here just discovery + `/healthz`.)

**Per-task review** focus 🔒: no plaintext secrets in compose; healthchecks actually gate `depends_on`; readiness genuinely probes dependencies; volumes persist across restarts (durability prereq).

**Consolidated Phase Review.** `docker compose up` cold-start to all-green; kill a container and confirm `/readyz` flips and recovers.

**Exit Gate.** All services healthy from cold start; `/readyz` green; persistence verified across `down/up`.

---

## Phase 3 — Data Layer, Persistence & RAG Ingestion

**Objective.** Seed the relational store and the five Chroma collections from the JSON files; expose a typed repository layer and a `Retriever` port.

**Pre-flight.** Re-read PRD §13 (data model & grounding) and design §7 (RAG), §14 (synthetic data). Skim `jobs.json`, `candidates.json`, `leveling_comp.json`, `company_knowledge.json`, `compliance_rules.json`, `interview_rubrics.json`.

**Tasks**
- **T-3.1 — SQLAlchemy models + Alembic.** Tables: `requisitions`, `candidates`, `applications`, `comp_bands`, `compliance_rules`, `interview_rubrics`, `reasoning_log` (append-only), `approvals`, `agent_runs`, `tenants`. Tenant-scope every table 🔒. First migration.
- **T-3.2 — Seed loader.** `seed/` loads all JSON into the DB **idempotently** (upsert by id), preserving the canonical demo records (REQ-2026-0431/0455/0478/0490; CAND-90105 Amara, CAND-90120 Wei, etc.). Validate referential integrity (every `applied_to` → real req; every `comp_band_ref` → real band).
- **T-3.3 — Repository layer.** Typed async repositories with pagination (must handle 1000 candidates / 100 reqs without loading all). No raw SQL string-building 🔒.
- **T-3.4 — RAG `Retriever` port + Chroma store.** Abstract retriever interface; Chroma implementation; **embeddings** via Voyage (`voyage-3`) or local `bge-large`. Chunk (~300–500 tokens) with metadata (`req_id`, `level`, `location`, `clearance`, `category`, `doc_id`).
- **T-3.5 — Collection ingestion.** Build the five collections: `job_knowledge`, `candidate_corpus`, `company_knowledge`, `compliance_kb`, `interview_kb`. **Metadata-filter-first** retrieval (e.g. `clearance=Secret` before semantic rank — verify Helena Brandt surfaces, clearance-less profiles excluded).
- **T-3.6 — RAG smoke tests.** Query each collection; assert grounded, cited results (e.g. company_knowledge answers a benefits/visa question; candidate_corpus respects the clearance filter).

**Per-task review** focus 🔒⚖️: append-only enforcement on `reasoning_log` (no UPDATE/DELETE path); tenant isolation on every query; metadata filter applied **before** similarity, not after; idempotent seeding; pagination correctness; embeddings model swappable behind the port.

**Consolidated Phase Review.** Re-run seeding twice (idempotent, no dupes); verify counts (100 reqs / 1000 candidates / 300+ KB docs); run all RAG smoke tests; confirm clearance filter and citation behavior.

**Exit Gate.** Migrations apply clean; seed idempotent with integrity checks green; all five collections queryable with correct metadata filtering; repository pagination tested.

---

## Phase 4 — LangGraph Core: State, Checkpointer & Streaming

**Objective.** A minimal but real durable graph with shared state, Postgres checkpointing, conditional branching, HITL interrupt, and streaming that reaches the browser as SSE.

**Pre-flight.** Re-read PRD §7 (workflow), §12.1/§12.3 (real-time, durability), design §13.1 (state + branch + HITL code skeleton).

**Tasks**
- **T-4.1 — `RecruitingState`.** TypedDict with `req_id, candidate_id, stage, screen_score, assessment_score, reasoning_log (append-only via reducer), messages`. Document every field.
- **T-4.2 — Checkpointer.** `PostgresSaver` wired; thread id = `REQ:CAND`. Prove **resume-after-crash**: start a run, kill the process, resume from last checkpoint with no lost `reasoning_log` entries.
- **T-4.3 — 3-node skeleton graph.** `screen → route_after_screen → {advance | human_review | reject}` with the **borderline = threshold−0.10** band exactly (Wei 0.63 vs 0.70 → human_review; Marcus Webb 0.58 → reject path but **not** executed without human). Conditional edges per design §13.1.
- **T-4.4 — HITL interrupt.** `human_review` node calls `interrupt()`; expose a resume endpoint that injects the human decision and continues. ⚖️ No path advances/rejects without it.
- **T-4.5 — Streaming bridge.** Graph `stream()` events → Redis pub/sub → FastAPI **SSE** endpoint emitting the typed event union. First token < 1s (PRD §12.1).
- **T-4.6 — SSE endpoint + reconnect.** `/stream/{thread}` SSE with heartbeat and resumable cursor; tolerate disconnect/reconnect without losing state (server keeps running).

**Per-task review** focus ⚖️: there is **no** edge from screen→advance/reject that bypasses `human_review` for consequential moves; reducer makes `reasoning_log` truly append-only; checkpointer resume is real (test by killing mid-run); SSE event names exactly match the shared enum; back-pressure handled.

**Consolidated Phase Review.** End-to-end: drive a candidate through screen→interrupt→resume; verify reasoning_log entries, SSE event stream shape, and crash-resume. Run with all three score archetypes (advance/borderline/reject).

**Exit Gate.** Durable resume proven; HITL interrupt blocks consequential transitions; SSE streams typed events to a test client; branch logic matches thresholds exactly.

---

## Phase 5 — Core Funnel Agents + HITL Approvals

**Objective.** The working funnel: Scribe (intake/JD), Scout (parallel sourcing), Sift (sequential screening), Echo (engagement), with model tiering and the Approvals Inbox backend.

**Pre-flight.** Re-read PRD §7.2 (stage-by-stage), §8.3/§8.4 (Intake, Approvals), §9.1–§9.5 (capabilities), design §4.1 (agent detail), §13.2 (ADK skeletons).

**Tasks**
- **T-5.1 — `model_for(role)` resolver + prompt registry.** Central mapping (Atlas/Gauge/Aegis→Opus; Scribe/Scout/Sift-score/Echo-chat/Lens/Verify-summarize→Sonnet; Sift-parse/Echo-notify/Sync/Verify-orchestrate/Bridge→Haiku). Prompts versioned in `prompts/`.
- **T-5.2 — Scribe (Intake & JD).** ADK LLM agent; RAG over `job_knowledge` (leveling + comp band + past JDs + inclusive rules); emits JD **artifact**; produces an `approval.required` (JD). ⚖️ Aegis bias scan must pass before the draft surfaces (full Aegis in Phase 6 — here a stub gate + interface).
- **T-5.3 — Scout (Sourcing, ParallelAgent).** Fan out to internal `candidate_corpus` (vector + metadata filter), a LinkedIn **MCP** tool (mock acceptable now), and referrals; de-dupe/merge. Surfaces results into "Sourced". Clearance metadata filter for REQ-2026-0490.
- **T-5.4 — Sift (Screening, SequentialAgent).** `parse (Haiku) → extract (Haiku) → score (Sonnet) → rank`; output ranked sub-scores; feed the Phase-4 conditional edge. ⚖️ Writes a per-candidate `reasoning_log` entry; never auto-rejects.
- **T-5.5 — Echo (Engagement).** Sonnet chat + Haiku notifications; RAG over `company_knowledge` with streaming; long-term memory (interaction history) per candidate; **nurture LoopAgent** for quiet/below-threshold (Jordan Lee 90-day). ⚖️ Surfaces the LL144 notice + alternative-process option (COMP-LL144-02).
- **T-5.6 — Approvals API + Inbox backend.** Endpoints to list/inspect/resolve approvals of types **JD / shortlist / borderline / offer / advance**; each carries recommendation + rationale + reasoning-log excerpt; resolve → resume graph + emit `stage.changed`. ⚖️ No auto-decision anywhere.
- **T-5.7 — REST commands.** `POST /reqs/{id}/jd/approve`, `/shortlist/approve`, `/candidates/{id}/advance`, etc. — all resume paused threads.

**Per-task review** focus ⚖️🔒: every advance/reject/offer is human-gated; model tiering correct per role (cost lever); Scout truly parallel (not serial); Sift sequential order correct; Echo never sends a candidate message without the notice; RBAC on approval endpoints; rationale + reasoning-log present on every approval item.

**Consolidated Phase Review.** Run REQ-2026-0431 end-to-end through intake→sourcing→screening→engagement with Amara (0.91 advance), Wei-style borderline (→ inbox), Marcus Webb (0.58 → reject path, human-confirmed). Verify Approvals Inbox contents, model assignments (via traces), and no auto-decisions.

**Exit Gate.** Full core funnel runs on seed data; Approvals Inbox shows all five item types with rationale; PRD §14 acceptance for Intake/Approvals/Screening/Engagement pass; model tiering verified in LangSmith.

---

## Phase 6 — Governance: Gauge, Aegis, Reasoning Log & Time-Travel

**Objective.** Assessment loop + the compliance backbone: Aegis guardrails, reasoning log, PII segmentation, bias scan, impact-ratio math, and time-travel replay.

**Pre-flight.** Re-read PRD §8.4/§8.8 (Approvals, Compliance/Audit), §9.11 (governance), §13.4 (compliance rules), §17 of the design doc (compliance backbone), `compliance_rules.json`, `interview_rubrics.json`.

**Tasks**
- **T-6.1 — Gauge (Assessment, LoopAgent, Opus).** Generate/score assessments against `interview_kb` rubrics (RUB-ENG-SYSDESIGN etc.); `max_iterations=3` loop until rubric satisfied; support live/bidirectional streaming hooks. Weighted scoring matches rubric weights exactly.
- **T-6.2 — Aegis PII gate** 🔒⚖️ (Custom/deterministic). Architectural segmentation: demographic data physically cannot reach scoring/ranking agents (COMP-PII-01). Prove with a test that scoring inputs exclude protected attributes.
- **T-6.3 — Aegis bias scan** ⚖️ (Opus + rules from `compliance_kb`). Scan JDs and outbound messages for biased/exclusionary language (COMP-BIAS-01); block on violation; one-click fixes surfaced to UI.
- **T-6.4 — Aegis human-decision boundary** ⚖️. Wrap every consequential edge: assert no auto-reject/auto-hire system-wide (COMP-EUAI-02). Add a test that fails if any new edge bypasses the gate.
- **T-6.5 — Reasoning log** ⚖️. Append-only, model+timestamp-stamped entries for every ranking/advancement/rejection (COMP-EUAI-01); reviewer-overridable; exportable.
- **T-6.6 — Impact-ratio / 4/5ths math** ⚖️ (Lens-supported, Aegis-owned). Selection rates + impact ratios across protected groups (COMP-LL144-01); flag < 0.80. Deterministic, unit-tested math.
- **T-6.7 — Time-travel / replay.** Reconstruct any candidate's decision path from checkpoints; expose a replay API the Compliance screen will use.
- **T-6.8 — EEOC job-relatedness + IL VIA consent** ⚖️. Flag must-haves that could disparately filter (COMP-EEOC-01); capture video-AI consent (COMP-ILVIA-01) before any video analysis.

**Per-task review** focus ⚖️🔒: PII gate is architectural (not advisory); bias scan blocks (not warns); no edge bypasses Aegis; reasoning log immutable + complete; impact-ratio math correct against a hand-computed fixture; replay reproduces exact state. This is a **high-risk phase → run an independent `/security-review` + `/code-review`** second pass.

**Consolidated Phase Review.** Drive a cohort through the funnel; export a reasoning log; replay Wei Chen's borderline decision; compute impact ratios and confirm the 0.74-style flag triggers review; attempt (and fail) to make scoring see a demographic attribute.

**Exit Gate.** All compliance rules (`compliance_rules.json`) have an enforcing code path with a test; PII gate proven; reasoning log + replay working; PRD §14 Compliance/Analytics acceptance pass; independent review findings closed.

---

## Phase 7 — External World via A2A: Sync, Verify, Bridge + Mock Vendors

**Objective.** Cross-vendor A2A: scheduling, checks, and onboarding handoff against **locally runnable** mock vendor agents, with signed-card verification, version negotiation, and push.

**Pre-flight.** Re-read PRD §7.2 (scheduling/checks/handoff), §9.7–§9.10, §13.3, design §9 (A2A detail), `external_agent_cards.json` (note the local URLs/ports/mock commands).

**Tasks**
- **T-7.1 — Mock vendor agents (`mock-a2a/`).** Implement each vendor from `external_agent_cards.json` as a small local A2A server with its declared skills, **signed card** at `/.well-known/agent.json`, `/healthz`, and the local port. Include the expanded set (calendar, bgcheck, reference, hris, itprov, sourcing, video-proctor, coding-sandbox, identity, e-sign, comms, etc.). `docker compose -f mock-a2a/compose.yml up`.
- **T-7.2 — A2A client** 🔒. Fetch card → **verify signature** against `signature_domain` → auth per scheme (OAuth2 scopes / api_key / mtls) → open task. SSRF-safe URL allow-list. Local vs prod registry switch.
- **T-7.3 — Sync (Scheduling, Haiku).** `find_mutual_slots → book_interview` for multi-party (Amara's 5-interviewer onsite); **push** callback → `task.completed` → SSE toast. Reschedule path.
- **T-7.4 — Verify (Checks, ParallelAgent).** Run `BackgroundCheckProvider` + `ReferenceCheckAgent` **concurrently** with candidate **consent** ⚖️; **version negotiation** down to v0.3 for the reference vendor (compatibility shim); long-running + push for bg-check; **poll** where push unsupported; Sonnet summarizes reports. Offer gated on result.
- **T-7.5 — Bridge (Onboarding handoff, Parallel).** On offer accept, fire `HRISOnboarding.create_employee` + `ITProvisioning.provision_by_role` in parallel; push status back.
- **T-7.6 — Push callbacks + idempotency.** `/a2a/callbacks/{task}` endpoint; verify, de-duplicate, and fan to SSE. Retries/timeouts/backoff on all vendor calls.

**Per-task review** focus 🔒: signature verification cannot be skipped; failed/timed-out tasks surface and recover (PRD §11); version-negotiation fallback actually works; consent required before checks; callbacks idempotent and authenticated; no SSRF.

**Consolidated Phase Review.** Cold-start the mock-a2a stack; run scheduling, parallel checks (incl. v0.3 negotiation + a forced timeout), and onboarding handoff for Amara/Helena; verify signed-card rejection of a tampered card; confirm offer stays gated while a check is `in_progress`.

**Exit Gate.** All A2A flows green against local mocks; signed-card verify + version negotiation + push/poll proven; failure/timeout recovery tested; PRD §14 Scheduling/Checks/Onboarding acceptance pass.

---

## Phase 8 — Analytics, Atlas Routing & Evaluation

**Objective.** Lens analytics + Atlas dynamic routing (fallback + A/B) + LangSmith evaluation harness.

**Pre-flight.** Re-read PRD §8.9 (Analytics), §9.12, design §4.1 (Atlas, Lens), §10 (model strategy), §5.1 (LangSmith).

**Tasks**
- **T-8.1 — Lens metrics.** Funnel conversion, time-to-stage, source effectiveness, recruiter-hours-saved — as artifacts/SQL over the app DB; correct against seed data.
- **T-8.2 — What-if fairness simulator.** Re-compute impact ratio + yield for a proposed threshold before applying; applying routes through an approval gate ⚖️.
- **T-8.3 — Atlas routing.** Dynamic coordinator: **fallback** sourcing strategy when yield is low; **A/B** route of two sourcing strategies with yield comparison feeding back.
- **T-8.4 — LangSmith eval harness.** Trace all agent trajectories; an eval suite scoring decision quality + model-tier A/B; gate regressions before deploy.

**Per-task review** focus: metric math correct & paginated for scale; A/B routing deterministic + logged; eval harness catches a deliberately degraded prompt.

**Consolidated Phase Review.** Generate analytics for REQ-2026-0431; run the what-if at 0.72→0.68 and confirm impact-ratio movement; run an A/B sourcing comparison; run the eval suite and confirm a seeded regression is flagged.

**Exit Gate.** Analytics correct vs seed; Atlas fallback + A/B working and logged; LangSmith traces + eval green; PRD §14 Analytics acceptance pass.

---

## Phase 9 — Frontend Foundation & Design System

**Objective.** The app shell, auth, RBAC-aware nav, design system, and the real-time client layer — the chrome every screen shares (PRD §6, §8.1).

**Pre-flight.** Re-read PRD §6 (surface map), §8.1 (shell/login/home), §10 (roles), §12 (NFRs incl. a11y).

**Tasks**
- **T-9.1 — App shell & chrome.** Wordmark, global command/search bar, notifications bell, six-tab nav, Agent Activity rail, approvals badge — consistent across screens (match the PRD §8 wireframes).
- **T-9.2 — Auth.js OIDC + role selection** 🔒. Login screen (Google/Okta/Entra), session, role claim (Recruiter/HM/Compliance/Admin) read on every call.
- **T-9.3 — RBAC in the UI** 🔒. Route/component guards reflecting the PRD §10 matrix (server-enforced too); hide/disable unauthorized actions.
- **T-9.4 — API client + generated types.** Typed client from the backend OpenAPI; TanStack Query setup; error/loading conventions.
- **T-9.5 — SSE + WebSocket hooks.** `useEventStream` (typed event union, auto-reconnect, resumable cursor) and `useCandidateChat` (WS). ♿ announce feed events via ARIA live region.
- **T-9.6 — Design system (shadcn/ui).** Build the design system on **[shadcn/ui](https://ui.shadcn.com)** — initialize with the shadcn CLI (`components.json`), install the components the screens need (button, card, dialog, dropdown-menu, tabs, table, badge, sonner/toast, command, sheet, tooltip, skeleton, avatar, scroll-area, etc.), theme via CSS variables. Compose kanban primitives (dnd-kit), score meters, charts, and status glyphs as shadcn-based components. Storybook stories incl. empty/loading/error states. **All later screens compose only shadcn/ui primitives** for visual consistency.
- **T-9.7 — Workspace home.** The recruiter landing (PRD §8.1.2): my reqs (◆mine/▸team ownership per data), agenda, KPIs, feed preview.

**Per-task review** focus 🔒♿: RBAC enforced server-side too (not just hidden in UI); SSE reconnect loses nothing; live-region announces without spamming; types regenerate with no drift; keyboard nav across shell.

**Consolidated Phase Review.** Log in as each role; confirm nav/actions match the RBAC matrix; kill/restore the SSE connection and confirm re-sync; run axe on the shell (0 serious violations).

**Exit Gate.** Shell + auth + RBAC + real-time clients working against the live backend; Storybook covers states; a11y clean on the shell; PRD §8.1 acceptance pass.

---

## Phase 10 — Frontend Screens (Mission-Control Suite)

**Objective.** Build every recruiter screen from PRD §8 — composed **entirely from [shadcn/ui](https://ui.shadcn.com) components** for visual consistency — wired to the backend with live updates.

**Pre-flight.** Re-read **all of PRD §8** (8.2–8.10) and §9 (capability catalog) — match layouts, data, and actions to the wireframes. Build every screen from the shadcn/ui primitives set up in Phase 9 (no ad-hoc/bespoke components).

**Tasks** (one per screen; each: build → per-task review → fix → test)
- **T-10.1 — Pipeline / Mission Control (§8.2).** Live kanban (Sourced→Offer), candidate cards (score, glyphs, aging, ★ recommended, ✗ below-threshold), right-rail feed, cross-req roster, pending-approvals strip. `stage.changed` moves cards; drag gated by approval.
- **T-10.2 — Intake & JD Builder (§8.3).** Two-pane: Scribe chat (streaming) + side-by-side JD editor; Aegis inclusive-language banner with one-click fixes; **Approve & Open Req** resolves the interrupt.
- **T-10.3 — Approvals Inbox (§8.4).** Mixed-type queue (JD/shortlist/borderline/offer/advance); detail pane with recommendation + rationale + sub-scores + reasoning-log excerpt + replay link; Approve/Override/Request-changes; ⛔ no-auto banner.
- **T-10.4 — Sourcing / Candidate Search (§8.5).** Semantic box + metadata filter chips (clearance=Secret…), ranked results with source + match, A/B yield panel, channel effectiveness, dedupe, add-to-pipeline.
- **T-10.5 — Candidate 360 (§8.6).** Identity header + recommendation; score meters vs threshold; resume/skills; assessment vs rubric dimensions; interaction-history timeline; scheduling + bg panels; compliance flags + reasoning-log link; message/nudge actions.
- **T-10.6 — Engagement / Echo monitor (§8.7).** Conversation list (channel glyphs, ghost-risk, nurture) + streaming thread with RAG citations; UX-negotiation escalate-to-video; AI-assist notice; take-over/send-on-behalf/nurture.
- **T-10.7 — Command bar / copilot + notifications (§8.10).** Cmd-K NL queries routed to Atlas; toast stack from A2A `task.completed` + `approval.required`.

**Per-task review** focus ⚖️♿: drag cannot bypass approval; Approvals UI never exposes an auto-decision; every screen handles empty/loading/error states (PRD §11); live events update without refresh; keyboard + screen-reader paths; data matches seed (real names/scores/reqs).

**Consolidated Phase Review.** Full click-through of the recruiter day-in-the-life (PRD §7.1): clear the inbox, watch Sift stream, nudge a quiet candidate, observe a push toast. Run Playwright e2e across all screens; axe on each.

**Exit Gate.** All §8 screens functional, live, and a11y-clean; e2e green; PRD §9 capabilities demonstrably available; visual parity with wireframes.

---

## Phase 11 — Compliance UI, RBAC Surfaces & Candidate-Facing App

**Objective.** The oversight surface (Dana) and the public candidate chat, completing the compliance story end-to-end.

**Pre-flight.** Re-read PRD §8.8 (Compliance & Audit), §10 (roles), §12.7 candidate surface in design §12.7, §13.4, the compliance backbone (design §17).

**Tasks**
- **T-11.1 — Compliance & Audit screen (§8.8).** Reasoning-log viewer (model+ts), time-travel replay scrubber over checkpoints, bias-audit dashboard (selection rates + impact ratios + 0.80 line), notice + PII indicators, LL144 export. Compliance-role gated.
- **T-11.2 — Analytics screen (§8.9).** Funnel, time-to-stage, source effectiveness, hours-saved, what-if fairness slider (Apply → approval gate).
- **T-11.3 — Candidate chat surface.** Public app: 24/7 Echo chat (streamed, channel-aware), **AI-assists/human-decides notice**, one-click **request-an-alternative-process** (LL144), status lookup.
- **T-11.4 — RBAC end-to-end audit** 🔒. Verify the full PRD §10 matrix server- and client-side, including delegation logging (delegator/delegate/scope/time → reasoning_log).

**Per-task review** focus ⚖️🔒: replay reproduces exact state; export is complete & regulator-ready; candidate notice + alternative path always reachable; Compliance role cannot edit pipeline / advance candidates (auditor independence); delegation attributed to the human who clicked.

**Consolidated Phase Review.** As Dana: replay a decision, export an LL144 audit, run the what-if, confirm no edit access. As a candidate: chat, see the notice, request an alternative.

**Exit Gate.** Compliance + candidate surfaces complete; RBAC matrix verified both sides; PRD §14 Compliance acceptance pass; LL144 export validated.

---

## Phase 12 — Quality Hardening: Testing, Security, Accessibility, Performance, Observability

**Objective.** Raise the whole system to production quality across all NFRs (PRD §12).

**Pre-flight.** Re-read PRD §12 (all NFRs) and §11 (edge cases).

**Tasks**
- **T-12.1 — Test depth.** Backend unit+integration to a coverage target; frontend unit + Playwright e2e covering every PRD §11 edge case (borderline, below-threshold, vendor timeout, version negotiation, clearance exclusion, resume-after-crash, SLA breach, offline reconnect, alternative-process).
- **T-12.2 — Security review** 🔒. Run `/security-review`; authZ on every endpoint; secret-scan; dependency audit; SSRF on A2A/MCP; signed-card + scope checks; multi-tenant isolation tests; rate limiting.
- **T-12.3 — Accessibility** ♿. WCAG 2.1 AA across all screens; keyboard-only run; screen-reader pass on the agent feed; axe in CI gating.
- **T-12.4 — Performance & load.** p95 page load < 2.5s; SSE first token < 1s; the 200+/week high-volume path (REQ-2026-0478) and 12–18 concurrent reqs don't degrade; query plans checked (no N+1); pagination everywhere.
- **T-12.5 — Observability.** Structured logs + OTel traces + LangSmith dashboards; error budgets; health/readiness; alerting hooks.
- **T-12.6 — Resilience drills.** Kill Postgres/Redis/Chroma/a vendor mid-flow; confirm durable resume and graceful degradation.

**Per-task review** focus: the edge cases actually fail correctly and recover; no endpoint unauthenticated; a11y violations = 0 serious; load targets met with evidence (numbers, not vibes).

**Consolidated Phase Review.** Full regression of Phases 3–11; security + a11y + perf reports attached; resilience drills pass.

**Exit Gate.** All NFR targets met with evidence; security/a11y/perf gates green in CI; zero open blockers/majors across the whole system.

---

## Phase 13 — CI/CD, Deployment & Acceptance Validation

**Objective.** Ship it: containerized deploy, environments, and a full pass over the PRD §14 acceptance criteria.

**Pre-flight.** Re-read PRD §14 (acceptance criteria — every item) and §12 NFRs; design §11 deployment, §15 roadmap.

**Tasks**
- **T-13.1 — Containerization & IaC.** Reuse the **same multi-stage Dockerfiles** (prod target) used in dev — **dev/prod image parity**, non-root, pinned digests, CI-scanned — and deploy to the chosen target (Cloud Run / GKE / LangGraph Platform / Vertex Agent Engine); object storage for artifacts, managed Postgres/Redis/vector store, secrets via secret manager.
- **T-13.2 — CD pipeline.** Build → test → scan → deploy to staging → smoke → promote to prod; migrations run safely; rollback path.
- **T-13.3 — Multi-tenancy & config** 🔒. Per-tenant isolation verified in a deployed env; env-specific config; A2A registry points at real or mock vendors per env.
- **T-13.4 — Full acceptance pass.** Execute **every** PRD §14 checkbox against the deployed system on seed data; record pass/fail with evidence.
- **T-13.5 — Demo script & docs.** README, runbooks, the recruiter day-in-the-life demo (PRD §7.1), and an operator guide.
- **T-13.6 — Roadmap handoff.** Capture PRD §15 future extensions as backlog (internal mobility, voice screening, offer modeling, ghost-risk monitor, A2A-server inbound, localization).

**Per-task review** focus 🔒: no secrets in images; migrations reversible; staging≈prod; rollback tested; every acceptance item genuinely verified (not assumed).

**Consolidated Phase Review.** Fresh deploy to staging from scratch following only the docs; run the full §14 acceptance suite; security re-scan of the deployed surface.

**Exit Gate.** Deployed, smoke-green, **100% of PRD §14 acceptance criteria pass** with evidence; rollback proven; docs complete. **Ship.**

---

## 14. Cross-cutting requirement → phase traceability

| Requirement (PRD) | Primary phase(s) | Verified again in |
|---|---|---|
| Human-decision boundary, no auto-reject/hire (COMP-EUAI-02) ⚖️ | 4, 5, 6 | 10, 11, 12, 13 |
| Reasoning log + time-travel (COMP-EUAI-01) ⚖️ | 4, 6 | 11, 13 |
| PII segmentation (COMP-PII-01) 🔒⚖️ | 3, 6 | 12 |
| Bias scan on JD/messages (COMP-BIAS-01) ⚖️ | 5, 6 | 10 |
| LL144 impact ratios + notice + alternative (COMP-LL144-01/02) ⚖️ | 6, 8, 11 | 13 |
| IL VIA video consent (COMP-ILVIA-01) ⚖️ | 6, 7 | 11 |
| EEOC job-relatedness (COMP-EEOC-01) ⚖️ | 6 | 12 |
| Durable execution / resume-after-crash | 4 | 7, 12 |
| Signed A2A cards + version negotiation + push 🔒 | 7 | 12 |
| Real-time SSE/WS targets | 4, 9 | 10, 12 |
| RBAC (4 roles) 🔒 | 9, 11 | 12, 13 |
| Model tiering (Opus/Sonnet/Haiku) | 5 | 8 |
| RAG metadata-filter-first (clearance) | 3 | 5, 7 |
| Accessibility WCAG 2.1 AA ♿ | 9, 10 | 11, 12 |
| Performance & scale (1000 cands / 200+wk) | 3, 12 | 13 |

---

## 15. Per-phase logs (templates to fill as you go)

**Defect Log (per task and per consolidated review)**
```
Phase: <n>   Task/Review: <T-/R->   Reviewer-hat: Expert Code Reviewer
[D1] blocker  correctness  graph/edges.py:88  advance edge bypasses human_review for score>=thr
     → Fix: route all advances through human_review interrupt; add edge test
     → Verified: pytest tests/graph/test_no_autodecision.py::test_advance_gated  ✅
[D2] major     security     api/approvals.py:40  resolve endpoint missing RBAC check
     → Fix: add require_role(RECRUITER|HM); server-side enforce
     → Verified: tests/api/test_rbac.py ✅
```

**Decision Log**
```
[DEC-<phase>-<n>] <decision> — <rationale> — <alternatives rejected> — <date>
```

**Phase sign-off**
```
Phase <n> — EXIT GATE: PASS
- Tasks closed: <list>   Open blockers/majors: 0
- Suites green: backend ✅  frontend ✅  e2e ✅  lint/type ✅
- PRD §14 items verified: <list>
- Security/a11y/perf: <evidence links>
```

---

## 16. Risk register (seed; extend as you build)

| Risk | Impact | Mitigation |
|---|---|---|
| A path silently auto-decides (regresses COMP-EUAI-02) | Legal/regulatory | Global "no-auto-decision" test that fails on any unguarded consequential edge; Aegis wraps every edge (Phase 6) |
| PII leaks into scoring | Compliance breach | Architectural segmentation + test that scoring inputs exclude protected attrs (Phase 6) |
| TS↔Python contract drift | Runtime breakage | Generated types only; CI fails on regenerate-diff (Phase 1) |
| A2A vendor unavailable/forged card | Broken/insecure integration | Local mocks + signature verification + retries/timeouts + version negotiation (Phase 7) |
| Scale (1000 cands / 200+wk) regressions | UX/perf | Pagination everywhere + load tests + query-plan review (Phases 3, 12) |
| Model cost/latency creep | $$$ | `model_for(role)` tiering + LangSmith A/B eval (Phases 5, 8) |
| Long-lived runs lose state | Data loss | Postgres checkpointer + resume-after-crash drills (Phases 4, 12) |

---

## 17. Master command reference (wire these in Phase 1)

```
make doctor        # verify Docker/Compose, images build, env, service health
make dev           # docker compose: web + api (+ deps) in containers, hot reload
make up / make down# docker compose: FULL stack (web, api, postgres, redis, chroma, mock-a2a)
make build-images  # build web + api docker images (dev & prod targets)
make shell-api     # exec a shell into the running api container
make seed          # idempotently load JSON into DB + Chroma (runs in a container)
make generate-types# OpenAPI → TS types (single source of truth)
make test          # backend pytest + frontend vitest + playwright
make lint          # ruff + eslint
make typecheck     # mypy --strict + tsc
make a11y          # axe across screens
make review        # /code-review on the current diff (per-task & per-phase)
make security      # /security-review (Phases 6, 7, 12)
```

---

### Final reminder

> **Before every phase: re-read [`PRD.md`](PRD.md).** Implement a task, then put on the
> **Expert Code Reviewer** hat, find and list the bugs, fix them all, and re-test. Review each
> task, then review the whole phase together. **Only a fully green Exit Gate unlocks the next
> phase.** Speed is welcome; skipping the review or the gate is not.
