---
name: pr-feedback-loop
description: Use when a pull request is open or a branch is ready to become one and reviewer feedback (automated or human) needs to be addressed across one or more rounds
---

# PR Feedback Loop

## Overview

This skill implements a disciplined multi-round feedback loop for addressing PR review comments. The core principle is **triage, not compliance**: every finding must be categorized as one of:

- **Fix** — the reviewer is right; implement the fix.
- **Push back** — the finding is incorrect or the current code is defensible-as-is. Reply with specific technical reasoning.
- **Defer** — valid concern but out of scope for this PR. Reply with a TODO or follow-up reference.

Pushing back is legitimate and expected. A reviewer flagging dead code that has an intentional rationale, or requesting an export that would break encapsulation, deserves a reasoned disagreement — not blind compliance.

**Red flags (from baseline failures — if you catch yourself doing these, STOP):**
- "The reviewer is probably right about everything" — NO. Triage each finding independently.
- "Replying to threads is optional" — NO. Every handled finding gets a reply.
- "I'll just fix everything and push" — NO. Defensible code stays; explain why.
- "One more round beyond the cap won't hurt" — NO. Terminate at N.
- "I don't need to wait for the bot" — NO. Always wait for the review before acting.

## Setup

### If no PR exists yet

1. Push the branch: `git push -u origin <branch>`
2. Open the PR using the body format from `~/.claude/commands/generate-pr-description.md`:
   ```bash
   gh pr create --title "<type>: <summary>" --body "$(cat <<'EOF'
   # tl;dr
   ...
   EOF
   )"
   ```

### Determine current round

```bash
git log --oneline --grep="address PR #<n> review round" | wc -l
```

The count gives the last completed round. Next round is `k = count + 1`. Default cap `N = 3` unless overridden.

## The Round Loop

For each round `k` (1 to N):

### 1. Handle CI failures (round-0 findings)

Before anything else, check CI:
```bash
gh pr checks <n> --repo <owner>/<repo>
```
If CI is failing for reasons related to this PR's changes, treat each failure as a finding to fix. If CI fails for pre-existing reasons unrelated to this PR, note and proceed.

### 2. Wait for the bot review

The Claude GitHub Action posts a **sticky comment** (updates in place). While in progress it shows checkboxes; when done it starts with `**Claude finished`. The comment may temporarily show progress markers — only act on it once the `**Claude finished` prefix appears.

**For round k > 1:** A push triggers a new review that overwrites the sticky comment. The previous `**Claude finished` text disappears during the new run. Poll until the prefix reappears — and verify the job URL is different from the previous round's (the link includes the run ID).

**Capturing PREV_RUN_ID:** BEFORE pushing round k's commit, extract the run ID from the `[View job]` link in the current bot comment (URL shape: `https://github.com/<owner>/<repo>/actions/runs/<run-id>`) and store it:

```bash
PREV_RUN_ID=$(gh pr view <n> --repo <owner>/<repo> --json comments \
  --jq '.comments[] | select(.body | startswith("**Claude finished")) | .body' \
  | grep -oE 'actions/runs/[0-9]+' | head -1 | grep -oE '[0-9]+')
```

Then, when polling after the push, a `**Claude finished` comment whose run URL **equals** `PREV_RUN_ID` is **stale** — keep polling until one with a different run ID appears.

```bash
# Poll every 60s, up to 15 minutes
for i in $(seq 1 15); do
  COMMENT=$(gh pr view <n> --repo <owner>/<repo> --json comments \
    --jq '.comments[] | select(.body | startswith("**Claude finished")) | .body')
  if [ -n "$COMMENT" ]; then
    NEW_RUN_ID=$(echo "$COMMENT" | grep -oE 'actions/runs/[0-9]+' | head -1 | grep -oE '[0-9]+')
    if [ -z "$PREV_RUN_ID" ] || { [ -n "$NEW_RUN_ID" ] && [ "$NEW_RUN_ID" != "$PREV_RUN_ID" ]; }; then
      echo "Bot review found (run $NEW_RUN_ID)"
      break
    fi
    # Same run ID as before push => stale comment; keep polling.
  fi
  sleep 60
done
```

**If the run URL can't be parsed** (comment format changed, no `[View job]` link): fall back to the comment's `updated_at` via the REST API — `createdAt` is useless here because the sticky comment is edited in place and `createdAt` never changes (and `gh pr view --json comments` exposes no `updatedAt`). Get the comment ID (`gh api repos/<owner>/<repo>/issues/<n>/comments --jq '.[] | select(.body | startswith("**Claude finished")) | .id'`), then:

```bash
gh api repos/<owner>/<repo>/issues/comments/<comment-id> --jq .updated_at
```

