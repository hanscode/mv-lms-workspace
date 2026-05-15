# McKinney-Vento LMS

Learning management system used to deliver mandatory training (and track completion) for the McKinney-Vento Homeless Assistance Act program. Targets state coordinators, regional coordinators, district liaisons, school liaisons, essential staff, and community partners across a hierarchical state network.

Production is auto-deployed via Laravel Forge to DigitalOcean droplets (staging and prod).

## Repository layout

This directory is a workspace containing two independently versioned repos:

| Directory | Stack | Purpose |
|---|---|---|
| [`mckinney-vento-lms-api/`](./mckinney-vento-lms-api/) | Laravel 12, PHP 8.4, MySQL 8.0 | REST API at `/api/v1/*`, magic-link auth, nightly report aggregation, Excel exports |
| [`mckinney-vento-lms-fe/`](./mckinney-vento-lms-fe/) | Nuxt 3 (SPA mode), Vue 3, Pinia, Tailwind | Web UI for participants, liaisons, coordinators, and admins |

The two apps talk over HTTP using Laravel Sanctum (stateful cookie auth with custom `XSRF-TOKEN-MVLMS` headers).

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| PHP | 8.4+ | `brew install php` |
| Composer | 2.x | `brew install composer` |
| Node | 18.x (per `.nvmrc`) | `nvm install 18 && nvm use 18` — newer versions usually work but aren't pinned |
| Bun or Yarn | latest | Bun preferred (`bun.lockb` present); Yarn also supported |
| Docker Desktop | any recent | Only used for the MySQL container — Laravel itself runs natively |

You do **not** need Laravel Sail or a system-wide MySQL install. The MySQL server runs in a single Docker container pinned to the exact prod version.

## First-time setup

### 1. MySQL (Docker)

From the workspace root (this directory):

```bash
docker compose up -d
```

This starts `mysql:8.0.45` on host port **3307**. The compose file lives at the workspace root (not inside the API repo) so it stays out of the team's repo. The data is persisted in a named volume (`mvlms_mvlms-mysql-data`).

### 2. Backend (Laravel API)

```bash
cd mckinney-vento-lms-api

# PHP deps
composer install

# .env (already present on first clone — copy if missing)
cp -n .env.example .env

# Generate app key (only if APP_KEY is empty)
php artisan key:generate

# Migrate + seed (MySQL container must already be running)
php artisan migrate:fresh --seed
```

