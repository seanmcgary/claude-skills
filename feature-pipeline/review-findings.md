# Review Findings — the review/remediation handoff contract

`producing-review-findings` and `executing-review-findings` are two independent skills joined by
one artifact: a **findings comment** on the pull request. The reviewer writes it; the executor
reads it and fixes what it says. `reviewing-commits` composes both in one session and passes the
artifact in memory rather than through GitHub.

This file is the single definition of that artifact. All three skills reference it; none
redefines it.

## Why the artifact is prescriptive, not descriptive

Measured across eight real review runs, the reviewer fan-out was 8% of the cost and the
remediation that followed was 92%. A quarter of the remediation's tool calls were **re-reading
and re-searching the same code the reviewers had already read** — because the findings came back
as prose, so whoever fixed them had to re-locate every issue itself, in a context that only grew.

The artifact removes that work. Every finding carries the file, the line, and a **prescribed
fix specific enough that a fresh subagent can apply it after reading only the named file**. A
finding that does not meet that bar is not finished; write it again before you post it.

The exception is a fix that cannot be written at line level: a migration plus the code that reads
it, or a change across several layers. That finding goes in `Fix — cross-cutting` with a goal and
acceptance criteria, and a senior-tier subagent applies it. Difficulty moves a finding to the
senior tier. It never moves a finding to Defer.

## Format

The comment body:

````markdown
## PR Review Findings — round 1

<!-- agent-utils:findings v1 round=1 -->

Reviewed `<head-ref>` at `<sha>` against `<base>`. Mechanical gates green.
53 findings: 47 Fix (1 cross-cutting), 5 Reject, 1 Defer.

### Fix

#### `internal/config/entryloop.go`

Tests: `go test ./internal/config/...`

| # | Line | Sev | Dim | Finding | Prescribed fix | Rule |
|---|------|-----|-----|---------|----------------|------|
| 3 | 119 | HIGH | sec | `other.terminal` compared case-sensitively on one path | Replace `l.trigger == other.review` at line 131 with `strings.EqualFold(l.trigger, other.review)`, matching line 119 | conventions.md § Label matching ignores case |
| 7 | 146 | LOW | qual | Error text names only the repo | Append `l.trigger` to the `%s` list in the `fmt.Errorf` at line 146 | codebase precedent: `internal/engine/engine.go:212` |

#### `internal/engine/engine.go`

Tests: `go test ./internal/engine/...`

| # | Line | Sev | Dim | Finding | Prescribed fix | Rule |
|---|------|-----|-----|---------|----------------|------|
| 12 | 88 | HIGH | sec | `tickIssue` reads then inserts with no conflict handling | Wrap the insert at line 94 in the duplicate-key retry used by `upsertSession` (`internal/store/session.go:140`); do not log the caught conflict | reviewing-commits § Race conditions |

### Fix — cross-cutting

#### 14 — Callback verifier is first-writer-wins (HIGH, sec)

Files: `internal/store/migrations/`, `internal/store/verifier.go`, `internal/webhook/verify.go`
Tests: `go test ./internal/store/... ./internal/webhook/...`

- **Finding:** `RecordCallbackVerifier` keeps one slot, so an anonymous caller who posts first
  owns the verifier.
- **Goal:** keep every verifier the handshake receives, and let `verify` pick the newest valid one.
- **Acceptance criteria:**
  1. A migration adds one row per received verifier.
  2. `GetLatestCallbackVerifier` is replaced by a newest-first list.
  3. A test posts two verifiers and asserts the legitimate one verifies.
- **Rule:** reviewing-commits § Authentication boundaries

### Reject

| # | File:Line | Finding | Why rejected |
|---|-----------|---------|--------------|
| 21 | `cmd/agent-utils/main.go:44` | Missing input length cap | The value is a parsed flag, not user input over a wire; there is no unbounded read |

### Defer

| # | File:Line | Finding | Why it is a major refactor | Issue |
|---|-----------|---------|----------------------------|-------|
| 30 | `internal/store/migrate.go:200` | Migration list should be generated | Generating it replaces the registration of all 60 migrations and the loader that reads them | #212 |

### Decisions needed

| # | Question | Options | Recommendation |
|---|----------|---------|----------------|
| 40 | The seeded prices are archived; what does the landing page show? | (a) the new prices from #119; (b) no price line | (b) until #119 merges |

### Pre-merge checks

| # | Check | Why an agent cannot run it |
|---|-------|----------------------------|
| 41 | VoiceOver reads the new card's rate as one element | No simulator in the headless worktree |
````

The example summary line counts every section: 53 = 47 Fix + 5 Reject + 1 Defer, and the
`Decisions needed` and `Pre-merge checks` rows are listed in addition to that count.

## Rules

- **A real finding is Fix.** Apply "Implement, don't defer" in `conventions.md` to every finding.
  "Pre-existing," "outside the diff," "the plan did not cover it," and "it needs a migration" are
  not reasons to Defer.
- **Group by FILE, not by finding.** One `####` heading per file, holding every Fix finding in
  that file. This is the whole reason for the format: 40+ findings collide on far fewer files,
  and two subagents editing one file is a lost write. The heading is a repo-relative path in
  backticks.
