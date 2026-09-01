---
name: executing-review-findings
description: Use when a review has already produced a findings artifact (from producing-review-findings or a pr-review loop) and the fixes still need to be applied — fans out one fresh-context subagent per file group, runs targeted tests per group, then the full suite once.
---

## Overview

Remediation, not review. A review already happened: every finding is located, severity-tagged, and
carries a prescribed fix. This skill applies them and nothing else.

**It does not re-review.** It does not re-read the diff looking for what the reviewers missed, and
it does not second-guess a `Reject` or promote a `Defer`. Re-reviewing here is the ratchet
`conventions.md` warns about, paid for twice.

**Why this is its own skill.** Measured across eight real review runs, remediation was 92% of the
cost of review, because it ran in the reviewing session's context — which grew from 66k to 321k
tokens while it worked — and because a quarter of its tool calls re-located issues the reviewers
had already located. The fix is a fresh context per group, and an artifact prescriptive enough
that the group's subagent never re-derives anything.

## Input

The findings artifact defined in `$SKILLS_ROOT/feature-pipeline/review-findings.md`. Read that
file before this one; it is the contract.

**Fetch it by ID, never by search.** Pipeline State (`pipeline-state.md`) carries
`findings comment`:

```bash
gh api repos/{owner}/{repo}/issues/comments/<id> --jq .body
```

If the body carries a `### Parts` list, fetch each listed ID in order and concatenate. A pull
request under review also carries bot reviews, human comments, and earlier rounds; listing and
searching them is how you end up fixing the wrong round.

**No `findings comment` field, or the comment does not exist: PARK.** Do not reconstruct the
artifact by re-reviewing the branch — that is the expensive session this skill exists to remove,
and the result would not be the round the human read.

## Process

### 1. Form the groups

The artifact is already grouped: each `####` heading under `### Fix` is one group. Take them as
written. Two adjustments, and no others:

- **Merge groups that name each other.** A finding whose prescribed fix says `Also touches:
  <other file>` binds those two groups into one. Merging is mandatory, not a judgement call.
- **Merge groups whose `Tests:` commands are identical and whose files sit in one package**, when
  that leaves fewer than three groups' worth of work. Two subagents that must both keep one
  package compiling are not independent.

**One file appears in exactly one group.** This is the invariant the whole fan-out rests on: two
subagents editing one file is a lost write, and nothing in the harness will tell you it happened.
Before dispatching, list every file across every group and confirm there is no repeat.

### 2. Fan out — one subagent per group

**Send every Agent call in a single message.** Background tasks are disabled under an unattended
loop, so a subagent is an ordinary blocking tool call: several dispatched in one turn run
concurrently, one dispatched per turn runs strictly serially. Dispatching them one at a time
rebuilds the very serial, context-accumulating session this skill replaces.

Each subagent receives, and nothing more:

- Its group's file list and its group's findings table, verbatim from the artifact.
- Its group's `Tests:` command.
- The path to the conventions doc.
- These instructions:

  > Apply every finding in your table. The `Prescribed fix` column tells you the change; a review
  > already derived it, so implement it rather than re-deriving it. Read only the files in your
  > list plus whatever they force you to read.
  >
  > **Your file list is your blast radius.** Edit no file outside it. If a fix genuinely cannot be
  > made without editing a file you were not given, STOP and report that back — do not edit it.
  > Another subagent may hold that file right now.
  >
  > Run your `Tests:` command and fix until it passes. Do not run the full suite; the orchestrator
  > runs it once after every group returns.
  >
  > If a finding is wrong — the code does not say what the finding says it says — do not implement
  > it and do not invent a substitute. Report it back by its number with one line of reasoning.
  >
  > Report: findings applied by number, findings not applied by number with the reason, files
  > changed, and your test command's final result.

**Do not commit inside a subagent.** The orchestrator commits, once per group, after that group
returns — concurrent subagents sharing one index is the same lost-write hazard as sharing a file.

### 3. Place the deferred TODOs

For each `### Defer` row, write a `TODO` comment citing the finding at the location the artifact's
`Place TODO at` column names. Do this in the orchestrator, after the fan-out, so it cannot collide
with a subagent's edit.

### 4. Full suite, once

Run the project's full test suite **once**, after every group has returned. This is the run that
catches what the per-group targeted runs could not see: interaction between groups. A failure here
is a normal outcome of the cadence, not evidence a group did its job badly.

**Whoever runs it fixes what it surfaces** — that is this session, inline. A cross-group failure is
by definition not any one group's, so there is no group to send it back to.

Then re-run format and lint and commit their output.

### 5. Report

Post one comment covering the whole round:

- Every `Fix` finding by number: applied, or not applied with the reason.
- Every finding a subagent reported wrong, with its reasoning, and your decision.
- The deferred TODOs you placed.
- The full suite's result and anything you fixed after it.

Do not restate the `Reject` and `Defer` reasoning — the findings artifact already carries it and
the human has read it. Link to the artifact comment instead.

## Test cadence

Two weights, and the boundary between them is the group boundary:

- **Per group: targeted only.** The group's `Tests:` command plus the fast gates. A group that
  cannot make its own targeted tests pass is a blocker, not something to push into the full run.
- **Full suite: once, at the end, by the orchestrator.** Never per group, and never twice.

This is the same shape `conventions.md` describes for the pipeline as a whole — targeted while the
work is fresh, full once in the phase that owns whole-branch verification.

## Blockers

PARK rather than guess when:

- Pipeline State names no `findings comment`, or the comment it names does not exist.
- A group reports it cannot fix a finding without editing a file held by another group, and the
  merge that would resolve it is not obvious from the artifact.
- The full suite is still red after your fixes.
- A finding's prescribed fix contradicts the code, and the correct change is a design decision
  rather than a repair.

Parking states what you need and stops. It does not turn into a re-review.
