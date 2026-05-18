# PROJECT_STATUS.md

**Last Updated**: 2026-05-08
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

*Add completed tasks here as they are finished.*

---

## Known Issues / Deferred Items

- Legacy `.vue` composables in `mckinney-vento-lms-fe/composables/` (`useApiFetch.vue`, `useGetCurrentUser.vue`, etc.) — replace when touching nearby code
- API `.env` has historically held live secrets — rotate before sharing the repo publicly
- Some legacy backup pages exist (`pages/admin/index-backup.vue`, `pages/admin/programs-backup/`) — do not refactor unless explicitly asked

---

## Key Architecture Notes

- All environments use **MySQL 8.0.45** — pinned deliberately (Pulse `md5()` generated column breaks on 8.4+)
- Reports are **pre-computed nightly** at 12:01 AM ET — API reads, never computes on-demand
- Magic-link auth only — no passwords. For local dev use `MAIL_MAILER=log` and grab the link from logs
- Custom XSRF cookie/header names (`XSRF-TOKEN-MVLMS`) — must stay matched on both sides

---

## Quick Reference

### Branch Strategy

- `develop` / `dev` — feature development
- `staging` — auto-deploys to staging (Forge)
- `production` — auto-deploys to production (Forge)

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