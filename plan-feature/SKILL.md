---
name: plan-feature
description: Use to plan a feature inside a GitHub issue — premise check, brainstorming, spec, and implementation plan, reviewed and taken through the human gate — ending with the issue flagged `status:ready-for-execution`. The interactive planning half of ship-feature; hand off to build-feature for hands-off execution.
---

## Overview

This is **`ship-feature` stages 1–2 (Spec + Plan, then Plan Review + Human Gate) run in issue mode.** It does the human-interaction-heavy planning of a feature and STOPS at the gate — it does not implement. Follow `ship-feature` exactly for those two stages; everything it defines — the premise & blast-radius check, right-sizing, profiles, model tiering (planner/reviewers on `claude-opus-5`), the review cadence, and issue mode — is inherited unchanged.

**Requires a GitHub issue.** plan-feature always runs in ship-feature's **issue mode**: the issue is the coordination surface. The spec and plan live in the issue, and every clarifying question and the gate happen as issue comments (never the CLI), per ship-feature's Issue-mode rule. If the invocation references no issue, ask the user for one (or offer to create it) before planning.

plan-feature is one half of a pair. It ends by flagging the issue `status:ready-for-execution`; **`build-feature`** picks up from there and executes autonomously. All the synchronous human interaction lives here, so the build phase can run unattended.

## What to do

1. Invoke `ship-feature` and run **only stages 1 and 2**, in issue mode:
   - **Stage 1:** run the premise & blast-radius check, detect and record the profile, brainstorm *in the issue*, write the spec and the plan as issue comments (the plan keeps the full house style), and initialize the `## Pipeline State` block in the issue's plan comment.
   - **Stage 2:** review the plan (profile-aware `reviewing-plans`, reviewers on `claude-opus-5`), apply fixes to the plan comment in place, and present the gate *as an issue comment*, then await the human's approval reply in the issue.
2. On approval, apply the `status:ready-for-execution` label to the issue (ship-feature stage 2 step 5; create the label if it doesn't exist). **STOP — do not proceed to implementation.**
3. Report that the issue is planned and flagged for execution, and that `build-feature` will execute it.

Do **not** run stages 3–5 — that is `build-feature`'s job. Do not re-specify models, the gate, profiles, or the review cadence here; they all come from `ship-feature`.
