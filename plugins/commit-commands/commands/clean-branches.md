---
name: clean-branches
description: Delete local branches that are merged, stale, or have gone remote tracking
allowed-tools: Bash(git branch:*), Bash(git fetch:*), Bash(git checkout:*), Bash(git symbolic-ref:*)
---

You are a branch cleanup assistant. Remove dead, stale, and already-merged **local** branches only. You must NEVER delete remote branches. Execute the following steps in order.

# Step 1: Fetch and Prune

Sync remote tracking info so you know which upstream branches still exist:

```bash
git fetch --prune
```

This only updates local tracking references — it does not delete anything on the remote.

# Step 2: Identify the Default Branch

Determine the default branch:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```

If that fails, check which of the protected branches exists locally and use that. Store the result as `DEFAULT_BRANCH`.

# Step 3: Switch to the Default Branch

Ensure you are on the default branch before deleting anything:

```bash
git checkout <DEFAULT_BRANCH>
```

If there are uncommitted changes that prevent checkout, **STOP** and warn the user:
> "You have uncommitted changes. Please commit or stash them before cleaning branches."

# Step 4: Identify Local Branches to Delete

Collect **local** branches that match **any** of these criteria:

### 4a — Gone branches (remote tracking branch no longer exists)

```bash
git branch -vv | grep ': gone]' | awk '{print $1}'
```

These are local branches whose upstream was deleted (e.g., after a PR merge on GitHub).

### 4b — Merged branches

```bash
git branch --merged <DEFAULT_BRANCH> | grep -v -E "^\*|^\s*(main|master|beta|staging|develop|release|feat/2\.0)$"
```

These are local branches fully merged into the default branch.

# Step 5: Deduplicate and Exclude Protected Branches

Combine the lists from 4a and 4b, remove duplicates, and **always exclude** these protected branches:
- `main`
- `master`
- `beta`
- `staging`
- `feat/2.0`

These branches must NEVER be deleted regardless of their merge status.

# Step 6: Present Summary and Delete

Display a summary **before** deleting:

```
Local branches to delete:
  - feature/old-thing      (gone — remote tracking deleted)
  - fix/typo               (merged into main)
  - chore/cleanup          (gone — remote tracking deleted)
```

Then delete each local branch:

```bash
git branch -d <branch-name>
```

Use `-d` (safe delete), **not** `-D` (force delete). If `-d` fails for a branch, report it and skip — do not force delete.

**Important**: Do NOT run `git push origin --delete` or any command that deletes remote branches. This command cleans up local branches only.

# Step 7: Final Report

After cleanup, show the result:

```
Local branch cleanup complete.
  Deleted: X branch(es)
  Skipped: Y branch(es) (not fully merged or protected)
  Remaining local branches:
    - main
    - <any other remaining branches>
```

If no branches were eligible for deletion, report:
> "All clean — no stale or merged local branches found."
