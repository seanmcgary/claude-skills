---
name: ship-feature
description: Use when taking a feature from idea or spec all the way to a review-ready pull request in a repo, without a driving GitHub issue — spec and plan are committed as docs behind a draft PR. For issue-driven work use plan-feature (planning) and build-feature (execution) instead.
---

## Overview

A five-stage pipeline taking a feature from idea (or an existing spec) to a review-ready pull
request. There is exactly ONE human gate: after stage 2 (plan review), before any
implementation code is written. After the gate, the pipeline runs autonomously to completion
with the exceptions in the Autonomy Contract. **It never merges** — it ends with a status report.

**Scope: repo mode.** The spec and plan are committed as files under `docs/superpowers/` and
published on a **draft PR**. All human interaction — clarifying questions and the gate — happens
in the CLI.

**If a GitHub issue is driving the work, do not use this skill.** Use `plan-feature` (planning →
human gate) and then `build-feature` (execution → review-ready PR). Those two coordinate through
the issue and its `status:*` labels, keep the spec and plan as issue comments, and sync design
assets before the gate. This skill has no issue mode.

Shared reference — **read these at the paths below** (see the portability section in
`conventions.md`); do not redefine their contents here:

- `$SKILLS_ROOT/feature-pipeline/conventions.md` — profiles, right-sizing, model tiering,
  review cadence, test cadence, repos without a review bot, owner notifications
- `$SKILLS_ROOT/feature-pipeline/profiles/{backend,frontend,seam}.md` — per-domain slices

**Stages:**

1. **Spec + Plan** — verify the premise, brainstorm, write the plan in house style, commit it
   and open a draft PR.
2. **Plan Review** — review the plan; present it to the human for approval.
3. **Implementation** — execute the plan; subagents per task, or inline for small changes.
4. **Commit Review** — the one whole-diff review before push.
5. **PR Feedback Loop** — mark the draft ready, then address reviewer feedback for up to N rounds.

Profiles, Right-Sizing, Model Tiering, and the Review Cadence all come from `conventions.md`.
Select the profile per that file — or take the pin from the entry skill (`ship-frontend` →
`frontend`; `ship-fullstack` → `frontend` + `backend` + `seam`) and skip inference.

## Stage sequence

```dot
digraph pipeline {
  rankdir=LR;
  node [shape=box, style=rounded];

  spec [label="1. Spec + Plan"];
  review [label="2. Plan Review"];
  gate [label="HUMAN GATE", shape=diamond, style="filled", fillcolor="#ffcccc"];
  impl [label="3. Implementation"];
  commits [label="4. Commit Review"];
  pr [label="5. PR Feedback\nLoop (N=3)"];
  done [label="Status Report", shape=doubleoctagon];
  stop_ambig [label="STOP:\nambiguous\ncomment", shape=octagon, style="filled", fillcolor="#ffffcc"];
  stop_deviate [label="STOP:\narchitecture\ndeviation", shape=octagon, style="filled", fillcolor="#ffffcc"];
  stop_cap [label="STOP:\ncap hit +\nopen findings", shape=octagon, style="filled", fillcolor="#ffffcc"];

  spec -> review;
  review -> gate;
  gate -> impl [label="approved"];
  gate -> spec [label="rejected / rework"];
  impl -> commits;
  commits -> pr;
  pr -> done [label="clean round\nor cap reached"];
  pr -> stop_ambig;
  pr -> stop_deviate;
  pr -> stop_cap;
}
```

### Stage 1: Spec + Plan

**Skip condition:** if the user provides a path to an approved spec or plan whose
`## Pipeline State` block shows stage >= 2, skip to the indicated stage (see Resume).

**0. Premise & blast-radius check — do this FIRST, by reading code, not assumption.** Run the
shared check at `$SKILLS_ROOT/feature-pipeline/premise-check.md`: what to answer, the three
outcomes, and what is NOT a premise failure. Do not restate it here.

Specific to this skill: the human is present, so a contradiction between the user's framing and
the code is usually resolved by asking in one exchange rather than parking. Outcome 2 there —
premise holds, details are stale — is reconciled in the plan and is never a stop.

