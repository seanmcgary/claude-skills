---
name: ship-fullstack
description: Use when taking a feature that spans both UI and server (e.g. a new screen plus the API route, business logic, or database change behind it) from idea or spec to a review-ready pull request in a repo, without a driving GitHub issue. Same pipeline as ship-feature, with the frontend and backend profiles plus the seam addendum pinned. Best for light-to-moderate backend work (tweaking a schema, adding routes and business logic) alongside UI — not for architecting a new backend subsystem.
---

## Overview

This is **`ship-feature` with both the `frontend` and `backend` profiles pinned, plus the `seam` addendum.** It is not a separate pipeline. Follow `ship-feature` exactly — its five stages, the single human gate, the autonomy contract, and everything it inherits from `$SKILLS_ROOT/feature-pipeline/conventions.md` (right-sizing, model tiering, review cadence) apply unchanged.

The only difference: **both** profiles are active, and the `seam` addendum (the client/server contract) applies on top. Skip ship-feature's profile-inference step — the profiles are pinned by this entry point.

**Issue-driven work does not belong here.** If a GitHub issue is driving the feature, use `plan-feature` then `build-feature`; they infer `frontend+backend+seam` from the touched surfaces and additionally sync the design source into the repo before the gate.

"Full-stack" here means a feature with a UI and the light-to-moderate backend it needs — a new route, business logic, a schema tweak — not a from-scratch backend subsystem. The **weight** of the backend part is handled by ship-feature's right-sizing (a small schema tweak is a Small/Standard change), independent of the fact that both profiles are on. If the backend part is genuinely a new subsystem, treat it as Large and decompose per ship-feature stage 1.

## What to do

1. Invoke `ship-feature` and run its pipeline as written.
2. Set the Pipeline State `profile` field to `frontend+backend+seam`.
3. At the start of stages 1, 3, and 4, read `$SKILLS_ROOT/feature-pipeline/profiles/frontend.md`, `$SKILLS_ROOT/feature-pipeline/profiles/backend.md`, and `$SKILLS_ROOT/feature-pipeline/profiles/seam.md`, and apply **all three** matching slices:
   - **Planner (stage 1):** author both frontend and backend task shapes; run right-sizing with both trigger sets; fix the API contract first (seam) so both sides build to it.
   - **Executor (stage 3):** verify each task per its profile (tests for server tasks, render/screenshot for UI tasks), and verify the seam **end-to-end** — real UI action → real API → real data layer.
   - **Reviewer (stages 2 & 4):** run the combined rubric — backend security + frontend a11y/responsive/design-fidelity/client-security + the seam's contract-consistency/error-propagation/validation-parity dimensions.

Do **not** re-specify the models, gate, cadence, or stage sequence here — they come from `ship-feature` and `feature-pipeline/conventions.md`.
