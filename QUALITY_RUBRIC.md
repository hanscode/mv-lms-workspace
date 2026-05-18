# QUALITY_RUBRIC.md

> Quality criteria for implementing, reviewing, and validating changes in mv-lms.

---

## Purpose

This document defines the quality standards for all changes in this project.

It ensures that work is:

- correct
- understandable
- safe
- maintainable
- aligned with the system's architecture

This rubric is **practical, opinionated, and grounded in real production constraints**.

---

## Core Principles (Non-Negotiable)

### 1. Understand Before Changing

- Read the existing code fully before making changes
- Trace execution paths (controller → service → frontend → side effects)
- Identify which repo owns the behavior before writing a line
- Never assume behavior — verify it in code

> If the current behavior is not understood, changes must not proceed.

---

### 2. KISS — Keep It Simple

- Prefer the simplest solution that works
- Avoid unnecessary abstractions or indirection
- Prefer explicit logic over "smart" solutions

---

### 3. DRY — Applied Judiciously

- Extract shared logic only when duplication is real and stable
- Avoid premature abstraction
- Duplication is acceptable during exploration or isolated fixes

> Duplication is safer than the wrong abstraction.

---

### 4. Avoid Over-Engineering

- Do not build for hypothetical future use cases
- Do not expand scope beyond the task
- Do not introduce new patterns without strong justification

---

### 5. Narrative Flow

- Code should read clearly from top to bottom
- Meaningful, self-documenting names
- Comments explain *why*, not *what*

---

## Repo Boundary Safety

Every change must begin with:

- Which repo owns this behavior — API or Frontend?
- Does this change cross the repo boundary?
- If cross-repo: are both sides being updated atomically?

**Never assume the other repo will "figure it out."** Mismatched API contracts are a common source of silent bugs.

---

## Backend Standards (Laravel API)

### Controllers

- Keep controllers thin — move logic to `app/Services/*`
- No business logic inside controllers
- Access control via middleware, not in-controller checks

### Services

- One responsibility per service
- Services are the source of truth for business rules
- Side effects (emails, jobs, events) belong in services, not controllers

### Access Control & Geo Scope

Every change must evaluate:

- Who can call this endpoint?
- What geo scope does the user have access to?
- Does the response correctly filter to the user's scope?

Any ambiguity here is a **Blocker**.

### Reports

- Report endpoints read from `reports_*` pre-computed tables
- Never add real-time aggregation to report endpoints
- If new report data is needed, add it to the `app:reports` nightly command
- Test report changes locally with `php artisan app:reports` or `php artisan app:fake`

### Scheduled Jobs

- Changes to scheduled commands must consider: what happens if the job runs twice?
- Idempotent where possible
- Failures must be logged and scoped — one failure must not block other jobs

### Database

- Migrations must be backward-compatible where possible
- Do not use MySQL-specific SQL — CI runs on SQLite
- Verify SQL portability before committing
- `./vendor/bin/pint` must pass before committing

### Excel Exports

- Exports are queued jobs — they must not block the request
- Verify the job writes correctly to storage and emails the requesting user
- Test with a real user account locally before merging

---

## Frontend Standards (Nuxt 3 / Vue 3)

### Composables

- New composables in `.ts` files — not `.vue` files
- Use `useSanctumFetch` and `useSanctumUser` — not legacy composables
- Handle errors via `composables/useApiError.ts`

### Components

- Keep components focused and readable
- Reach for `components/Admin/ListLayout/` before hand-rolling tables
- Avoid unnecessary re-renders
- Keep business logic out of UI components — move to composables or stores

### State (Pinia)

- One store per domain area
- Stores should not contain UI logic
- Side effects (API calls) belong in store actions, not component methods

### Forms

- New forms use `vee-validate` + `yup` — not plain refs
- Validation messages must be user-friendly, not technical

### TypeScript

- TypeScript is loose (`strict: false`) — don't over-invest in types for existing code
- New code should have reasonable type coverage
- Avoid `any` in new code unless justified

---

## Testing

### Backend (PHPUnit)

- Add tests when behavior is critical or error-prone
- Test the service layer, not just HTTP responses
- Be aware: CI uses SQLite, local uses MySQL — verify portability
- Cover: happy path, edge cases, access control violations

### Frontend

- Vitest for unit tests on composables and utilities
- Playwright for E2E flows that involve auth or multi-step interactions
- The `auth.setup.ts` E2E fixture exercises magic-link login — keep it working

---

## Pull Request Quality Bar

Every change must satisfy:

- [ ] Is the intent clear?
- [ ] Does it stay within scope?
- [ ] Is repo ownership clear — did we touch the right repo(s)?
- [ ] Is geo-scope access control safe?
- [ ] Are report endpoints still reading pre-computed data only?
- [ ] Does it simplify or clarify the system?
- [ ] Are side effects understood (emails, jobs, events)?
- [ ] Does the SQL work on both SQLite and MySQL?

If any answer is "no", the change requires revision.

---

## When to Refactor vs When to Stop

Refactor when:

- behavior is understood
- complexity is reduced
- improvement is clear and contained

Stop when:

- changes spread into unrelated areas
- complexity increases
- solving hypothetical problems

> A good change simplifies the system.

---

## Red Flags

Be cautious if a change:

- touches `User::boot()` or user type logic (auto-detaches geo assignments)
- modifies report endpoints to add real-time computation
- changes the XSRF cookie/header names
- touches the MySQL Docker version pin
- adds logic to a controller instead of a service
- introduces a new `.vue` composable instead of `.ts`
- bypasses the `nuxt-auth-sanctum` module for auth

---

## Final Guiding Principle

This system supports real training workflows used by school district staff across the US.

Clarity > Cleverness
Correctness > Speed
Stability > Novelty

If a change improves these, it is likely the right decision.

---

*Human in the loop. Always.*