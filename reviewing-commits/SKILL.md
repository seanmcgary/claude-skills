---
name: reviewing-commits
description: Use when a feature branch is locally complete and needs security, quality, and standards review plus lint/test gates before pushing or opening a PR
---

## Overview

The branch author cannot effectively review their own commits. Fresh-context subagents reviewing
along a single dimension each will catch defects that a single generalist pass rationalizes away
or buries in noise. The goal: round 1 of PR review should find nothing mechanical.

**This skill is a composition of two others.** It runs them back to back in one session:

1. **`producing-review-findings`** — mechanical gates (format, lint, then the full test suite as a
   precondition), the reviewer fan-out over the branch diff, triage into Fix / Reject / Defer, and
   the findings artifact.
2. **`executing-review-findings`** — one fresh-context subagent per file group applying that
   artifact's Fix findings, targeted tests per group, then the full suite once.

Follow each of those skills exactly. This file adds only what composing them changes.

## What composing them changes

- **The artifact stays in the session.** `producing-review-findings` normally posts the findings
  as a pull request comment and records its ID in Pipeline State. Here there may be no pull
  request yet, so hold the artifact in the session and hand it straight to
  `executing-review-findings`. Write it in the format
  `$SKILLS_ROOT/feature-pipeline/review-findings.md` defines regardless — the grouping and the
  prescribed-fix column are what make the fan-out work, not the transport.
- **One full-suite run, not two.** `producing-review-findings`'s gates run the suite as a
  precondition and `executing-review-findings` runs it again after its fixes. In one session,
  fixes land minutes after the precondition run, so run it **once, at the end**, per the "Test
  cadence" section in `conventions.md`. Keep format and lint as a precondition — they are cheap
  and they block the reviewers on noise.
- **Fold the fixes into the branch.** Once every group has returned and the suite is green:
  - If a fix belongs to exactly one existing commit, use `git commit --fixup=<sha>`, then
    re-resolve `$BASE` as `producing-review-findings` step 1 describes and run
    `git rebase -i --autosquash $(git merge-base "$BASE" HEAD)` to fold it in.
  - If a fix spans multiple commits or is new work, create a conventional commit (e.g.,
    `fix: add cross-tenant test for GetSecret`).
  - Re-run format, lint, and the suite after the rebase. The branch must be green before you stop.
- **Output the findings-disposition table**, which is the findings artifact plus what the executor
  did with each Fix row:

  ```
  | # | File:Line | Finding | Dimension | Severity | Disposition | Notes |
  |---|-----------|---------|-----------|----------|-------------|-------|
  ```

## When to use this instead of the two halves

Use **this** skill when the branch is fresh, the finding count is small, and one session can carry
both halves without its context becoming the cost — which is the case for `build-feature` phase 3
and `ship-feature` stage 4, where the branch was written minutes ago by the same pipeline.

Use the **two halves separately** when review and remediation are separate dispatches: an
unattended loop, a large branch, or any run where the findings will outlive the session that found
them. Measured across eight real review runs, remediation was 92% of review's total cost, all of
it paid in a context that had already absorbed the whole diff and every reviewer's output.
Splitting exists for exactly that case.
