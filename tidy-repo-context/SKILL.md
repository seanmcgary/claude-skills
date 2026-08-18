---
name: tidy-repo-context
description: Use to shrink a bloated repo context file (CLAUDE.md, AGENTS.md, GEMINI.md) into a navigable map plus topic docs, when it has grown large enough that every session and every subagent pays for detail most of them never need. Relocates content, never deletes it, and verifies nothing was lost.
---

## Overview

A repo context file is loaded **in full, into every session and every subagent**, whether or not
any of it is relevant to the task. It is the only file in the repo with that property, which makes
it the one place where length has a recurring cost rather than a one-time one.

Left alone it grows monotonically: each session that learns something appends a paragraph, nothing
ever removes one, and a file that started as orientation becomes a reference manual nobody reads
end to end but everybody pays for.

This skill converts it into a **map**: repo layout, the rules that silently break things, and
pointers to topic docs that hold the depth. Content is **relocated, never deleted** — the detail
was expensive to learn and is still correct; it just should not be resident in every context
window.

## When to run it

Worth running when the context file is **over ~15KB**, or when you can point at whole sections
that only matter to one subsystem. A 5KB file is already a map; leave it alone.

Not a substitute for editing: if the file contains something *wrong*, fix it. This skill assumes
the content is correct and only badly placed.

## Phase 1 — Measure

```bash
wc -c CLAUDE.md AGENTS.md GEMINI.md 2>/dev/null
grep -n "^#\{1,4\} " <file>          # section map with line numbers
ls docs/ 2>/dev/null                  # existing docs and their naming convention
```

Report the cost in the terms that matter: bytes ÷ 4 ≈ tokens, paid on **every** session and
**every** subagent dispatch. Note which sections are largest — those are the extraction candidates.

**Read the repo's own file-naming convention before creating any file.** The context file usually
states it (`docs/**` is kebab-case, or camelCase, or something else). Getting this wrong on the
first new file is an easy and embarrassing miss, and the convention is right there in the document
you are reading.

## Phase 2 — Classify

Two questions decide where every section goes.

**Does this stay?** Only if it passes one of these:

1. **A map** — repo layout, where things live, what each member is.
2. **A rule that bites** — violating it breaks the build, corrupts data, or drifts silently, AND
   an agent would plausibly get it wrong by default. "Never run `expo prebuild`", "these three
   files must change together", "`--filter` matches the package name, not the directory".
3. **An index** — topic → which doc has the depth.

**Everything else moves.** Subsystem internals, architectural archaeology, command listings,
"why we chose X over Y", detailed data models, migration histories. All of it valuable; none of
it needed by a session working on an unrelated subsystem.

The sharpest test for a rule: **state it in one or two lines with a link, or it is not a rule, it
is documentation.** A rule that needs three paragraphs to explain is a doc with a rule at the top —
extract the doc, keep the top line.

## Phase 3 — Relocate

Group the extracted sections by **how someone would look them up**, not by where they appeared in
the original. Typical shape:

- `docs/<area>-architecture.md` — one per major subsystem
- `docs/conventions.md` — naming, formatting, commit rules, with full rationale
- `docs/<feature>.md` — a subsystem substantial enough to own a doc

Aim for a handful of docs, not a dozen. Each gets a one-line intro saying what it covers and a
link back to the context file.

Move text **intact**. Rewriting while relocating risks losing meaning that took real work to
establish, and makes the verification in phase 5 unable to tell a lost fact from a reworded one.
Fix prose in a later pass if you want to.

Where a doc already exists for an area, extend it rather than creating a rival.

## Phase 4 — Rewrite the context file

Structure:

1. One paragraph: **this file is a map**, depth lives in `docs/`, do not re-expand it.
2. **Layout table** — path, what it is, which doc has the depth.
3. **Index table** — topic → doc.
4. **Rules that bite** — the list from phase 2, each one or two lines with a link.
5. Anything genuinely load-bearing and short (known-broken scaffolding, active caveats).

## Phase 5 — Verify (mandatory, not optional)

You have just moved most of a file. Prove nothing fell out.

**Byte accounting** — the sum of the new docs plus the new context file should slightly exceed the
original (headers and intros are additive). Materially less means content was dropped.

**Distinctive-string sampling** — pick 15–20 strings that appear nowhere else: function names,
constants, config keys, error strings, unusual phrases. Every one must still resolve.

```bash
for s in "SomeConstant" "someFunction" "UNUSUAL_KEY"; do
  grep -rql -- "$s" CLAUDE.md docs/*.md || echo "LOST: $s"
done
```

**Link integrity** — every path referenced from the context file must exist.

```bash
grep -o '](\([^)]*\))' CLAUDE.md | sed 's/](\(.*\))/\1/' \
  | while read -r p; do [ -e "$p" ] || echo "MISSING: $p"; done
```

**Duplication check** — extraction by line range easily copies a trailing section into two files.
Grep a string from each boundary section and confirm it appears once.

Report all four results. "I moved it and it looks fine" is not verification.

## Phase 6 — Guard against regrowth

The failure mode is not this edit; it is the next twenty sessions each appending one more
important thing. Two defenses:

- State the rule **in the file itself**, at the top: detail added here is loaded into every
  session and every subagent, so it belongs in a topic doc.
- If the repo generates plans or specs, check whether they **restate** the context file. They
  usually do, and it is the same waste one layer down: the executor already has the context file
  loaded, so a plan that copies it duplicates what the reader has and pays for it again on every
  execution run. A plan should cite the rules that bite for *that* change, not transcribe them.

## Finish

Commit on a branch, do not merge. The diff is large and mostly moves, so give the commit message
the before/after byte counts and the list of new docs — that is what makes it reviewable.

## Red flags

1. **"I'll summarize this section instead of moving it."** That is deletion with extra steps. The
   detail is correct and expensive; relocate it intact and let the doc be long.
2. **"This rule is important, I'll keep the full explanation inline."** Importance is not the test —
   *recurrence* is. An important rule that is explained in one line and linked stays; the same rule
   with three paragraphs of rationale becomes a link plus a doc.
3. **"The new docs are named `myNewDoc.md`."** Check the repo's naming convention first. It is
   stated in the file you are editing.
4. **"Nothing looks missing."** Run phase 5. Line-range extraction drops and duplicates content in
   ways that are invisible to reading.
5. **"I'll also fix the wording while I'm in here."** Two changes in one diff, one of which is
   unreviewable. Move first, edit later.
6. **"It's under 15KB but I'll tidy it anyway."** A short context file is already doing its job.
   Splitting it adds navigation cost for no saving.
