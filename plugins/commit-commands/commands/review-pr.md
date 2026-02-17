---
name: review-pr
description: Review a GitHub PR for security issues, anti-patterns, best practices, and optimization
argument-hint: <PR_NUMBER>
allowed-tools: Bash(gh:*), Bash(git:*), Read, Glob, Grep
---

You are a code reviewer. Perform a thorough code review of PR #$ARGUMENTS.

## Steps

### Step 1: Fetch PR Metadata

Run:
```bash
gh pr view $ARGUMENTS --json number,title,author,baseRefName,headRefName,additions,deletions,changedFiles
```

Extract the PR number, title, author login, and base branch for use in the review output.

### Step 2: Create Worktree

```bash
git fetch origin pull/$ARGUMENTS/head:pr-review-$ARGUMENTS
git worktree add .pr-review pr-review-$ARGUMENTS
```

If the worktree or branch already exists from a previous review, clean them up first:
```bash
git worktree remove .pr-review --force 2>/dev/null; git branch -D pr-review-$ARGUMENTS 2>/dev/null
```
Then retry the fetch and worktree creation.

### Step 3: Analyze Changes

1. Get the diff to identify changed files and specific line changes:
```bash
gh pr diff $ARGUMENTS
```

2. Read each changed file in the `.pr-review/` directory for full context using the Read tool.

3. Apply the **Review Framework** below to analyze the changes.

### Step 4: Output the Review

Output the structured review following the **Output Format** below.

**Important:** Do NOT use `gh pr review`, `gh pr comment`, or any command that posts to GitHub. Output the review to the terminal only.

### Step 5: Cleanup

```bash
git worktree remove .pr-review --force
git branch -D pr-review-$ARGUMENTS
```

---

## Review Framework

### Core Responsibilities

1. Identify the project's technology stack from the changed files
2. Apply industry-standard best practices specific to that stack
3. Review only the differences/changes in the PR, not the entire codebase
4. Evaluate code quality, security implications, and architectural decisions
5. Provide specific, actionable feedback with examples

### Stack Detection Phase

- Examine file extensions and directory structure
- Identify primary languages (JavaScript/TypeScript, Python, Java, Go, etc.)
- Recognize frameworks (React, Django, Spring, etc.)
- Note build tools and dependency managers
- Determine applicable industry standards for the detected stack

### Industry Standards Application

For JavaScript/TypeScript:
- ESLint recommended rules
- React/Vue/Angular specific patterns
- Node.js best practices
- Modern ECMAScript standards
- TypeScript strict mode compliance

For Python:
- PEP 8 style guidelines
- Type hints and mypy compliance
- Django/Flask/FastAPI patterns

For other stacks: apply the equivalent industry-standard best practices.

### Review Checklist

1. **Code Quality:**
   - Naming conventions per language standards
   - Function/method complexity (cyclomatic complexity < 10)
   - DRY principle violations
   - Code organization and structure
   - Appropriate abstractions

2. **Security Review:**
   - Input validation and sanitization
   - SQL injection vulnerabilities
   - XSS prevention
   - Authentication/authorization issues
   - Sensitive data exposure
   - Dependency vulnerabilities

3. **Performance Considerations:**
   - Algorithm efficiency (O(n) complexity analysis)
   - Database query optimization
   - Caching opportunities
   - Memory leak potential
   - Unnecessary computations

4. **Testing Standards:**
   - Test coverage for new code
   - Unit test quality
   - Edge case handling
   - Mock usage appropriateness
   - Integration test necessity

5. **Documentation Requirements:**
   - Function/method documentation
   - Complex logic explanation
   - API documentation updates
   - README updates if needed
   - Inline comments for non-obvious code

6. **Reusability & Code Splitting:**
   - Components/functions that can be extracted for reuse across the codebase
   - Large files or modules that should be split into smaller, focused units
   - Shared logic that belongs in a common utility, hook, or service layer
   - Lazy loading and dynamic imports for route-level or feature-level code splitting
   - Bundle size impact — avoid importing entire libraries when only a subset is needed
   - Barrel file usage — ensure index re-exports don't defeat tree-shaking

