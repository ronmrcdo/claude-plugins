---
name: daily-standup
description: Generate a daily standup message from your GitHub PR activity
argument-hint: "[--all]"
allowed-tools: Bash(gh:*), Bash(date:*), Bash(git:*)
---

You are a standup assistant that generates daily standup messages from GitHub PR activity. Execute the following steps in order.

# Step 1: Determine Dates

Calculate today's and yesterday's date, with weekend awareness:

```bash
TODAY=$(date +%Y-%m-%d)
DOW=$(date +%u)
```

Use the day-of-week number (`%u`: 1=Monday, 7=Sunday) to resolve "yesterday":
- **Monday (1):** yesterday = last Friday (`date -d '3 days ago' +%Y-%m-%d`)
- **Sunday (7):** yesterday = Friday (`date -d '2 days ago' +%Y-%m-%d`)
- **Saturday (6):** yesterday = Friday (`date -d '1 day ago' +%Y-%m-%d`)
- **Otherwise:** yesterday = calendar yesterday (`date -d '1 day ago' +%Y-%m-%d`)

If today is Monday, note "(Looking back to Friday)" in the output later.

# Step 2: Resolve Repository Scope

Determine whether to scope queries to the current repository or query across all repos.

1. If `$ARGUMENTS` contains `--all`, skip repo scoping — do **not** pass `--repo` to any `gh` command.
2. Otherwise, run:
   ```bash
   git remote get-url origin
   ```
   Extract the `owner/repo` slug from the URL (strip `.git` suffix, handle both HTTPS and SSH formats). Use `--repo owner/repo` on all `gh` commands in subsequent steps.
3. If `git remote get-url origin` fails (not a git repo), fall back to querying all repos without `--repo`.

# Step 3: Fetch Yesterday's PR Activity

Run these three queries to capture all authored PR activity from yesterday. Append `--repo owner/repo` if scoped (from Step 2).

**3a. PRs merged yesterday:**
```bash
gh pr list --author @me --state merged --search "merged:YESTERDAY_DATE" --json number,title,url,mergedAt
```

**3b. PRs created yesterday:**
```bash
gh pr list --author @me --state all --search "created:YESTERDAY_DATE" --json number,title,url,state,isDraft
```

**3c. PRs updated yesterday (but not created yesterday — indicates review cycles or pushes):**
```bash
gh pr list --author @me --state all --search "updated:YESTERDAY_DATE -created:YESTERDAY_DATE" --json number,title,url,state
```

If any `gh` command fails with an authentication error, **stop immediately** and tell the user:
> "GitHub CLI is not authenticated. Run `gh auth login` first."

# Step 4: Fetch Today's PR Activity

Run these two queries. Append `--repo owner/repo` if scoped.

**4a. Currently open PRs (in-progress work):**
```bash
gh pr list --author @me --state open --json number,title,url,isDraft
```

**4b. PRs created today:**
```bash
gh pr list --author @me --state all --search "created:TODAY_DATE" --json number,title,url,state,isDraft
```

# Step 5: Fetch Local Commits (No PR Yet)

Skip this step if `$ARGUMENTS` contains `--all` or if not in a git repository.

Fetch the git user's name and email for author matching:
```bash
git config user.name
git config user.email
```

**5a. Commits from yesterday not on a PR branch:**
```bash
git log --all --author="USER_NAME" --since="YESTERDAY_DATE" --until="TODAY_DATE" --format="%h %s [%D]"
```

**5b. Commits from today not on a PR branch:**
```bash
git log --all --author="USER_NAME" --since="TODAY_DATE" --format="%h %s [%D]"
```

For each commit, check its branch. Exclude commits that belong to branches already associated with a PR found in Steps 3-4. The remaining commits represent local work not yet submitted as a PR — group them by branch.

# Step 6: Deduplicate and Classify

1. **Deduplicate** PRs by PR number across all queries.
2. **Yesterday bucket** — assign each PR the first matching label:
   - **Merged**: appeared in query 3a (merged yesterday)
   - **Opened**: appeared in query 3b (created yesterday, not merged)
   - **Updated**: appeared in query 3c only (pushed to or received reviews)
3. **Today bucket** — assign each PR the first matching label:
   - **Opened today**: appeared in query 4b (created today)
   - **Working on draft**: open and `isDraft` is true
   - **Continuing**: open and not a draft (carried over from previous days)
4. **Local commits (no PR)** — group by branch. These are commits from Step 5 on branches with no associated PR. Summarize by branch name and number of commits.

# Step 7: Generate Standup Message

Output a formatted standup message like this:

```
## Daily Standup — [TODAY_DATE]

### Yesterday ([YESTERDAY_DATE])
- Merged PR #42: "Add user auth flow"
- Opened PR #45: "Fix pagination bug"
- Updated PR #40: "Refactor API middleware"
- Local: 3 commits on `feat/search` (no PR yet)

### Today
- Continuing PR #45: "Fix pagination bug"
- Working on draft PR #48: "Add search feature"
- Opened today PR #50: "Update CI config"
- Local: 1 commit on `fix/typo` (no PR yet)

### Blockers
- None
```

**Formatting rules:**
- Use verb prefixes (Merged, Opened, Updated, Continuing, Working on draft, Opened today) so each line reads naturally
- Include PR number and title for quick identification
- Include the GitHub URL for each PR so links are clickable in Slack/Teams
- For local commits without a PR, show as "Local: N commits on `branch-name` (no PR yet)" with the most recent commit message
- Always include the "Blockers" section with "None" as default — the user can edit before pasting
- Show resolved dates in the section headers so the user can verify correctness

# Step 8: Handle Edge Cases

- **No PRs found for yesterday:** Output "No PR activity found for [YESTERDAY_DATE]." under the Yesterday section.
- **No open PRs for today:** Output "No open PRs. Starting fresh today." under the Today section.
- **Monday standup:** Append "(Looking back to Friday)" next to the yesterday date header.
- **Not in a git repository (and no --all flag):** Fall back to querying all repos without `--repo`, skip local commit detection, and note "Showing PRs across all repositories" at the top of the output.
- **No local commits outside PRs:** Omit the "Local:" lines — do not show an empty local section.
