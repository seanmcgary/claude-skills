---
name: build-feature
description: Use to execute a feature that has already been planned and flagged `status:ready-for-execution` in a GitHub issue, autonomously through to a review-ready pull request — implementation, commit review, and PR feedback. The hands-off execution half of ship-feature; after the flag it only requests human input on a blocker it cannot resolve on its own.
---

## Overview

This is **`ship-feature` stages 3–5 (Implementation, Commit Review, PR Feedback Loop) run in issue mode, with NO human gate.** The gate already happened during `plan-feature`, and the issue's `status:ready-for-execution` label IS that approval. build-feature reads the approved plan from the issue and drives it to a review-ready PR with minimal human interaction. Follow `ship-feature` exactly for those three stages; profiles, model tiering (executor on `claude-sonnet-5`, reviewers/orchestration on `claude-opus-5`), and the review cadence are inherited unchanged.

**Precondition — the target issue MUST carry `status:ready-for-execution`.** That label is build-feature's authorization to execute, exactly as ship-feature's gate-approval token authorizes the Implementer. If the issue lacks the label, or has no approved plan, REFUSE and direct the user to `plan-feature`. Never plan, never re-open the gate here. (Normally the plan and a `## Pipeline State` block were written by plan-feature; if the plan was written into the issue by hand with no Pipeline State block, reconstruct one — infer the profile, assess the right-sizing class — before starting.)

## Autonomy: hands-off after the flag

Once the issue is flagged, proceed **fully autonomously** through stages 3–5. Do NOT ask permission to run tests, push, open the PR, or address a bot finding — triage and proceed per the sub-skills. The ONLY reason to stop and request human input is a blocker you cannot resolve on your own — ship-feature's Autonomy Contract, here comprising:

- an implementation **BLOCKED** status you cannot fix (the plan is wrong or impossible, or a task cannot be completed as specified),
- an **ambiguous or contentious** human comment (on the PR or the issue),
- a required **architecture deviation** from the approved plan, or
- the **PR round cap (N=3)** reached with actionable findings still open.

When you hit a blocker: post it as an issue comment, set `status:blocked` on the issue (removing `status:ready-for-execution`; create the label if missing), and STOP with a status report. The run resumes when a human resolves the blocker and re-applies `status:ready-for-execution`.

## What to do

1. **Check the precondition and load state.** Confirm the issue has `status:ready-for-execution`. Read the approved plan and the `## Pipeline State` block (profile, class, branch) from the issue, and materialize a local, UNCOMMITTED working copy of the plan for the executor tooling (`task-brief` / `review-package`). Ensure the feature branch (name from Pipeline State) exists — create it from the default branch if this is a fresh session/checkout; issue-mode planning produced no commits, so nothing is lost by (re)creating it.
2. **Invoke `ship-feature` and RESUME at stage 3,** treating `status:ready-for-execution` as the gate-approval token:
   - **Stage 3:** execute the plan (implementer subagents on `claude-sonnet-5`), applying the profile's executor slice and the Review Cadence (per-task review only for `review: yes` tasks; no separate SDD final review).
   - **Stage 4:** the one authoritative whole-diff fan-out — profile-aware `reviewing-commits`, reviewers on `claude-opus-5`, scope-bounded re-reviews.
   - **Stage 5:** open the PR (linking the issue via `Closes #<issue>`), flip the label `status:ready-for-execution` → `status:ready-for-review`, and run `pr-feedback-loop` (N=3).
3. End with the final status report. **Never merge** — that stays a human decision.

Do not re-specify models, profiles, or the cadence here; they come from `ship-feature`.
