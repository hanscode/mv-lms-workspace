---
name: write-update-message
description: Draft a clear, professional update message for stakeholders (e.g., Dan via Teamwork or Slack)
---

# Write Update Message

## Invocation

This skill runs ONLY when explicitly invoked by the user.
Do NOT self-trigger based on perceived task completion or context.

Typical triggers are an explicit `/write-update-message` invocation, or the user
directly asking for the skill to be run (e.g. "use the update-message skill to
draft this"). Claude MAY suggest invoking it when drafting an outbound message
seems appropriate — the human decides whether to run it.

---

## Objective

Produce a **clear, concise, and professionally natural English update message** that:

- reflects the current state of the work accurately
- is easy to scan and understand
- aligns with real project context (PRs, staging, production, validation)

The message should sound like a **thoughtful engineer communicating clearly**, not like AI-generated text.

---

## Audience Awareness

Assume the audience is:

- Dan (technical, strategic)
- internal stakeholders (Teamwork / Slack)

Write accordingly:

- professional but natural
- direct but not abrupt
- confident but not absolute unless verified

---

## Language Requirement

All output MUST be:

- natural, fluent, professional English
- free of robotic phrasing
- consistent with real workplace communication

Avoid:

- overly formal or stiff language
- exaggerated claims
- unnecessary filler

---

## Structure

Use a clear and predictable structure:

### 1. Opening (optional but preferred)

- "Hey Dan," or "Hi Dan," depending on tone

---

### 2. Summary (always first)

Start with a quick, high-level statement of what changed:

- what was implemented
- what was fixed
- current status (e.g., deployed to staging)

---

### 3. Details (scannable)

Use short bullet points when helpful:

- what was done
- what the feature allows
- what changed behavior-wise

Keep bullets focused and relevant.

---

### 4. Validation / Current State

Clearly state:

- where it is deployed (staging / production)
- what has been verified
- what is ready for testing

Use phrases like:

- "ready for testing"
- "verified on staging"
- "behaves as expected"

---

### 5. Instructions (if applicable)

If testing is needed, include simple steps:

- how to access
- what to check

---

### 6. Next Step / Ask

Clearly state what you need from the recipient:

- confirmation
- validation
- approval

Examples:

- "Let me know if everything behaves as expected"
- "Once confirmed, I’ll proceed to production"

---

## Style Rules

### Do

- be concise but complete
- use clean, readable formatting
- prefer clarity over cleverness
- sound like a real engineer communicating status
- keep sentences natural and fluid
- use numbers only when they add clarity or reinforce a conclusion
- group related data points so they are easy to understand
- prefer stakeholder terminology over internal implementation terms when describing features or user-facing behavior

### Do NOT

- sound robotic or templated
- over-explain obvious context
- assume things are correct without validation
- use absolute language unless confirmed
- include unnecessary technical noise
- overload sections with too many numbers or metrics
- present raw data without explaining what it means
- prioritize completeness over readability

---

## Tone Calibration

Overall direction:

- less structured and rigid
- more conversational and natural
- avoid sounding self-congratulatory
- read like a real person writing to a colleague

Prefer:

- natural phrasing over formal wording ("I looked into" vs "I investigated")
- leading with reassurance when applicable
- soft, non-alarmist language when describing issues
- keep sections clear, but avoid overly rigid formatting
- smooth narrative flow over dense technical explanation

Avoid:

- overly formal or rigid phrasing
- unnecessarily technical or internal terms (e.g., "bug") in stakeholder communication

---

## Context Awareness

When available, incorporate:

- PR numbers
- branch names
- staging/production status
- feature scope
- limitations or constraints

If context is missing, do not invent it.

---

## Output

Return only the final message, ready to copy and paste.

Do NOT include explanations, analysis, or meta commentary.

---

## Final Rule

The message must feel like it was written by a professional engineer communicating clearly in a real project context — not by an AI.

Human tone. Clear thinking. Minimal friction.
