---
name: reviewing-plans
description: Use when an implementation plan draft exists and needs review for security, quality, and standards compliance before execution begins or before presenting it for approval
---

## Overview

The plan's author cannot review their own plan effectively. Fresh-context subagents reviewing along a single dimension each will catch defects that a single generalist pass rationalizes away or buries in noise.

This skill dispatches three parallel reviewer subagents — security, quality, and standards — each with a focused rubric. Their findings are triaged, fixes applied inline to the plan, and a summary table produced.

## Project conventions doc

This skill stays at the workflow level; every project-specific rule (coding style, auth model, schema conventions, commit format) lives in repo-level files, NOT here. Resolve them once, up front:

- **Prose conventions — the "conventions doc":** the first of these that exists at the repo root: `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `STANDARDS.md`, `STYLEGUIDE.md`/`STYLE.md`. If more than one exists, read them in that order and treat later ones as supplements (a primary agent-instructions file may link out to a detailed `STANDARDS.md` / `docs/` guide — follow those links). "Read the conventions doc" below means this resolved set.
- **Machine-enforced style — also consult:** `.editorconfig` and the project's linter/formatter configs. These are the source of truth for formatting/lint rules the plan must respect.

If no conventions doc exists, note that in the findings table and review against the universal rubric categories alone. To generate one from the codebase for future runs, use the `identify-standards` skill.

## Process

1. **Dispatch three reviewers in parallel** — send three Agent tool calls in a single message, one per dimension. Resolve the project's conventions doc as described in "Project conventions doc" above. Each subagent receives: the plan file path, the matching rubric below (verbatim), the path to the conventions doc, and the instruction: "Read the conventions doc first. For every finding, quote the specific project rule by name that is violated (or cite the codebase precedent if the rule is architecture-derived rather than written down). Output findings as a structured list: `| task/step | defect description | severity | rule cited |`." If a reviewer returns garbage (no structured findings, off-topic, or fewer than 3 sentences), re-dispatch that single reviewer once with a more explicit prompt. If the re-dispatch also returns garbage or off-topic output, perform that dimension's review yourself inline in the current context using the same rubric, and note the fallback in the findings table.

2. **Collect findings** — gather all findings into a unified list with columns: `task/step`, `defect`, `dimension` (security/quality/standards), `severity` (high/medium/low).

3. **Triage each finding** — for each finding, decide:
   - **Fix:** edit the plan inline to resolve the defect. State what was changed.
   - **Reject:** the finding is incorrect or not applicable. State why in one line.
   - **Defer:** the finding is valid but out of scope for this plan; record it in the findings table with a pointer to where it should be handled (follow-up plan, backlog, or a TODO task added to the plan).

4. **Re-check interfaces after fixes** — if any fix changed a task's Interfaces (Produces/Consumes) section, re-dispatch ONLY the quality reviewer on the updated plan to verify the fix didn't create new interface mismatches.

5. **Output a findings-disposition table** — final output format:

   ```
   | # | Task/Step | Defect | Dimension | Severity | Disposition | Notes |
   |---|-----------|--------|-----------|----------|-------------|-------|
   ```

## Reviewer Rubrics

> **Adapting the rubrics to your project:** The categories below are universal, but the concrete rule for each category lives in the project's conventions doc and codebase. Before dispatching, skim the conventions doc and note the project's specifics (its routing/handler convention, auth mechanism, ORM/schema conventions, logging style, migration registration, doc-sync requirements, commit-message format). Feed those specifics to the reviewers alongside the category. Where a category does not apply to the project (e.g., no multi-tenancy, no CLI), the reviewer skips it. Examples in each bullet are illustrative — replace them with the project's actual rule.

> **Profile-driven dimensions (when invoked by the feature pipeline):** If a domain **profile** is active (`$SKILLS_ROOT/feature-pipeline/profiles/*.md` per the portability note in `conventions.md`), its reviewer slice is the source of truth for *which* dimensions to run. The three rubrics below are the `backend` profile. The `frontend` profile substitutes accessibility / responsive / design-fidelity / client-security for the Security rubric and keeps Quality + Standards; full-stack runs both sets plus the seam's contract-consistency / error-propagation / validation-parity checks. Read the active profile and dispatch one fresh reviewer per dimension it names (group related dimensions where sensible — the "three parallel reviewers" count is a default, not a fixed set); keep each reviewer fresh-context and single-focus.

### Security Reviewer

You are reviewing an implementation plan for security defects. Read the project's conventions doc first. Check every task and step against these categories:

- **Authn/authz on every new route/endpoint:** Every externally- or internally-reachable route MUST specify its auth mechanism. If the project has more than one server surface (e.g., a public API and an internal service), each has its own auth requirement — check the conventions doc and codebase precedent for how routes are registered as protected. This is often an architecture-derived rule not written down explicitly; if so, cite the codebase precedent (the file where existing routes register their auth) rather than a doc quote. There are NO exceptions based on data sensitivity. If a task registers a route and does not mention auth configuration, that is a HIGH severity finding.
- **No secrets/keys/signatures in logs:** No step may log a full request or response object, because requests carry auth context (API keys, tokens, signatures). Logging must use specific named fields only. If a step logs a whole request/response object or similar full-object serialization, that is a HIGH finding regardless of what data the service "primarily" handles. The concern is the auth fields present on every request, not the domain data.
- **Tenant isolation on every query (if the project is multi-tenant):** Every data-access method that takes a resource ID must also scope by tenant (org/account/user). If a plan describes a query by resource ID without mentioning authorization that the caller owns that resource, flag it.
- **Fail-closed defaults:** If the plan is ambiguous about whether auth is required, the answer is yes. Never rationalize "this data isn't sensitive" as a reason to skip auth.
- **Input validation:** Every user-supplied field must have validation specified (length limits, format regex, etc.).

### Quality Reviewer

You are reviewing an implementation plan for quality defects. Read the project's conventions doc first. Check every task against these rules:

- **Interface consistency (Produces/Consumes):** For EVERY method/type listed in a task's "Consumes" section, verify that EXACTLY that name appears in another task's "Produces" section. If a consumer references a symbol from Task 2, then Task 2's Interfaces/Produces section must explicitly list that symbol. Implementation details in step bodies do NOT count — only the formal Interfaces section matters. A mismatch is a HIGH finding. Check EVERY consumer against EVERY producer systematically; do not skip any.
- **Test coverage per task:** Every task that produces a function, method, or handler must have a corresponding test step — either in the same task or in a dedicated test task that explicitly lists it. Validation/parsing functions MUST have dedicated unit tests; integration tests alone are insufficient. If a function is introduced without a test step, that is a MEDIUM finding.
- **Error paths specified:** Every operation that can fail must describe what happens on failure (return error, retry, rollback, etc.).
- **No scope creep:** Every task must be necessary for the stated Goal. If a task adds functionality not justified by the Goal or Architecture sections, flag it.
- **Dependency ordering:** If Task B consumes from Task A, Task A must appear earlier in the dependency graph. Circular dependencies are HIGH.

### Standards Reviewer

You are reviewing an implementation plan for violations of the project's documented conventions. Read the project's conventions doc first. For EVERY finding, you MUST quote the specific section title and rule text that is violated. The categories below are the ones project conventions most commonly govern — check each against what the conventions doc actually says, and skip any the project does not have a rule for:

- **Routing/handler convention:** Many projects mandate one way to define routes (e.g., protobuf/IDL-defined services, a specific router, a code-generation step) and forbid hand-written handlers that bypass it. If the conventions doc states such a rule and a task violates it, that is a HIGH finding.
- **Schema/enum conventions:** Projects often constrain how categorical values, status fields, and columns are modeled (e.g., lookup tables vs. raw enum columns, required column types). Flag migrations/models that deviate from the documented pattern.
- **ORM/model tag conventions:** If the conventions doc restricts which ORM/serialization struct tags are permitted, flag any tag outside the allowed set.
- **Migration registration:** If the project requires new migrations to be registered in a central list/registry, every migration task must include that step.
- **Doc-sync requirements:** If changing a CLI command, API, or public interface requires updating specific docs and/or running a generation command (and CI fails on drift), a task that changes those without the corresponding doc/generation steps is a HIGH finding. Enumerate ALL required targets from the conventions doc.
- **Logging convention:** Verify logging examples in the plan match the project's documented logger API and style.
- **Commit-message convention:** Check the plan's commit-message guidance matches the project's format (e.g., conventional commits, trailer restrictions).

## Common Mistakes

These are common failure modes in plan reviews — watch for them and do NOT repeat them:

1. **Severity rationalization based on data sensitivity:** "The data is just metadata/tags, so auth is lower priority." WRONG. Auth requirements are structural and apply regardless of what data a service handles. Every route needs auth; no exceptions.
2. **Interface-consistency blindness:** Reviewing tasks in isolation without cross-referencing Produces/Consumes declarations across the full plan. You MUST systematically check every Consumes entry against producer Interfaces sections.
3. **Diluting findings with noise:** Focus on real defects per the rubric. Do not flag stylistic preferences or speculative concerns that aren't backed by a specific project rule or rubric item.
4. **Conflating step-body implementation with formal interface contracts:** A method that appears in a task's code snippet but is NOT listed in the task's Interfaces/Produces section is NOT formally produced. Consumers that reference it have a dangling dependency.
