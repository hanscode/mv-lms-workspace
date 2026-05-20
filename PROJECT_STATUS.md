# PROJECT_STATUS.md

**Last Updated**: 2026-05-20
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

*No active tasks at this time.*

---

## Recently Completed

- **2026-05-20** — Staging API migration completed. Old Forge site (`lms-rebuild-api.zenclients.com`, stuck in manual-deploy mode) deleted; new Forge site renamed onto the canonical `lms-rebuild-api.zenclients.com` domain. FE staging env var updated accordingly. Staging deployment is fully automated again.
- **2026-05-19** — Full-name user search released to production. Multi-token queries like "Dan Jordan" now match across the admin user list, client users tab, and published program users tab. Companion FE fix resets pagination on search. See `docs/archive/2026-05-19-full-name-user-search.md`.
- **2026-05-19** — Long-running staging↔production merge-commit divergence resolved on both repos via fast-forward after the release. New workflow step: fast-forward `staging` to `production` after every release to keep them aligned.
- **2026-05-18** — Workspace setup and remote migration. Both team repos now point at the MV-Learning-LLC org. Workspace-level `docker-compose.yml` runs MySQL 8.0.45 in Docker for local dev.

---

## Known Issues / Deferred Items

- **Sail re-introduction (pending)** — README in `mckinney-vento-lms-api/` describes a Sail-based local setup, but Sail isn't installed in the repo. Planning to add Sail back as the canonical local setup for one-command onboarding + bundled Mailpit. Conversation with David done; he's OK with it.
- **GitHub branch protection unavailable on free org plan** — MV-Learning-LLC needs GitHub Team ($4/user/month) to enable branch protection on private repos. Asked Dan; pending decision.
- **Sendgrid SMTP credentials in `mckinney-vento-lms-api/.env` are out of credits** — for local dev, `MAIL_MAILER=log` is used and links can be grabbed from `storage/logs/`. Production/staging use their own Forge env vars.
- **DigitalOcean OAuth in Forge has been unstable since the migration** — Dan engaged Forge support for the provider state issue.
- **Legacy `.vue` composables in `mckinney-vento-lms-fe/composables/`** (`useApiFetch.vue`, `useGetCurrentUser.vue`, etc.) — replace when touching nearby code.
- **API `.env` has historically held live secrets** — rotate before sharing the repo publicly.
- **Legacy backup pages** (`pages/admin/index-backup.vue`, `pages/admin/programs-backup/`) — do not refactor unless explicitly asked.

---

## Key Architecture Notes

- All environments use **MySQL 8.0.45** — pinned deliberately (Pulse `md5()` generated column breaks on 8.4+)
- Reports are **pre-computed nightly** at 12:01 AM ET — API reads, never computes on-demand
- Magic-link auth only — no passwords. For local dev use `MAIL_MAILER=log` and grab the link from logs
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
# From workspace root
docker compose up -d                        # Start MySQL

# API (mckinney-vento-lms-api/)
php artisan serve                           # http://localhost:8000
php artisan test                            # Run test suite
php artisan app:reports                     # Run nightly aggregation manually
php artisan app:fake                        # Generate fake report data
./vendor/bin/pint                           # Format PHP

# Frontend (mckinney-vento-lms-fe/)
bun run dev                                 # http://localhost:3000
bun run test                                # Vitest unit tests
bun run playwright                          # E2E tests
bun run lint                                # Prettier format
```