---
name: identify-standards
description: Use when a repo has no conventions doc (or an incomplete one) and you need to produce or refresh a STANDARDS.md capturing its coding styles, conventions, and review rules by observing the actual codebase
---

## Overview

The review and pipeline skills (`reviewing-plans`, `reviewing-commits`, `ship-feature`, `pr-feedback-loop`) all read a project's **conventions doc** to turn universal rubric categories into concrete, citable rules. This skill produces that doc: a `STANDARDS.md` at the repo root derived from **evidence in the actual codebase**, not from what standards a project "should" have.

**Core principle: every rule must cite its evidence.** A standard you cannot trace to a config file, a documented rule, or a dominant code pattern is a guess — and a guessed standard is worse than no standard, because downstream reviewers will enforce it as if it were real. If you cannot cite it, you do not write it.

**Machine-enforced rules are referenced, never duplicated.** Formatting and lint rules already live in `.editorconfig`, `.prettierrc`, `.golangci.yml`, `ruff.toml`, etc. STANDARDS.md points at those files as the source of truth; it does not restate their contents (they drift, and the config always wins).

## When to Use

- A repo has no `AGENTS.md` / `CLAUDE.md` / `CONTRIBUTING.md` / `STANDARDS.md` and you want the review skills to have something concrete to enforce.
- An existing conventions doc is thin, stale, or silent on categories the review rubrics care about (auth, schema, logging, testing, commits).
- You are onboarding the review pipeline to a new project.

**When NOT to use:**
- A complete, current conventions doc already exists → the review skills read it directly; don't duplicate it into a second file.
- You want to *invent* new conventions for a greenfield repo → that is a design decision for the humans/architects, not an observation task. This skill documents what IS, not what SHOULD BE.

## The Iron Law

```
NO RULE WITHOUT CITED EVIDENCE
```

Every rule in the output carries an **Evidence** note: a config file path, a doc quote, or a file:line pattern with an observed prevalence. No evidence → the item goes in the "Open questions" section as a question for humans, NOT in the rules as a fact.

## Process

### 1. Inventory the evidence sources

Gather, in this order (later sources refine earlier ones):

- **Existing prose docs** — any `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `README.md`, `docs/`, `STYLEGUIDE.md`. These are authoritative where they exist; quote them.
- **Machine-enforced configs** — `.editorconfig`, linter/formatter configs (`.eslintrc*`, `.prettierrc*`, `biome.json`, `.golangci.yml`, `rustfmt.toml`, `ruff.toml`/`pyproject.toml`, `.rubocop.yml`, `checkstyle.xml`, etc.), and pre-commit/CI hooks (`.pre-commit-config.yaml`, `.github/workflows/`).
- **Build/task manifests** — `Makefile`, `package.json` scripts, `Cargo.toml`, `pyproject.toml`, `justfile` — to record the format/lint/test/build commands.
- **The code itself** — the dominant patterns (see step 3).
- **Commit history** — `git log` for the commit-message convention (conventional commits? trailers? scopes?).

### 2. Detect the stack and pick which categories apply

Identify languages, frameworks, and architecture from the manifests and directory layout. Then decide which of the standard categories (see Category Checklist below) the repo actually has. A category with no evidence is **skipped**, not fabricated — a repo with no database has no schema conventions.

### 3. Derive conventions from dominant patterns (with prevalence)

For each applicable category, find the **prevailing** pattern, not a single example. A convention is a pattern the codebase follows consistently; one occurrence is an anecdote.

- Sample broadly (grep/search across the tree), count the majority pattern vs. deviations, and record the ratio as evidence (e.g., "logging: `slog` used in 34/37 packages; 3 legacy uses of `log.Printf` in `internal/legacy/`").
- Prefer newer/actively-changed code over legacy corners — check `git log` recency if two patterns compete.
- Distinguish **enforced** (a linter/config guarantees it) from **conventional** (humans follow it but nothing checks it). Mark which is which; enforced rules are stronger.
- When two patterns genuinely compete with no clear majority, that is an **Open question**, not a rule.

### 4. Write STANDARDS.md

Use the template in [templates/STANDARDS.template.md](templates/STANDARDS.template.md). Rules:

- One rule per bullet, in the imperative ("Use X", "Every route must Y").
- Each rule ends with an **Evidence** parenthetical: config path, doc quote, or `pattern — N/M files`.
- For machine-enforced style, write a pointer, not the rules: "Formatting is enforced by `.prettierrc` + `.editorconfig`; run `npm run format`. Do not restate specific rules here."
- Anything uncertain goes under `## Open questions` as a question, never as a rule.
- Keep it scannable — the review skills read this to extract specifics; headings should match the Category Checklist so reviewers can map category → rule quickly.

