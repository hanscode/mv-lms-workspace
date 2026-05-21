# Pre/Post Quiz Scores Added to Reports

**Status**: Active
**Owner**: Hans
**Requested by**: Dan
**Created**: 2026-05-20

---

## Problem

For essential staff training, every course has a **Pre-Quiz** (before the lesson content) and a **Post-Quiz** (after). Today these quizzes are barriers — users have to pass to proceed — but the **scores are not surfaced anywhere outside the immediate post-submission UI**. Dan wants both scores displayed on the essential staff reports so admins can see learning improvement.

## Context

### Where quizzes live today

- **`quiz_sections`** (`app/Models/Sections/QuizSection.php`) — one row per quiz, with `quiz_threshold` (passing percentage, nullable, defaults to a constant in `CourseService`)
- **`quiz_questions`** and **`quiz_answers`** — questions and their possible answers, with `is_correct` on the answer
- **No pre/post distinction in the schema** — every quiz uses `SectionType::Quiz = 'quiz_section'` (`app/Enums/SectionType.php:9`). Pre vs Post is currently a convention based on section ordering within a lesson, not enforced by any code or column.

### Where scores live today

- Scores are computed at submission time in `CourseService::answerQuiz()` (`app/Services/CourseService.php:694-770`) and persisted as **JSON** in `lesson_progresses.quiz_data`:
  ```json
  { "choices": [...], "results": { "correct": 8, "total": 10, "pct": 80 } }
  ```
- This JSON is consumed for one purpose only: gating "did you pass" against `quiz_threshold` (`CourseService.php:761-770`). It is **never read by any report or aggregation**.
- No `quiz_attempts` table exists. Multiple attempts overwrite the same JSON (history is lost).

### Where essential staff reports live today

- Endpoints: `GET /api/v1/reports/{state|region|district|school}/{id}/essential_staff` and matching `/export` variants (`routes/api.php:102, 384, 395, 404`)
- Backing table: `reports_user_overview` — per-user-per-school-year aggregate, rebuilt nightly by `TrainingReportService::run()` (`app/Services/Reports/TrainingReportService.php:35-109`). Truncates and re-populates every run, so adding new fields is cheap.
- FE display: `components/Admin/reports/StaffReport.vue` renders the table; columns are defined in `composables/useReportTableColumns.ts:109-151`.
- Export job: `UserOverviewExportJob` → `UserOverviewExport.php` writes XLSX.

## Expected behavior

For each essential staff user on the report:

1. **Pre-Quiz Score** column showing the user's pre-quiz score as a percentage (e.g., `40%`)
2. **Post-Quiz Score** column showing the user's post-quiz score as a percentage (e.g., `80%`)
3. **Improvement** column showing the difference in percentage points (`post − pre`, e.g., `+40 pp`). Can be negative if a user scored worse on the post-quiz.

For users who took multiple courses in the same school year, scores are averaged across the courses (matches the existing per-user-per-school-year aggregation pattern on `reports_user_overview`).

Existing essential staff who already completed training are included via a one-shot backfill from existing `lesson_progresses.quiz_data` JSON.

## Current behavior

- Quiz scores are stored as JSON in `lesson_progresses.quiz_data` and never aggregated for reporting.
- The reports table shows: name, email, role, completed (Y/N), completion date, time spent. No score columns.
- Multiple quiz attempts overwrite the JSON — no per-attempt history is preserved.

## Approach summary

Five coordinated changes:

1. **Schema** — add `kind` enum on `quiz_sections`, create a `quiz_attempts` table, add 3 score columns to `reports_user_overview`.
2. **Submission write** — `CourseService::answerQuiz()` writes a row to `quiz_attempts` on every submission (in addition to the existing JSON write; the JSON stays in place for now to avoid touching the threshold-check code path).
3. **Nightly aggregation** — `TrainingReportService` reads `quiz_attempts`, picks the right attempt per kind, computes user averages, writes to `reports_user_overview`.
4. **Admin UI** — quiz section form gets a "Kind" dropdown so admins can mark pre/post/standalone explicitly.
5. **Backfill** — one-shot Artisan command extracts existing `lesson_progresses.quiz_data` JSON into `quiz_attempts`. Existing quiz sections still need their `kind` set by admins (or via a separate convention-based heuristic if Dan confirms one).

The report's FE columns and export are extended last, after the data is reliably populated.

## Options considered

### Option 1 — Convention-based pre/post identification (rejected)

Treat the first quiz section in a course as the pre-quiz and the last as the post-quiz, no schema change.

- ✅ No migration, no admin UI work
- ❌ Silent failure mode: if a course has one quiz, three quizzes, or admins reorder sections, reports show wrong data with no signal
- ❌ Conflicts with `QUALITY_RUBRIC.md` "Failure containment — errors should be local, not systemic"

### Option 2 — Read scores from `lesson_progresses.quiz_data` JSON directly during nightly aggregation, no `quiz_attempts` table (smaller scope)

Skip the new table. `TrainingReportService` parses JSON, identifies pre/post by the new `kind` column, computes averages.

