---
name: ship-frontend
description: Use when taking a frontend/UI feature from idea or spec all the way to a review-ready pull request. Same pipeline as ship-feature, with the frontend profile (visual verification, design review, accessibility) pinned.
---

## Overview

This is **`ship-feature` with the `frontend` profile pinned.** It is not a separate pipeline. Follow `ship-feature` exactly — its five stages, the single human gate, the autonomy contract, right-sizing, model tiering (senior planner on `claude-opus-5`, mid-level executor on `claude-sonnet-5`, senior reviewers on `claude-opus-5`), the review cadence, and team mode are all inherited unchanged.

The only difference: the active profile is **`frontend`**, not the default `backend`. Skip ship-feature's profile-inference step — the profile is pinned by this entry point.

## What to do

1. Invoke `ship-feature` and run its pipeline as written.
2. Set the Pipeline State `profile` field to `frontend`.
3. At the start of stages 1, 3, and 4, read `ship-feature/profiles/frontend.md` and apply its matching slice (planner guidance / executor verification pack / reviewer rubric).

Do **not** re-specify the models, gate, cadence, or stage sequence here — they come from `ship-feature`. If the change turns out to also involve backend/server work (a new API route, business logic, a schema change) to make the UI function, that is a full-stack change: use `ship-fullstack` instead so the backend profile and the seam addendum are also applied.
