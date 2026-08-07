# Report Format

One unified report. Deduplicate findings that multiple agents raised for the same `file:line` —
keep the highest severity and merge the fixes. Do not staple nine reports together.

```
# PR Review — #<number>: <title>
<owner>/<repo> · @<author> · <base> ← <head> · +<additions>/−<deletions> across <n> files · <STATE>
Stack: <detected stack>          Agents dispatched: <n>

## Verdict: <REQUEST CHANGES | APPROVE>
<n> Critical, <n> High blocking.        # or: No blocking findings.

### Blocking findings
| # | Sev | Category | file:line | Issue | Fix |
|---|-----|----------|-----------|-------|-----|

### Suggestions (non-blocking)
| # | Sev | Category | file:line | Suggestion |
|---|-----|----------|-----------|------------|

### Findings by category
<one section per dispatched agent, in dispatch order; "No issues found" when clean>

### Summary
| Category | Critical | High | Medium | Low |
|----------|----------|------|--------|-----|
| ...      |          |      |        |     |
| **Total**|          |      |        |     |

### Strengths
- <positive patterns worth keeping>

### Coverage notes
- <files reviewed diff-only, and why>
- <agents that failed to return>
- <nothing to note: "Full content reviewed for all changed files; all agents returned.">
```

## Verdict rule

```
Critical > 0 or High > 0   →  REQUEST CHANGES
only Medium / Low          →  APPROVE (with comments)
no findings                →  APPROVE
```

Every blocking finding appears in the Blocking table with its `file:line`. The verdict must
always trace to the specific findings that produced it — never state a verdict the tables do
not support.

## Qualifiers

Append to the verdict line when they apply. They stack.

| Qualifier | When |
|---|---|
| **(provisional)** | An agent failed or returned nothing |
| **(partial)** | The size guard reduced any file to diff-only |
| **(advisory)** | The PR is already merged or closed |

## Closing line

End every report with, verbatim:

> This is a recommendation only — nothing was posted to the PR.
