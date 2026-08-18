# Plan structure

Shared by `plan-feature` and `ship-feature`. The calling skill sets **where the plan lives** and
may add requirements of its own; everything below applies to both.

Write all plan prose in **Simplified Technical English (ASD-STE100)** — see the writing-style
section in `conventions.md`.

## Global Constraints preamble

Immediately after the Architecture block, a `## Global Constraints` section **citing** the binding
conventions this feature can touch — one line per rule: its name, where it is enforced, and a
link. Do NOT copy their text.

The conventions doc (the first that exists at the repo root: `AGENTS.md`, `CLAUDE.md`,
`CONTRIBUTING.md`, `STANDARDS.md`/`STYLEGUIDE.md`) is already loaded into every agent session in
that repo, executors included, so reproducing it in the plan duplicates what the reader already
has — and pays for it again on every execution run. The value of this section is the
**selection**: which rules bite for THIS change. That survives at a fraction of the size.

```markdown
## Global Constraints

Binding rules this change can touch (see CLAUDE.md):
- React hooks ban outside `withRouter.tsx` — enforced by `frontend/eslint.config.js`
- camelCase filenames (URL-slug exception does not apply here)
- New public route ⇒ three registrations: `sitemap.ts`, `routeHead.ts`, `robots.ts`
```

Two exceptions still quote verbatim: a rule the plan deliberately departs from, and a rule whose
exact wording the change contests.

**A constraint that would be identical in the next plan is not plan content.** If it is missing
from the conventions doc, say so and recommend adding it there — do not compensate by pasting it
into every plan.

## `Verified external API (do not re-derive)`

A section with that exact title, listing exact signatures, types, and behaviors of external and
library APIs the plan depends on. Pin these by reading actual source, never from memory.

Scope it to facts **specific to this change**. A fact that outlives it and will be re-derived by
the next plan belongs in the repo as dated, sourced reference material; cite it as `path:line`
here instead of restating it.

## Checkbox tasks with the agentic-worker header

Every task uses `- [ ]`. The plan begins with:

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.

## Profile-shaped tasks

Per the active profile's planner slice, each with a `review: yes|no` tag and concrete **acceptance
criteria** — the exact checks that prove it done. For UI tasks that means the breakpoints to
render, the tokens to match, and the keyboard/focus/reduced-motion checks. These criteria are what
let a mid-level executor verify its own work and what reviewers grade against.

For a full-stack change, fix the API contract first (seam addendum) so both sides build to it.
