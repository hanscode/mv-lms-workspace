# CLAUDE.md

## Purpose

This file defines the **mental model, boundaries, and operating rules** for working in this multi-repository workspace.
It is not setup documentation and not deep architecture documentation.

Think of this file as:

> *"How to reason about the system before writing or changing code."*

---

## Workspace Layout

```text
mv-lms/
├── mckinney-vento-lms-api/   # Laravel 12 REST API — magic-link auth, reports, programs
└── mckinney-vento-lms-fe/    # Nuxt 3 SPA — participant UI, admin panel, reports
```

Each repository has a **distinct responsibility and ownership boundary**.
Crossing those boundaries unintentionally is a common source of bugs.

---

## High-Level System Model

- **API** owns all business logic, data persistence, access control, and report aggregation
- **Frontend** owns UI rendering, user interaction, and state management
- **Reports** are pre-computed nightly — the API reads them, never computes them on-demand

This is a **hierarchical multi-client system** with nightly batch jobs.
Always assume:

- Access cascades downward through the geo hierarchy (State → Region → District → School)
- Reports reflect yesterday's data — real-time computation does not happen at request time
- Magic-link auth means no passwords — session management is Sanctum cookie-based

---

## Repo-Specific Rules & Guardrails

### `mckinney-vento-lms-api/` (Laravel API)

**Role**
- REST API at `/api/v1/*`
- Business logic, access control, report aggregation, Excel exports, scheduled jobs

**Rules**
- Controllers stay thin — push logic into `app/Services/*`
- Access control is middleware-based, not Policy-based — match the existing pattern
- Reports are pre-computed — never add real-time computation to report endpoints
- Custom XSRF cookie/header names (`XSRF-TOKEN-MVLMS`) must stay matched — do not normalize to Sanctum defaults
- `User::boot()` auto-detaches geo assignments on user type change — verify side effects when touching users
- Excel exports are queued jobs — they write XLSX, upload to storage, and email the user
- CI runs tests on SQLite — if a test passes locally (MySQL) but fails in CI, suspect MySQL-specific SQL
- Run `./vendor/bin/pint` before committing

**Critical**: Do not change the MySQL version. Production runs `mysql:8.0.45`. Laravel Pulse uses `md5()` in a generated column that breaks on MySQL 8.4+. The pinned version is deliberate.

---

### `mckinney-vento-lms-fe/` (Nuxt 3 SPA)

**Role**
- Web UI for participants, liaisons, coordinators, and admins
- SPA mode (SSR disabled) — no server-rendered pages

**Rules**
- Use `useSanctumFetch` and `useSanctumUser` from `nuxt-auth-sanctum` — not legacy `.vue` composables
- New composables go in `.ts` files — not `.vue` files
- Handle API errors via `composables/useApiError.ts`
- Reach for `components/Admin/ListLayout/` before hand-rolling tables
- Do not refactor legacy backup pages unless explicitly asked
- TypeScript is loose (`strict: false`) — don't over-invest in type coverage for existing code
- `bun` is the preferred package manager (`bun.lockb` is current)

---

## Cross-Cutting Principles (Non-Negotiable)

1. **KISS** — simplest solution that works
2. **DRY** — no duplication without intent
3. **Understand before changing** — read first, then modify
4. **Iterative quality** — make it work, then refine
5. **Avoid over-engineering** — no premature abstractions
6. **Match existing patterns** — consistency over novelty
7. **Narrative flow** — code should read clearly
8. **Framework-first** — use Laravel/Vue/Nuxt features before custom solutions
9. **Failure containment** — errors should be local, not systemic

---

## Working Model

This project must be worked using a **human-in-the-loop approach**.

AI is an assistant, not the decision-maker.

The human is responsible for:

- defining the problem
- validating assumptions
- choosing the implementation approach
- reviewing side effects
- approving production changes
- confirming final behavior

Do NOT behave like an autonomous agent.

Do NOT make product assumptions without evidence from:

- code
- existing behavior
- explicit task context

When uncertain:

1. trace the flow
2. identify which repo owns the behavior
3. propose options
4. avoid guessing

### When Context Is Insufficient

If the flow cannot be traced with available information:

- Stop
- State what is missing
- Ask the human before proceeding

Do NOT proceed with assumptions.

---

## AI Collaboration Rules

### Do

- trace flows before changing code
- identify which repo owns the behavior before making changes
- separate hypothesis vs confirmed cause
- analyze access control and geo-scope implications explicitly
- propose multiple approaches when needed
- favor minimal, safe changes

### Do NOT

- guess behavior
- assume access scope without verifying middleware
- introduce unnecessary abstractions
- change code across repo boundaries casually or without justification
- deploy without human approval
- present guesses as facts

---

## Collaboration Philosophy