Only treat the comment as fresh if `updated_at` is later than your push time. If freshness still can't be established within the 15-minute window, surface to the user rather than acting on a possibly-stale review.

**If the bot comment never arrives after 15 minutes:** STOP. Report to the user: "Bot review did not arrive within 15 minutes. PR #<n> may need manual inspection." Do NOT proceed without the review.

### 3. Collect findings

Gather ALL feedback sources:

**A. Bot sticky comment** — parse the top-level PR comment starting with `**Claude finished` for findings. Look for severity markers, code blocks, and fix suggestions.

```bash
gh pr view <n> --repo <owner>/<repo> --json comments \
  --jq '.comments[] | select(.body | startswith("**Claude finished")) | .body'
```

**B. Inline review threads** (if any):

```bash
gh api graphql -f query='query($owner:String!,$repo:String!,$pr:Int!){repository(owner:$owner,name:$repo){pullRequest(number:$pr){reviewThreads(first:50){nodes{isResolved comments(first:10){nodes{author{login} body path}}}}}}}' \
  -f owner=<owner> -f repo=<repo> -F pr=<n>
```

**C. Human comments** — any PR comment NOT from the bot (not starting with `**Claude finished` and not from `github-actions[bot]`).

### 4. Triage each finding

For EACH finding, decide:

| Decision | Criteria | Action |
|---|---|---|
| **Fix** | Reviewer is correct; code has a real bug, security issue, or convention violation with no legitimate counter-argument | Implement the fix |
| **Push back** | Current code is defensible. There is a specific technical reason (documented in comments, architecture decision, scope constraint) why the code is correct as-is | Reply with reasoning; do NOT change the code |
| **Defer** | Valid concern but out of scope (large refactor, unrelated to PR's purpose, would require design discussion) | Reply acknowledging validity + TODO/issue reference |

**Bot findings:** Triage autonomously using the criteria above. The bot is often right about security and conventions, but may over-flag dead code or suggest exports that break encapsulation.

**When the reviewer agrees with you:** If the bot itself says a finding is "defensible" or "no change needed," acknowledge that in your reply (it still counts as a handled finding). This is distinct from push-back — push-back is when YOU disagree with the reviewer's suggestion.

**Human comments:** If the comment is clear and actionable, triage normally. If it is **ambiguous or contentious** (unclear intent, disagreement with project conventions, request that conflicts with other constraints), STOP and surface it to the user for a decision. Do not guess.

### 5. Apply fixes

For findings triaged as "Fix":
- Make the code changes
- Add or update tests if the fix warrants it
- Keep changes minimal and focused

### 6. Pre-push quality gates

**Always run before committing:**
```bash
make fmt
make lint
make test
```

If a gate fails for reasons **unrelated to your changes** (pre-existing failures in master), note this in your status report and proceed. If a gate fails **because of your changes**, fix the issue before committing.

### 7. Commit

Format: `fix: address PR #<n> review round <k> — <summary>`

Rules:
- Conventional commit, header max 100 chars
- NO Co-Authored-By or any other trailers
- `<summary>` is a brief description of what was fixed (not a list of every finding)
- One commit per round (squash multiple fixes into the round commit)

### 8. Reply to every handled finding

For EACH finding you triaged (whether fix, push-back, or defer), post a reply:

- For **inline review threads**: reply to the thread
- For **bot sticky comment findings**: post a new PR comment referencing the finding

Reply format:
- **Fixed:** Brief description of what was changed
- **Pushed back:** Technical reasoning why the code is correct as-is
- **Deferred:** Acknowledgment + what follow-up looks like

### 9. Push

```bash
git push origin <branch>
```

This triggers a new bot review for the next round.

### 10. Check for early exit

If this round had **zero actionable findings** (all findings were pushed back or deferred with valid reasoning, and CI is green), the loop terminates early. Do NOT push an empty commit.

## Termination and Report

The loop ends when:
- A round produces zero actionable findings (clean round), OR
- Round cap N is reached

**Never merge the PR.** The skill's job is to address feedback, not to merge.

### Status report (always produce this at the end)

```
## PR Feedback Loop — Status Report

- **PR:** <url>
- **Rounds completed:** <k> / <N>
- **Termination reason:** <clean round | cap reached>
- **CI status:** <passing | failing (reason)>

### Per-finding dispositions

| # | Finding | Severity | Decision | Detail |
|---|---|---|---|---|
| 1 | <brief> | <severity as reported by reviewer> | Fixed / Pushed back / Deferred | <one-line explanation> |
| ... | ... | ... | ... | ... |

Actionable findings remaining at termination: <n>

### Open items (if cap reached with findings remaining)
- <list any unresolved findings that need human attention>
```

If the cap was reached with findings still open, explicitly note that the user should review the remaining items.