### 5. Verify against the codebase (adversarial self-check)

Before finalizing, try to **falsify each rule you wrote**: search for counter-examples. If a rule says "all routes use decorator X" and you find three that don't, either the rule is wrong (downgrade to "most", cite the exceptions) or those three are bugs (note them as findings). A rule that its own codebase violates in 30% of cases is not a standard.

### 6. Present, don't impose

STANDARDS.md is a durable, repo-shaping artifact. Show the human the drafted file and the Open-questions list before treating it as authoritative. Let them correct fabrications, resolve open questions, and ratify. Only commit it once they approve.

## Category Checklist

Derive a rule for each category the repo has evidence for; skip the rest. These mirror the review-skill rubrics so the output plugs straight into them.

| Category | What to capture | Where evidence usually lives |
|----------|-----------------|------------------------------|
| Formatting & lint | Pointer to the enforcing configs + the commands | `.editorconfig`, linter/formatter configs, CI |
| Language/style idioms | Naming, error handling, module layout, idiomatic constructs | dominant code patterns |
| Routing / handlers | How endpoints are defined; forbidden bypasses | router setup, IDL/proto files, existing handlers |
| Auth / authz | Mechanism, where routes register protection | middleware/interceptor/guard code |
| Multi-tenancy / data scoping | Whether queries scope by tenant/owner and how | data-access layer |
| Schema / migrations | Column/enum conventions, migration registration | migration dir, ORM models |
| ORM / model conventions | Permitted struct tags, model patterns | model definitions |
| Logging | Logger API and style | dominant logging calls |
| Testing | Framework, required coverage, unit-vs-integration expectations | test files, CI config |
| Docs / codegen sync | What must be regenerated/updated on API/CLI changes | codegen scripts, CI drift checks |
| Commit / PR conventions | Message format, trailers, branch naming | `git log`, CONTRIBUTING, commit-lint config |
| Security defaults | Secrets handling, input validation, size caps | dominant patterns, security docs |

## Relationship to the review skills

STANDARDS.md sits at the **top of the resolution chain** those skills use (`AGENTS.md` → `CLAUDE.md` → `CONTRIBUTING.md` → `STANDARDS.md` → `STYLEGUIDE.md`). If a repo already has an `AGENTS.md`/`CLAUDE.md`, prefer **augmenting that file** over creating a competing STANDARDS.md — two conventions docs that disagree is worse than one. Create STANDARDS.md when no primary agent-instructions file exists, or link to it from the primary file so there is a single entry point.

## Red Flags — STOP

These mean you are about to write fiction. Stop and either find evidence or move the item to Open questions:

- **"This is a common best practice, so I'll add it."** — Common ≠ this repo's practice. Only document what THIS codebase does.
- **"The framework docs recommend X, so they probably do X."** — Check. Framework defaults are frequently overridden.
- **"I saw it once, that's the convention."** — One example is an anecdote. Establish prevalence.
- **"I'll restate the eslint rules in prose so they're all in one place."** — No. Configs drift; the config wins. Point at it.
- **"The user asked for standards, so more rules is better."** — A short doc of true rules beats a long doc of plausible guesses. Downstream reviewers enforce every line.
- **"I don't need to check for counter-examples."** — Step 5 is mandatory. A rule the codebase violates is not a standard.

## Common Mistakes

1. **Fabricating completeness.** Filling every category even when the repo has no evidence for some. Skip empty categories; note them as Open questions if they seem important but unresolved.
2. **Duplicating machine-enforced rules.** Restating formatter/linter settings in prose. Reference the config file instead.
3. **Confusing legacy with current.** Documenting an old pattern that new code has moved away from. Weight by recency and prevalence.
4. **Imposing without ratification.** Committing STANDARDS.md as authoritative before a human has corrected the guesses. Always present first.