→ See `AI_ENGINEERING_WORKFLOW.md` for the full framework.

---

## Skills

Skills are defined in `.claude/skills/`.
Skills run ONLY when explicitly invoked by the user.
Do NOT self-trigger any skill based on perceived task completion.
Claude MAY suggest invoking a skill when appropriate — the human decides.

---

## Task Workflow

1. Problem
2. Context
3. Expected behavior
4. Current behavior
5. Root cause / hypothesis
6. Options
7. Recommendation
8. Validation
9. Side effects
10. Outcome

---

## Task Documentation

- Active tasks: `docs/tasks/`
- Completed tasks: `docs/archive/`

---

## Code Quality Expectations

All code changes must follow the standards defined in:

→ `QUALITY_RUBRIC.md`

Before proposing or applying any change, evaluate against `QUALITY_RUBRIC.md`.
This is mandatory, not optional.

Claude must:

- evaluate code against the rubric
- propose improvements
- avoid low-quality changes

---

## Branching & Deployment

| Branch | Auto-deploys to |
|--------|----------------|
| `staging` | Staging (Forge → DigitalOcean) |
| `production` | Production (Forge → DigitalOcean) |

Feature work branches off `staging`. PRs go back to `staging` (auto-deploys to staging server).
When staging is verified, `staging` → `production` (auto-deploys to production).

---

## Local Dev Environment

MySQL runs in **Docker** (single container, workspace root). API and frontend run natively.

```bash
# MySQL — from workspace root
docker compose up -d        # mysql:8.0.45 on host port 3307

# API
cd mckinney-vento-lms-api
php artisan serve            # http://localhost:8000

# Frontend
cd mckinney-vento-lms-fe
bun run dev                  # http://localhost:3000
```

**Do not use Laravel Sail** — this workspace uses a single MySQL Docker container (not Sail).
The `docker-compose.yml` lives at the workspace root — it is not committed to the team repo.

---

## Logging & Observability Mindset

- Success paths should be quiet (debug-level)
- Errors should be visible but scoped
- Logs are for operators, not for reconstructing execution flow
- Report endpoints read pre-computed data — never log inside report queries

---

## Claude Code – Commit Rules (Mandatory)

- NO AI attribution in commit messages
- NO co-author lines such as:
  - `Co-Authored-By: Claude`
  - `Co-Authored-By: Anthropic`
  - `Generated with AI`
  - `Assisted by AI`
- All commits must have a single human author
- The human developer fully owns and is responsible for the code

---

## Claude Code – Outgoing Artifact Rules (Mandatory)

Outgoing artifacts mean anything that leaves my private workspace and reaches other people or external systems: **commit messages, PR titles, PR descriptions, PR comments, GitHub issues, Slack/Teamwork messages, release notes, code comments that ship.**

When creating any outgoing artifact, Claude MUST NOT reference my private workflow tooling. Specifically:

- **Local documentation paths** — no `docs/tasks/`, `docs/archive/`, `docs/deferred/`, `PROJECT_STATUS.md`, or any other path under my workspace `docs/` tree. These are my private working notes, not durable repository content.
- **Internal workflow concepts** — no "Phase 1 / Phase 2 / Phase 4", "investigation findings", "task doc", "deferred items", or similar terminology that only makes sense to me.
- **Private skills** — no `/final-strict-review`, `/write-update-message`, "strict review", or any other personal workflow skill. These are my tools, not project artifacts.
- **References to prior internal iterations** — phrases like "v1 implementation", "earlier draft", "after the strict review" leak my workflow into the project history.

What outgoing artifacts CAN reference:

- The code itself (file paths within the repo being committed to, line numbers, symbol names)
- Public artifacts (PR numbers, issue numbers, public commits)
- The actual problem being solved and how the change addresses it
- Testing performed and validation plan
- Anyone reading the artifact months from now without my private context should understand it fully

If I want to track my private workflow context, I do that locally in my workspace docs. The repository's public history should read like it was written by a contributor who only had access to the repository itself.

---

## What Does NOT Belong in This File

- Step-by-step setup → `README.md`
- Task progress tracking → `PROJECT_STATUS.md`
- In-flight implementation details → `docs/tasks/`

---

## Related Docs

- `QUALITY_RUBRIC.md`
- `AI_ENGINEERING_WORKFLOW.md`
- `PROJECT_STATUS.md`
- `README.md` — human setup walkthrough
- `docs/tasks/` — active task documents
- `docs/archive/` — completed tasks and historical reference

---

## Living but Stable

This file changes **infrequently**.
If you find yourself wanting to add large explanations here, they probably belong elsewhere.

When in doubt:

> Keep this file short, opinionated, and directional.

---

**Status**: Stable
**Maintained by**: Hans
**Last updated**: 2026-05-18