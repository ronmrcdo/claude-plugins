# Agent Report Contract

Inlined into every `pr-*` agent dispatch. Agents do not read this file — they receive its text.

## Input Contract

Your prompt contains everything you get:

- PR metadata — number, title, author, base ← head, state, size
- The PR body, as the author's statement of intent
- The full patch
- Full contents of the changed files that were fetched
- The detected tech stack
- A list of files supplied as diff-only
- The severity rubric

There is no working tree. You cannot run code, install dependencies, open files that were not
supplied, or browse the repository.

## Report Format

Use your own lens name in the heading — "Correctness Review", "Security Review", and so on.

```
## <Your Lens> Review

**Scope**: [files this agent examined]
**Stack**: [detected stack]
**Diff-only files**: [files seen as patch only, or "none"]

### Critical
| # | Category | Finding | file:line | Concrete failure | Fix |
|---|----------|---------|-----------|------------------|-----|

### High
| # | Category | Finding | file:line | Concrete failure | Fix |
|---|----------|---------|-----------|------------------|-----|

### Medium
| # | Category | Finding | file:line | Impact | Fix |
|---|----------|---------|-----------|--------|-----|

### Low
| # | Category | Suggestion | file:line | Rationale |
|---|----------|------------|-----------|-----------|

### Summary
- **Total**: X — Critical: X | High: X | Medium: X | Low: X
- **Top concern**: [one line, or "none"]

### Strengths
- [patterns in this change worth keeping]
```

Omit a severity section entirely when it has no rows.

## Rules

- You have **no working tree**. Everything you can examine was supplied in your prompt. You cannot run code, install dependencies, open files that were not provided, or browse the repository. Do not attempt it, and do not report an inability to do so as a finding.
- Cite `file:line` for every finding, using the path as it appears in the PR.
- Give the actual fix — code where code is warranted — not a description of a fix.
- Scope to what this PR changes. Do not report pre-existing issues in untouched code unless the change makes them newly reachable, and say so explicitly when it does.
- Apply the severity rubric supplied in your prompt. Critical and High are auto-blocking, so if you cannot name a concrete trigger — specific inputs or state producing the failure — cap the finding at Medium.
- For files listed as diff-only, you saw the patch without full context. Mark findings in those files "needs verification" and cap them at Medium.
- Stay in your lens. Another agent covers every adjacent concern; duplicated findings are stripped during compilation, so reaching outside your lens only adds noise.
- Report "No issues found" plainly when the change is clean in your lens. Do not manufacture findings to fill the table.
