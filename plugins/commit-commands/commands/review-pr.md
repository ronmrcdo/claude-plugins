---
name: review-pr
description: Review a GitHub PR in an isolated git worktree via superpowers:requesting-code-review, then tear the worktree down
argument-hint: <PR_NUMBER>
allowed-tools: Bash(git:*), Bash(gh:*), Read, Skill, Task
---

# Review PR #$ARGUMENTS

Review GitHub PR **#$ARGUMENTS** by checking it out into a throwaway git worktree, delegating the actual review to the `superpowers:requesting-code-review` skill, then removing the worktree.

**Worktree lifecycle guarantee:** the worktree created in Step 3 MUST be removed in Step 6 — on success, on error, and if the user cancels the review at any point. If you are interrupted mid-run, execute Step 6 before doing anything else. Step 1 also reclaims any leftover worktree from a previously cancelled run, so a missed teardown self-heals on the next invocation.

## Step 0: Validate input

If `$ARGUMENTS` is empty or not a positive integer, stop and print the usage: `/review-pr <PR_NUMBER>` (e.g. `/review-pr 1542`). Do not proceed.

## Step 1: Reclaim stale state

A prior run may have been cancelled before cleanup. Remove any leftover worktree and branch for this PR first (the `|| true` swallows the "nothing there" case):

```bash
git worktree remove .worktrees/pr-review-$ARGUMENTS --force 2>/dev/null || true
git worktree prune
git branch -D pr-review-$ARGUMENTS 2>/dev/null || true
```

## Step 2: Fetch PR metadata

```bash
gh pr view $ARGUMENTS --json number,title,body,author,baseRefName,headRefName,additions,deletions,changedFiles
```

Capture the **title**, **author login**, **baseRefName** (base branch), and **body** for use in Step 5. If this fails (PR not found or `gh` not authenticated), stop and surface the error verbatim — no worktree exists yet, so there is nothing to clean up.

## Step 3: Create the worktree

Ensure the worktree directory is ignored so it never pollutes `git status`, then fetch the PR head plus the base branch and check the PR out into an isolated worktree:

```bash
git check-ignore -q .worktrees || printf '\n.worktrees/\n' >> .gitignore
git fetch origin pull/$ARGUMENTS/head:pr-review-$ARGUMENTS
git fetch origin <baseRefName>
git worktree add .worktrees/pr-review-$ARGUMENTS pr-review-$ARGUMENTS
```

Replace `<baseRefName>` with the base branch from Step 2. If any command here fails, jump straight to Step 6 to remove whatever was partially created, then report the error.

## Step 4: Compute the review range

```bash
BASE_SHA=$(git merge-base origin/<baseRefName> pr-review-$ARGUMENTS 2>/dev/null || git rev-parse origin/<baseRefName>)
HEAD_SHA=$(git rev-parse pr-review-$ARGUMENTS)
```

`BASE_SHA..HEAD_SHA` is exactly the PR's diff. Note both values for the next step.

## Step 5: Run the review

Invoke the **`superpowers:requesting-code-review`** skill (Skill tool, `skill: "superpowers:requesting-code-review"`). When it dispatches the reviewer subagent from its `code-reviewer.md` template, fill the placeholders with:

- **DESCRIPTION** — `PR #$ARGUMENTS — <title>` by `<author>`, plus a one-line summary of the PR body.
- **PLAN_OR_REQUIREMENTS** — the PR body from Step 2 (what the PR claims to do). If the body is empty, say so and have the reviewer judge against the changes' apparent intent.
- **BASE_SHA** — the value computed in Step 4.
- **HEAD_SHA** — the value computed in Step 4.
- Add one extra instruction to the reviewer: the checked-out PR sources live in `.worktrees/pr-review-$ARGUMENTS/` — read files there for full context. The diff range (`git diff $BASE_SHA..$HEAD_SHA`) works from anywhere since worktrees share the object store.

Relay the reviewer's **Strengths / Issues / Assessment** back to the user in full.

## Step 6: Cleanup (mandatory — always run)

Run this whether the review finished, errored, or was cancelled:

```bash
git worktree remove .worktrees/pr-review-$ARGUMENTS --force 2>/dev/null || true
git worktree prune
git branch -D pr-review-$ARGUMENTS 2>/dev/null || true
```

Confirm to the user that the temporary worktree and branch were removed.
