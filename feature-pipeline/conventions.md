# Feature pipeline conventions

Shared reference for `plan-feature` and `build-feature`. This file defines **how** the two
skills scale their work and address the owner. It contains no stages and orchestrates nothing —
each skill runs its own phases end to end and loads this for the rules below.

Companion files: [`pipeline-state.md`](pipeline-state.md) (the handoff contract) and
[`profiles/`](profiles/) (per-domain slices).

## Profiles: scale to the domain

Right-Sizing (below) scales ceremony to the *weight* of a change. **Profiles** scale it to the
*domain*. The axes are orthogonal — every change has both. "Full-stack" is not a third profile;
it is `frontend` + `backend` both active, plus the `seam` addendum.

- `profiles/backend.md` — server, data, APIs. **The default.**
- `profiles/frontend.md` — UI, client state, styling, interaction, accessibility.
- `profiles/seam.md` — thin addendum, active whenever both of the above are.

Each profile supplies three slices: **planner** (used by `plan-feature`), **executor** (used by
`build-feature`), and **reviewer** (used by both — plan review and commit review).

**Selection:** infer from the surfaces the premise check enumerated — UI/client files →
`frontend`; server/data/API files → `backend`; both → both + `seam`. Default to `backend` when
the signal is absent or ambiguous, and say so in the plan so the human can redirect before the
gate. `plan-feature` records the result in Pipeline State's `profile`; both skills load the
file(s) and apply the matching slice.

## Right-Sizing: scale the ceremony to the change

**Classify by risk, not file count.** Renaming a config var across 12 files is trivial;
reimplementing one file's internals is not. Score on three dimensions and take the *highest*
triggered. The active profile gives each dimension its domain meaning — read the profile's
planner slice for concrete triggers.

- **New surface** — adds a dependency, DB table/migration, public API/route, config var, or
  subsystem? (adds surface → at least Standard; new subsystem/schema/cross-repo contract → Large)
- **Risky boundary** — auth/authz, payments/money, data integrity, concurrency, migrations, or a
  security-relevant path? (yes → at least Standard, usually Large regardless of size)
- **Mechanical vs. semantic** — does correctness follow from a *uniform, interface-preserving*
  edit, or does it require *per-site reasoning about new behavior*? Mechanical → Small even
  across many files; semantic → Standard+ even in one file.

When unsure, pick the heavier class.

| | **Small** | **Standard** | **Large** |
|---|---|---|---|
| **Trigger** | mechanical / interface-preserving, adds no surface, touches no risky boundary | adds surface OR is semantic/behavioral OR touches a risky boundary in a contained way | new subsystem, schema/migration, security or money boundary, cross-repo contract, or a semantic change spanning many consumers |
| **Spec** | SKIP the written spec — the premise check is the correctness gate and the plan doubles as the spec | brainstorm → written spec | brainstorm → written spec (decompose if needed) |
| **Brainstorm** | premise check + confirm approach in one exchange | full brainstorming | full brainstorming + decompose |
| **Plan** | lightweight (still house style; tasks may be coarse) | full plan | full plan |
| **Plan review** | ONE reviewer pass (combined dimensions), unless the premise check flagged risk | three-dimension `reviewing-plans` | three-dimension `reviewing-plans` |
| **Implementation** | inline (`superpowers:executing-plans`), no per-task subagent relay | subagent-driven, batch trivial tasks | subagent-driven, one task per subagent |
| **Commit review** | mechanical gates + one self-review against the profile rubric | the whole-diff fan-out: profile-aware `reviewing-commits` | the whole-diff fan-out |

You may **right-size** a review; you may never **skip** it. Neither may you skip the premise
check — it is *most* valuable on changes that look small, because a small-looking change built
on a wrong assumption produces a fast, confident, completely wrong PR that every later layer
then blesses.

## Model tiering: senior plans, mid executes, senior reviews

| Work | Tier | Model | Effort |
|------|------|-------|--------|
| Planning — premise check, brainstorm, design sync, plan authoring (`plan-feature`; `ship-feature` stage 1) | senior | `claude-opus-5` | **high** |
| Reviewer subagents — plan review and commit review | senior | `claude-opus-5` | high |
| Per-task implementation (`build-feature`; `ship-feature` stage 3) | mid | `claude-sonnet-5` | default |
| Orchestration + review triage | senior | `claude-opus-5` | high |

