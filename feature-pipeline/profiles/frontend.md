# Profile: frontend

UI and client work — components, client state, styling, interaction, and accessibility. This profile changes what the planner authors, how the executor verifies, and which dimensions the reviewers grade; it does **not** change the engine's stages, gate, autonomy contract, or model tiering.

A profile supplies three slices, each consumed by a different phase. **`plan-feature` loads the planner slice (planning) and the reviewer slice (plan review); `build-feature` loads the executor slice (implementation) and the reviewer slice (commit review); `ship-feature` loads all three, at its stages 1, 3, and 4.** Load this file at the start of each and apply the matching slice.

## Planner slice (stage 1)

**Compose `frontend-design` when the change introduces or reshapes UI.** The senior planner does the taste-heavy work up front and pins the result so the mid-level executor implements to a fixed spec — do not leave aesthetic choices to the executor. Produce and record in the plan a compact token system: color (4–6 named hex values), type (display + body + utility roles), layout concept, and the one signature element. These tokens become the executor's acceptance criteria and the reviewer's design-fidelity yardstick.

**Domain right-sizing triggers** — what Small/Standard/Large *mean* here:

- **New surface:** a new route/page/screen, a new global store/context/provider, a new design-system token or primitive, or a new client dependency. Adds surface → at least Standard; a new screen wired to new global state → Large.
- **Risky boundary (frontend-specific):** login/auth UI, anything that displays or collects PII/credentials/money, destructive or irreversible actions (delete, publish, pay) that need a confirmation step, and any surface that renders untrusted/user-supplied content (an XSS sink). Touching one → at least Standard, usually Large, and tag it `review: yes`.
- **Mechanical vs. semantic:** a restyle, a copy change, moving a component, or a prop-preserving refactor is Mechanical; new interaction logic, new state, or a new data dependency is semantic.

**Task shapes to author:** build component (structure + props), wire client state, implement **each render state explicitly** (empty / loading / error / success / edge: long text, zero items, many items), apply tokens, add the interaction + its keyboard path, and a component/interaction test step. Enumerate the **responsive breakpoints** the feature must hold and the **a11y contract** (keyboard reachability, focus order, ARIA/semantics, reduced-motion).

**Per-task `review:` tag:** `review: yes` for risky-boundary tasks (auth UI, destructive actions, untrusted-content rendering, forms submitting sensitive data); `review: no` for presentational/mechanical tasks — the whole-diff fan-out still covers them.

**Per-task acceptance to bake in (this is what lets a mid-level executor verify itself):** name the concrete checks, e.g. "renders without overflow at 375 / 768 / 1440; every control keyboard-reachable with visible focus; colors/spacing match tokens `--x`/`--y`; empty + loading + error states present; respects `prefers-reduced-motion`."

## Executor slice (stage 3)

Backend verification is "tests green." Frontend adds **observe the rendered result** — the executor must actually look at the UI, not just pass unit tests. Discover the project's tooling rather than assuming one (repo-agnostic):

1. **Run the app.** Use the `run` skill (or the project's dev/build script) to get the feature on screen.
2. **Drive and observe.** Use the `verify` skill to exercise the real flow. For visual/interaction capture use `claude-in-chrome` or `playwright` if available; otherwise the project's e2e tool (Cypress/Playwright) or a component harness (Storybook). Screenshot at **each breakpoint the plan named** and check the render against the task's acceptance criteria — states present, no overflow, tokens matched.
3. **Component/interaction tests.** Discover and use the project's test tooling (Vitest/Jest + Testing Library, Playwright, Cypress). Cover the interaction and each non-trivial state.
4. **Accessibility check.** Tab through the feature: every control reachable, focus visible, order sensible. Run the project's a11y linter/`axe` if present.
5. **Mechanical gates.** Run format/lint/test/typecheck as the backend profile does.

A task is done when its screenshots match the acceptance criteria, its component tests pass, the a11y check is clean, and the gates are green. When the environment cannot render the UI at all, that is a blocker to surface — do not mark a visual task done on unit tests alone.

## Reviewer slice (stages 2 & 4)

Replace the backend security/quality/standards rubric with the dimensions below (keep **standards** and the shared **quality** items — test coverage, error/empty-state handling, commit-message format, doc-sync — from the existing rubrics). When both profiles are active, keep the backend security rubric too and add the `seam` addendum.

- **Accessibility (a11y):** every interactive element keyboard-reachable with a **visible focus** indicator; correct semantics/ARIA (buttons are buttons, not click-handling divs); images have `alt`; inputs have associated labels; color contrast meets WCAG AA; `prefers-reduced-motion` respected. Missing keyboard access or focus is **HIGH**.
- **Responsive:** renders without overflow or broken layout at every breakpoint the plan named; no horizontal scroll on mobile; touch targets adequately sized. A broken mobile layout is **HIGH**.
- **Design fidelity:** matches the plan's token system — no hardcoded colors/spacing/type that bypass the tokens; every render state (empty/loading/error/success) is implemented, not just the happy path. Stray hardcoded values or a missing state are **MEDIUM**.
- **Client security:** no untrusted input reaching an HTML sink (`dangerouslySetInnerHTML`, `v-html`, `innerHTML`, `eval`) without sanitization — **HIGH** (XSS); no secrets/keys/tokens shipped in the client bundle — **HIGH**; auth-gated UI is defense-in-depth only — the real check must exist server-side (flag a client-only gate as **MEDIUM**); destructive actions require an explicit confirmation step.
- **Quality (shared):** component/interaction test coverage for new components; no uncaught console errors; error and empty states handled — same severities as the backend quality rubric.
- **Standards (shared):** the project's component conventions, styling approach, commit-message format, and doc-sync — from the existing standards rubric.

Drop the backend-only categories (tenant isolation on queries, migration registration, raw-query injection) when the change is pure-frontend.
