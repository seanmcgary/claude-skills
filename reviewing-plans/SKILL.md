---
name: reviewing-plans
description: Use when an implementation plan draft exists and needs review for security, quality, and standards compliance before execution begins or before presenting it for approval
---

## Overview

The plan's author cannot review their own plan effectively. Fresh-context subagents reviewing along a single dimension each will catch defects that a single generalist pass rationalizes away or buries in noise.

This skill dispatches three parallel reviewer subagents — security, quality, and standards — each with a focused rubric. Their findings are triaged, fixes applied inline to the plan, and a summary table produced.

## Process

1. **Dispatch three reviewers in parallel** — send three Agent tool calls in a single message, one per dimension. Each subagent receives: the plan file path, the matching rubric below (verbatim), the path to CLAUDE.md (`/Users/seanmcgary/Code/ecloud-platform/CLAUDE.md`), and the instruction: "Read CLAUDE.md first. For every finding, quote the specific CLAUDE.md rule by name that is violated. Output findings as a structured list: `| task/step | defect description | severity | CLAUDE.md rule cited |`." If a reviewer returns garbage (no structured findings, off-topic, or fewer than 3 sentences), re-dispatch that single reviewer once with a more explicit prompt. If the re-dispatch also returns garbage or off-topic output, perform that dimension's review yourself inline in the current context using the same rubric, and note the fallback in the findings table.

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

### Security Reviewer

You are reviewing an implementation plan for security defects. Read CLAUDE.md first. Check every task and step against these rules:

- **Authn/authz on every new route:** Every route — both main-server AND internal-server — MUST specify its auth mechanism. Main-server routes use JWT/API-key auth. Internal-server routes MUST be listed in the `protectedMethods` map for the `ApiKeyUnaryInterceptor` (static-key auth). This is an architecture-derived rule: CLAUDE.md does not state it explicitly, so do NOT hunt for a CLAUDE.md quote — cite the codebase precedent instead: the protected-methods allowlist in `pkg/rpcServer/internalServer.go`, where every existing internal route is registered in `protectedMethods` and gated by `ApiKeyUnaryInterceptor` (allowlist parameterized in commit ccd4640). There are NO exceptions based on data sensitivity. If a task registers a route and does not mention auth configuration, that is a HIGH severity finding.
- **No secrets/keys/signatures in logs:** No step may log a full request or response object, because requests carry auth context (API keys, JWTs, signatures). Logging must use specific named fields only (e.g., `"stackId", stackID`). If a step logs `"request", req` or similar full-object serialization, that is a HIGH finding regardless of what data the service "primarily" handles. The concern is the auth fields present on every request, not the domain data.
- **Tenant isolation on every query:** Every data-access method that takes a resource ID must also scope by tenant (org/account). If a plan describes a query by `stack_id` without mentioning authorization that the caller owns that stack, flag it.
- **Fail-closed defaults:** If the plan is ambiguous about whether auth is required, the answer is yes. Never rationalize "this data isn't sensitive" as a reason to skip auth.
- **Input validation:** Every user-supplied field must have validation specified (length limits, format regex, etc.).

### Quality Reviewer

You are reviewing an implementation plan for quality defects. Read CLAUDE.md first. Check every task against these rules:

- **Interface consistency (Produces/Consumes):** For EVERY method/type listed in a task's "Consumes" section, verify that EXACTLY that name appears in another task's "Produces" section. If a consumer references `GetTagsByStack` from Task 2, then Task 2's Interfaces/Produces section must explicitly list `GetTagsByStack`. Implementation details in step bodies do NOT count — only the formal Interfaces section matters. A mismatch is a HIGH finding. Check EVERY consumer against EVERY producer systematically; do not skip any.
- **Test coverage per task:** Every task that produces a function, method, or handler must have a corresponding test step — either in the same task or in a dedicated test task that explicitly lists it. Validation/parsing functions (e.g., `ValidateTagKey`, `ValidateName`) MUST have dedicated unit tests; integration tests alone are insufficient. If a function is introduced without a test step, that is a MEDIUM finding.
- **Error paths specified:** Every operation that can fail must describe what happens on failure (return error, retry, rollback, etc.).
- **No scope creep:** Every task must be necessary for the stated Goal. If a task adds functionality not justified by the Goal or Architecture sections, flag it.
- **Dependency ordering:** If Task B consumes from Task A, Task A must appear earlier in the dependency graph. Circular dependencies are HIGH.

### Standards Reviewer

You are reviewing an implementation plan for violations of project conventions documented in CLAUDE.md. Read CLAUDE.md first. For EVERY finding, you MUST quote the specific CLAUDE.md section title and rule text that is violated. Check:

- **Protobuf-only routes (CLAUDE.md: "Protobuf & gRPC"):** "All HTTP and gRPC routes must be defined as protobuf services with `google.api.http` annotations — never add hand-written HTTP handlers directly to the server." If any task adds `mux.HandleFunc(...)` or a direct HTTP handler, that is a HIGH finding.
- **Enum Lookup Tables (CLAUDE.md: "Enum Lookup Tables"):** "Status fields and other categorical values use lookup tables instead of raw TEXT columns." If a migration adds a `TEXT` or `VARCHAR` column for a status/type/category field, that is a HIGH finding. The correct pattern requires: lookup table with `id SERIAL PRIMARY KEY` + `name VARCHAR(50) NOT NULL UNIQUE`, FK column on the referencing table, Go model with typed string constants.
- **Minimal GORM tags (CLAUDE.md: "GORM struct tags"):** "Keep `gorm:` tags minimal." Only `primaryKey`, `default:uuid_generate_v1()`, and `autoIncrement` are permitted. Any other tag (`type:`, `not null`, `uniqueIndex`, `foreignKey`, etc.) is a finding.
- **Migration registered in GetMigrations() (CLAUDE.md: "Adding Migrations"):** Every migration must include a step to "add the migration to `pkg/postgres/migrations/migrations.go:GetMigrations()`."
- **CLI doc-sync (CLAUDE.md: "Bundled agent skill" + "User-facing documentation"):** "After adding or changing any CLI command or flag, run `make skill` and commit the result — CI fails on drift." ALSO: "CLI command changes must update `ecloud-ui/src/content/docs/cli.md` and `ecloud-cli/README.md`." If a task adds/modifies CLI commands without a step for ALL THREE (README, cli.md, `make skill`), that is a HIGH finding.
- **Sugared-logger convention (CLAUDE.md: "Logging"):** "Always use `logger.Sugar()` with the `w`-suffixed methods and pass key-value pairs." Verify logging examples use the correct pattern.
- **Conventional commits without trailers (CLAUDE.md: "Commit messages"):** "Do NOT add `Co-Authored-By` or any other trailers to commit messages."

## Common Mistakes

These are failure modes observed in baseline reviews — watch for them and do NOT repeat them:

1. **Severity rationalization based on data sensitivity:** "The data is just metadata/tags, so auth is lower priority." WRONG. Auth requirements are structural and apply regardless of what data a service handles. Every route needs auth; no exceptions.
2. **Interface-consistency blindness:** Reviewing tasks in isolation without cross-referencing Produces/Consumes declarations across the full plan. You MUST systematically check every Consumes entry against producer Interfaces sections.
3. **Diluting findings with noise:** Focus on real defects per the rubric. Do not flag stylistic preferences or speculative concerns that aren't backed by a specific CLAUDE.md rule or rubric item.
4. **Conflating step-body implementation with formal interface contracts:** A method that appears in a task's code snippet but is NOT listed in the task's Interfaces/Produces section is NOT formally produced. Consumers that reference it have a dangling dependency.
