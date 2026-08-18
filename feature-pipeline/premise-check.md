# Premise & blast-radius check

Shared by `plan-feature` and `ship-feature`. Run it FIRST, by reading code, before designing any
detail. A plan built on a wrong premise passes every downstream review: three fresh reviewers all
bless a correct-looking implementation of the wrong thing, because the error is baked equally into
the plan and the spec. Review layers cannot catch a shared blind spot.

**The expected result is that the premise holds and you proceed.** That is the normal case, not a
failure to detect something. Read this whole file before concluding otherwise — a false premise
failure costs a human round-trip and, in the issue-driven pipeline, a parked slot.

## What to answer

Answer each by grepping and reading, recording findings with `file:line` evidence in the spec (or,
for a Small change that skips the spec, in the plan's Global Constraints preamble):

- **Entry path** — how is the thing I am changing actually reached? Who calls this
  endpoint/function/flag? Is there a proxy, gateway, BFF, or other indirection in front of it?
- **Blast radius** — what else lives on this path? Enumerate every repo, service, and surface the
  change touches or that consumes its output. If the answer includes another repo, that repo is IN
  SCOPE for this run, not a follow-up.
- **Prior art** — is there an existing config, pattern, or host to reuse instead of inventing one?
- **Profile + class** — set the active profile(s) and the Right-Sizing class per `conventions.md`,
  and record both in Pipeline State with a one-line reason.

If an answer is unknown after a reasonable search, that is itself a finding — surface it, do not
paper over it with a plausible guess.

## Three outcomes, in descending frequency

**1. Premise holds — proceed.** Most runs end here.

**2. Premise holds, details are stale — reconcile and proceed.** The issue's line numbers,
filenames, or descriptions have drifted, or a feature built since it was written changed the
shape. Fix it yourself, note the correction in one line in the plan, and continue. **This is not a
block.** A project's premise evolves through the features built against it; an issue written three
weeks ago describing the code as it was then is normal, not wrong.

**3. Premise fails — park.** Only these four count:

- the thing **already exists and works**;
- building it would **violate a documented invariant** the issue did not account for;
- the blast radius is **several times the ask**;
- it is genuinely **several issues wearing one hat**.

Post the finding with your reasoning and a recommendation, and STOP. Do not close the issue, do not
silently reshape it into a different feature, and do not invent scope it does not support. The
human decides.

## Not premise failures

- **The code does not yet contain the feature.** That is the point of the request. An issue
  describes a desired end state and the code describes the current one; they differ by
  construction.
- **A stale path, line number, or symbol name.** Outcome 2.
- **The issue proposing an approach you would implement differently.** Implement the better one and
  say why in the plan.
- **A gap you could close by reading more code.** Read it.

If you can state the correction with confidence, the correction is the plan — not a question, and
not a block.