**1. Brainstorm + spec.** Invoke `brainstorming` (the host's design-exploration skill) to
explore design space, requirements, and edge cases, informed by the step-0 findings, then write
a spec in **Simplified Technical English (ASD-STE100)** — see the writing-style section in
`conventions.md`.
**Small changes:** skip both brainstorming and the written spec once the premise check is clean
and the approach is unambiguous — the plan doubles as the spec. If a spec already exists (user
provides a path, or one exists at `docs/superpowers/specs/`), skip brainstorming and go to the
plan — but still run the premise check against it. For a borderline Small/Standard change whose
approach is clear but not trivial, confirm the approach in one exchange rather than the full
one-question-at-a-time ceremony.

**2. Write the plan.** Invoke `writing-plans` (the host's plan-authoring skill), layering the
**house plan style** on top, in **Simplified Technical English (ASD-STE100)**. **The plan's required structure is
shared** — see `$SKILLS_ROOT/feature-pipeline/plan-structure.md` for the Global Constraints
preamble, `Verified external API`, the checkbox/agentic-worker header, and profile-shaped tasks.
Do not restate it here.

One requirement is specific to this skill:

- **File location:** `docs/superpowers/plans/YYYY-MM-DD-<topic>.md`.

**This skill has no design-sync phase, and that is deliberate.** It runs interactively with the
human present in the CLI, so a wrong design surfaces immediately in conversation. `build-feature`
needs `plan-feature`'s committed design assets because it runs autonomously and nobody sees the UI
until the PR. Do not import that phase here.

**3. Initialize the Pipeline State block** (see below).

**4. Publish for review.** Create the feature branch at the START of stage 1, as soon as the
premise check has fixed the topic. As each artifact is produced: commit the spec
(`docs: <topic> spec`), `git push -u origin <branch>`, and open a **draft** PR against the
default branch (`gh pr create --draft --title "<type>: <topic>" --body "spec/plan under review;
implementation follows after the human gate"`); then commit the plan (`docs: <topic> plan`) and
push. For a Small change the plan is the first artifact and is what opens the draft PR. Record
branch + PR number in Pipeline State. If the repo has no GitHub remote, commit locally and skip
the draft PR.

The draft PR does **not** start the stage-5 feedback loop and does **not** replace the human
gate. Bot or human comments on it are inputs to the **stage-2** plan review and the gate, not
stage-5 rounds. It stays a draft until stage 5.

### Stage 2: Plan Review + Human Gate

1. Review the plan per the Right-Sizing class, using the active profile's **reviewer slice** for
   dimensions, with reviewers at the `senior` tier (resolved per `conventions.md` → `tiers.md`):
   **Standard/Large** invoke `reviewing-plans`; **Small** run one combined-dimension pass, unless the premise check flagged risk, in which
   case escalate to the full `reviewing-plans`. A review of some weight always runs before the
   gate.
2. Apply the review's fixes to the plan inline.
3. **THE GATE.** Present the reviewed plan to the human in the CLI:

   > The plan has been reviewed across the active profile's dimensions. Please review and approve
   > to proceed with implementation, or provide feedback for revision.

4. **STOP. Write no implementation code until the human explicitly approves.** If the human
   requests changes, revise the plan, re-run the review, and present again.
5. On approval, record it in Pipeline State (`gate: approved <date>`) and proceed to stage 3. The
   approved plan on the draft PR is the durable signal.

### Stage 3: Implementation

1. Execute the plan per the Right-Sizing class, spawning implementer subagents at the `mid`
   tier (resolved per `conventions.md` → `tiers.md`): **Standard/Large** invoke
   `subagent-driven-development` (fresh subagent per task, batching only trivial tasks);
   **Small** execute inline with `executing-plans`. Either way, follow the plan's checkbox tasks
   and apply the
   profile's **executor slice** to verify each — backend: TDD + targeted tests green (the full
   suite runs once at stage 4; see the "Test cadence" section in `conventions.md`); frontend: run
   the app, drive it, screenshot at the plan's breakpoints, a11y check, fast gates; full-stack:
   also verify the seam end-to-end (real UI action → real API → real data layer).
2. Apply the **Review Cadence**: dispatch a per-task reviewer (at the `senior` tier) ONLY for
   tasks tagged `review: yes`; `review: no` tasks are gated by their own acceptance criteria. Do
   NOT run subagent-driven-development's final whole-branch review — stage 4 is the single
   fan-out. Any re-review is scope-bounded to the fix.
3. Update Pipeline State.

### Stage 4: Commit Review

1. This is the **one authoritative whole-diff fan-out**:
   - **Standard/Large:** invoke `reviewing-commits` on the feature branch — format and lint,
     then three parallel **profile-aware** reviewers over the branch diff at the `senior`
     tier; triage into a findings artifact, then one fresh subagent per file group applying
     it, with the **full** test suite once at the end (the run's one full-suite pass; per-task
     runs were targeted). Fold the fixes in as commits. Re-reviews are scope-bounded.
     Stage 3 skipped SDD's final review precisely so this is the single fan-out — do not look for
     a prior
     whole-branch review to dedupe against.
   - **Small:** mechanical gates plus a single self-review against the profile's reviewer rubric.
2. **Check whether this repo has a PR review bot** before settling on the lighter path — see
   `conventions.md`. Where there is no bot, "the bot will catch it" is not available as a reason
   to review less: run the fan-out, or at minimum gates plus a genuine self-review. Never leave a
   diff with no review at all.
3. Update Pipeline State.

### Stage 5: PR Feedback Loop

1. **Ready the PR.** The draft already exists from stage 1: push the implementation + stage-4
   commits, update the PR body to the full description (format:
   `$SKILLS_ROOT/../commands/generate-pr-description.md`, the host's PR-description generator),
   mark it ready (`gh pr ready <n>`), assign
   `@seanmcgary` (`gh pr edit <n> --add-assignee seanmcgary`), and @-mention him in the
   ready-for-review note. Fall back to `gh pr create` only if no PR exists (a no-remote run that
   skipped the stage-1 draft). If the `### Decisions` list has entries, the body carries a
   **"Deviations from the approved plan"** section built from it — one bullet each, naming what
   the plan said, what you did, and why. A run with deviations and no such section has hidden
   them.
2. Invoke `pr-feedback-loop` with N=3. It waits for review, triages findings, applies fixes,
   replies to threads, and repeats for up to N rounds. (It sees the PR already open and skips its
   own create-PR setup; the `docs:` spec/plan commits do not match its round-counter grep, so
   round numbering is unaffected.)
3. Update Pipeline State after each round.
4. On termination (clean round or cap), produce the final status report.

**Do not block on slow CI for a clean-round exit.** Wait for what actually drives the loop and
for any check *required for the exit decision* (a branch-freshness gate, a red check that changes
your triage); do not sit in a blocking poll for multi-minute test/build jobs when local gates
already passed. Report completion and flag CI as in-progress. In a repo with **no review bot**,
the only signals are CI status and human comments — never poll for a review that will not arrive.

## Pipeline State

After each stage transition, update a `## Pipeline State` block in the plan document:

```markdown
## Pipeline State

| Field   | Value                                     |
|---------|-------------------------------------------|
| stage   | 3 (implementation)                        |
| class   | small (mechanical rename, no new surface)  |
| profile | frontend                                  |
| branch  | feat/<topic>                              |
| pr      | #<n>                                      |
| gate    | approved 2026-07-29                       |
| round   | 0                                         |
| decisions | 2 (see Decisions below)                 |
```

- **`stage`** — current stage number and name.
- **`class`** — Right-Sizing class plus a one-line reason, set at stage 1 start.
- **`profile`** — active domain profile(s), set at stage 1 start.
- **`branch`** — the feature branch, created at stage 1 start so the draft PR can be opened.
- **`pr`** — the PR number: opened as a DRAFT in stage 1 carrying the spec/plan, marked ready in
  stage 5. `n/a` only before the first artifact is pushed, or on a no-remote run.
- **`gate`** — `pending`, or `approved <date>` once the human approves at stage 2. A respawn must
  never infer approval from the stage number alone.
- **`round`** — current feedback round within stage 5 (`0` before stage 5).
- **`decisions`** — how many entries the `### Decisions` list below the table holds (`0` when
  there are none). Every deviation from the approved plan is logged there rather than escalated
  — see the Autonomy Contract — and the list must be complete before the PR is readied, because
  it is what stage 5's "Deviations from the approved plan" section is written from.

**Resume:** on invocation, if the plan doc already has a Pipeline State block, read it and resume
from the indicated stage rather than starting over. Never resume into stage 3+ unless `gate`
shows approved.

## Autonomy Contract

After the gate, proceed **without asking permission** except in exactly two cases:

1. **Ambiguous or contentious review comment** — unclear in intent, contradicts project
   conventions, or requires a judgment call that could go either way. STOP and surface it.
2. **Round cap hit with actionable findings still open.** STOP and report the remaining items.

Everything else proceeds autonomously. Do NOT ask "should I continue?" Do NOT ask permission to
run tests, push, or open a PR. Do NOT ask whether to address a finding — triage it per
`pr-feedback-loop`'s criteria.

### Deviating from the plan — decide, log, surface

**A deviation from the approved plan is NOT a reason to stop.** The gate approved the outcome,
not a literal transcript: no plan specifies everything, so a finding whose fix the plan does not
describe is the normal case. Decide it yourself, however structural it is.

For each one, append an entry to a `### Decisions` list under Pipeline State — the task, what
the plan said, what you did instead, and why — bump the `decisions` count, and render the whole
list as a **"Deviations from the approved plan"** section in the PR body at stage 5. The human
reviews these at the PR, with the diff in front of them, which is a better review than the same
question answered blind mid-run.

The one thing a deviation may never do is quietly widen scope. If the fix takes the feature
somewhere the request did not ask to go, say so plainly in the entry and in the PR body.

**The pipeline NEVER merges.** It ends with a status report. Merging is a human decision.

## Red Flags

1. **"The plan is simple enough to skip review."** Wrong. Every plan gets a review before the
   gate. Right-Sizing scales the review's *weight* — a Small change gets one combined pass
   instead of three — but never to zero, and it never removes the gate. You may right-size; you
   may never skip.
2. **"This is a small change, so I can skip the premise check."** Backwards. The premise check is
   MOST valuable on changes that look small: a small-looking change built on a wrong assumption
   produces a fast, confident, completely wrong PR that every review layer then blesses.
3. **"I'll just ask the user to be safe."** After the gate you proceed autonomously. The two
   exceptions are the only reasons to stop. "The plan doesn't cover this" is not one of them —
   decide it, log it under `### Decisions`, surface it in the PR body.
4. **"I can skip brainstorming and the spec for a small feature."** Acceptable for a Small change
   once the premise check is clean, or if a spec already exists. But the premise check is never
   optional, the plan is never skipped, and the gate never goes away.
5. **"The draft PR is open, so I can start implementing (or mark it ready)."** Wrong. The stage-1
   draft only exposes the spec/plan for review — it is NOT gate approval. Implementation waits
   for the explicit gate, and the PR stays a draft until stage 5.
6. **"The tests pass so the code is fine."** Passing tests prove the happy path. Stage 4 exists
   to catch what tests miss: security boundaries, convention violations, missing error paths, and
   the profile's domain risks (tenant isolation; keyboard/focus a11y and XSS sinks; contract
   drift across the seam).
7. **"I'll review every task / re-scan the whole diff to be thorough."** This is the stall the
   Review Cadence prevents. A `review: no` task is gated by its acceptance criteria; a re-review
   verifies only the fix. If a task needs its own reviewer, tag it `review: yes` **in the plan** —
   decided up front, not improvised mid-execution.
8. **"PR is open, my job is done."** Opening the PR starts stage 5; the pipeline ends with a
   status report after the loop terminates.
9. **"I'll wait for CI to be fully green before reporting."** Wrong for a clean-round exit. Wait
   for what drives the loop and for any check required for the exit decision; report completion
   with CI flagged in-progress.
10. **"There's a GitHub issue, I'll just use issue mode."** This skill has no issue mode. Use
    `plan-feature` then `build-feature`.
