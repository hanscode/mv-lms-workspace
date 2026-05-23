---
name: final-strict-review
description: Run a strict final review before commit or merge
---

# Final Strict Review

## Invocation

This skill runs ONLY when explicitly called by the user via `/final-strict-review`.
Do NOT self-trigger. Do NOT run autonomously. Wait for the developer to invoke it.

## Objective

Evaluate the implementation strictly against:

- `CLAUDE.md`
- `QUALITY_RUBRIC.md`
- intended behavior / spec for the change under review

The goal is to ensure the change is correct, consistent, and production-ready.

## Repo Ownership Verification

Before reviewing code quality, first determine:

1. Which repository owns the behavior being changed — API or Frontend?
2. Whether the implementation is single-repo or cross-repo in impact
3. Whether the change touches:
   - access control or geo-scope middleware
   - report endpoints (must remain pre-computed reads only)
   - scheduled jobs or nightly aggregation
   - XSRF cookie/header configuration
   - User type logic or geo assignment side effects
   - Excel export jobs

If repo ownership or cross-repo impact is unclear, stop and state that clearly before proceeding.

## Verification Protocol

Before classifying ANY finding as Blocker or Improvement:

1. **Spec coverage check** — reread the task doc and verify each planned
   step against this PR:
   - ✅ Implemented in this PR
   - ⏭️ Explicitly deferred to a follow-up (must be stated)
   - ❌ Missing — not implemented and not acknowledged as deferred
     If any step is ❌, that is a Blocker regardless of code quality.

2. Read the actual file — locate the specific line
3. Confirm the issue exists in code, not assumption
4. Check whether it is already handled elsewhere
5. If cross-repo, verify both sides are in sync

If you cannot cite file + line number, it is NOT a Blocker or Improvement.

## Review Focus

Perform a strict review focused on:

- DRY opportunities worth fixing now
- narrative flow and readability
- consistency with existing patterns
- side effects (emails, queued jobs, events, geo-scope changes)
- access control and geo-scope safety
- report endpoint integrity — pre-computed reads only, no real-time aggregation
- SQL portability — works on both SQLite (CI) and MySQL (local + production)
- frontend/backend contract alignment (API shape matches what the frontend expects)
- Sanctum XSRF configuration untouched
- performance concerns worth addressing now
- critical testing gaps

## Classification

Classify findings as:

- **Blocker** → verified in code, must fix before commit (requires file + line citation)
- **Improvement (fix now)** → verified, low-risk, high-value (requires file + line citation)
- **Optional** → can be deferred

If no issues exist in a category, explicitly state `None`.
Do not invent issues to fill categories.

## Rules

- Do not optimize for encouragement or momentum
- Do not assume behavior — verify it in code
- Do not treat suspicious code as a bug without validation specifying file + line citation
- Do not assume single-repo impact without checking the other repo
- False positives erode trust and waste developer review time
- Prefer correctness and clarity over speed
- Avoid over-engineering during this phase

## Output

Provide:

1. Summary
2. Repo ownership / cross-repo impact
3. Blockers (with file + line)
4. Improvements (fix now) (with file + line)
5. Optional
6. Final verdict (ready / not ready)

## Final Rule

Do not recommend commit or merge unless:

- no blockers remain
- no important improvements should reasonably be addressed now
- cross-repo impact has been verified when relevant

Human in the loop. Always.
