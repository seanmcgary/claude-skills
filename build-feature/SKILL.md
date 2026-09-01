---
name: build-feature
description: Use to execute a feature that has already been planned and flagged `status:ready-for-execution` in a GitHub issue, autonomously through to a review-ready pull request — implementation, commit review, and PR feedback. The execution half of the feature pipeline; after the flag it only stops on a blocker it cannot resolve on its own.
---

## Overview

`build-feature` takes an approved plan from a GitHub issue to a review-ready pull request. It
runs standalone — it does not delegate to `ship-feature` — and it runs **hands-off**: the human
gate already happened in `plan-feature`, and the `status:ready-for-execution` label IS that
approval.

Shared reference — **read these at the paths below** (they are siblings of this skill, under
`$SKILLS_ROOT/feature-pipeline/`; see the portability section in `conventions.md`); do not
redefine their contents here:

- `$SKILLS_ROOT/feature-pipeline/conventions.md` — profiles, right-sizing, model tiering,
  review cadence, test cadence, no-bot repos, owner notifications
- `$SKILLS_ROOT/feature-pipeline/pipeline-state.md` — the handoff contract with
  `plan-feature`
- `$SKILLS_ROOT/feature-pipeline/profiles/{backend,frontend,seam}.md` — per-domain slices

**Model tiering:** orchestration and review triage at the `senior` tier (high effort); reviewer
subagents at `senior`; per-task implementer subagents at `mid`. Resolve each model per
`$SKILLS_ROOT/feature-pipeline/tiers.md` and pass it explicitly when dispatching — or omit it
to inherit the invoking session's model when resolution falls back to `inherit`.

## Preconditions

1. **The issue carries `status:ready-for-execution`.** That label is your authorization to write
   code. Without it, REFUSE and direct the user to `plan-feature`. Never plan here, never
   re-open the gate, never apply the label to yourself.
2. **The issue has an approved plan with a `## Pipeline State` block.** If the block is missing,
   REFUSE — do not reconstruct it. The fields you would have to guess (`branch`,
   `design assets`) are precisely the ones that cause silent data loss when guessed wrong.

## Phase 0 — Load state and resolve the branch

Read the plan and the `## Pipeline State` block: `class`, `profile`, `branch`, `pr`,
`design assets`, `stage`. Load the profile file(s). Materialize a local, UNCOMMITTED working
copy of the plan for executor tooling (`task-brief` / `review-package`).

**Resolve the branch — this is where work gets destroyed if you get it wrong:**

```
git fetch origin
```

- **If `branch` exists on origin: check it out.** It may already carry commits — `plan-feature`
  creates the branch during planning and commits design assets onto it. Re-creating it from the
  default branch silently discards them, and the build then has no design to match.
- **Only if it genuinely does not exist on origin**, create it from the default branch.
- Then rebase onto the default branch before new work. **If the rebase conflicts, STOP** and
  report the conflicted files — never force, never guess a resolution.

**Resolve the PR:** if `pr` names a draft, that PR is yours to finish — push to its branch and
promote it at stage 5. Do **not** open a second PR for the same branch. If `pr` is `n/a`, you
open the PR at stage 5.

## Phase 1 — Design fidelity (before you write UI code)

Every design source this plan depends on is **already a committed file on your branch**, listed
in Pipeline State's `design assets` and cited by path in the plan. Read those files and
implement what they specify.

- **You do not fetch design.** No browser, no design tool, no design project. That sync happened
  during planning, deliberately, so it could be reviewed before the gate.
- **You do not reconstruct a design** from prose, screenshots, the issue title, or taste.
- **If `design assets` is `none`,** the planner has asserted this feature has no visual surface.
  Proceed.
- **If the plan says "match the design" and the cited file is missing — or no path is cited, or
  `design assets` is empty on a feature that plainly has UI — that is a BLOCKER.** Stop and ask
  (see Autonomy contract). Shipping an approximation is a defect, not a partial success: it
  costs a full rework round and reads to the reviewer as if the design was ignored.

## Phase 2 — Implementation

Execute the plan's checkbox tasks per the Right-Sizing class, applying the profile's **executor
slice** to verify each one:

- **Standard/Large:** `subagent-driven-development` — a fresh implementer subagent per task at
  the `mid` tier (batch only trivial tasks).
- **Small:** execute inline with `executing-plans`.

Apply the **Review Cadence** from `conventions.md`: dispatch a per-task reviewer ONLY for tasks
the plan tagged `review: yes` (at the `senior` tier); `review: no` tasks are gated by their own
acceptance criteria. Do **not** run subagent-driven-development's separate final whole-branch
review — the single authoritative fan-out is phase 3.

