# Profile: backend

The **default** profile. Server, data, and API work — routes, handlers, business logic, schema/migrations, background jobs, integrations. This profile is the pipeline's original baseline; it makes the default behavior explicit so the other profiles can differ from a named starting point.

A profile supplies three slices, each consumed by a different phase. **`plan-feature` loads the planner slice (planning) and the reviewer slice (plan review); `build-feature` loads the executor slice (implementation) and the reviewer slice (commit review); `ship-feature` loads all three, at its stages 1, 3, and 4.** Load this file at the start of each and apply the matching slice.

## Planner slice (stage 1)

**Domain right-sizing triggers** — what the engine's Small/Standard/Large dimensions *mean* here:

- **New surface:** a new route/endpoint, DB table or migration, config var, message/queue topic, external integration, or subsystem. Adds surface → at least Standard; new subsystem/schema/cross-repo contract → Large.
- **Risky boundary:** auth/authz, payments/money, PII or credentials, data integrity, transactions, concurrency, migrations, or anything on a security-relevant path. Touching one → at least Standard, usually Large.
- **Mechanical vs. semantic:** a signature-preserving refactor, a rename, or a default-off flag is Mechanical even across many files; anything requiring per-site reasoning about new behavior is semantic.

**Task shapes to author:** migration (+ registration), request/response type + validation, route/handler wired to its auth mechanism, service method, data-access method scoped by owner/tenant, and a test step per producing task. Order so schema precedes the code that reads it.

**Per-task `review:` tag** — flag `review: yes` for tasks that touch a risky boundary (auth, money, migrations, raw queries, concurrency, cross-tenant data); flag `review: no` for mechanical/transcription tasks whose correctness the whole-diff fan-out will still cover. See `conventions.md`'s Review Cadence for how the tag is used.

**Per-task acceptance to bake in:** the covering test(s) pass; error paths return/handle explicitly; the change respects the Global Constraints copied into the plan.

## Executor slice (stage 3)

Verification is **tests + mechanical gates green**. Follow TDD: write the covering test, make it pass, then the project's format/lint/test gates. A task is done when its named tests pass and the gates are clean — no browser, no screenshots. This is the baseline the mid-level executor already knows how to reach; the frontend profile is where verification diverges.

## Reviewer slice (stages 2 & 4)

The backend reviewer rubric **is** the security / quality / standards rubric already written into `reviewing-plans` (stage 2) and `reviewing-commits` (stage 4). Use those rubrics verbatim — they are the backend dimensions:

- **Security:** auth on every route + a test proving it; tenant/owner isolation on every query + a cross-tenant rejection test; no injection; validated request fields; no secrets in logs; race conditions in read-then-write paths; crypto hygiene; metadata-only reads on sensitive resources.
- **Quality:** test per new function; error-path completeness; deterministic user-facing output; interface consistency (Produces/Consumes).
- **Standards:** routing/handler convention, schema/enum conventions, migration registration, doc-sync/codegen, logging style, commit-message format.

No profile-specific additions or removals — this profile is the rubric's native domain.
