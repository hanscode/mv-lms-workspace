# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this workspace.

For a human-oriented setup walkthrough, see [`README.md`](./README.md).

## Workspace layout

This directory contains two independently versioned repos. Always `cd` into the right one before running commands.

- `mckinney-vento-lms-api/` — Laravel 12 / PHP 8.4 REST API, magic-link auth, MySQL.
- `mckinney-vento-lms-fe/` — Nuxt 3 SPA (SSR disabled), Vue 3, Pinia, Tailwind.

## What this project is

A learning management system for the McKinney-Vento Homeless Assistance Act program. It delivers mandatory training to a hierarchical network of users (state coordinators → regional coordinators → district liaisons → school liaisons → essential staff / community partners), tracks course/lesson progress, and produces aggregated reports for each tier of that hierarchy.

Core domain entities:

- **Geo hierarchy**: `State` → `Region` → `District` → `School` (cascading access)
- **Client**: organization tenant, attached to one or more geo nodes
- **Course → Lesson → Section** (sections are polymorphic across `app/Models/Sections/*` — text, image, video, file, quiz, embed)
- **Program**: per-client per-school-year bundle of courses with deadline + reminder schedule. Has draft (editable) and published (read-only, notifications active) phases
- **Reports**: `Reports*` models hold pre-computed aggregations; the API reads them, the nightly `app:reports` command writes them

## Local dev environment

MySQL runs in **Docker** (single container, no Sail). The API and frontend run **natively** on the host.

```bash
# MySQL — from the workspace root (this directory)
docker compose up -d                  # mysql:8.0.45 on host port 3307

# API
cd mckinney-vento-lms-api
php artisan serve                     # http://localhost:8000

# Frontend
cd mckinney-vento-lms-fe
bun run dev                           # http://localhost:3000
```

The `docker-compose.yml` lives at the workspace root, not inside the API repo, so it doesn't pollute the team repo. The API `.env` is already configured for the Docker MySQL (`DB_HOST=127.0.0.1`, `DB_PORT=3307`, `DB_DATABASE=lms_rebuild`).

This workspace's local setup intentionally **does not use Laravel Sail**, even though the API repo's own `README.md` and `quick-guide.txt` describe a Sail-based workflow (which David and others on the team use). My `docker-compose.yml` lives only locally — it runs a single MySQL 8.0 container and isn't committed to the team repo. Don't push it.

**Do not change the MySQL version.** Production runs `8.0.45`. Laravel Pulse uses `md5()` in a generated column which breaks on MySQL 8.4+ / 9.x — the pinned version is deliberate.

## Common commands

### Backend (`mckinney-vento-lms-api/`)

```bash
# docker compose commands run from the WORKSPACE ROOT (../), not inside the API repo
docker compose up -d                          # Start MySQL  (workspace root)
docker compose stop                           # Stop MySQL   (workspace root)
php artisan serve                             # API at :8000
php artisan migrate:fresh --seed              # Reset DB
php artisan test                              # PHPUnit (also: --filter=TestName)
./vendor/bin/pint                             # Format PHP
php artisan scribe:generate                   # Regenerate API docs at /docs
php artisan app:reports                       # Manually run nightly aggregation
php artisan app:fake                          # Generate fake report data
php artisan cache:clear && php artisan config:clear && php artisan route:clear
```

### Frontend (`mckinney-vento-lms-fe/`)

```bash
bun install                                   # or yarn install
bun run dev                                   # Dev server :3000
bun run build                                 # Production build
bun run test                                  # Vitest unit tests
bun run playwright                            # Playwright E2E
bun run lint                                  # Prettier format
bun run nuke                                  # Wipe node_modules + .nuxt + .output
```

## Architecture notes

### API (Laravel)

- All API routes live in `routes/api.php` and are prefixed with `/api/v1`. Route file is ~415 lines, grouped by area (auth, admin users, course-management, programs, options, reports, registration).
- **No password auth** — magic links only. `app/Http/Controllers/Auth/LoginController.php` + `app/Services/MagicLinkService.php`. Sanctum manages the session via stateful cookies. For local dev set `MAIL_MAILER=log` and grab the link from `storage/logs/laravel-*.log`.
- **Sanctum CSRF** uses custom cookie/header names: `XSRF-TOKEN-MVLMS` / `X-XSRF-TOKEN-MVLMS`. Both API and FE config must match — don't "normalize" them.
- **Service layer**: business logic lives in `app/Services/*` (17 services). Controllers stay thin. Report-related services are under `app/Services/Reports/`.
- **Access control** is middleware-based, not policy-based: `admin`, `participant`, `reports:permission_type`, `resource-library`. See `app/Http/Middleware/`.
- **User model side-effects**: `User::boot()` auto-detaches geo assignments when the user type changes. Easy to miss when debugging "missing" relationships.
- **Reports are pre-computed**: API endpoints under `/api/v1/reports/*` read from `reports_*` tables. The nightly `app:reports` command (00:01 ET) regenerates them via `ReportService`. To test report changes locally, run that command or `app:fake` first.
- **Excel I/O** via `maatwebsite/excel`. Exports are queued jobs (`app/Jobs/Exports/*`) that write XLSX, upload to storage, and email the requesting user.
- **Scheduled tasks** in `routes/console.php` — magic-link expiry, idle timelog cleanup, program reminders/reports/publishing, nightly aggregation. Run any one manually via `php artisan <command>`.
- **CI runs tests on SQLite** (`.github/workflows/testsuite.yml`). Local dev uses MySQL. If a test passes locally but fails in CI, suspect MySQL-specific SQL.