Verification is domain-specific, and per task it runs **targeted tests only** — the new and
affected tests plus format/lint/typecheck; the full suite runs once at phase 3 (see the "Test
cadence" section in `conventions.md`). Backend: TDD + targeted tests green; frontend
additionally requires
**observing the rendered result** — run the app, drive the real flow, screenshot at every
breakpoint the plan named, check against the task's acceptance criteria, and tab through for
keyboard reachability and visible focus. A visual task is never done on unit tests alone; if the
environment cannot render the UI at all, that is a blocker to surface.

Update Pipeline State on completion — including any `### Decisions` entries for plan deviations
made during the tasks.

## Phase 3 — Commit review

This is the **one authoritative whole-diff fan-out**.

- **Standard/Large:** invoke `reviewing-commits` on the feature branch — format, lint and
  typecheck, then three parallel **profile-aware** reviewers over the branch diff at the
  `senior` tier, triage into a findings artifact, then one fresh subagent per file group
  applying it. The **full** test suite runs once at the end of that (the run's one full-suite
  pass; per-task runs were targeted). Fold the fixes in as commits. Re-reviews are
  scope-bounded to each fix, never a fresh whole-diff pass.
- **Small:** mechanical gates plus a real self-review against the profile's reviewer rubric.

**Check for a PR review bot before deciding the weight** (see `conventions.md` → repositories
without a PR review bot). If the repo has none, "the bot will catch it" is not available as a
reason to review less: run the fan-out, or at absolute minimum gates + a genuine self-review.
Never leave a diff with no review at all.

Update Pipeline State on completion. A triaged finding whose fix departs from the approved plan
is a `### Decisions` entry, not a park — this phase is where most of them are made.

## Phase 4 — PR + feedback loop

**Ready the PR:**

- **If Pipeline State names a draft `pr`:** push the branch, update the PR body to the full
  description (format: `$SKILLS_ROOT/../commands/generate-pr-description.md`, the host's
  PR-description generator), and promote it —
  `gh pr ready <n>`.
- **If `pr` is `n/a`:** push and open it now — `gh pr create --assignee seanmcgary` with the full
  description, linking the issue via `Closes #<issue>`.

Either way: if the `### Decisions` list has entries, the PR body carries a **"Deviations from
the approved plan"** section built from it — one bullet per entry, each naming what the plan
said, what you did, and why. This is the whole payoff of deciding rather than asking: the human
sees every judgment call in one place, next to the diff that implements it. A run with
deviations and no such section has hidden them.

Then: assign `@seanmcgary`, @-mention him in the ready-for-review comment, and flip the
issue's labels — remove `status:ready-for-execution`, add `status:ready-for-review`.

**Run the feedback loop** (`pr-feedback-loop`, N=3 default). Triage findings, apply fixes, reply
to threads, repeat until a clean round or the cap.

**In a repo with no review bot, the only signals are CI status and human comments** — do not
poll for a bot review that will never arrive. And in every repo, **do not block on slow CI for a
clean-round exit**: wait for what actually drives the loop and for any check required for the
exit decision; report completion with CI noted as in-progress rather than sitting in a blocking
poll on multi-minute jobs.

Update Pipeline State after each round, and end with a status report.

## Autonomy contract

After the gate, proceed **without asking permission**. Do NOT ask whether to run tests, push,
open the PR, or address a finding — triage and proceed. Stop only for a blocker you cannot
resolve on your own:

1. **An implementation blocker you cannot fix** — the plan is wrong or impossible, or a task
   cannot be completed as specified. "The plan did not anticipate this" is not that; see
   *Deviating from the plan* below.
2. **An ambiguous or contentious human comment** on the PR or the issue.
3. **The round cap reached** with actionable findings still open.
4. **A missing or uncited design artifact** — the plan requires matching a design and the file
   isn't there (see phase 1).

### Deviating from the plan — decide, log, surface

**A deviation from the approved plan is NOT a blocker. Decide it yourself.** Approving the plan
authorized the outcome, not a literal transcript: no plan specifies everything, so a review
finding whose fix the plan does not describe is the normal case, not an exception. Adding
middleware the plan omitted, changing a bootstrap order, introducing a request-invalidation
scheme to close a defect review found — all of these you resolve on your own, however structural
they are.

For each one, **log it and keep going**:

1. Append an entry to the `### Decisions` list under Pipeline State — the task, what the plan
   said, what you did instead, and why (see `pipeline-state.md`). Write it when you make the
   decision, not from memory at the end.
2. Bump the `decisions` count in the Pipeline State table.
3. At phase 4, render the whole list as a **"Deviations from the approved plan"** section in the
   PR body.

The human reviews these **at the PR**, with the diff in front of them — which is a better review
than the same question answered blind in an issue comment, and it does not cost a full dispatch
of latency. Surface the decision; do not ask for it.

The one thing a deviation may never do is quietly widen scope. If your fix takes the feature
somewhere the issue did not ask to go, log that plainly in the same entry and say so in the PR
body — an unflagged scope change is what this rule is protecting against, not the deviation
itself.

When you hit one: commit and **push** your work first, then post the blocker as an issue comment
@-mentioning `@seanmcgary`, set `status:needs-execution-input` (removing
`status:ready-for-execution`), and STOP with a status report. The run resumes when a human
resolves it and re-applies `status:ready-for-execution`.

**Never merge.** That stays a human decision.

## Red flags

1. **"The branch doesn't matter, I'll just make it from master."** This deletes the design
   assets `plan-feature` committed. Check out the remote branch.
2. **"There's no design file, but I can tell what it should look like."** You cannot. That is
   the exact failure mode this pipeline was restructured to prevent. Park and ask.
3. **"I'll open a PR."** Check `pr` first — if a draft exists, promote it. Two PRs for one branch
   is a mess someone else has to clean up.
4. **"The tests pass so the code is fine."** Passing tests prove the happy path. Commit review
   exists to catch what tests miss: security boundaries, convention violations, missing error
   paths, a11y gaps, contract drift.
5. **"I'll wait for the bot review."** Check whether this repo has one. If it doesn't, you will
   wait forever.
6. **"PR is open, my job is done."** Opening the PR starts the feedback loop; it does not end
   the run.
7. **"The plan doesn't cover this, so I should park and ask."** Almost always wrong. Parking
   costs a full human round-trip to answer a question the reviewer could have answered from the
   diff. Decide it, log it under `### Decisions`, surface it in the PR body. Park only for the
   four blockers above.
8. **"I made the call, so it doesn't need writing down."** An undocumented deviation is worse
   than parking: the human never learns the plan was departed from, and reviews the diff
   believing it matches what they approved.
