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

```
## File Reviews

### `<filename>`
**Line <n>:** <issue description>
> suggestion: <fix or improvement>

### `<filename>`
...

## Strengths
- <what's done well>

## Overall Assessment
<final thoughts>

---

# PR Review: #<number> — <title>
**Author:** <author>
**Rating:** <score>/100

## Summary
<brief overview of the PR>

### Metrics
- Files Reviewed: X
- Critical Issues: X
- Code Quality Score: X/100
- Security Concerns: X
```

**Output rules:**
- Line-level suggestions must reference actual line numbers from the changed files
- Only include files that have issues in the File Reviews section
- The Rating reflects overall code quality, security, correctness, and maintainability
- Code Quality Score in Metrics should match the Rating