- **A finding that spans two or three files** goes under its primary file, with `Also touches:`
  and the other paths named in the Prescribed fix cell. `executing-review-findings` merges any two
  groups that name each other, so the pair is dispatched to one subagent.
- **`Fix — cross-cutting`** holds a finding whose fix spans layers or several file groups, or
  needs design work: a migration plus the code that reads it, a proto field plus both clients, a
  shared token plus its consumers. Give each one a `####` heading with its number, its files, its
  `Tests:` command, the goal, and numbered acceptance criteria. Do not force it into line-level
  prescriptions. `executing-review-findings` gives each one a senior-tier subagent after the file
  groups return. A cross-cutting finding is never a reason to Defer.
- **`Tests:`** — one line under each file heading naming the command that runs the tests covering
  that file, and nothing wider. This is the group's blast radius. Write the narrowest command
  that covers it; never the full suite.
- **`Sev`** is `HIGH` / `MEDIUM` / `LOW`. **`Dim`** is the reviewer dimension abbreviated —
  `sec`, `qual`, `std`, `a11y`, `design`, `seam`.
- **`Prescribed fix`** names the symbol, the line, and the concrete change. Write it the way a
  planner writes a task: the executor must not have to re-derive what you already worked out.
  The banned words are `consider`, `improve`, `handle properly`, `review`, `ensure` — every one
  of them pushes the derivation back onto the executor. If the reviewer's prescription is wrong
  or vague, the triager writes the correct one. If the fix cannot be written at line level, the
  finding is cross-cutting. It is never a Defer.
- **A product or design choice with no plan or design behind it is a Fix.** The triager picks the
  option, writes the fix, and starts the Finding cell with `Decision:` and the reason. The
  executor logs it as a deviation. Only a question an agent does not own alone (a public price,
  new recurring cost, a legal or compliance surface) goes in `Decisions needed`.
- **`Pre-merge checks`** holds checks that cannot run headless: a device, a simulator,
  VoiceOver, a real vendor build. They are not deferrals. The executor copies them into the PR
  body.
- **A branch-level finding** (a commit message, branch history) is Reject when the repository
  squash-merges, because the PR title becomes the commit. Otherwise it is cross-cutting.
- **`Rule`** is the conventions-doc section title, or `codebase precedent: <file>:<line>`, or the
  rubric section that names the category. Never empty.
- **Reject and Defer are decisions, not questions.** They are recorded so the human reads one
  table instead of re-reviewing the diff, and so nothing silently disappears.
- **Defer is only for a major refactor**, as `conventions.md` defines it. The triager files the
  GitHub issue with `gh issue create` before it posts the artifact, and writes the issue number
  in the `Issue` column. A Defer row with no issue is not finished. No one writes a `TODO`
  comment for a deferred finding.
- **Numbering is continuous across every section** and never reused within a round. The
  executor reports back by number.
- **The HTML comment marker is required.** It is how a later round, or a resumed session, finds
  the artifact when Pipeline State has been lost.
- **Omit an empty section** other than `### Fix`.

## Where the artifact is named

Pipeline State (see [`pipeline-state.md`](pipeline-state.md)) carries `findings comment` — the
numeric comment ID. The executor fetches exactly that one comment:

```bash
gh api repos/{owner}/{repo}/issues/comments/<id> --jq .body
```

It does **not** list and search a pull request's comments. A pull request under review carries
bot reviews, human comments, and earlier rounds; searching them is how the executor picks up the
wrong round.

## The size cap and overflow

**Verified against the GitHub API:** an issue comment body of 262,144 characters is accepted and
262,145 is rejected. The 422 message says `Body is too long (maximum is 65536 characters)`,
which is **wrong** — do not size against that number. The real cap is 2^18.

Measured reviewer output is 35–70KB, so a single round fits with room to spare. Overflow is still
defined, because a large branch reviewed by five dimensions is not hypothetical:

- **Budget 200,000 characters** for the assembled body. The margin covers multi-byte characters
  in quoted code, since the cap counts bytes.
- **Under budget:** post one comment. It is the artifact.
- **Over budget:** split into parts.
  1. Split the `### Fix` and `### Fix — cross-cutting` sections at `####` boundaries. **Never split a group** — a
     group is one subagent's whole assignment. A single group over budget is itself the finding:
     that file needs its own issue, not a comment.
  2. Post each part as its own comment, in order, each carrying
     `<!-- agent-utils:findings v1 round=<n> part=<k>/<m> -->`.
  3. Post the **manifest last**: the summary line, the full `### Reject`, `### Defer`, `### Decisions needed` and
     `### Pre-merge checks` tables,
     and a `### Parts` ordered list of the part comment IDs.
  4. Pipeline State's `findings comment` names the **manifest**, never a part.
- **The executor** fetches `findings comment`, and if the body carries a `### Parts` list, fetches
  each listed ID in order and concatenates before grouping. One ID in Pipeline State either way,
  so nothing downstream has to know whether the round overflowed.