**`plan-feature` runs entirely on `claude-opus-5` at high effort — the orchestrating session and
every subagent it dispatches.** Planning is the highest-leverage step in the pipeline: a
well-specified plan is what lets a mid-level executor produce correct code without heavy
per-task review. There is no cheap path through planning, and no task within it is "mechanical
enough" to downgrade.

**Always specify the model explicitly when dispatching a subagent.** An omitted model inherits
the session's model and silently defeats this tiering — nothing errors, you just get worse
output. Note the Agent tool's `model` parameter takes a short alias, not a full model ID:
pass `"opus"` where this table says `claude-opus-5` and `"sonnet"` where it says
`claude-sonnet-5`.

If the executor struggles, the fix is a better-specified plan — not a more expensive executor.

## Review cadence: one authoritative pass; converge, don't ratchet

Reviews are the pipeline's main stall risk. Fresh-context reviews of the same diff each surface
their own subjective findings, so fixing round N's findings hands round N+1 a changed diff to
find NEW findings in. This cadence keeps review value high while making the loop terminate.

1. **Per-task review is conditional.** Only tasks the plan tags `review: yes` get a per-task
   reviewer. `review: no` tasks are gated by their own acceptance criteria (tests, or the
   frontend profile's render/screenshot check). A well-specified plan makes most tasks `review: no`.
2. **Exactly ONE authoritative whole-diff fan-out per run**, at commit review. Implementation
   does NOT additionally run subagent-driven-development's separate final whole-branch review.
3. **Re-reviews are scope-bounded.** After a fix, the re-review verifies ONLY that the fix
   resolves its finding and introduces no regression in the touched lines. It does NOT re-scan
   the whole diff. This is what stops the ratchet.
4. **The PR feedback loop is capped** (default N=3); a clean round exits immediately.

If a task feels like it needs its own reviewer, the fix is to tag it `review: yes` **in the
plan** — decided by the senior planner up front, not improvised mid-execution.

## Repositories without a PR review bot

Some repos (including `mcgarylabs/lawndominator-monorepo`) have **no automated PR reviewer**.
Detect this once, at the start of commit review, by checking whether recent merged PRs carry
bot-authored reviews. When there is no bot:

- **Never** downgrade the whole-diff fan-out on the assumption a bot will catch it. A Small
  change still gets mechanical gates plus a real self-review against the profile's reviewer
  rubric; when in doubt run the full fan-out. "The bot will catch it" is not available as a
  reason to review less.
- **Never** poll for a bot review that will not arrive. In the PR feedback loop, the only
  signals are **CI status** and **human comments**. Push, confirm CI, and exit on a clean round
  rather than waiting out a timeout.

## Owner notifications

`@seanmcgary` owns these runs.

- **@-mention `@seanmcgary` in every issue or PR comment you author** that awaits his input or
  signals a state he should see — clarifying questions, the gate, blockers, ready-for-review.
  (Bot-authored comments are not yours to tag.)
- **Assign the PR to `@seanmcgary`** when it is readied for review:
  `gh pr edit <n> --add-assignee seanmcgary`, or `--assignee seanmcgary` when opening it fresh.

## Writing style: Simplified Technical English (ASD-STE100)

Write every document you author — specs, plans, design-reference docs, and the issue/PR prose
that carries them — in **Simplified Technical English (ASD-STE100)**, the controlled language for
technical documentation. It exists so a plan reads the same way to every executor and reviewer,
with no room to misread intent. Apply its core rules:

- **Short sentences, one idea each.** Keep instruction sentences to about 20 words and
  descriptive sentences to about 25; split anything longer. Write one instruction per sentence.
- **Active voice; imperative for instructions.** Write "Add the migration," not "the migration
  should be added." Name the actor in descriptive text.
- **Simple tenses** (present, past, future). Do not use progressive (`-ing`) or compound tenses
  where a simple tense says the same thing.
- **One term for one thing.** Choose a single word for each concept and reuse it verbatim; do not
  vary it for style (an "endpoint" stays an "endpoint," not sometimes a "route," a "handler," or
  an "API"). Keep the articles ("a" / "the").
- **Plain, approved vocabulary.** Use no jargon, idiom, slang, or needless synonyms. Prefer the
  simplest word that is exact.
- **Vertical lists** for sequences, conditions, and sets of options, instead of long
  comma-strung sentences.
- **Say it explicitly.** State the actor, the object, and the condition; do not make the reader
  infer them.

STE governs the document *prose*. It does not touch code, identifiers, commit-message format,
copied-verbatim constraints (Global Constraints stay word-for-word), or literal API signatures.

## The pipeline never merges

Merging is a human decision, in every mode, at every stage, without exception.
