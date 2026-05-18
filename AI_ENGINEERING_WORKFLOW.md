# AI Engineering Workflow

> A personal cognitive framework for software development with AI assistance.

---

## Interpretation & Usage

This document defines how I (the human engineer) approach problem solving, decision-making, and software development when working with AI.

It is not a set of instructions for the AI to execute independently.

Claude should interpret this document as:

- a description of the human's workflow
- a set of constraints that guide collaboration
- a definition of how decisions are made and validated

The human remains responsible for:

- defining problems
- validating assumptions
- making architectural decisions
- approving all production changes

Claude's role is to assist within this framework — not to replace it.

---

## Overview

This workflow is designed to maintain full control over system behavior while leveraging AI to accelerate analysis and implementation.

The objective is not speed alone, but **correctness, clarity, and system integrity**.

---

## Core Principles

- **AI is an assistant, not the decision-maker**
- All critical logic must be understood before being changed
- Every fix must be traceable: `problem → cause → solution`
- Production behavior is the source of truth
- Speed is secondary to correctness and system integrity

---

## Workflow

### 1. Problem Definition

- Clearly describe the issue
- Capture all relevant context (logs, environment, dataset, etc.)

### 2. System Understanding

- Identify involved components (API, DB, cache, jobs, etc.)
- Trace data flow end-to-end before making assumptions

### 3. Hypothesis

- Define possible root causes
- Validate assumptions using logs, code inspection, or controlled tests

### 4. AI-Assisted Analysis

Use AI as a tool to:

- explore code paths
- surface edge cases
- generate hypotheses for validation
- review logic critically

Do not accept AI output blindly — all reasoning must be validated.

### 5. Fix Strategy

- Define the minimal, safe change required
- Prefer low-risk solutions in production systems
- Avoid unnecessary refactors during incident resolution

### 6. Validation

- Confirm behavior before and after the fix
- Test in staging when possible
- Validate in production when necessary

### 7. Documentation

- Record root cause, fix, and impact
- Update task documentation for traceability

---

## Task Documentation Template

> Spec Driven Development (SDD)

```markdown
## Problem

## Context

## Expected Behavior

## Observed Behavior

## Root Cause

## Fix Strategy

## Validation

## Outcome
```

---

## AI Usage Guidelines

AI is used as a thinking and analysis tool, not as an autonomous system.

AI is encouraged to propose ideas, alternatives, and optimizations — but always within a collaborative, human-validated process.

**Use AI to:**

- Analyze code and execution flow
- Explore possible hypotheses
- Review logic and edge cases
- Assist in drafting structured solutions

**Do not rely on AI to:**

- Generate production code without understanding it
- Make architectural decisions without human validation
- Apply changes without review and verification

> AI can propose solutions — the human must decide and validate.

---

## Quality Standards

All changes must:

- Be understandable
- Be traceable
- Minimize side effects
- Maintain backward compatibility
- Include validation steps

---

## Artifacts

| File                | Purpose                                |
| ------------------- | -------------------------------------- |
| `CLAUDE.md`         | Project context and AI behavior rules  |
| `QUALITY_RUBRIC.md` | Code quality and engineering standards |
| `Task documents`    | Per-issue structured documentation     |
| `PROJECT_STATUS.md` | Overall system and project state       |

---

## When to Use This Approach

This workflow is especially useful for:

- Debugging complex systems
- Working with legacy code
- Production issue resolution
- Multi-component systems (API, DB, cache, jobs)

---

_Human in the loop. Always._