The default seeded admin user is `admin@example.com`. Login is by magic link — see [Authentication](#authentication) below for the local-dev workaround.

### 3. Frontend (Nuxt SPA)

```bash
cd mckinney-vento-lms-fe
bun install           # or: yarn install
bun run dev           # or: yarn dev
```

Open http://localhost:3000.

## Running day-to-day

| What | Command (from repo root) |
|---|---|
| Start MySQL | `docker compose up -d   # from workspace root` |
| Stop MySQL | `docker compose stop    # from workspace root` |
| Start API | `cd mckinney-vento-lms-api && php artisan serve` |
| Start frontend | `cd mckinney-vento-lms-fe && bun run dev` |
| Reset DB | `cd mckinney-vento-lms-api && php artisan migrate:fresh --seed` |
| Run API tests | `cd mckinney-vento-lms-api && php artisan test` |
| Run FE unit tests | `cd mckinney-vento-lms-fe && bun run test` |
| Run E2E tests | `cd mckinney-vento-lms-fe && bun run playwright` |
| Format PHP | `cd mckinney-vento-lms-api && ./vendor/bin/pint` |
| Format JS/Vue | `cd mckinney-vento-lms-fe && bun run lint` |
| Re-generate API docs | `cd mckinney-vento-lms-api && php artisan scribe:generate` (served at http://localhost:8000/docs) |

## Local environment

| Service | URL | Notes |
|---|---|---|
| Frontend | http://localhost:3000 | Nuxt dev server |
| API | http://localhost:8000 | `php artisan serve` |
| API docs (Scribe) | http://localhost:8000/docs | Regenerate after route changes |
| MySQL | `127.0.0.1:3307` | Docker, user/db `lms_rebuild`, password `password` |

The frontend reads `NUXT_PUBLIC_API_BASE_URL` and `NUXT_PUBLIC_API_REFERER` from the environment (defaults to the two URLs above).

## Authentication

There are no passwords. Login is magic-link only:

1. Frontend posts an email to `POST /api/v1/login`.
2. API generates a UUID token in `magic_links`, emails a link to the user.
3. User clicks the link → frontend posts to `POST /api/v1/login/verify` with `email`, `token`, `device_id`.
4. API calls `Auth::login()` → Sanctum cookie issued.

For local dev, set `MAIL_MAILER=log` in `mckinney-vento-lms-api/.env` so the link is written to `storage/logs/laravel-YYYY-MM-DD.log` instead of being sent. Then grab the link from the log and paste it into the browser.

## Domain orientation

A 60-second tour for anyone new to the codebase:

- **Geo hierarchy**: `State` → `Region` → `District` → `School`. Users are assigned to one or more nodes; access cascades downward.
- **Clients**: Organizations using the system, attached to one or more geo nodes.
- **Courses**: Made of `Lesson`s, each with ordered `Section`s. Section content is polymorphic across 10 type-specific models (`TextSection`, `VideoSection`, `QuizSection`, etc.).
- **Programs**: Per-client per-school-year bundles of courses with deadlines, reminder schedules, and bulk user enrollment. Have a draft phase (editable) and a published phase (read-only with notifications).
- **Reports**: Pre-computed nightly at 12:01 AM ET (`app:reports`). Aggregates live in `reports_*` tables — API endpoints read these, no real-time computation.
- **User types**: Admin, State Coordinator, Regional Coordinator, District Liaison, School Liaison, Essential Staff, Community Partner. Each has its own access scope.

## Scheduled jobs

Run by Laravel Scheduler on a single cron entry (`* * * * * php artisan schedule:run`):

| Command | Timing | Purpose |
|---|---|---|
| `app:expire-links` | every minute | Delete expired magic links |
| `app:close-idle-timelogs` | every minute | Close idle course sessions |
| `app:program-reminders` | 6:00 AM ET | Email program participants |
| `app:program-reports` | 6:00 AM ET | Email progress to program leaders |
| `app:program-publish` | 8:00 AM ET | Auto-publish scheduled programs |
| `app:reports` | 12:01 AM ET | Nightly aggregation of all report tables |

Run any of them manually with `php artisan app:reports` etc. For populating reports with fake data locally, use `php artisan app:fake`.

## Known gotchas

- The MySQL Docker container binds host port **3307**, not 3306, so a brew-installed MySQL can coexist on 3306 without conflicting. The API `.env` reflects this.
- Laravel Pulse uses `md5()` in a generated column. This works on MySQL 5.7 and 8.0.x but **fails on MySQL 8.4 and 9.x** — that's why we pin `mysql:8.0.45` (matches production).
- A `.env` checked into the API repo currently contains live email credentials. Rotate before sharing the repo publicly.
- The frontend `nuxt.config.ts` uses non-standard `XSRF-TOKEN-MVLMS` / `X-XSRF-TOKEN-MVLMS` cookie/header names. Don't "fix" them to the Sanctum defaults — the API expects this rename.
- Some composables in `mckinney-vento-lms-fe/composables/` are still `.vue` files (e.g., `useApiFetch.vue`, `useGetCurrentUser.vue`). These are legacy; new code should use `useSanctumFetch` / `useSanctumUser` from the `nuxt-auth-sanctum` module.
- CI runs the test suite on SQLite (`.github/workflows/testsuite.yml`), but local dev uses MySQL. If a test passes locally but fails in CI, check for MySQL-specific behavior.

## Branching and deploys

| Branch | Auto-deploys to |
|---|---|
| `staging` | Staging droplet (Forge) |
| `production` | Production droplet (Forge) |

Feature work branches off `develop`/`dev`, PRs go back to that branch. Forge handles the deploy pipeline.

## Useful references

- API documentation (local): http://localhost:8000/docs
- Laravel 12 docs: https://laravel.com/docs/12.x
- Nuxt 3 docs: https://nuxt.com/docs
- `nuxt-auth-sanctum`: https://manchenkoff.gitbook.io/nuxt-auth-sanctum

## Project-internal notes

Workspace-level files you may also want to read:
- `CLAUDE.md` — guidance for Claude Code when working in this workspace
- `specs.txt`, `quick-guide.txt`, `task-list-with-micro-steps.md` — historical context from earlier collaborators (some details may be out of date relative to this README)
