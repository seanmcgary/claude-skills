---
name: reviewing-commits
description: Use when a feature branch is locally complete and needs security, quality, and standards review plus lint/test gates before pushing or opening a PR
---

## Overview

The branch author cannot effectively review their own commits. Fresh-context subagents reviewing along a single dimension each will catch defects that a single generalist pass rationalizes away or buries in noise. The goal: round 1 of PR review should find nothing mechanical.

This skill runs mechanical gates first, then dispatches three parallel reviewer subagents (security, quality, standards) over the branch diff, triages their findings, applies fixes as commits, re-runs the gates, and produces a findings-disposition table.

## Mechanical Gates (run first, in order)

> **Team-mode banner:** If team tools are available (SendMessage / shared task list / spawn mechanism) and you are the QA teammate, the gate COMMIT actions ("commit them", "fix them and commit"), step 4 (Fold fixes), and step 5 (Re-run gates as commits) are SUSPENDED -- see "Running as a team-mode teammate" at the end of the Process section; you review READ-ONLY and articulate, the Implementer applies and commits.

These gates run BEFORE dispatching reviewers. A red gate blocks reviewer dispatch -- there is no point reviewing a broken branch.

1. **`make fmt`** -- run it; if any files change, commit them (`fix: format code`).
2. **`make lint`** -- run it; must produce 0 issues. If there are findings, fix them and commit (`fix: resolve lint findings`). Re-run until clean.
3. **`make test`** -- run the full test suite (`make test`). If tests fail, fix and commit. Re-run until green.

Only proceed to the reviewer dispatch once all three gates are green.

## Process

> **Team-mode banner:** If team tools are available (SendMessage / shared task list / spawn mechanism) and you are the QA teammate, the gate COMMIT actions ("commit them", "fix them and commit"), step 4 (Fold fixes), and step 5 (Re-run gates as commits) are SUSPENDED -- see "Running as a team-mode teammate" at the end of this section; you review READ-ONLY and articulate, the Implementer applies and commits.

### 1. Compute the diff

```bash
git diff $(git merge-base origin/master HEAD)...HEAD
```

This is the scope the reviewers examine. If the diff exceeds 3000 lines, chunk it by package directory for each reviewer so they can review in focused passes rather than rationalizing "the diff is too large to review fully."

### 2. Dispatch three reviewers in parallel

Send three Agent tool calls in a single message (one per dimension). Each subagent receives:
- The diff scope (the command above, or the chunked file list)
- The path to CLAUDE.md: `/Users/seanmcgary/Code/ecloud-platform/CLAUDE.md`
- Its dimension-specific rubric (below, verbatim)
- The instruction: "Read CLAUDE.md first. Run `git diff $(git merge-base origin/master HEAD)...HEAD` to see the full diff. For every finding, cite the specific rule violated (CLAUDE.md section title or codebase precedent). Output findings as a structured list: `| # | File:Line | Finding | Severity | Rule Cited |`. Do not modify any files."

If a reviewer returns garbage (no structured findings, off-topic, or fewer than 3 sentences), re-dispatch that single reviewer once with a more explicit prompt. If the re-dispatch also returns garbage or off-topic output, perform that dimension's review yourself inline in the current context using the same rubric, and note the fallback in the findings table.

### 3. Triage findings

