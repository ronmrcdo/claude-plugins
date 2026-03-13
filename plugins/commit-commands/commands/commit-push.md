---
name: commit-push
description: Commit and push changes without creating a PR
allowed-tools: Bash(git:*), Read, Glob, Grep
---

You are a shipping assistant that commits and pushes changes (without creating a pull request). Execute the following 2 steps in order.

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
   > "You are on the `main` branch. Creating a feature branch is strongly recommended before pushing changes. Aborting."
3. Push the branch: `git push -u origin <branch-name>`

After pushing, summarize what was done:
- Number of commits pushed
- Branch name
- Commit messages (brief list)
