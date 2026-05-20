# McKinney-Vento LMS

Learning management system used to deliver mandatory training (and track completion) for the McKinney-Vento Homeless Assistance Act program. Targets state coordinators, regional coordinators, district liaisons, school liaisons, essential staff, and community partners across a hierarchical state network.

Production is auto-deployed via Laravel Forge to DigitalOcean droplets (staging and prod).

## Repository layout

This directory is a workspace containing two independently versioned repos:

| Directory | Stack | Purpose |
|---|---|---|
| [`mckinney-vento-lms-api/`](./mckinney-vento-lms-api/) | Laravel 12, PHP 8.4, MySQL 8.0.45 (via Sail) | REST API at `/api/v1/*`, magic-link auth, nightly report aggregation, Excel exports |
| [`mckinney-vento-lms-fe/`](./mckinney-vento-lms-fe/) | Nuxt 3 (SPA mode), Vue 3, Pinia, Tailwind | Web UI for participants, liaisons, coordinators, and admins |

The two apps talk over HTTP using Laravel Sanctum (stateful cookie auth with custom `XSRF-TOKEN-MVLMS` headers).

## Prerequisites

| Tool | Notes |
|---|---|
| Docker Desktop | Required — the API stack (PHP, MySQL, Mailpit) runs via Laravel Sail in containers |
| Node 18.x (per `.nvmrc`) | For the Nuxt frontend, which runs natively |
| Bun | Preferred package manager for the FE (`bun.lockb` is current); Yarn also works |

The backend runs entirely in Docker via Sail — no native PHP/Composer install needed. The frontend runs natively for fast HMR.

## First-time setup

### Backend (Laravel API via Sail)

See [`mckinney-vento-lms-api/README.md`](./mckinney-vento-lms-api/README.md) for the full setup. Short version:

```bash
cd mckinney-vento-lms-api
cp .env.example .env
# Install Sail's PHP dependencies (uses a one-shot Composer container — no host PHP needed):
docker run --rm -u "$(id -u):$(id -g)" -v "$(pwd):/var/www/html" -w /var/www/html laravelsail/php84-composer:latest composer install --ignore-platform-reqs
./vendor/bin/sail up -d
./vendor/bin/sail artisan key:generate
./vendor/bin/sail artisan migrate:fresh --seed
```

Optionally add a shell alias so you can use `sail` instead of `./vendor/bin/sail`:

```bash
echo "alias sail='[ -f sail ] && bash sail || bash vendor/bin/sail'" >> ~/.zshrc
source ~/.zshrc
```

The default seeded admin is `admin@example.com`. Magic-link login works locally via the bundled Mailpit container — see [Authentication](#authentication) below.

### Frontend (Nuxt SPA)

```bash
cd mckinney-vento-lms-fe
bun install
bun run dev
```

Open http://localhost:3000.

## Running day-to-day

| What | Command |
|---|---|
| Start API stack | `cd mckinney-vento-lms-api && sail up -d` |
| Stop API stack | `cd mckinney-vento-lms-api && sail down` |
| Start frontend | `cd mckinney-vento-lms-fe && bun run dev` |
| Reset DB | `cd mckinney-vento-lms-api && sail artisan migrate:fresh --seed` |
| Run API tests | `cd mckinney-vento-lms-api && sail artisan test` |
| Run FE unit tests | `cd mckinney-vento-lms-fe && bun run test` |
| Run E2E tests | `cd mckinney-vento-lms-fe && bun run playwright` |
| Format PHP | `cd mckinney-vento-lms-api && sail bin pint` |
| Format JS/Vue | `cd mckinney-vento-lms-fe && bun run lint` |
| Re-generate API docs | `cd mckinney-vento-lms-api && sail artisan scribe:generate` |

## Local environment

| Service | URL | Notes |
|---|---|---|
| Frontend | http://localhost:3000 | Nuxt dev server (native) |
| API | http://localhost:8000 | Sail's `laravel.test` container (bound via `APP_PORT=8000` in `.env`) |
| API docs (Scribe) | http://localhost:8000/docs | Regenerate after route changes |
| Mailpit (catches outbound mail) | http://localhost:8025 | All magic-link emails land here in local dev |
| MySQL | `127.0.0.1:3306` | Sail's `mysql` container, user/db `lms_rebuild`, password `password` |

The frontend reads `NUXT_PUBLIC_API_BASE_URL` and `NUXT_PUBLIC_API_REFERER` from the environment (defaults to the two URLs above).

## Authentication

There are no passwords. Login is magic-link only:

1. Frontend posts an email to `POST /api/v1/login`.
2. API generates a UUID token in `magic_links`, emails a link to the user.
3. User clicks the link → frontend posts to `POST /api/v1/login/verify` with `email`, `token`, `device_id`.
4. API calls `Auth::login()` → Sanctum cookie issued.

For local dev, **the link is delivered to Mailpit** (http://localhost:8025) — no external SMTP needed. Open Mailpit's web UI, find the email, click the link.

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

Run any of them manually with `sail artisan app:reports` etc. For populating reports with fake data locally, use `sail artisan app:fake`.

## Known gotchas

- Laravel Pulse uses `md5()` in a generated column. This works on MySQL 5.7 and 8.0.x but **fails on MySQL 8.4 and 9.x** — that's why `compose.yaml` pins `mysql:8.0.45` (matches production).
- The frontend `nuxt.config.ts` uses non-standard `XSRF-TOKEN-MVLMS` / `X-XSRF-TOKEN-MVLMS` cookie/header names. Don't "fix" them to the Sanctum defaults — the API expects this rename.
- Some composables in `mckinney-vento-lms-fe/composables/` are still `.vue` files (e.g., `useApiFetch.vue`, `useGetCurrentUser.vue`). These are legacy; new code should use `useSanctumFetch` / `useSanctumUser` from the `nuxt-auth-sanctum` module.
- CI runs the API test suite on SQLite (`.github/workflows/testsuite.yml`), but local dev uses MySQL via Sail. `phpunit.xml` is configured to use SQLite in-memory for tests so local matches CI — don't change that.

## Branching and deploys

| Branch | Auto-deploys to |
|---|---|
| `staging` | Staging droplet (Forge) |
| `production` | Production droplet (Forge) |

Feature work branches off `staging`, PRs go back to `staging`. After a `staging → production` release, fast-forward `staging` to `production` to keep them aligned.

## Useful references

- API documentation (local): http://localhost:8000/docs
- Laravel 12 docs: https://laravel.com/docs/12.x
- Laravel Sail: https://laravel.com/docs/12.x/sail
- Nuxt 3 docs: https://nuxt.com/docs
- `nuxt-auth-sanctum`: https://manchenkoff.gitbook.io/nuxt-auth-sanctum

## Project-internal notes

Workspace-level files you may also want to read:
- `CLAUDE.md` — guidance for Claude Code when working in this workspace
- `PROJECT_STATUS.md` — current project state, recently completed work, known issues
- `QUALITY_RUBRIC.md`, `AI_ENGINEERING_WORKFLOW.md` — quality standards and collaboration framework
- `specs.txt`, `quick-guide.txt`, `task-list-with-micro-steps.md` — historical context from earlier collaborators (some details may be out of date relative to this README)