- ✅ Smaller PR — fewer files touched
- ✅ No write-path changes in `CourseService::answerQuiz()`
- ❌ Loses attempt history (only the latest submission is preserved per quiz)
- ❌ JSON-only storage doesn't scale to future questions like "show me the user's first attempt" or "audit individual quiz answers"
- ❌ JSON parsing during nightly aggregation is awkward to query for ad-hoc analysis

### Option 3 — Add `quiz_attempts` table, explicit `kind` on `quiz_sections`, aggregate per-user (**chosen**)

The recommended approach above. Bigger than Option 2 by one migration + one model + one write-site change, but pays back permanently.

### Option 4 — Per-user-per-course granularity (deferred)

Create a new `reports_user_course_overview` table so each user's report shows one row per course. Lets the UI display detailed per-course pre/post.

- ✅ More accurate when a user takes multiple courses
- ❌ New table, new FE rendering for multiple rows per user, no existing pattern in the codebase for per-course report rows
- ❌ Marginal benefit for the typical case (essential staff take one mandatory training per school year)

**Deferred to follow-up.** Captured under "Follow-ups" below.

## Recommendation

Option 3. Specific design choices:

### Pre/Post identification

New `kind` column on `quiz_sections`. Backed by a `QuizKind` enum (`app/Enums/QuizKind.php`) with cases `Pre`, `Post`, `Standalone`. Existing quizzes default to `Standalone` and are explicitly retagged by admins (or via a separate one-shot script if Dan confirms a recognizable naming convention).

### Granularity

Three new columns on `reports_user_overview`:

- `pre_quiz_score` — decimal(5,2), nullable. User's average pre-quiz score across all courses taken in the school year, 0-100.
- `post_quiz_score` — decimal(5,2), nullable. Same for post-quizzes.
- `quiz_improvement_pct` — decimal(5,2), nullable. `post_quiz_score - pre_quiz_score`. Signed.

Nullable columns let us represent "this user has no quiz data" cleanly (e.g., still in progress, or course had no tagged pre/post).

### Improvement metric

Percentage-points subtraction: `post − pre`. Signed decimal. Negative values are valid (some users score worse on the retake — important data to surface).

### Attempt selection (when a user has multiple attempts at the same quiz)

- **Pre-quiz**: take the **first** attempt. Represents incoming knowledge before exposure to the material.
- **Post-quiz**: take the **passing** attempt (the one that crossed `quiz_threshold` and unlocked progression). Best representation of the user's final demonstrated knowledge. Falls back to the latest attempt if no attempt has yet passed.

These rules live in `TrainingReportService` and can be tuned without touching schema.

### `quiz_attempts` table shape

```
id                       bigint unsigned PK
user_id                  fk users.id, indexed
quiz_section_id          fk quiz_sections.id, indexed
lesson_id                fk lessons.id, indexed (denormalized for query convenience)
course_id                fk courses.id, indexed (denormalized)
attempt_number           unsigned tinyint (1-based, per user per quiz)
score_pct                decimal(5,2), 0-100
passed                   boolean
created_at, updated_at
```

Index on `(user_id, quiz_section_id, attempt_number)` for the "find attempt N for user U on quiz Q" lookups during aggregation.

### Backfill

One-shot Artisan command `app:backfill-quiz-attempts` reads every `lesson_progresses` row with non-null `quiz_data`, parses the JSON, and writes a `quiz_attempts` row per quiz answered. Idempotent (truncates `quiz_attempts` then re-inserts). Run once after the migration deploys.

The existing JSON is **not deleted** by the backfill — it stays in place as the legacy structure. Removing it is out of scope for this task (separate cleanup if anyone wants it later).

## Implementation plan

### Phase 1 — API: schema + write path + backfill

1. **Migration 1**: add `kind` enum column to `quiz_sections` (nullable, default `standalone`)
2. **Migration 2**: create `quiz_attempts` table
3. **Migration 3**: add 3 score columns to `reports_user_overview` (all nullable)
4. **Enum**: create `App\Enums\QuizKind` with `Pre`, `Post`, `Standalone`
5. **Model**: create `App\Models\QuizAttempt` with relationships to `User`, `QuizSection`, `Lesson`, `Course`
6. **Update `QuizSection`**: cast `kind` to `QuizKind` enum, add to `$fillable`
7. **Update `CourseService::answerQuiz()`**: after the existing JSON write, also insert a row into `quiz_attempts` with attempt_number = current count + 1
8. **Update `Admin/CreateSectionRequest` and `UpdateSectionRequest`**: validate `kind`
9. **Update `SectionService::createSection`/`updateSection`**: persist `kind`
10. **Add Artisan command**: `php artisan app:backfill-quiz-attempts`
11. **Update `TrainingReportService`**: in the per-user loop, look up `quiz_attempts` filtered by the user's courses, pick first-attempt for pre and passing-attempt for post per quiz section's kind, average across the user's courses, populate the three new columns on `reports_user_overview`
12. **Tests** (PHPUnit, SQLite):
    - Feature test: submitting a quiz creates a `quiz_attempts` row with correct fields
    - Feature test: nightly aggregation populates the three new score columns correctly for a user who took pre + post quizzes
    - Unit test: attempt-selection rule for pre (first) and post (passing)
    - Feature test: backfill command populates `quiz_attempts` from JSON correctly
