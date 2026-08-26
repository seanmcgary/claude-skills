---
name: reviewing-commits
description: Use when a feature branch is locally complete and needs security, quality, and standards review plus lint/test gates before pushing or opening a PR
---

## Overview

The branch author cannot effectively review their own commits. Fresh-context subagents reviewing along a single dimension each will catch defects that a single generalist pass rationalizes away or buries in noise. The goal: round 1 of PR review should find nothing mechanical.

This skill runs mechanical gates first, then dispatches three parallel reviewer subagents (security, quality, standards) over the branch diff, triages their findings, applies fixes as commits, re-runs the gates, and produces a findings-disposition table.

## Project conventions doc

This skill stays at the workflow level; every project-specific rule (coding style, auth model, schema conventions, commit format, gate commands) lives in repo-level files, NOT here. Resolve them once, up front:

- **Prose conventions — the "conventions doc":** the first of these that exists at the repo root: `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `STANDARDS.md`, `STYLEGUIDE.md`/`STYLE.md`. If more than one exists, read them in that order and treat later ones as supplements (a primary agent-instructions file may link out to a detailed `STANDARDS.md` / `docs/` guide — follow those links). "Read the conventions doc" below means this resolved set.
- **Machine-enforced style — always also consult:** `.editorconfig` and the project's linter/formatter configs (`.eslintrc`/`.prettierrc`, `.golangci.yml`, `rustfmt.toml`, `ruff.toml`/`pyproject.toml`, etc.). These are the source of truth for formatting/lint rules; the mechanical gates run them.
- **Gate commands** are discovered as described in Mechanical Gates below.

If no conventions doc exists, note that in the findings table and review against the universal rubric categories alone (plus whatever the linter/formatter configs enforce). To generate one from the codebase for future runs, use the `identify-standards` skill.

## Mechanical Gates (run first, in order)

These gates run BEFORE dispatching reviewers. A red gate blocks reviewer dispatch -- there is no point reviewing a broken branch.

Discover the project's format, lint, and test commands from its conventions doc (see "Project conventions doc" above), its `Makefile`, or its package manifest (`package.json` scripts, `Cargo.toml`, `pyproject.toml`, etc.). Typical shapes: `make fmt`/`make lint`/`make test`, `npm run format`/`npm run lint`/`npm test`, `cargo fmt`/`cargo clippy`/`cargo test`. Run them in this order:

1. **Format** -- run the formatter; if any files change, commit them (`fix: format code`).
2. **Lint** -- run the linter; must produce 0 issues. If there are findings, fix them and commit (`fix: resolve lint findings`). Re-run until clean.
3. **Test** -- run the full test suite. This is the pipeline's **single full-suite run**
   (per-task runs were targeted — see "Test cadence" in `conventions.md`); it exists to catch
   cross-task regressions. If tests fail, fix and commit. Re-run until green.

Only proceed to the reviewer dispatch once all three gates are green.

## Process

### 1. Compute the diff

Determine the repo's default branch, then diff against the merge-base:

```bash
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/@@')
BASE=${BASE:-origin/main}   # fall back to origin/main, or origin/master on older repos
git diff $(git merge-base "$BASE" HEAD)...HEAD
```

This is the scope the reviewers examine. If the diff exceeds 3000 lines, chunk it by directory/package for each reviewer so they can review in focused passes rather than rationalizing "the diff is too large to review fully."

### 2. Dispatch three reviewers in parallel

Send three Agent tool calls in a single message (one per dimension). Resolve the project's conventions doc as described in "Project conventions doc" above. Each subagent receives:
- The diff scope (the command above, or the chunked file list)
- The path to the conventions doc
- Its dimension-specific rubric (below, verbatim)
- The instruction: "Read the conventions doc first. Run the diff command above to see the full diff. For every finding, cite the specific rule violated (conventions-doc section title or codebase precedent). Output findings as a structured list: `| # | File:Line | Finding | Severity | Rule Cited |`. Do not modify any files."

