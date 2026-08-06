# Profile addendum: seam

**Active only when both `backend` and `frontend` profiles are on** (full-stack). This is not a standalone profile — it is a thin addendum layered on top of both. It governs the one thing neither pure profile owns: the **contract between client and server**. Load it alongside both profiles and apply its slice at the matching stage.

## Planner slice (stage 1)

**The API contract is the shared artifact both sides build to — pin it before either side builds against it.** In the plan, define the request/response shapes, status codes, and error semantics for every endpoint the UI calls, and place them in a section the server task **Produces** and the client task **Consumes** (the same Produces/Consumes discipline the quality plan-reviewer already enforces, applied across the seam). Record it in the plan's `Verified external API (do not re-derive)` section so both sides pin the same signatures.

- **Share types where the stack allows** — a generated client, an OpenAPI/IDL source, or a shared types package — rather than hand-writing the shape twice. If types can't be shared, the contract section in the plan is the single source both tasks copy from.
- **Order the tasks** so the contract (and any shared types) is fixed first, then the server implements it, then the client consumes it. A client task must never precede the contract it depends on.

## Executor slice (stage 3)

Verify **across the seam, end-to-end** — not the client against a mock and the server against `curl` in isolation. Drive the real UI (frontend executor slice) so a user action issues the real request, hits the real handler and data layer (backend executor slice), and the response renders in the UI. Use the `verify` skill for the end-to-end drive. The full-stack task is done only when the round trip works against the real backend, not a stub.

## Reviewer slice (stages 2 & 4)

Add these to the combined frontend+backend rubric:

- **Contract consistency:** the client Consumes exactly what the server Produces — field names, types, optionality, and status codes match. A drift (client reads `userId`, server sends `user_id`; client expects 200, server returns 204) is **HIGH**.
- **Error propagation:** every server error path the contract defines has a corresponding client render state — a 4xx/5xx surfaces the error state, not a crash or a silent swallow. Missing error rendering is **MEDIUM**.
- **Validation parity, server-authoritative:** validation exists on the **server** for every field (never client-only); the client may mirror it for immediate feedback, but a client-only check with no server enforcement is **HIGH**. Flag drift where client and server disagree on what is valid.
