# Severity Rubric

Apply these definitions exactly. The review verdict is computed from severity counts, so
inconsistent rating across agents makes the verdict meaningless.

| Severity | Meaning |
|---|---|
| **Critical** | Exploitable vulnerability, data loss or corruption, breaks production on merge, or silently wrong results on a common path |
| **High** | A bug real users will hit on a reachable path; missing authorization on a non-trivial surface; a breaking change to a consumed contract; no tests for new business-critical logic |
| **Medium** | Degraded behavior in edge cases, notable performance regression, weak error handling, meaningful test gap, WCAG AA violation on a non-critical control |
| **Low** | Style, naming, minor duplication, hardening suggestions, nits |

## Anti-inflation rules

Critical and High are auto-blocking. Rate accordingly.

1. **Name the trigger.** If you cannot state the specific inputs or state that produce the
   failure, cap the finding at Medium.
2. **Unverifiable caps at Medium.** If the finding cannot be confirmed from the context supplied
   in your prompt, mark it "needs verification" and cap it at Medium. This applies to every
   finding in a file listed as diff-only.
3. **Rate on exploitability and blast radius**, not on which category the issue belongs to. A
   theoretical SQL injection behind an admin-only feature flag is not Critical; a missing null
   check on the primary request path may be.
4. **Reachability matters.** Dead code, test fixtures, and unreachable branches cap at Low.
