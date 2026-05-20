# Full-Name User Search

**Completed**: 2026-05-19
**Owner**: Hans
**Released via**: `MV-Learning-LLC/mckinney-vento-lms-api#13`, `MV-Learning-LLC/mckinney-vento-lms-fe#15`

---

## Problem

Dan reported that the admin user search only matched on a single field at a time — typing "Dan Jordan" returned no results because no single column (`first_name`, `last_name`, `email`) contained the full string. The support team had to search by either first or last name separately, then scan the list manually.

## Context

Three places in the admin call `User::search()` through Laravel Scout (database driver):

- `app/Services/UserService.php:149` — main user list (`GET /api/v1/users`)
- `app/Http/Controllers/Admin/ClientController.php:242` — client → users tab
- `app/Http/Controllers/Admin/ProgramController.php:198` — published program → users tab

Scout's database engine treats the keys of `User::toSearchableArray()` as actual DB column names and runs `WHERE col LIKE %term%` against each one. The searchable array only exposed `first_name`, `last_name`, and `email` — no combined field.

## Expected Behavior

Typing "Dan Jordan" matches users whose first name is "Dan" AND last name is "Jordan" (combined). Single-name and email searches keep working as before.

## Current Behavior

Multi-token queries returned no results across all three search endpoints. Single-token queries worked.

## Root Cause / Hypothesis

The `database` Scout driver builds `LIKE` clauses per column from `array_keys(toSearchableArray())`. With no combined `full_name` representation, multi-word search terms could never match any single column.

## Options

- **A — generated column on `users.full_name`** (driver-branched expression: `CONCAT(...)` for MySQL, `||` for SQLite). Keep Scout integration; one migration + one-line model change; fixes all three call sites.
- **B — tokenize the search term in `UserService::searchUsers`** and bypass Scout for users. More code, more flexible matching (handles reversed order, partial tokens), but only fixes one of three call sites and diverges from the project's Scout pattern.

## Recommendation

**Option A.** Smaller blast radius, matches the existing Pulse-migration pattern in this repo for driver-branched generated columns, and fixes all three call sites via one schema change without touching service code.

## Validation

- Added three feature tests (one per affected controller path), all asserting that a combined-name search returns the right user and excludes decoys sharing only the first or only the last name.
- Verified locally on MySQL 8.0.45 (Docker container) with seed data, then with a ~120k-user prod-like dump.
- Verified on staging UI by searching real combined names ("Laura Gillan") across multiple pages. Dan validated and approved.
- Verified on production after release.

## Side Effects

- Schema change: `users.full_name` is a VIRTUAL generated column. No data backfill, no table lock of any duration that matters, no impact on existing reads.
- Negligible per-row cost: one `CONCAT` + one `LIKE` evaluated at query time per searched user. At current scale (~120k users) this is invisible in practice.
- Search behavior for single-token queries is unchanged.

## Outcome

- Released to production via API PR #13 and FE PR #15.
- Companion FE PR #14 (pagination reset on search) was discovered during validation — the page wasn't resetting to 1 when the search term changed, making the new full-name search appear empty on page 2+ of the user list. Fixed in the same release.
- Both repos' `staging` branches were fast-forwarded to `production` after release, eliminating long-running merge-commit divergence and establishing a clean post-release sync as a recurring workflow step.

## Known Limitations

Reversed-order ("Jordan Dan") and partial-token ("Dan J") queries are not supported. Both were outside the original ask. If they come up later, Option B (tokenization) is the natural extension.
