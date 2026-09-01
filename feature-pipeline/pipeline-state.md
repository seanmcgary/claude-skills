# Pipeline State — the handoff contract

`plan-feature` and `build-feature` are two independent skills joined by one artifact: a
`## Pipeline State` block kept in the **issue's plan comment**, edited in place. `plan-feature`
writes it; `build-feature` reads it, then keeps it current. The `status:*` labels are a coarse
public mirror of this block, never a replacement for it.

This file is the single definition of that block. Both skills reference it; neither redefines it.

## Format

```markdown
## Pipeline State

| Field         | Value                                            |
|---------------|--------------------------------------------------|
| stage         | 3 (implementation)                               |
| class         | standard (adds a public route, contained)        |
| profile       | frontend                                         |
| issue         | #69                                              |
| branch        | feat/hardiness-zone-tool                         |
| pr            | #76 (draft)                                      |
| findings comment | 5495839830                                    |
| design assets | docs/design/hardinessZone.dc.html                |
| round         | 0                                                |
| decisions     | 2 (see Decisions below)                          |
```

When `decisions` is non-zero, a `### Decisions` list follows the table in the same comment:

```markdown
### Decisions

- **Task 3 — checkout request invalidation.** Plan: `clearCurrentCheckout()` clears state and
  nothing else. Did: added a request-generation counter that invalidates in-flight checkout
  requests. Why: an earlier `createCheckout` could settle after the clear and restore
  `paymentClientSecret`, leaking a payment credential across sessions.
```

## Fields

- **`stage`** — current phase, number + name. `plan-feature` owns `1 (spec + plan)` and
  `2 (plan review)`; `build-feature` owns `3 (implementation)`, `4 (commit review)`,
  `5 (pr feedback loop)`. Both RESUME from this value rather than starting over.
- **`class`** — the Right-Sizing class (`small` / `standard` / `large`) plus a one-line reason.
  Set by `plan-feature` at the start of planning; consumed by `build-feature` to weight
  implementation and commit review. See `conventions.md`.
- **`profile`** — active domain profile(s): `backend` (default), `frontend`, or
  `frontend+backend+seam`. Set by `plan-feature`; both skills load the matching slices from
  `profiles/`.
- **`issue`** — the GitHub issue driving the run. Always present: both skills are issue-mode
  only.
- **`branch`** — the feature branch. **Created by `plan-feature` at the start of planning, and
  it may already carry commits** (design assets). `build-feature` CHECKS IT OUT; it does not
  create it unless the branch genuinely does not exist on origin.
- **`pr`** — the pull request. `#<n> (draft)` when `plan-feature` opened one to carry design
  assets — `build-feature` PROMOTES that PR (`gh pr ready`) rather than opening a second one.
  `n/a` when no design assets were synced, in which case `build-feature` opens the PR itself at
  stage 5.
- **`findings comment`** — the numeric ID of the review's **findings comment** (see
  [`review-findings.md`](review-findings.md)), or `n/a` before a review has run. Written by
  `producing-review-findings`; read by `executing-review-findings`, which fetches exactly that one
  comment via `gh api repos/{owner}/{repo}/issues/comments/<id>`. **The ID is the field's whole
  purpose.** A pull request under review carries bot reviews, human comments, and earlier rounds,
  so an executor that lists and searches instead of fetching by ID will eventually fix the wrong
  round — silently, because every round looks alike. When a round overflows the comment size cap
  and is split into parts, this names the **manifest**, never a part.
- **`design assets`** — newline- or comma-separated repo paths of every design source committed
  for this feature, or the literal `none`. **`none` is a positive assertion by the planner that
  this feature has no visual surface** — it is what lets `build-feature` distinguish "nothing to
  match" from "someone forgot to sync the design." An empty or missing value is neither, and
  `build-feature` treats it as a blocker on any feature with UI.
- **`round`** — current feedback round within stage 5 (`0` before stage 5).
- **`decisions`** — how many entries the `### Decisions` list holds (`0` when there are none).
  Every deviation `build-feature` makes from the approved plan is logged there rather than
  parked — see build-feature's Autonomy contract. Each entry names the task, what the plan
  said, what was done instead, and why, in that order. The list is the source the PR body's
  "Deviations from the approved plan" section is written from, so it must be complete before
  the PR is readied: the human reviews these decisions at the PR, not at the moment they are
  made.

## Resume

On invocation, read this block and resume from `stage`. `plan-feature` refuses anything at
stage >= 3 (that run belongs to `build-feature`); `build-feature` refuses anything below
stage 3 with no gate label (that run belongs to `plan-feature`).

If the block is absent entirely, `plan-feature` initializes it. `build-feature` does **not**
reconstruct a missing block — it refuses and directs the user to `plan-feature`, because the
fields it would have to guess (`branch`, `design assets`) are exactly the ones that cause
silent data loss when guessed wrong.