For each finding from all three reviewers, decide:
- **Fix:** implement the fix. State what was changed.
- **Reject:** the finding is incorrect or not applicable (same concept pr-feedback-loop calls "push back" -- disagree with reasoning, don't apply). State why in one line.
- **Defer:** the finding is real but out of scope for this branch. Create a TODO comment in the relevant file citing the finding.

### 4. Fold fixes into the branch

- If a fix belongs to exactly one existing commit, use `git commit --fixup=<sha>` and then `git rebase -i --autosquash $(git merge-base origin/master HEAD)` to fold it in.
- If a fix spans multiple commits or is new work, create a conventional commit (e.g., `fix: add cross-tenant test for GetSecret`).

### 5. Re-run the three gates

After all fixes are applied, re-run in order: `make fmt` -> `make lint` -> `make test`. Fix any regressions. The branch must be green before producing the final output.

### 6. Findings-disposition table

Output the final table:

```
| # | File:Line | Finding | Dimension | Severity | Disposition | Notes |
|---|-----------|---------|-----------|----------|-------------|-------|
```

### Running as a team-mode teammate

> **⚠️ UNVALIDATED-BY-LIVE-TEAM:** Team mode ships behind the experimental agent-teams flag (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) and has NOT been validated by a live team run. The RED/GREEN ground-truth re-run (QA-as-teammate re-reviewing the PR #273 diff vs. the logged 13/13 subagent baseline, on both catch-rate and token cost) was NOT performed. Do not treat the teammate reviewer path as proven equivalent to the standalone subagent path until that run is done.

**When this applies.** This subsection applies ONLY when team tools (SendMessage / shared task list / the spawn mechanism) are AVAILABLE and you were spawned as the **QA teammate** (model `claude-opus-4-8[1m]`) by the lead session running `ship-feature` stage 4. Detect team mode by the AVAILABILITY of the team tools, NOT by reading the env var (a settings.json env var is not reliably exported into the Bash shell). **If team tools are absent, ignore this subsection entirely** -- the standalone behavior above is UNCHANGED: the invoker runs the writing mechanical gates (`make fmt`/`make lint`/`make test` and commits their fixes), dispatches the reviewers, applies fixes, and folds them via steps 4-5 itself.

You are a teammate, not the lead. You never spawn teammates (teammates cannot). You never orchestrate the pipeline or advance it past stage 4. Ignore `ship-feature`'s `## Team Mode` section -- it is a lead-only orchestration script.

**Core rule: QA reviews READ-ONLY and articulates; the Implementer applies and commits.** You NEVER edit code, never run the writing gates, never commit. File ownership: the Implementer (model `claude-sonnet-5`) writes all source and all local commits; QA writes NO files.

The following are SUSPENDED for you: the Mechanical Gates' commit actions ("if any files change, commit them", "fix them and commit"), Process **step 4 (Fold fixes into the branch)**, and Process **step 5 (Re-run the three gates)** as commits. You do the read-only equivalents below instead.

**1. Mechanical gates -- READ-ONLY, and the red-gate-blocks-dispatch invariant is preserved.** Run the gates as read-only checks and treat any failure as a finding -- never edit, never commit:
- `make fmtcheck` (the non-mutating form of `make fmt`)
- `make lint`
- `make test`

If ANY gate is red, do NOT dispatch the reviewers. Articulate the failures to the Implementer via SendMessage (create one shared task item per failure), and WAIT for the Implementer to signal a green re-verify before dispatching the three reviewer subagents. This preserves the base skill's "a red gate blocks reviewer dispatch" invariant -- there is no point reviewing a broken branch.

**2. Act only on a QUIESCED tree.** You and the Implementer share one working tree and git index. Run your read-only gates and compute the branch diff ONLY after the Implementer signals `commit N complete, tree quiesced` via SendMessage. NEVER run git operations that overlap the Implementer's -- do not read the tree while it is mid-commit or mid-rebase. If the harness offers git worktrees (see the `using-git-worktrees` skill), use one worktree per teammate to make the parallelism truly race-free; otherwise rely strictly on the quiescence signal.

**3. Dispatch the three fresh reviewer subagents UNCHANGED.** Dispatch the security, quality, and standards reviewers exactly as written in step 2 above, over the branch diff. Because teammates cannot spawn teammates, these MUST be classic Agent-tool subagents (not team members) -- which is exactly what the base skill already uses, so fresh-context review is preserved.

**4. Triage, then ARTICULATE each Fix (do not apply it).** Triage per step 3 (Fix / Reject / Defer). For each **Fix** finding: create ONE shared task-list item and SendMessage the Implementer the precise change (file:line, what to change, why, and the rule cited). Do NOT edit. The Implementer claims the task, applies the fix, re-runs the writing gates to green, commits it (fixup+autosquash where the fix maps cleanly to one existing commit, else a conventional commit), and signals `commit <sha> complete, tree quiesced`, then WAITS.
- **Reject:** record the reason (one line) in the disposition table; do not create a task.
- **Defer:** SendMessage the Implementer to add a TODO comment citing the finding; record it as deferred.

**5. VERIFY each completion and loop to convergence.** After each `commit <sha> complete` signal, re-check the tree (quiesced) and verify the fix against the ORIGINAL finding. If unsatisfied, re-articulate to the Implementer and keep the task open. Loop until all Fix findings are closed.

**6. Whole-PR pre-open pass, then report review-ready.** Once converged, run the whole-PR pre-open pass (the full read-only gate + diff review over the complete branch) and then SendMessage the LEAD that the branch is review-ready, including the findings-disposition table from step 6.

**Architecture-deviation findings -> flag the LEAD and BLOCK.** If a finding would require deviating from the approved plan's architecture (autonomy exception 2), do NOT decide it yourself and do NOT articulate a fix -- SendMessage the LEAD and BLOCK until the lead acknowledges with a decision. Because SendMessage delivery is not battle-tested, the fail-safe default on non-delivery is to STALL (never proceed).

**"Surface to the user" is redefined for teammates.** Everywhere this skill (and its Red Flags) says to surface, escalate, or report to the user, a teammate instead **SendMessages the LEAD**. Teammates NEVER address the human directly for pipeline decisions; the lead is the sole human-facing router.

## Reviewer Rubrics

### Security Reviewer

You are reviewing a branch diff for security defects. Read CLAUDE.md first. Examine every new/changed file in the diff against these rules:

- **Auth on every new route (both main + internal):** Every route must specify its auth mechanism. Main-server routes use JWT/API-key auth. Internal-server routes MUST be in the `protectedMethods` map for `ApiKeyUnaryInterceptor`. This is architecture-derived: cite codebase precedent at `pkg/auth/middleware/middleware.go` (`ApiKeyProtectedMethods()` and `ApiKeyUnaryInterceptor`) and `pkg/rpcServer/internalServer.go` where existing internal routes are registered. If a new route is added without auth configuration, that is HIGH.
- **Auth TEST coverage for new routes:** It is not enough that a route has auth configured -- there must be a TEST that proves unauthenticated/wrong-key requests are rejected. Specifically: if a new internal-server route is added, check whether any test file verifies that calling the route WITHOUT the API key (or with a wrong key) returns Unauthenticated. The interceptor unit test (`middleware_test.go`) proves the interceptor works generically, but there must ALSO be a test proving the route is actually IN the protected map. If no such test exists, that is HIGH.
- **Tenant isolation on every query + cross-tenant TEST:** Every data-access method that takes a resource ID must scope by tenant (org/account). Additionally, there MUST be a test that attempts cross-tenant access and verifies it is rejected for EVERY endpoint (not just some). If the handler test covers Set/List/Delete cross-tenant but NOT Get, that is a gap. Check every RPC method individually. If any endpoint scopes by tenant but has no cross-tenant rejection test, that is HIGH.
- **SQL injection (no raw SQL from user input):** Examine every raw SQL construction in the diff. Query text must NEVER be built by string concatenation or `fmt.Sprintf` with user-supplied values, and `gorm.Raw`/`Exec` calls must use `?` placeholder binding for every dynamic value -- never interpolation. Codebase precedent: the models layer uses GORM with parameterized queries and subqueries (e.g., `WHERE status_id = (SELECT id FROM release_status WHERE name = ?)` per CLAUDE.md's "Enum Lookup Tables" pattern). Any deviation from placeholder binding where a user-controllable value reaches query text is HIGH. Table/column names built from variables are also HIGH unless validated against a hardcoded allowlist.
- **Unvalidated proto fields:** Every field of a new/changed proto request message must be validated before use: required-field presence (non-empty IDs), format/length checks (codebase precedent: `secretsService.ValidateName` -- regex + length limit applied at the top of the handler), and enum/range checks for numeric or enum fields. Check every handler in the diff: a handler that passes `req` fields straight to service/model calls without validation is a finding. Severity by sink: HIGH if the unvalidated field reaches a query, crypto operation, or filesystem/exec path; MEDIUM otherwise.
- **No secrets/keys in logs:** No code may log a full request/response object (they carry auth context). Logging must use specific named fields only. If code logs `"request", req` or similar full-object serialization, that is HIGH.
- **Race conditions in concurrent paths:** Any read-then-write pattern (SELECT then INSERT/UPDATE) without proper locking or conflict handling is HIGH. Specifically examine every "upsert" or "get-or-create" function: if it does SELECT first and then INSERT on not-found, two concurrent calls for the same key BOTH get "not found" (no row exists to lock), BOTH attempt INSERT, and one hits the unique constraint. If the duplicate-key error is not caught and retried (or ON CONFLICT is not used), this is an unhandled 500 in production. Also look for: map access without mutex in goroutines, shared state modified in parallel loops. Every model method that does SELECT-then-INSERT must be checked for this pattern.
- **Encryption/crypto hygiene:** Ciphertext output MUST include a version/algorithm identifier byte so encryption can be rotated without breaking existing data. Hardcoded dev/test keys MUST always produce a visible warning (not gated behind --verbose or debug flags). If either is missing, that is HIGH.
- **Input validation + size caps:** All user-supplied input (stdin, file reads, request fields) must have size limits. Unbounded reads (io.ReadAll without LimitReader) are MEDIUM. Missing client-side validation that forces a round-trip to the server for rejection is MEDIUM.
- **Metadata-only reads (check the FULL data path, not just the wire):** User-facing list/get endpoints for sensitive resources (secrets, keys, credentials) should return metadata only (names, timestamps, IDs) -- not the actual secret values. The full value should require a separate explicit retrieval path or be internal-only. If a user-facing RPC returns secret values in its standard response, that is HIGH. CRITICALLY: it is NOT enough that the response proto omits the value -- trace the query all the way to the database. If the handler's service/model call SELECTs or loads the sensitive column (e.g., a GORM `Find` on the full model struct including the encrypted value) on a path that only needs metadata, that is a least-privilege violation: the value transits the DB connection, driver buffers, and process memory for no reason. Metadata endpoints must use metadata-only queries (explicit column selection omitting the sensitive column). Codebase precedent: commit 66206cb added `ListSecretMetadata`/`GetSecretMetadata` service methods backed by metadata-only model queries because the user-facing read paths were loading full rows including the encrypted value. Flag as HIGH.

### Quality Reviewer

You are reviewing a branch diff for quality defects. Read CLAUDE.md first. Examine every new/changed file against these rules:

- **Test coverage for every new function:** Every new exported function, method, or handler must have a corresponding test. Validation/parsing functions (e.g., `ValidateName`, `decodeKey`, dotenv parsing) MUST have dedicated unit tests -- integration tests alone are insufficient. Missing tests are MEDIUM.
- **Cross-tenant rejection test:** If the diff adds data-access methods scoped by tenant, there must be a test that specifically attempts access with a different tenant's credentials and asserts rejection. Missing cross-tenant tests are HIGH.
- **Error path completeness:** Every operation that can fail must handle the error explicitly. Silent drops (`_ = err`) or missing error returns are HIGH.
- **Non-deterministic behavior:** Code that iterates Go maps and produces user-visible output (error messages, logs, API responses) has non-deterministic ordering. If bulk operations process items from a map and report partial failures, the error messages will be non-reproducible. Flag non-deterministic user-facing output as MEDIUM.
- **Log noise from handled errors:** If code catches and retries an error (e.g., a duplicate-key conflict in an upsert), it should NOT also log the error at ERROR/WARN level -- that creates confusing noise for operators who see errors that were actually handled. Specifically: if an upsert function logs an error when the initial SELECT or INSERT fails, but then retries or falls through to an alternate path, the first-attempt log is noise. Check every error log in upsert/retry paths. Flag as LOW.
- **Client-side validation:** If the server validates input (name format, value length, etc.), the CLI client should perform the same validation locally before making the RPC call, for immediate user feedback. Missing client-side validation is MEDIUM.

### Standards Reviewer

You are reviewing a branch diff for violations of project conventions documented in CLAUDE.md. Read CLAUDE.md first. For every finding, quote the specific CLAUDE.md section title and rule text that is violated. Check:

- **Protobuf-only routes (CLAUDE.md: "Protobuf & gRPC"):** "All HTTP and gRPC routes must be defined as protobuf services with `google.api.http` annotations -- never add hand-written HTTP handlers directly to the server." If the diff adds `mux.HandleFunc(...)` or a direct HTTP handler, that is HIGH.
- **Enum Lookup Tables (CLAUDE.md: "Enum Lookup Tables"):** "Status fields and other categorical values use lookup tables instead of raw TEXT columns." If a migration adds a TEXT/VARCHAR column for a status/type/category field, that is HIGH.
- **Minimal GORM tags (CLAUDE.md: "GORM struct tags"):** "Keep `gorm:` tags minimal." Only `primaryKey`, `default:uuid_generate_v1()`, and `autoIncrement` are permitted. Any other tag is a finding.
- **Migration registered in GetMigrations() (CLAUDE.md: "Adding Migrations"):** Every new migration must be registered in `pkg/postgres/migrations/migrations.go:GetMigrations()`.
- **CLI doc-sync (CLAUDE.md: "Bundled agent skill" + "User-facing documentation"):** "After adding or changing any CLI command or flag, run `make skill` and commit the result." ALSO: CLI changes must update `ecloud-ui/src/content/docs/cli.md` AND `ecloud-cli/README.md`. If new CLI commands exist without ALL THREE being updated, that is HIGH.
- **Sugared-logger convention (CLAUDE.md: "Logging"):** "Always use `logger.Sugar()` with the `w`-suffixed methods (`Debugw`, `Infow`, `Warnw`, `Errorw`) and pass key-value pairs -- never use `zap.Field` helpers (e.g., `zap.String`, `zap.Uint64`) with the sugared logger." If new code uses `zap.Error()`, `zap.String()`, or the non-sugared logger, that is MEDIUM.
- **Conventional commits (CLAUDE.md: "Commit messages"):** "Do NOT add `Co-Authored-By` or any other trailers to commit messages." Check commit messages in the branch for trailer violations.
- **`make skill` drift:** If the diff adds/modifies CLI commands or flags, verify that the generated skill file (`ecloud-cli/cmd/skill/assets/SKILL.md`) was regenerated. If the diff includes CLI changes but no corresponding skill-file update, that is HIGH.
- **golangci-lint clean:** Run `make lint` (or `golangci-lint run ./...`) against the diff scope. Any findings (unused variables, ineffectual assignments, shadow declarations, etc.) are MEDIUM. The linter is the first mechanical gate and MUST pass before push.

## Red Flags (from baseline failure modes)

These are rationalization patterns observed in baseline reviews. If you catch yourself thinking any of these, STOP and re-examine:

1. **"The implementation looks correct, so no test is needed."** WRONG. Correct implementations still need tests that prove the correctness and prevent regressions. Missing tests are always a finding.
2. **"Tenant isolation is implemented correctly, so cross-tenant access is fine."** WRONG. Implementation correctness and test coverage are independent concerns. A cross-tenant rejection test MUST exist even if the implementation looks solid.
3. **"The diff is too large to review fully."** WRONG. Chunk by package and review each chunk. Never skip files due to diff size.
4. **"This data isn't sensitive, so auth/security is lower priority."** WRONG. Auth requirements are structural and apply regardless of data sensitivity. Every route needs auth; no exceptions.
5. **"Tests pass, so the code is fine."** WRONG. Tests passing proves the happy path works. It does not prove security boundaries hold, error paths are handled, or conventions are followed.
6. **"The values are encrypted, so returning them to users is safe."** WRONG. User-facing list/get endpoints should return metadata only. Even encrypted values should not be exposed in standard responses if the design can separate metadata reads from value retrieval.
7. **"The response proto omits the value, so the metadata-only requirement is satisfied."** WRONG. Check what the query LOADS, not just what the response SENDS. A handler that maps to a metadata-only proto but whose service call fetches full rows (including the sensitive column) from the database still violates least-privilege data fetch. Trace handler -> service -> model -> SQL.
