# PROJECT_STATUS.md

**Last Updated**: 2026-05-22
**Maintained by**: Hans

---

## Production Health

- **Status**: Stable
- **API**: Laravel 12, PHP 8.4, MySQL 8.0.45
- **Frontend**: Nuxt 3 SPA
- **Hosting**: DigitalOcean (Forge auto-deploy)
- **Production branch**: `production`
- **Staging branch**: `staging`

---

## Active Tasks

- **Pre/Post Quiz Scores in Reports** (requested by Dan, 2026-05-20) — API side fully on staging end-to-end. FE companion PR open (`mckinney-vento-lms-fe#16`) awaiting merge. After merge, validate in UI, then cut staging→production release for both repos and run `app:backfill-quiz-attempts` on production. See `docs/tasks/2026-05-20-pre-post-quiz-scores-in-reports.md`.

---

## Recently Completed

- **2026-05-22** — Pre/post quiz scores feature shipped to staging (API side). `mckinney-vento-lms-api#15` added the `quiz_attempts` table, `kind` column on `quiz_sections`, three score columns on `reports_user_overview`, the nightly aggregation, and the `app:backfill-quiz-attempts` command. `mckinney-vento-lms-api#16` followed up to expose `kind` and the score fields in the API resources. Verified end-to-end on staging via tinker: `QuizSection.kind` round-trips, `ReportsUserOverview` returns the three score fields. FE companion PR (`mckinney-vento-lms-fe#16`) open. New workflow rules adopted in `CLAUDE.md` (task scope acknowledgement) and `.claude/skills/final-strict-review/SKILL.md` (spec coverage check) to prevent silent narrowing of multi-phase work.
- **2026-05-21** — Sail PR merged to staging and production
- **2026-05-20** — Sail re-introduction PR opened (`MV-Learning-LLC/mckinney-vento-lms-api#14`). Canonical local setup is now: `sail up -d` from the API repo brings up PHP 8.4 + MySQL 8.0.45 + Mailpit. Workspace-level `docker-compose.yml` deleted. Workspace `README.md` and `CLAUDE.md` updated to reflect Sail.
- **2026-05-20** — Staging API migration completed. Old Forge site (`lms-rebuild-api.zenclients.com`, stuck in manual-deploy mode) deleted; new Forge site renamed onto the canonical `lms-rebuild-api.zenclients.com` domain. FE staging env var updated accordingly. Staging deployment is fully automated again.
- **2026-05-19** — Full-name user search released to production. Multi-token queries like "Dan Jordan" now match across the admin user list, client users tab, and published program users tab. Companion FE fix resets pagination on search. See `docs/archive/2026-05-19-full-name-user-search.md`.
- **2026-05-19** — Long-running staging↔production merge-commit divergence resolved on both repos via fast-forward after the release. New workflow step: fast-forward `staging` to `production` after every release to keep them aligned.
- **2026-05-18** — Workspace setup and remote migration. Both team repos now point at the MV-Learning-LLC org. Workspace-level `docker-compose.yml` runs MySQL 8.0.45 in Docker for local dev.

---

## Known Issues / Deferred Items

- **GitHub branch protection unavailable on free org plan** — MV-Learning-LLC needs GitHub Team ($4/user/month) to enable branch protection on private repos. Asked Dan; pending decision.
- **Sendgrid SMTP credentials in `mckinney-vento-lms-api/.env` are out of credits** — local dev uses Mailpit via Sail (no external SMTP needed). Production/staging use their own Forge env vars; those credentials need rotation when Sendgrid quota or account is restored.
- **DigitalOcean OAuth in Forge has been unstable since the migration** — Dan engaged Forge support for the provider state issue.
- **Legacy `.vue` composables in `mckinney-vento-lms-fe/composables/`** (`useApiFetch.vue`, `useGetCurrentUser.vue`, etc.) — replace when touching nearby code.
- **API `.env` has historically held live secrets** — rotate before sharing the repo publicly.
- **Legacy backup pages** (`pages/admin/index-backup.vue`, `pages/admin/programs-backup/`) — do not refactor unless explicitly asked.

---

## Key Architecture Notes

- All environments use **MySQL 8.0.45** — pinned deliberately (Pulse `md5()` generated column breaks on 8.4+)
- Reports are **pre-computed nightly** at 12:01 AM ET — API reads, never computes on-demand
- Magic-link auth only — no passwords. In local dev, Mailpit catches every outbound email at http://localhost:8025
- Custom XSRF cookie/header names (`XSRF-TOKEN-MVLMS`) — must stay matched on both sides

---

## Quick Reference

### Branch Strategy

- Feature work branches off `staging` and PRs back into `staging`
- `staging` — auto-deploys to staging (Forge → DigitalOcean)
- `production` — auto-deploys to production (Forge → DigitalOcean)
- Note: `development` (API) and `dev` (FE) are legacy integration branches — pending review for deprecation since Forge only watches `staging` and `production`

### Useful Commands

```bash
# API stack (from mckinney-vento-lms-api/) — Sail runs PHP + MySQL + Mailpit
sail up -d                                  # API on :8000, Mailpit on :8025
sail down                                   # Stop the stack (data preserved in volume)
sail artisan test                           # Run test suite (SQLite in-memory)
sail artisan app:reports                    # Run nightly aggregation manually
sail artisan app:fake                       # Generate fake report data
sail bin pint                               # Format PHP

# Frontend (from mckinney-vento-lms-fe/) — runs natively
bun run dev                                 # http://localhost:3000
bun run test                                # Vitest unit tests
bun run playwright                          # E2E tests
bun run lint                                # Prettier format
```