13. **Pint**: run `./vendor/bin/pint --dirty` before commit
14. **PR**: target `staging`, request review

### Phase 2 — API: report endpoint exposes new fields

15. **Update Eloquent API Resource** (whichever one shapes the essential-staff endpoint response) to include `pre_quiz_score`, `post_quiz_score`, `quiz_improvement_pct`
16. **Update `UserOverviewExport.php`** headings and `map()` to include the three new columns
17. **Tests**: feature tests for the report endpoints asserting the new fields are present
18. **PR**: stacked on or following Phase 1

### Phase 3 — Frontend: admin UI + report columns

19. **Admin quiz section form** (`components/Admin/CourseManagement/SectionForms/QuizSection.vue`): add `kind` dropdown (Pre / Post / Standalone)
20. **Report columns** (`composables/useReportTableColumns.ts`): add three columns conditional on `staff_type === 'essential_staff'` (or always; cheap if scores are null)
21. **Custom display component(s)** if needed (e.g., `ReportsImprovementCell.vue` to show the signed `+40 pp` formatting)
22. **Tests**: Vitest unit test for the new display component if added; Playwright E2E test for the admin form change is optional
23. **PR** on the FE repo, target `staging`

### Phase 4 — Deployment + backfill

24. **Deploy Phase 1 + 2 to staging** via merge to `staging`. Migrations run automatically via Forge.
25. **Run backfill on staging**: `sail artisan app:backfill-quiz-attempts` (or `php artisan` on the staging droplet)
26. **Deploy Phase 3 to staging**, validate end-to-end in the staging UI
27. **Admin task**: an admin (Hans or someone with access) needs to tag existing `quiz_sections` with their correct `kind`. Otherwise the report shows null for users who only have `standalone` quizzes.
28. **Verify on staging** with Dan, then promote to production via release PRs
29. **Run backfill on production** after the migrations deploy

## Validation

- Backend tests pass: every new write site has a test, attempt-selection logic has a unit test, backfill has a test
- Staging smoke test: pick a known essential staff user who completed training, manually verify pre/post scores in the report match their actual quiz submissions
- Negative case: user who never took the post-quiz shows null in `post_quiz_score` and null in `quiz_improvement_pct` (not zero, not "−40")
- Backfill idempotency: running the command twice produces the same `quiz_attempts` state

## Side effects

- **Performance**: `quiz_attempts` will accumulate one row per quiz submission per user. With ~120k users and 2 quizzes per course, expect ~240k rows for a full school year. Indexed lookups are fast. Nightly aggregation adds a `quiz_attempts` join, but the join is on indexed columns and scoped per user — negligible impact on the `app:reports` runtime.
- **`lesson_progresses.quiz_data` JSON stays in place**. The existing threshold-check code in `CourseService::answerQuiz()` continues to read it. No regression risk on the quiz-passing gate.
- **Admin onboarding**: existing `quiz_sections` default to `Standalone`. An admin needs to retag them as `Pre` / `Post` for the report to populate. Until that happens, the new report columns show null for users on those courses. This is documented behavior, not a bug.
- **CI tests run on SQLite** (per `CLAUDE.md`). All new migrations and queries verified portable to SQLite via the test suite.
- **No breaking API changes**. New fields are added to existing response shapes; old fields are unchanged. No FE-side breakage if FE deploys before API.

## Follow-ups (not in scope for this task)

- **Per-user-per-course report rows**: if Dan ever wants to see "this user's pre/post on Course A, and separately on Course B," create a `reports_user_course_overview` table and a corresponding FE view. The data is already structured for this once `quiz_attempts` is in place.
- **Retire JSON quiz storage**: `lesson_progresses.quiz_data` becomes redundant once `quiz_attempts` is the source of truth. Migration path: switch the threshold-check in `answerQuiz()` to read from `quiz_attempts`, then drop the JSON column. Probably 1-2 hours of work for someone already in the codebase, plus a migration.
- **Per-question analytics**: `quiz_attempts.choices` could be stored (currently we capture only the score). If Dan wants to know "which questions are users getting wrong most often?", that's a small extension once `quiz_attempts` exists.
- **Attempt-selection tuning**: if "first attempt for pre" or "passing attempt for post" turns out wrong for Dan's mental model, the rules are isolated to one method in `TrainingReportService` and can be tweaked without schema changes.
- **Pre/post tagging of existing quizzes**: a heuristic-based one-shot command could try to auto-tag (e.g., quizzes whose title contains "pre" → `Pre`, "post" → `Post`). Worth doing only if there's a recognizable convention in the existing data — needs investigation against the actual `quiz_sections.title` rows.

## Outcome

To be filled in once the work is complete.
