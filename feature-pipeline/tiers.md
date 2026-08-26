# Model tiers — the single resolution point

Every skill in this pipeline names a **tier** (`senior` or `mid`), never a concrete model.
This file is the **only** place a tier becomes a concrete model. It defines the resolver that
turns a tier into the model to dispatch, and it is provider-agnostic: under Claude, OpenAI,
Gemini, or any other agent, the same tier meaning holds.

Skill prose references `tiers.md` when it needs a model. Do **not** restate model names in a
skill file or a profile — put them here, once.

## The two tiers

| Tier | Typical work | Effort intent |
|------|--------------|---------------|
| `senior` | planning (premise check, brainstorm, design sync, plan authoring), every reviewer, orchestration + review triage | **high** — the taste and judgment work |
| `mid` | per-task implementation, verification | default — execute to a fixed spec |

The tier table in `conventions.md` assigns each pipeline phase to a tier. `plan-feature` runs
entirely at `senior`; `build-feature` orchestrates at `senior` and implements at `mid`.

## The resolver: tier → model

Resolve a tier in this exact order, and stop at the first hit:

1. **Provider row** — if `AGENT_PROVIDER` names a known key, use that provider's model for this
   tier (the table below).
2. **Env override** — else if `<TIER>_MODEL` is set (e.g. `SENIOR_MODEL`, `MID_MODEL`), use it.
   This pins one tier for a run without editing this file.
3. **Inherit** — else resolve to `inherit`: **omit the model on dispatch** so the subagent
   inherits the invoking agent's session model. This is the intended fallback, not a silent
   defect — for an agent/provider this pipeline has no mapping for, the invoking model IS the
   right choice.

## Provider rows

| `AGENT_PROVIDER` | `senior` | `mid` |
|------------------|----------|-------|
| `claude` | `claude-opus-5` | `claude-sonnet-5` |
| `openai` | `gpt-5.1-codex-max` | `gpt-5.1-codex` |
| `gemini` | `gemini-2.5-pro` | `gemini-2.5-flash` |

Add a row for any provider you use. A row is optional — an absent provider simply falls through
to the env override or to `inherit`.

## How the running agent is detected

The resolver uses one explicit contract: the `AGENT_PROVIDER` environment variable
(`claude`, `openai`, `gemini`, ...). Set it once in each agent's environment so the pipeline
knows which provider row to use:

- **Claude Code:** put `export AGENT_PROVIDER=claude` in your shell profile or in the project's
  environment.
- **Pi / other agents:** export it the same way in that tool's environment.
- Pipelines may also *hint* detection from host variables when present (e.g. a Claude-specific
  variable, a Pi-specific variable) and map them to a provider — but `AGENT_PROVIDER` is the
  unambiguous contract and wins.

## Dispatching with a resolved model

When you dispatch a subagent for work at a tier:

1. Resolve the tier with the order above.
2. If the result is a model name, pass it **explicitly** as the subagent's model so the tiering
   holds. Mind aliasing: some agents take a short alias rather than a full model ID (Claude's
   agent tool accepts `opus` / `sonnet`). Use the form the running agent expects.
3. If the result is `inherit`, **omit** the model parameter — the subagent inherits the invoking
   session's model, which is the intended behavior for unknown providers.

## Why tier, not model, in skills

- One edit in this file re-tiers the whole pipeline (e.g. "bump every senior planner to X").
- Skills stay provider-agnostic: the same `ship-feature` runs under Claude, OpenAI, or Gemini
  without an edit.
- The fallback to `inherit` means an agent you have **not** configured still works correctly —
  it just uses its own model.