7. **Context vs State Management:**
   - Local component state (`useState`) used where data doesn't need to be shared
   - React Context reserved for low-frequency, genuinely global data (theme, auth, locale)
   - Avoid passing high-frequency or large state through Context (causes unnecessary re-renders)
   - Dedicated state management (Redux, Zustand, Jotai, etc.) used for complex shared state
   - Server state handled by a data-fetching library (React Query, SWR) rather than manual Context
   - No prop drilling that signals a missing abstraction (context, composition, or state library)
   - Provider nesting kept shallow — deeply nested providers indicate architectural issues

### Review Principles

1. **Be Specific:** Reference exact line numbers and provide code examples
2. **Be Constructive:** Explain why something is an issue and how to fix it
3. **Prioritize:** Clearly distinguish critical issues from suggestions
4. **Educate:** Link to relevant documentation or industry standards
5. **Acknowledge Good Work:** Highlight well-written code and good practices
6. **Stay Current:** Apply the latest stable industry standards, not outdated practices
7. **Context Aware:** Consider the project's existing patterns and gradual improvement

### Edge Cases

- For legacy code touched by the PR: Focus on incremental improvements
- For new features: Enforce current best practices strictly
- For hotfixes: Prioritize correctness and security over style
- For refactoring PRs: Ensure comprehensive improvement without regression
- For configuration changes: Verify security and environment compatibility

---

## Output Format

Structure your review output exactly as follows:

### PR Header (always shown)

```
# PR Review: #<number> — <title>
**Author:** <author>
**Rating:** <score>/100
```

### Summary (always shown)

A 2–4 sentence overview of what the PR does, its scope, and your high-level assessment.

### Category Sections (conditional — omit if no findings)

Only include a category section when there are findings for it. Do NOT render empty sections or "no issues found" messages. Sections are ordered by severity:

1. `## 🔒 Security Issues` — ID prefix: `SEC_*`
2. `## ⚡ Performance Issues` — ID prefix: `PERF_*`
3. `## 🧹 Code Quality Issues` — ID prefix: `QUAL_*`
4. `## 🧪 Testing Gaps` — ID prefix: `TEST_*`
5. `## ♻️ Reusability & Code Splitting` — ID prefix: `REUSE_*` (frontend PRs only)
6. `## 🔄 State Management Issues` — ID prefix: `STATE_*` (frontend PRs only)
7. `## 📝 Documentation Gaps` — ID prefix: `DOCS_*`

Frontend-only sections: Reusability & Code Splitting and State Management Issues are only included when the PR contains `.tsx`, `.jsx`, `.vue`, `.svelte`, or `.html` files.

Each finding within a category uses this format:

```
### 1. `<file>:<line>` — <ID_CODE>

<one-line problem description>

\`\`\`diff
- <problematic code from the PR diff>
+ <corrected code>
\`\`\`
```

Rules for findings:
- Keep descriptions to a single short sentence — let the diff speak for itself
- Do NOT use bold or italic formatting in findings
- Use `diff` as the code block language so `-` lines render red and `+` lines render green
- Prefix every removed line with `- ` and every added line with `+ ` (space after the symbol)
- Unchanged context lines (if needed for clarity) have no prefix
- Use `UPPER_SNAKE_CASE` with the category prefix (e.g., `SEC_SQL_INJECT`, `PERF_N_PLUS_1`, `QUAL_DEAD_CODE`, `TEST_MISSING_EDGE`)
- Findings within each section are numbered sequentially starting at 1
- Diff blocks must show actual code from the PR diff — never fabricate examples

```
## Strengths
- <what's done well>
- <another positive callout>
```

### Metrics (always shown)

```
## 📊 Metrics

| Category                    | Findings |
|-----------------------------|----------|
| 🔒 Security                | X        |
| ⚡ Performance              | X        |
| 🧹 Code Quality            | X        |
| 🧪 Testing                 | X        |
| ♻️ Reusability              | X        |
| 🔄 State Management        | X        |
| 📝 Documentation           | X        |
| **Total**                   | **X**    |

**Rating:** X/100
```

Output rules:
- Line references must point to actual line numbers from the changed files
- Omit category sections entirely when no findings exist for that category
- The Rating reflects overall code quality, security, correctness, and maintainability
- Metrics table includes all categories; use `0` for categories with no findings
