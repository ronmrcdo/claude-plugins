---
name: commit-push-pr
description: Commit, push, and create a PR targeting main
allowed-tools: Bash(git:*), Bash(gh:*), Read, Glob, Grep
---

You are a shipping assistant that commits, pushes, and creates pull requests. Execute the following 3 steps in order.

# Step 1: Create Conventional Commits

First, assess the current state of the working tree:

1. Run `git status` to see all modified, staged, and untracked files
2. Run `git diff` to see unstaged changes and `git diff --cached` to see staged changes
3. If there are **no changes at all** (nothing staged, modified, or untracked), stop and tell the user: "Nothing to ship — no changes detected."

Then, create commits:

1. Analyze all changes and group them by **logical unit** (e.g., a new feature, a bug fix, a refactor, test additions)
2. For each logical group:
   - Stage the relevant files with `git add <specific files>`
   - Create a commit following the **Conventional Commits** format:

```
type(scope): subject

- bullet point describing the change (optional, for multi-file changes)
```

**Commit types:**
- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code refactoring (no functional changes)
- `docs`: Documentation changes
- `chore`: Maintenance tasks, tooling, configs
- `test`: Test changes
- `style`: Code formatting, whitespace
- `perf`: Performance improvements

**Scope guidelines:**
- Use file/directory name or feature area (e.g., `auth`, `api`, `commands`)
- Keep it short and meaningful

**Subject guidelines:**
- Use imperative mood ("add feature" not "added feature")
- Don't capitalize first letter
- No period at the end
- Be concise but descriptive
- Never include metrics or hype language

**Important rules:**
- Do NOT use `git add .` or `git add -A` — always add specific files
- Do NOT include `.env`, credentials, or secret files
- Do NOT add a `Co-Authored-By` line
- If all changes are logically related, a single commit is fine
- Use a HEREDOC for multi-line commit messages:
```bash
git commit -m "$(cat <<'EOF'
type(scope): subject

- detail line 1
- detail line 2
EOF
)"
```

# Step 2: Push to Remote

1. Get the current branch name: `git branch --show-current`
2. **Safety check**: If the branch is `main` or `master`, **STOP immediately** and warn the user:
   > "You are on the `main` branch. Create a feature branch first before shipping. Aborting."
3. Push the branch: `git push -u origin <branch-name>`

# Step 3: Create Pull Request

1. Run `git log main..HEAD --oneline` to see all commits being included
2. Run `git diff main..HEAD --stat` to see a summary of file changes
3. Run `git diff main..HEAD` to read the full diff

Analyze the changes and create the PR:

- **PR title**: A concise title that captures the main theme of ALL the changes (not just one commit). This should read as the high-level "what" of the PR.
- **PR body**: Use the exact format below. Analyze the diff carefully to fill in each section.

Create the PR with:

```bash
gh pr create --base main --title "<pr title>" --body "$(cat <<'EOF'
## Summary
- <bullet point summarizing change 1>
- <bullet point summarizing change 2>
- <...etc>

## Changes

### Core Features
- <analysis of source code changes, grouped by feature or area>
- <describe what was added, modified, or removed and why>

### Refactor
- <analysis of refactoring changes — structural improvements, code reorganization>
- <describe what was refactored and the motivation>

### Cleanup
- <analysis of cleanup changes — dead code removal, formatting, lint fixes>
- <describe what was cleaned up>

### Tests
- <list changes in spec/test files>
- <describe what the tests cover and validate>

## Test Plan
- [ ] <step to verify the changes work correctly>
- [ ] <step to verify edge cases>
- [ ] <step to verify no regressions>
EOF
)"
```

**Guidelines for the PR body:**
- **Summary**: High-level bullet points of what changed — readable by someone skimming
- **Core Features**: Analyze the actual source code changes (non-test files). Group by feature area. Explain what each change does.
- **Refactor**: Analyze structural code changes that don't add new functionality (reorganization, extraction, pattern changes). **Only include this section if there are actual refactoring changes — omit it entirely if not applicable.**
- **Cleanup**: Analyze housekeeping changes (dead code removal, formatting fixes, lint fixes, dependency updates). **Only include this section if there are actual cleanup changes — omit it entirely if not applicable.**
- **Tests**: List test/spec file changes specifically. Describe what scenarios they cover. **Only include this section if there are test file changes — omit it entirely if not applicable.**
- **Test Plan**: Actionable checkbox items. Describe how a reviewer can manually verify the changes work.
- **General rule**: If a section has no relevant changes to analyze, **do not include that section in the PR body**. Only include sections that have meaningful content.

After the PR is created, output the PR URL so the user can see it.
