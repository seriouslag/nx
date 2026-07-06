---
name: review-pr-deep
description: Deep code review of a single open PR in nrwl/nx. Sets up an isolated worktree, runs the pr-review-toolkit (6 specialized review agents) plus the reproduce-verifier agent (grounds the review in the linked issues and, when runnable locally, executes the repro on master vs PR), surfaces only critical and important findings (plus strengths; nice-to-have suggestions are dropped), and saves a GitHub-flavored draft to ~/.nx-pr-reviews/<NUMBER>.md for the reviewer to read (nothing is posted). Use when you want a thorough review of one PR.
allowed-tools: Bash(gh pr *), Bash(gh issue *), Bash(gh api *), Bash(gh auth *), Bash(git -C *), Bash(git worktree *), Bash(git rev-parse *), Bash(mkdir -p *), Bash(rm -rf *), Bash(ls *), Bash(printf *), Bash(date *), Bash(cd *), Bash(test *), Bash(echo *), Bash(head *), Bash(tail *), Bash(cat *), Bash(jq *), Bash(grep *), Bash(wc *), Bash(sed *), Write(~/.nx-pr-reviews/**), Write(tmp/claude/pr-review-worktrees/**), Write(/tmp/**), Edit(~/.nx-pr-reviews/**), Edit(tmp/claude/pr-review-worktrees/**), Edit(/tmp/**), Read, Grep, Glob, Skill, Agent
argument-hint: '<PR_NUMBER> [--verify-repros]'
---

# Deep PR Review (review-pr-deep)

Wraps `/pr-review-toolkit:review-pr` for a remote PR in `nrwl/nx`. The toolkit reviews local changes, so this skill prepares an isolated worktree of the PR, invokes the toolkit, then collects the output into a draft suitable for posting on GitHub.

**Drafts only.** This skill never posts to GitHub. The draft is reading material for the reviewer; if they want any of it on the PR, they post it themselves (or ask in the session, e.g. via `gh pr review --body-file`).

## Inputs

- `<NUMBER>` — the PR number in `nrwl/nx`. Required.

## Configuration (env-overridable)

- `NX_REPO_PATH` — path to a local clone of nrwl/nx. Default: the repo you're in — `git rev-parse --show-toplevel` (this skill ships inside nrwl/nx)
- `WORKTREE_BASE` — where to put the temporary worktree. Default: `$NX_REPO_PATH/tmp/claude/pr-review-worktrees` (gitignored)
- `GH_USER` — your GitHub login. Default: derived at runtime via `gh api user --jq .login`
- `TRIAGE_DIR` — where drafts live. Default: `~/.nx-pr-reviews` (deliberately outside `~/.claude`, so drafts don't touch Claude Code's own config dir)

## Step 1: Pre-flight

```bash
gh auth status
git -C "$NX_REPO_PATH" rev-parse --git-dir   # nrwl/nx clone exists? (works for worktree-based clones too)
mkdir -p "$WORKTREE_BASE" "$TRIAGE_DIR"
```

If `gh` isn't authed or the nx clone is missing, fail fast with a clear message.

## Step 2: Fetch the PR metadata

```bash
gh pr view <NUMBER> \
  --repo nrwl/nx \
  --json number,title,author,headRefOid,headRefName,baseRefName,url,isDraft,additions,deletions,changedFiles,reviews \
  > /tmp/pr-<NUMBER>.json
```

Parse out:

- `title`, `author.login`, `headRefOid` (the head SHA), `headRefName`, `baseRefName`, `url`
- `isDraft` — if true, exit early (don't review drafts)
- `reviews[]` filtered to `author.login == $GH_USER` and most recent — if `commit == headRefOid`, the user already reviewed this PR at this SHA. Exit with no draft change; log "ALREADY_REVIEWED".

## Step 3: Set up an isolated worktree

```bash
git -C "$NX_REPO_PATH" worktree prune   # recover from a prior `git clean` that removed the worktree dir
git -C "$NX_REPO_PATH" fetch origin pull/<NUMBER>/head:pr-<NUMBER>
git -C "$NX_REPO_PATH" worktree add "$WORKTREE_BASE/pr-<NUMBER>" "pr-<NUMBER>"
```

Worktrees keep the main checkout untouched. The branch name `pr-<NUMBER>` makes the worktree easy to identify and clean up later.

## Step 4: Gather incremental-review context (only if a prior review exists)

If `$TRIAGE_DIR/<NUMBER>.md` already exists, this is a **re-review** triggered by new commits. Build context for the toolkit so it can be conversational instead of starting fresh.

1. Read the existing triage file. Extract:
   - The frontmatter `head_sha` (call it `$PRIOR_SHA`).
   - The `## Review draft` section (the most recent review). This becomes "the prior review."
   - The full `## Prior reviews` section (older reviews, if any). All of them — no cap on history.

2. Fetch `$PRIOR_SHA` so we can diff against it:

   ```bash
   git -C "$NX_REPO_PATH" fetch origin "$PRIOR_SHA"
   ```

   (If `$PRIOR_SHA` no longer exists on the remote — author force-pushed and orphaned it — skip this step and treat as a fresh review.)

3. Compute the incremental diff inside the worktree:

   ```bash
   git -C "$WORKTREE_BASE/pr-<NUMBER>" diff "$PRIOR_SHA".."<HEAD_REF_OID>" > /tmp/pr-<NUMBER>-incremental.diff
   ```

4. Write a context file at `$WORKTREE_BASE/pr-<NUMBER>/.review-context.md`:

   ```markdown
   # Re-review context

   This PR has been reviewed before. The prior review's verdict was: <PRIOR_VERDICT>.

   ## Most recent prior review (head_sha=$PRIOR_SHA)

   <PASTE THE PRIOR REVIEW DRAFT VERBATIM>

   ## All earlier reviews (oldest first)

   <PASTE THE FULL ## Prior reviews SECTION VERBATIM>

   ## Diff since last review (`$PRIOR_SHA..<HEAD>`)

   See /tmp/pr-<NUMBER>-incremental.diff for the new code added since the prior review.
   ```

## Step 4.5: Close-without-merge check

Before running the toolkit, do a cheap pass to answer: **"Should this PR be closed without merging?"** Two flavors:

- **Superseded** — master or another PR already addressed the goal.
- **Unnecessary** — the change shouldn't be merged at all (no real bug, abandoned, out of scope, duplicate of rejected work).

Both save the toolkit's effort on PRs that won't merge anyway. Signals 1–4 detect supersession; signals 5–8 detect unnecessary. Run the gh-only signals here. Signal 5 depends on the reproduce-verifier and is finalized after Step 5a.5.

### Supersession signals (gh-only, run now)

**1. Mergeability.** If master moved in the same files, the PR is stale.

```bash
gh pr view <NUMBER> --repo nrwl/nx --json mergeable,mergeStateStatus
```

Flag if `mergeable == "CONFLICTING"` or `mergeStateStatus == "DIRTY"`.

**2. Cross-references on linked issues.** Has another _merged_ PR referenced the same issue?
Parse `closingIssuesReferences` from the PR body + `gh pr view` (look for `Fixes #N`, `Closes #N`, `Resolves #N`). For each linked issue:

```bash
gh issue view <ISSUE> --repo nrwl/nx --json timelineItems --jq '.timelineItems[] | select(.__typename == "CrossReferencedEvent") | select(.source.__typename == "PullRequest") | {pr: .source.number, state: .source.state, merged: .source.merged, mergedAt: .source.mergedAt, title: .source.title}'
```

Flag any other PR with `merged: true` — that PR may have fixed the same issue.

**3. Same-file merged PRs since this PR opened.** Identify possibly-competing work.
Get the PR's `createdAt` and `files[].path`, then:

```bash
gh pr list --repo nrwl/nx --state merged --search "<FILE_PATH> merged:><PR_CREATED_AT>" --json number,title,mergedAt --limit 5
```

Run for the 2-3 most-touched files (not all — this is a cheap sanity check, not exhaustive). Flag any hits.

**4. Target-state check.** For small PRs (< 50 lines changed OR touches only `package.json` / `versions.ts` / `migrations.json`), peek at master to see if the target state is already there.

Read each changed file on master (`git -C $NX_REPO_PATH show origin/master:<path>`) and compare key lines against what the PR is trying to set. Example: if the PR changes `"@foo/bar": "^1.0.0"` → `"^2.0.0"` but master already has `"^2.3.3"`, flag it.

For larger PRs, skip this — the toolkit will catch subtler issues.

### Unnecessary signals

**5. Bug not reproducible + thin rationale.** Finalized after Step 5a.5. If the reproduce-verifier returns `BUG_NOT_REPRODUCED_ON_BASELINE` AND the PR body lacks a clear non-bug rationale (no design-doc link, no maintainer endorsement in the linked issue, no thumbs-up reaction count > 0 on the linked issue), flag as fixing a non-issue.

```bash
# Check the linked issue for maintainer endorsement / reaction count
gh issue view <ISSUE> --repo nrwl/nx --json reactions,comments --jq '{thumbsUp: .reactions.THUMBS_UP, commentAuthors: [.comments[].author.login]}'
```

If the linked issue has no thumbs-up and no nrwl-org member has commented affirmatively, the PR is "fixing" something that may not be a bug.

**6. Stale + abandoned + conflicted.** All three together:

- Last commit on the PR branch > 90 days ago: parse `commits[-1].committedDate` from `gh pr view ... --json commits`.
- Has merge conflicts (signal 1 fired).
- Has unanswered reviewer questions: most recent non-author comment is unanswered. Check via `gh pr view <NUMBER> --json comments --jq '.comments | map({author: .author.login, at: .createdAt}) | last'` — if the last commenter is not the author and the timestamp is > 30 days old, it's unanswered.

If all three fire, the PR is abandoned and unlikely to land.

**7. Duplicate of recently-closed-without-merge PR.** Search closed-but-not-merged PRs touching the same primary file in the last 6 months:

```bash
gh pr list --repo nrwl/nx --state closed --search "<MAIN_FILE_PATH> closed:>$(date -d '6 months ago' +%Y-%m-%d 2>/dev/null || date -v-6m +%Y-%m-%d)" --json number,title,closedAt,state,mergedAt --limit 10
```

Filter to entries where `mergedAt` is null (closed without merging). If any have a similar title or touched the same primary files, this PR may be re-attempting work the maintainers already declined.

**8. No linked issue + speculative scope.** All of:

- No `Fixes #N` / `Closes #N` / `Resolves #N` reference in body or commits.
- PR body < 200 chars of explanation.
- PR modifies > 100 lines OR touches public-API surface (`packages/*/src/index.ts`, files matching `*.public.ts`, anything under `packages/*/index.ts`).

Speculative refactors without a stated reason are usually closed. Advisory-strength signal — flag in the section, but don't on its own force a verdict.

### Emit

If any signal fires, prepend a `### Close-without-merge check` section to `$REVIEW_BODY` (above `### Reproduction verification`):

```markdown
### Close-without-merge check

<pick the strongest line — only one verdict-line, but multiple advisory lines OK:>

- 🛑 **Likely superseded.** <reason, with linked PR numbers / file evidence>
- 🛑 **Likely unnecessary.** <reason — name the signal(s) that fired: bug not reproduced, abandoned, duplicate of #N, etc.>
- ⚠️ **Stale branch.** Merge conflicts with master on <N> files; author should rebase before review lands.
- ⚠️ **Speculative scope.** No linked issue, thin rationale, > 100 lines changed.
- ✅ No close signals — PR is current and well-scoped.
```

**Verdict influence (Step 7):**

- **Superseded (strong)** → verdict `superseded`. "Strong" means ANY of: signal 2 fires (another merged PR closes the same issue), OR signals 3+4 both fire (same-file merged PR AND master already at/past the PR's target state). The section should include the specific superseding PR number(s) so whoever closes the PR has a concrete pointer to cite.
- **Unnecessary (strong)** → verdict `unnecessary`. "Strong" means ANY of: signal 5 fires (bug not reproduced + thin rationale), OR signal 6 fires (stale + abandoned + conflicted), OR signal 7 fires (duplicate of recently-closed-without-merge PR). The section should explain WHICH signals fired so whoever closes the PR can explain the reason.
- **Both fire** → supersession wins (more specific framing, gives the author a concrete pointer).
- **Stale branch alone** (only signal 1) → advisory; still run the toolkit, still pick a verdict normally.
- **Speculative scope alone** (only signal 8) → advisory; note it in the review body, don't force a verdict.
- **Clean** → no section emitted.

If all signals are cheap-negative, skip emitting the section entirely (no noise on healthy PRs).

## Step 5: Run the review toolkit

`cd` into the worktree and invoke the toolkit:

```
Skill(skill="pr-review-toolkit:review-pr", args="all")
```

The toolkit dispatches 6 specialized agents (code-reviewer, comment-analyzer, pr-test-analyzer, silent-failure-hunter, type-design-analyzer, code-simplifier) and aggregates results into Critical / Important / Suggestions / Strengths.

If `.review-context.md` exists in the worktree (from Step 4), instruct the toolkit to read it first so its agents are aware of the prior review and what's new.

Capture the toolkit's full output as `$RAW_REVIEW_BODY`.

### Trim to critical + important

**Only critical and important findings are kept.** After capturing `$RAW_REVIEW_BODY`, drop the entire **Suggestions** / nice-to-have section — discard those findings, do not downgrade or relocate them. Keep **Critical**, **Important**, and **Strengths**. The trimmed text is what flows into the steps below (reconciliation in Step 5b, formatting in Step 6).

## Step 5a.5: Run the reproduce-verifier agent

In parallel with (or right after) Step 5, dispatch the `reproduce-verifier` agent to ground the review in the reported bug:

Decide whether to opt in to Level 2 (expensive verdaccio-based external-repo reproduction, ~10-15 min per PR). Default is **off** — Level 2 only runs when:

- The caller of this skill explicitly requested deep verification (e.g. invoked with the `--verify-external-repros` flag, or a manual `/review-pr-deep <N> --verify-repros` pattern), OR
- `$NX_REVIEW_LEVEL_2=1` is set in the environment.

Level 2 is for deep-dive passes where you want end-user-level proof — each run takes ~10-15 minutes and ~0.5-1 GB of disk, so opt in deliberately.

```
Agent(
  subagent_type="reproduce-verifier",
  description="Verify PR <NUMBER> fixes linked issues",
  prompt="""
Verify that PR <NUMBER> in nrwl/nx actually fixes the issues it claims to close.

Inputs:
- PR_NUMBER: <NUMBER>
- WORKTREE_PATH: <WORKTREE_BASE>/pr-<NUMBER>
- HEAD_SHA: <HEAD_REF_OID>
- BASE_REF: <BASE_REF_NAME>
- RUN_LEVEL_2: <true|false — see gate above>

Follow your standard workflow (Level 0 always, Level 1 when applicable, Level 2 only when RUN_LEVEL_2=true AND classification is EXTERNAL_REPO or GENERATED_WORKSPACE). Return the structured report.
"""
)
```

Capture the agent's output as `$REPRO_REPORT`. Fold it into the final review body under a dedicated `### Reproduction verification` section, positioned above `### Critical` so readers see the grounding before the code findings. The agent's Level 1 / Level 2 verdicts feed into the overall verdict (Step 7):

**Level 1 verdicts:**

- `FIX_CONFIRMED` — evidence towards `lgtm`
- `FIX_DID_NOT_WORK` / `FIX_CHANGED_BEHAVIOR_BUT_NOT_RESOLVED` — strong push towards `needs-changes` regardless of toolkit findings
- `BUG_NOT_REPRODUCED_ON_BASELINE` — push towards `blocked` pending human check (could mean stale issue, wrong command, or the PR is unnecessary)
- `NOT_ATTEMPTED` — no effect on verdict; note it in the summary

**Level 2 verdicts (only present when opted in):**

- `PR_REPRO_PASSES` — strong evidence towards `lgtm` (PR verified against actual repro)
- `PR_REPRO_FAILS` / `PR_REPRO_FAILS_DIFFERENT` — strong push towards `needs-changes`
- `PR_REPRO_INCONCLUSIVE` / `SETUP_FAILED` — flag in summary; do not use for verdict

## Step 5b: Reconcile against prior reviews (only on re-review)

If a prior review exists, do a second pass _yourself_ (don't dispatch another agent — you already have all the context). Work only from the trimmed findings (critical / important — Suggestions were already dropped in Step 5). For each finding:

- Was the same concern raised in a prior review and now appears resolved? → move it under **Addressed since last review**.
- Was the same concern raised in a prior review and still present? → move it under **Still concerning** with a note like "raised in <date>".
- Is it a new finding (not in any prior review)? → keep under **New concerns**.

Reorganize the toolkit output into this structure:

```markdown
## Addressed since last review

- <findings the author has fixed since the prior review>

## Still concerning

- <findings raised before that haven't been addressed>

## New concerns

- <findings about code added since the prior review>

## Strengths

- <positive observations>
```

If this is the first review (no triage file existed), skip this step entirely — just use the toolkit output verbatim.

The reconciled (or fresh) text becomes `$REVIEW_BODY`.

## Step 6: Format for GitHub

`$REVIEW_BODY` is posted as-is — no header, footer, or tool attribution. It should read like a review a maintainer wrote. The review metadata (commit, date, attempt) lives in the triage file's frontmatter, not in the posted body.

## Step 7: Determine verdict

Check in this order (first match wins):

- Close-without-merge check emitted "Likely superseded" with strong evidence (see Step 4.5) → `verdict: superseded`
- Close-without-merge check emitted "Likely unnecessary" with strong evidence (see Step 4.5) → `verdict: unnecessary`
- Has any **Still concerning** or **New concerns** items rated critical → `verdict: needs-changes`
- Has 3+ items across Still concerning + New concerns → `verdict: needs-changes`
- Couldn't reach a clear conclusion → `verdict: blocked`
- Otherwise → `verdict: lgtm`

(For first reviews with no prior context, fall back to the toolkit's Critical/Important categories.)

**Verdict values:** `lgtm | needs-changes | blocked | superseded | unnecessary | failed`.

- `superseded` — the PR shouldn't merge because other work already landed; the draft carries a pointer to the superseding PR for whoever closes it.
- `unnecessary` — the PR shouldn't merge at all (no confirmed bug, abandoned, or duplicate of rejected work); the draft carries the reason from the close-without-merge check.

## Step 8: Write the triage file (preserving full history)

Write `$TRIAGE_DIR/<NUMBER>.md`. **If the file already exists** (re-review):

1. Read the existing file.
2. Move the existing `## Review draft` content into a new entry at the top of `## Prior reviews`, prefixed with a header like `### attempt <N-1> — head_sha=<PRIOR_SHA> — <PRIOR_DATE>`.
3. Preserve the `## Posted` and `## Failures` sections verbatim.
4. Replace `## Review draft` with the new `$REVIEW_BODY` (formatted in Step 6).
5. Update frontmatter: `head_sha`, `last_reviewed_at`, `verdict`, increment `attempt`. Preserve `posted_at` / `posted_url` (the user fills those in).

**No cap on history** — every prior review accumulates under `## Prior reviews`, oldest at the bottom, newest at the top.

Format:

```markdown
---
pr: <NUMBER>
title: <TITLE>
author: <AUTHOR>
url: <URL>
head_sha: <HEAD_REF_OID>
last_reviewed_at: <ISO_8601>
verdict: <lgtm|needs-changes|blocked|superseded|failed>
attempt: <N>
posted_at:
posted_url:
---

# PR #<NUMBER>: <TITLE>

<AUTHOR> · <ADDITIONS>+/<DELETIONS>- across <CHANGED_FILES> files
HEAD: `<HEAD_SHA_SHORT>` · base: `<BASE_REF>`

## Review draft

<FORMATTED_BODY_FROM_STEP_6>

## Prior reviews

### attempt <N-1> — head_sha=<PRIOR_SHA> — <PRIOR_DATE>

<the previous Review draft, verbatim>

### attempt <N-2> — head_sha=<EVEN_PRIOR_SHA> — <DATE>

<and so on — oldest at the bottom>

## Posted

(none yet, or whatever was already there)

## Failures

(none, or whatever was already there)
```

## Step 9: Cleanup

Always remove the worktree, even on failure:

```bash
git -C "$NX_REPO_PATH" worktree remove --force "$WORKTREE_BASE/pr-<NUMBER>"
git -C "$NX_REPO_PATH" branch -D "pr-<NUMBER>" 2>/dev/null
```

## Step 10: Commit the draft (if the triage dir is in a git repo)

Some maintainers keep their triage dir inside a synced git repo (e.g. dotfiles). If `git -C "$TRIAGE_DIR" rev-parse --is-inside-work-tree` succeeds:

```bash
git -C "$TRIAGE_DIR" add <NUMBER>.md
git -C "$TRIAGE_DIR" commit -m "review: drafted review for PR #<NUMBER> (attempt <N>)"
```

This makes the draft history visible (`git -C "$TRIAGE_DIR" log --oneline`) and gives a per-attempt audit trail.

If the triage dir is not inside a git work tree, skip this step silently — the file on disk is the record.

## On failure

If anything in Steps 3-7 errors:

1. Still write/update the triage file with `verdict: failed` and a `## Failures` entry containing the error.
2. Still preserve any prior `## Review draft` content into `## Prior reviews` so history isn't lost.
3. Still clean up the worktree (Step 9).
4. Commit with a `failed` message instead (same guard as Step 10 — only if the triage dir is inside a git work tree).
5. Return non-zero so the caller can tell the review failed.

## Returning the draft

Print to stdout the path to the saved triage file:

```
$TRIAGE_DIR/<NUMBER>.md verdict=<VERDICT>
```

The caller can grep this to know what happened without re-reading the file.