If a reviewer returns garbage (no structured findings, off-topic, or fewer than 3 sentences), re-dispatch that single reviewer once with a more explicit prompt. If the re-dispatch also returns garbage or off-topic output, perform that dimension's review yourself inline in the current context using the same rubric, and note the fallback in the findings table.

### 3. Triage findings

For each finding from all three reviewers, decide:
- **Fix:** implement the fix. State what was changed.
- **Reject:** the finding is incorrect or not applicable (same concept pr-feedback-loop calls "push back" -- disagree with reasoning, don't apply). State why in one line.
- **Defer:** the finding is real but out of scope for this branch. Create a TODO comment in the relevant file citing the finding.

### 4. Fold fixes into the branch

- If a fix belongs to exactly one existing commit, use `git commit --fixup=<sha>` and then re-resolve `$BASE` as in step 1 and run `git rebase -i --autosquash $(git merge-base "$BASE" HEAD)` to fold it in.
- If a fix spans multiple commits or is new work, create a conventional commit (e.g., `fix: add cross-tenant test for GetSecret`).

### 5. Re-run the three gates

After all fixes are applied, re-run the project's gates in order: format -> lint -> test. Fix any regressions. The branch must be green before producing the final output.

### 6. Findings-disposition table

Output the final table:

```
| # | File:Line | Finding | Dimension | Severity | Disposition | Notes |
|---|-----------|---------|-----------|----------|-------------|-------|
```

## Reviewer Rubrics

> **Adapting the rubrics to your project:** The categories below are universal, but the concrete rule for each lives in the project's conventions doc and codebase. Before dispatching, skim the conventions doc and note the project's specifics (auth mechanism and where routes register it, whether it is multi-tenant, its data-access/ORM layer, logging API, doc-sync/codegen requirements, commit-message format). Feed those specifics to the reviewers alongside the category, and cite codebase precedent (the file/function where the existing pattern lives) when the rule is architecture-derived rather than written down. Skip any category the project does not have (e.g., no multi-tenancy, no CLI, no encryption-at-rest). Examples in each bullet are illustrative — replace them with the project's actual API names, paths, and file references.

> **Profile-driven dimensions (when invoked by the feature pipeline):** If a domain **profile** is active (`$SKILLS_ROOT/feature-pipeline/profiles/*.md` per the portability note in `conventions.md`), its reviewer slice is the source of truth for *which* dimensions to run. The three rubrics below are the `backend` profile. The `frontend` profile substitutes accessibility / responsive / design-fidelity / client-security for the Security rubric and keeps Quality + Standards; full-stack runs both sets plus the seam's contract-consistency / error-propagation / validation-parity checks. Read the active profile and dispatch one fresh reviewer per dimension it names (group related dimensions into a single reviewer where sensible — the "three parallel reviewers" count is a default, not a fixed set); keep each reviewer fresh-context and single-focus.

### Security Reviewer

You are reviewing a branch diff for security defects. Read the project's conventions doc first. Examine every new/changed file in the diff against these categories:

- **Auth on every new route/endpoint:** Every route must specify its auth mechanism. If the project has more than one server surface (e.g., a public API and an internal service), each has its own auth requirement — find where routes register their auth (an interceptor/middleware allowlist, a decorator, a route guard) and confirm the new route is covered. This is often architecture-derived: cite the codebase precedent (the file where existing routes register auth) rather than a doc quote. If a new route is added without auth configuration, that is HIGH.
- **Auth TEST coverage for new routes:** It is not enough that a route has auth configured — there must be a TEST that proves unauthenticated/wrong-credential requests are rejected. A generic interceptor/middleware unit test proves the mechanism works, but there must ALSO be a test proving the specific new route is actually protected. If no such test exists, that is HIGH.
- **Tenant/owner isolation on every query + cross-tenant TEST (if multi-tenant):** Every data-access method that takes a resource ID must scope by tenant (org/account/user). Additionally, there MUST be a test that attempts cross-tenant access and verifies it is rejected for EVERY endpoint (not just some). If the tests cover create/list/delete cross-tenant but NOT get, that is a gap. Check every method individually. If any endpoint scopes by tenant but has no cross-tenant rejection test, that is HIGH.
- **Injection (no raw queries/commands from user input):** Examine every raw query or command construction in the diff. Query/command text must NEVER be built by string concatenation or interpolation with user-supplied values — use parameterized/placeholder binding for every dynamic value. Any deviation where a user-controllable value reaches query/command text is HIGH. Identifiers (table/column/path names) built from variables are also HIGH unless validated against a hardcoded allowlist.
- **Unvalidated request fields:** Every field of a new/changed request message must be validated before use: required-field presence (non-empty IDs), format/length checks, and enum/range checks for numeric or enum fields. A handler that passes request fields straight to service/data-layer/exec calls without validation is a finding. Severity by sink: HIGH if the unvalidated field reaches a query, crypto operation, or filesystem/exec path; MEDIUM otherwise.
- **No secrets/keys in logs:** No code may log a full request/response object (they carry auth context). Logging must use specific named fields only. If code logs a whole request/response object or similar full-object serialization, that is HIGH.
- **Race conditions in concurrent paths:** Any read-then-write pattern (read then insert/update) without proper locking or conflict handling is HIGH. Specifically examine every "upsert" or "get-or-create" function: if it reads first and then inserts on not-found, two concurrent calls for the same key BOTH see "not found", BOTH attempt insert, and one hits the unique constraint. If the duplicate-key error is not caught and retried (or an atomic upsert / ON CONFLICT is not used), this is an unhandled error in production. Also look for: shared map/state access without synchronization across concurrent tasks. Every method that does read-then-insert must be checked for this pattern.
- **Encryption/crypto hygiene (if the diff handles encryption):** Ciphertext output MUST include a version/algorithm identifier so encryption can be rotated without breaking existing data. Hardcoded dev/test keys MUST always produce a visible warning (not gated behind a verbose/debug flag). If either is missing, that is HIGH.
- **Input validation + size caps:** All user-supplied input (stdin, file reads, request fields) must have size limits. Unbounded reads (reading an entire stream without a limit) are MEDIUM. Missing client-side validation that forces a round-trip to the server for rejection is MEDIUM.
- **Metadata-only reads (check the FULL data path, not just the wire):** User-facing list/get endpoints for sensitive resources (secrets, keys, credentials) should return metadata only (names, timestamps, IDs) — not the actual sensitive values. The full value should require a separate explicit retrieval path or be internal-only. If a user-facing endpoint returns sensitive values in its standard response, that is HIGH. CRITICALLY: it is NOT enough that the response type omits the value — trace the query all the way to the data store. If the handler's service/data-layer call loads the sensitive column/field (e.g., loading the full record including the encrypted value) on a path that only needs metadata, that is a least-privilege violation: the value transits the connection, driver buffers, and process memory for no reason. Metadata endpoints must use metadata-only queries (explicit field selection omitting the sensitive column). Flag as HIGH.

### Quality Reviewer

You are reviewing a branch diff for quality defects. Read the project's conventions doc first. Examine every new/changed file against these rules:

- **Test coverage for every new function:** Every new exported function, method, or handler must have a corresponding test. Validation/parsing functions MUST have dedicated unit tests — integration tests alone are insufficient. Missing tests are MEDIUM.
- **Cross-tenant rejection test (if multi-tenant):** If the diff adds data-access methods scoped by tenant, there must be a test that specifically attempts access with a different tenant's credentials and asserts rejection. Missing cross-tenant tests are HIGH.
- **Error path completeness:** Every operation that can fail must handle the error explicitly. Silent drops (ignoring/swallowing an error) or missing error returns are HIGH.
- **Non-deterministic behavior:** Code that iterates an unordered collection (e.g., a hash map) and produces user-visible output (error messages, logs, API responses) has non-deterministic ordering. If bulk operations process items from such a collection and report partial failures, the messages will be non-reproducible. Flag non-deterministic user-facing output as MEDIUM.
- **Log noise from handled errors:** If code catches and retries an error (e.g., a duplicate-key conflict in an upsert), it should NOT also log the error at ERROR/WARN level — that creates confusing noise for operators who see errors that were actually handled. Check every error log in upsert/retry paths. Flag as LOW.
- **Client-side validation:** If the server validates input (name format, value length, etc.), a client (CLI/SDK/UI) should perform the same validation locally before the round-trip, for immediate user feedback. Missing client-side validation is MEDIUM.

### Standards Reviewer

You are reviewing a branch diff for violations of the project's documented conventions. Read the project's conventions doc first. For every finding, quote the specific section title and rule text that is violated. The categories below are the ones project conventions most commonly govern — check each against what the conventions doc actually says, and skip any the project has no rule for:

- **Routing/handler convention:** If the project mandates one way to define routes (e.g., IDL/protobuf-defined services, a specific router, a codegen step) and forbids hand-written handlers that bypass it, flag any diff that adds a bypassing handler. HIGH when the doc states such a rule.
- **Schema/enum conventions:** If the project constrains how categorical/status values and columns are modeled (e.g., lookup tables vs. raw enum columns, permitted column types), flag migrations/models that deviate.
- **ORM/model tag conventions:** If the conventions doc restricts which ORM/serialization struct tags are permitted, flag any tag outside the allowed set.
- **Migration registration:** If new migrations must be registered in a central list/registry, flag any migration not registered there.
- **Doc-sync / codegen drift:** If changing a CLI command, API, or public interface requires updating specific docs AND/OR running a generation command (with CI failing on drift), flag a diff that changes those without updating ALL required targets and regenerating. Enumerate every target from the conventions doc; a partial update is HIGH.
- **Logging convention:** If the diff uses a logging API/style other than the project's documented one, flag it (severity per the doc's emphasis; typically MEDIUM).
- **Commit-message convention:** Check the branch's commit messages against the project's format (e.g., conventional commits; trailer restrictions such as no `Co-Authored-By`). Flag violations.
- **Linter clean:** Run the project's linter against the diff scope. Any findings (unused variables, ineffectual assignments, shadowing, etc.) are MEDIUM. The linter is the first mechanical gate and MUST pass before push.