### Frontend (Nuxt 3)

- `ssr: false` — pure SPA. No SEO concerns; no server-rendered pages.
- **Auth: use `useSanctumFetch` and `useSanctumUser`** from the `nuxt-auth-sanctum` module. Several legacy `.vue` composables exist (`useApiFetch.vue`, `useGetCurrentUser.vue`, etc.) — do not write new code against them, and replace them when touching nearby code.
- **API errors**: handle via `composables/useApiError.ts`. Validation responses follow Laravel's `{ message, errors: { field: [string] } }` shape.
- **State**: Pinia stores in `store/`. Notable ones:
  - `programBuilderStore.ts` (large, multi-step program builder)
  - `useProgramStore.ts`, `useTakeCourseStore.ts`, `useLessonStore.ts`, `useRegistrationStore.ts`, `useReportsStore.ts`, `useOptionsStore.ts`, `optionManagementStore.ts`
- **Pages**: `pages/admin/*` is the admin section (24 subdirs). `pages/reports/*` is for liaison/coordinator views. `pages/course/[courseid]/lesson/[lessonid]/` is the participant course viewer. `pages/registration/[code]/` handles self-service signup.
- **Components**:
  - `components/ui/` — design system primitives (`BaseButton`, `BaseTextInput`, `BaseDatePicker`, `BaseRichTextEditor`, `BaseStepsPanel`, etc.)
  - `components/Admin/` — admin CRUD UIs grouped by entity
  - `components/Admin/ListLayout/` — the reusable table+pagination+search wrapper. Prefer it over hand-rolled tables.
  - `components/Participant/` — dashboard, course player, certificate, reports
- **Forms**: `vee-validate` + `yup`. The Program builder uses this pattern; older course/user forms use plain refs — when adding new forms, prefer vee-validate.
- **TypeScript is loose** (`strict: false` in `nuxt.config.ts`). Types in `types/` document API DTOs but aren't enforced at compile time.

## Conventions

### General
- DRY, KISS, narrative flow. Self-documenting names beat comments.
- Don't add comments explaining *what* code does. Only add comments for non-obvious *why* (a hidden constraint, a workaround).
- Don't add scaffolding for hypothetical future requirements.

### Vue / Nuxt
- Composition API with `<script setup>`.
- New composables go in `.ts` files, not `.vue`.
- Reach for `useSanctumFetch` before manual `useFetch` / axios.

### Laravel
- Controllers stay thin — push logic into `app/Services/*`.
- Run `./vendor/bin/pint` before committing.
- Custom middleware for access control, not Policies (matches existing pattern).

## Testing

- API: PHPUnit in `tests/Feature/` and `tests/Unit/`. ~36 test files. Run via `php artisan test`. CI uses SQLite.
- FE: Vitest unit tests in `test/`. Playwright E2E in `e2e/` and `playwright/`. The E2E `auth.setup.ts` exercises the magic-link login flow.

## Gotchas / things that bite

- The MySQL container binds host port **3307**. Both `.env` and the docker-compose file reflect this. If a brew MySQL is running on 3306, no conflict.
- Custom XSRF cookie/header names (`XSRF-TOKEN-MVLMS`) need to stay matched on both sides.
- The API `.env` has historically held live secrets (mail provider keys). Treat with care.
- Bun is the preferred package manager (`bun.lockb` is current). Yarn lockfile is also present but may lag behind.
- Some test pages and legacy backup pages exist (e.g. `pages/admin/index-backup.vue`, `pages/admin/programs-backup/`). Don't refactor them unless explicitly asked.

## Documentation

- API: Scribe-generated, served at http://localhost:8000/docs. Regenerate with `php artisan scribe:generate` after route changes.
- Workspace: see `README.md` for the human setup walkthrough.