## Red Flags (from baseline failure modes)

These are rationalization patterns observed in baseline reviews. If you catch yourself thinking any of these, STOP and re-examine:

1. **"The implementation looks correct, so no test is needed."** WRONG. Correct implementations still need tests that prove the correctness and prevent regressions. Missing tests are always a finding.
2. **"Tenant isolation is implemented correctly, so cross-tenant access is fine."** WRONG. Implementation correctness and test coverage are independent concerns. A cross-tenant rejection test MUST exist even if the implementation looks solid.
3. **"The diff is too large to review fully."** WRONG. Chunk by directory/package and review each chunk. Never skip files due to diff size.
4. **"This data isn't sensitive, so auth/security is lower priority."** WRONG. Auth requirements are structural and apply regardless of data sensitivity. Every route needs auth; no exceptions.
5. **"Tests pass, so the code is fine."** WRONG. Tests passing proves the happy path works. It does not prove security boundaries hold, error paths are handled, or conventions are followed.
6. **"The values are encrypted, so returning them to users is safe."** WRONG. User-facing list/get endpoints should return metadata only. Even encrypted values should not be exposed in standard responses if the design can separate metadata reads from value retrieval.
7. **"The response type omits the value, so the metadata-only requirement is satisfied."** WRONG. Check what the query LOADS, not just what the response SENDS. A handler that maps to a metadata-only response type but whose service call fetches full records (including the sensitive column) from the data store still violates least-privilege data fetch. Trace the full path: handler -> service -> data layer -> query.
