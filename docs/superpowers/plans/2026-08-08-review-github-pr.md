# `review-github-pr` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Paste a GitHub PR URL and get back a stack-aware multi-agent review with an Approve / Request Changes recommendation, without touching the local git repo or the PR.

**Architecture:** A skill in the `code-reviewer` plugin parses the URL, pulls PR metadata, the patch, and full file contents at the head SHA over `gh` (no git), detects the stack from changed extensions unioned with manifest dependencies, dispatches 5–9 `pr-*` agents in one parallel batch, then compiles a deduplicated report and applies a deterministic severity threshold to produce the verdict.

**Tech Stack:** Markdown specification files with YAML frontmatter, one JSON marketplace registry, GitHub CLI (`gh`). No build, no compiler, no test runner.

**Spec:** `docs/superpowers/specs/2026-08-08-review-github-pr-design.md`

## Global Constraints

- **No git, ever.** No `fetch`, `pull`, `clone`, `checkout`, `worktree`. All PR data comes from `gh`.
- **Read-only.** Never `gh pr review`, `gh pr comment`, `gh pr merge`, `gh pr close`, `gh pr edit`, `gh pr ready`; never `gh api` with `-X POST/PATCH/PUT/DELETE`, `--method <non-GET>`, or `-f`/`--field`.
- **Enforced at two layers.** The skill scopes itself via `allowed-tools`; each `pr-*` agent scopes itself via `tools: Read, Grep, Glob` frontmatter, because dispatched agents otherwise inherit every tool including Write and Edit.
- **Naming:** kebab-case files and directories. New agents are prefixed `pr-` so they never collide with the seven existing staged-changes agents.
- **Agent model:** `model: sonnet`, matching the existing convention.
- **Verdict rule, verbatim:** `Critical > 0 or High > 0 → REQUEST CHANGES`; `only Medium/Low → APPROVE (with comments)`; `no findings → APPROVE`.
- **The seven existing agents are not modified.** `/review-unstaged` keeps working unchanged.
- **No silent truncation.** Any file reviewed diff-only must be named in the report.

## File Structure

| Path | Responsibility |
|---|---|
| `plugins/code-reviewer/skills/review-github-pr/SKILL.md` | Orchestrator: parse → fetch → detect → dispatch → compile → rule |
| `plugins/code-reviewer/skills/review-github-pr/references/severity-rubric.md` | Severity definitions + anti-inflation rules; injected into every dispatch |
| `plugins/code-reviewer/skills/review-github-pr/references/agent-report-contract.md` | Input contract + agent report format + shared rules; injected into every dispatch |
| `plugins/code-reviewer/skills/review-github-pr/references/stack-detection.md` | Extension + dependency → agent mapping |
| `plugins/code-reviewer/skills/review-github-pr/references/report-format.md` | Unified report template + verdict rule + qualifiers |
| `plugins/code-reviewer/agents/pr-correctness.md` | Logic bugs, edge cases, intent match |
| `plugins/code-reviewer/agents/pr-security.md` | Vulnerabilities in changed code |
| `plugins/code-reviewer/agents/pr-performance.md` | Complexity, N+1, re-renders, memory, blocking I/O |
| `plugins/code-reviewer/agents/pr-test-coverage.md` | Test presence and assertion quality |
| `plugins/code-reviewer/agents/pr-integration.md` | Half-landed refactors, orphaned refs, breaking contracts |
| `plugins/code-reviewer/agents/pr-frontend.md` | Component/render/state/bundle concerns |
| `plugins/code-reviewer/agents/pr-accessibility.md` | WCAG 2.1 AA conformance |
| `plugins/code-reviewer/agents/pr-data-layer.md` | Queries, schema, migrations |
| `plugins/code-reviewer/agents/pr-infra.md` | Dockerfile, CI, terraform, k8s |
| `plugins/commit-commands/commands/review-pr.md` | **Deleted** |
| `README.md`, `.claude-plugin/marketplace.json`, `CLAUDE.md` | Updated |

## Shared Agent Contract — extracted, not duplicated

The input contract, report format, and shared rules are **identical across all nine agents**, so
they live in one file — `references/agent-report-contract.md`, created in Phase 1 Step 2 — and
the orchestrator inlines them into every dispatch prompt, exactly as it does the severity rubric.

**Agent files therefore contain only their unique content:** frontmatter, `# Title`, `## Purpose`,
`## What You Analyze`, and (where noted) `## Additional Rules`. Do not reproduce the contract in
any agent file.

The tradeoff, accepted deliberately: an agent dispatched directly, outside the skill, receives no
format instructions. These agents are skill-dispatched by design — their descriptions say so — and
one authoritative copy beats nine hand-synced ones.

The full contract text is in Phase 1 Step 2.

---

## Phase 1: Skill scaffold and references

Builds the orchestrator and the four reference files. Nothing dispatches yet — the deliverable is a skill that correctly parses a URL, fetches all PR data, detects the stack, and prints what it *would* dispatch.

**Files:**
- Create: `plugins/code-reviewer/skills/review-github-pr/SKILL.md`
- Create: `plugins/code-reviewer/skills/review-github-pr/references/severity-rubric.md`
- Create: `plugins/code-reviewer/skills/review-github-pr/references/agent-report-contract.md`
- Create: `plugins/code-reviewer/skills/review-github-pr/references/stack-detection.md`
- Create: `plugins/code-reviewer/skills/review-github-pr/references/report-format.md`

**Produces:** the reference file paths that Phase 2/3 agents' dispatch prompts inject, and the agent names Phases 2–3 must match exactly: `pr-correctness`, `pr-security`, `pr-performance`, `pr-test-coverage`, `pr-integration`, `pr-frontend`, `pr-accessibility`, `pr-data-layer`, `pr-infra`.

- [ ] **Step 1: Create `references/severity-rubric.md`**

```markdown
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
```

- [ ] **Step 2: Create `references/agent-report-contract.md`**

This is the single authoritative copy of what every `pr-*` agent is told about its input, its
output format, and the rules binding all of them. SKILL.md inlines it into every dispatch prompt.
No agent file reproduces it.

`````markdown
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
`````

- [ ] **Step 3: Create `references/stack-detection.md`**

```markdown
# Stack Detection

Two independent signals, unioned. Extensions alone are unreliable — a `.ts` file is frontend or
backend depending on whether the project depends on `next` or `@nestjs/core`.

## Signal A — changed file extensions

From the `files` array in the `gh pr view` metadata.

| Bucket | Extensions and paths |
|---|---|
| Frontend | `.tsx`, `.jsx`, `.vue`, `.svelte`, `.html`, `.css`, `.scss`, `.sass`, `.less` |
| Backend | `.ts`, `.js`, `.mjs`, `.py`, `.go`, `.java`, `.kt`, `.rb`, `.rs`, `.php`, `.cs` |
| Data | `.sql`, `.prisma`, any path containing `migrations/`, `migrate/`, or `schema/` |
| Infra | `Dockerfile*`, `docker-compose.y*ml`, `*.tf`, `*.tfvars`, `.github/workflows/*`, `k8s/`, `*.helm.y*ml`, `Chart.yaml` |
| Config | `.json`, `.yaml`, `.yml`, `.toml`, `.ini`, `.env*` |

## Signal B — manifest dependencies

Fetch each manifest at the PR head SHA the same way as changed files. **Best-effort: a 404 means
the project is not that stack, not an error.** Do not fail the review over a missing manifest.

`package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `Cargo.toml`, `composer.json`,
`Gemfile`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `*.csproj`, `Package.swift`,
`pubspec.yaml`

Read dependency names, not versions:

| Signal | Dependencies |
|---|---|
| Frontend | `react`, `react-dom`, `next`, `vue`, `svelte`, `@sveltejs/kit`, `@angular/core`, `solid-js`, `astro` |
| Backend (Node) | `express`, `@nestjs/core`, `fastify`, `koa`, `hono`, `@hapi/hapi` |
| Data | `prisma`, `@prisma/client`, `typeorm`, `sequelize`, `mongoose`, `drizzle-orm`, `knex`, `sqlalchemy`, `alembic`, `django`, `gorm.io/gorm`, `diesel`, `activerecord` |

## Dispatch table

| Agent | Fires when |
|---|---|
| `pr-correctness` | always |
| `pr-security` | always |
| `pr-performance` | always |
| `pr-test-coverage` | always |
| `pr-integration` | always |
| `pr-frontend` | Frontend extension changed **or** a Frontend dependency present |
| `pr-accessibility` | same condition as `pr-frontend` |
| `pr-data-layer` | Data extension/path changed **or** a Data dependency present |
| `pr-infra` | Infra path changed |

Range: 5 agents on a pure backend PR, 9 on a full-stack PR touching the database and CI.

State the detected stack in human terms in the report header — "Next.js + NestJS + Postgres",
not "has_frontend=true".
```

- [ ] **Step 4: Create `references/report-format.md`**

````markdown
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
````

- [ ] **Step 5: Create `SKILL.md`**

`````markdown
---
name: review-github-pr
description: Review a GitHub pull request from its URL using stack-aware parallel agents and return an approve / request-changes recommendation. Use when the user pastes a GitHub pull request URL, or asks to review, check, or assess a PR. Read-only — never comments on, approves, or modifies the PR, and never fetches or checks out code locally.
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api:*), Read, Glob, Grep, Task
model: sonnet
---

# Review a GitHub PR

Review a pull request from its URL and recommend Approve or Request Changes.

**You never modify anything.** Not the PR, not the local repository. See Rules.

## Step 1: Parse the URL

Match the URL's **path** against:

```
/(?<owner>[^/]+)/(?<repo>[^/]+)/pull/(?<number>\d+)
```

Path-only matching tolerates trailing segments (`/files`, `/commits`, `/checks`), fragments
(`#discussion_r123`), and query strings, and works on GitHub Enterprise hosts since `gh`
resolves the host from its own config.

No match → stop and print:

> Paste a GitHub PR URL, e.g. `https://github.com/owner/repo/pull/123`

Never infer a repository from the current directory. This skill reviews the PR at the URL given,
from wherever you happen to be standing.

## Step 2: Fetch metadata

```bash
gh pr view <number> --repo <owner>/<repo> --json number,title,body,author,state,isDraft,baseRefName,headRefName,headRefOid,additions,deletions,changedFiles,files,labels
```

On failure — PR not found, `gh` not authenticated, no access — stop and surface the error
verbatim. Nothing was created, so there is nothing to unwind.

`state` and `isDraft` are informational, never fatal. Draft, merged, and closed PRs are still
reviewed; state goes in the report header, and merged/closed PRs get the **(advisory)** verdict
qualifier.

## Step 3: Fetch the diff

```bash
gh pr diff <number> --repo <owner>/<repo> --patch
```

## Step 4: Fetch full file contents at the head SHA

The patch shows three lines of context, which is not enough to judge whether a function is
correct. Fetch each changed file whole, as it exists in the PR:

```bash
gh api "repos/<owner>/<repo>/contents/<url-encoded-path>?ref=<headRefOid>" -H "Accept: application/vnd.github.raw"
```

URL-encode the path — spaces and `#` in filenames will otherwise silently produce the wrong file
or a 404.

**Fetch nothing for these; the patch is enough:**

- Deleted files — no content exists at head
- Lockfiles: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb`, `Cargo.lock`, `poetry.lock`, `Gemfile.lock`, `composer.lock`, `go.sum`
- Generated or vendored: `dist/`, `build/`, `vendor/`, `node_modules/`, `*.min.js`, `*.min.css`, `*.map`, `*.snap`, `*.pb.go`, `*_pb2.py`
- Binaries: images, fonts, archives, PDFs
- Files over 500 KB

Renamed files: fetch at the **new** path. If any fetch fails — including GitHub's 1 MB ceiling on
the contents endpoint — fall back to diff-only for that file and record it for Coverage notes.

**Size guard.** Above **60 changed files** or **3000 changed lines**, fetch full content only for
the highest-signal files: rank by changed-line count, application source ahead of tests, tests
ahead of config. Every file left at diff-only goes in Coverage notes, and the verdict takes the
**(partial)** qualifier.

A report that quietly reviewed 40 of 120 files reads exactly like one that reviewed all 120.
Never let it.

## Step 5: Detect the stack

Read `references/stack-detection.md` and follow it. It unions changed-file extensions with
manifest dependencies and yields the agent dispatch list.

Manifest fetches use the same command as Step 4 and are best-effort — a 404 means the project is
not that stack, not an error.

## Step 6: Dispatch agents

Read `references/severity-rubric.md` and `references/agent-report-contract.md`.

Dispatch every agent the Step 5 table selected **in a single parallel batch** — one message,
multiple Task calls. Address each by its subagent type: `code-reviewer:pr-correctness`,
`code-reviewer:pr-security`, `code-reviewer:pr-performance`, `code-reviewer:pr-test-coverage`,
`code-reviewer:pr-integration`, and whichever of `code-reviewer:pr-frontend`,
`code-reviewer:pr-accessibility`, `code-reviewer:pr-data-layer`, `code-reviewer:pr-infra` apply.

Every agent gets the same payload:

1. **PR metadata** — number, title, author, base ← head, state, additions/deletions/changed files
2. **The PR body**, as the statement of intent to judge the diff against. Empty body → say so and
   instruct the agent to judge against the changes' apparent intent.
3. **The full patch** from Step 3
4. **Full contents** of every file fetched in Step 4
5. **The detected stack** from Step 5
6. **The list of diff-only files**, so agents scope their confidence
7. **The full text of `references/severity-rubric.md`, inlined**
8. **The full text of `references/agent-report-contract.md`, inlined** — the agents' input contract, report format, and shared rules

Inline items 7 and 8 rather than pointing at their paths. The verdict is computed from severity
counts, so agents applying different thresholds would make it meaningless; and an agent told to
go read a file may not, whereas text in its prompt is unconditional.

If an agent fails or returns nothing, continue with the rest. Record it in Coverage notes and
apply the **(provisional)** qualifier.

## Step 7: Compile and rule

Read `references/report-format.md` and produce exactly that report, applying the verdict rule and
any qualifiers.

Deduplicate across agents before writing the tables — several lenses will flag the same
`file:line`. Keep the highest severity and merge the fixes into one row.

## Rules

**Read-only. No exceptions.**

- Never `gh pr review`, `gh pr comment`, `gh pr merge`, `gh pr close`, `gh pr edit`, `gh pr ready`
- Never `gh api` with `-X POST`, `-X PATCH`, `-X PUT`, `-X DELETE`, `--method` and a non-GET verb, or `-f`/`--field` (which implies POST)
- Never any `git` subcommand — in particular `fetch`, `pull`, `clone`, `checkout`, `worktree`
- Never create, edit, or delete a file

The verdict is a recommendation delivered to the user in the terminal. Acting on it is the user's
call, not yours. If the user wants the review posted, they post it.

- Do not review pre-existing issues in untouched code unless the change makes them newly
  reachable — and say so explicitly when it does.
- One unified report. Never nine reports concatenated.
- If the size guard or an agent failure limited coverage, say so in Coverage notes and qualify
  the verdict. Never present partial coverage as complete.
`````

- [ ] **Step 6: Verify frontmatter and structure**

```bash
cd /home/ronmrcdo/Projects/vibe/claude-plugins
for f in plugins/code-reviewer/skills/review-github-pr/SKILL.md \
         plugins/code-reviewer/skills/review-github-pr/references/*.md; do
  echo "== $f"; head -1 "$f"; done
ls -R plugins/code-reviewer/skills
```

Expected: `SKILL.md` line 1 is `---`; all four reference files exist under `references/`.

- [ ] **Step 7: Dry-run the fetch and detection pipeline**

Pick a real PR you can read. Run Steps 1–5 manually and confirm each produces data:

```bash
gh pr view 1 --repo <owner>/<repo> --json number,title,headRefOid,changedFiles,files
gh pr diff 1 --repo <owner>/<repo> --patch | head -40
gh api "repos/<owner>/<repo>/contents/package.json?ref=<headRefOid>" -H "Accept: application/vnd.github.raw" | head -20
```

Expected: metadata JSON parses, patch prints, contents fetch returns raw file text. A 404 on the
manifest is a valid outcome — confirm it does not abort the run.

Then state which agents Step 5 would select for that PR and confirm the list matches the dispatch
table by hand. Nothing dispatches yet — the agents do not exist until Phase 2.

---

## Phase 2: The five always-on agents

**Files:**
- Create: `plugins/code-reviewer/agents/pr-correctness.md`
- Create: `plugins/code-reviewer/agents/pr-security.md`
- Create: `plugins/code-reviewer/agents/pr-performance.md`
- Create: `plugins/code-reviewer/agents/pr-test-coverage.md`
- Create: `plugins/code-reviewer/agents/pr-integration.md`

**Consumes:** the agent names and dispatch payload defined in Phase 1 Step 4.
**Produces:** subagent types `code-reviewer:pr-correctness`, `code-reviewer:pr-security`, `code-reviewer:pr-performance`, `code-reviewer:pr-test-coverage`, `code-reviewer:pr-integration`.

Every file in this phase has exactly this shape, and nothing more:

```
---frontmatter---
# <Title>
## Purpose          — one paragraph: the single lens this agent holds
## What You Analyze — the checklist below
## Additional Rules — only where this task specifies one
```

**Do not add an Input Contract, Report Format, or Rules section.** Those live in
`references/agent-report-contract.md` (Phase 1 Step 2) and are inlined into every dispatch
prompt by SKILL.md. Reproducing them here is the duplication this plan exists to avoid.

- [ ] **Step 1: Create `pr-correctness.md`**

Frontmatter:

```yaml
---
name: pr-correctness
description: Reviews a pull request for logic errors, unhandled edge cases, regressions, and mismatches between what the PR body claims and what the diff actually does.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: is this code correct? You judge whether the changed logic produces the right result for every input that can reach it, and whether it delivers what the PR says it delivers.*

`## What You Analyze`:

```markdown
### 1. Intent vs. implementation
- Does the diff do what the PR body claims? Name any claimed behavior with no corresponding code.
- Does the diff do things the PR body does not mention? Unannounced scope is a finding.
- Are stated constraints ("no breaking changes", "behind a flag") actually honored?

### 2. Logic errors
- Off-by-one in indices, slices, ranges, and pagination
- Inverted or short-circuited conditionals; `&&`/`||` precedence mistakes
- Wrong operator: `=` vs `==` vs `===`, `is` vs `==`, reference vs value equality
- Loop bounds, early `return`/`break`/`continue` that skips required work
- Assignment to a variable that is never read; a computed value that is discarded

### 3. Null, undefined, and empty
- Optional chaining or a null check on one access path but not the sibling path
- Empty array, empty string, and zero treated as absent (`if (!count)` when `0` is valid)
- Non-null assertions (`!`, `as`, `unwrap()`) on values that can genuinely be absent
- Default values applied after the value is already used

### 4. Types and boundaries
- Unsafe narrowing or casts that discard a union member that can occur at runtime
- Numeric precision, integer overflow, float comparison with `==`
- Timezone- and DST-sensitive date arithmetic; date-only vs. instant confusion
- Encoding assumptions on user input

### 5. Async and concurrency
- Missing `await`; a floating promise whose rejection is unhandled
- Race between read and write of shared state; check-then-act without a lock or transaction
- Unbounded parallelism over a user-controlled collection
- Cancellation and cleanup: aborted requests, unmounted components, closed connections

### 6. Error handling
- `catch` blocks that swallow the error or log and continue with invalid state
- Errors caught at a level that cannot meaningfully recover
- Partial mutation left behind when an operation fails midway — no rollback, no transaction
- A thrown error whose type callers do not handle

### 7. Regressions
- Changed behavior at a call site the diff does not update
- A modified default value or config that changes existing behavior silently
- Removed guard clauses, validation, or checks — was the removal justified?
```

This agent's `## Purpose` must name its lens as **Correctness**, since the injected contract tells it to head its report `## Correctness Review`. This agent needs no `## Additional Rules` section.

- [ ] **Step 2: Create `pr-security.md`**

Frontmatter:

```yaml
---
name: pr-security
description: Reviews a pull request for security vulnerabilities including injection, broken authentication and authorization, secret exposure, and insecure configuration in the changed code.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: what can an attacker do with this change? You assess only the code this PR touches, and only vulnerabilities with a plausible attack path.*

`## What You Analyze`:

```markdown
### 1. Injection
- SQL/NoSQL: string concatenation or interpolation into queries; Prisma `$queryRaw` / `$executeRaw` without `Prisma.sql`; Mongoose `$where` or unsanitized filters
- Command: user input reaching `exec`, `spawn`, `system`, `popen` — `execFile` with an explicit argv is the fix
- Path traversal: user input in file paths without normalization and a containment check
- Template, LDAP, and CRLF header injection

### 2. Cross-site scripting
- `dangerouslySetInnerHTML`, `v-html`, `innerHTML`, `outerHTML` with unsanitized content
- User-controlled values in `href`, `src`, or `action` — `javascript:` and `data:` URLs
- `eval`, `new Function`, dynamic `import()` with user-controlled input
- Markdown or SVG rendered without sanitization

### 3. Authentication and session
- Credentials, tokens, API keys, or private keys hardcoded in the diff — always Critical
- Secrets in URL query parameters, where they land in logs and referrers
- Weak or missing password hashing; missing salt
- JWT verified without an `algorithms` allowlist, or with `alg: none` accepted
- Session tokens without expiry or rotation; cookies missing `httpOnly`, `secure`, `sameSite`
- Missing rate limiting on authentication endpoints

### 4. Authorization
- A new endpoint, handler, or mutation with no authorization check
- IDOR: an identifier taken from the request and used without an ownership check
- Privilege escalation: role or permission derived from client-supplied data
- Client-side-only access control with no server-side enforcement
- CORS with `origin: '*'` alongside credentials

### 5. Data exposure
- Secrets, tokens, or PII written to logs
- Responses returning more fields than the caller needs — password hashes, internal IDs, PII
- Stack traces or database errors returned to the client
- Auth tokens in `localStorage`/`sessionStorage` rather than httpOnly cookies
- Secrets exposed through client-visible env vars (`NEXT_PUBLIC_*`, `VITE_*`, `EXPO_PUBLIC_*`)

### 6. Input validation
- Request body, params, or query used without schema validation (`zod`, `joi`, `class-validator`, Fastify schemas, pydantic)
- Mass assignment: whole request bodies spread into an entity without an allowlist
- File uploads without type, size, and destination-path validation
- Missing body size limits; XXE enabled in XML parsing; insecure deserialization

### 7. Cryptography and randomness
- MD5, SHA1, DES, RC4 used for security purposes
- Hardcoded keys or IVs; reused nonces
- `Math.random()`, `random()`, or timestamps used to generate tokens, IDs, or secrets
- Hand-rolled crypto instead of a vetted library

### 8. Dependencies and configuration
- New dependencies: typo-squattable names, unmaintained packages, known-vulnerable version ranges
- Debug modes, verbose errors, or permissive settings enabled in production config
- Removed or weakened security middleware (`helmet`, CSRF protection, rate limiters)
- `target="_blank"` without `rel="noopener noreferrer"`; `postMessage` handlers without origin checks
```

This agent's `## Purpose` must name its lens as **Security**, since the injected contract tells it to head its report `## Security Review`. Add an `## Additional Rules` section:

```markdown
- Reference the OWASP category where one applies.
- Any credential, token, or private key found in the diff is Critical regardless of apparent scope.
```

- [ ] **Step 3: Create `pr-performance.md`**

Frontmatter:

```yaml
---
name: pr-performance
description: Reviews a pull request for runtime inefficiencies including algorithmic complexity, N+1 queries, unnecessary re-renders, memory retention, and blocking I/O.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: what does this change cost at runtime, at production scale? You care about work that grows with input size, repeats needlessly, or blocks.*

`## What You Analyze`:

```markdown
### 1. Algorithmic complexity
- Nested iteration over collections that scale with data — O(n²) where a lookup map gives O(n)
- `Array.includes` / `indexOf` / `find` inside a loop instead of a `Set` or `Map`
- Repeated sorting, or sorting to obtain a single min/max
- Work inside a loop that is invariant and could be hoisted
- Unbounded recursion or missing memoization on an exponential call tree

### 2. Database and network
- N+1: a query, `fetch`, or RPC inside a loop or per-item `map`
- Missing eager loading (`include`, `select_related`, `JOIN`) where relations are then accessed
- `SELECT *` where a handful of columns are used
- Queries filtering or sorting on unindexed columns; no index accompanying a new query pattern
- Missing pagination on a collection endpoint
- Sequential awaits that are independent and could be `Promise.all`
- Missing caching on a hot, stable read

### 3. Rendering (frontend stacks)
- Object, array, or function literals created inline in JSX and passed as props
- Missing or wrong dependency arrays causing effects and memos to run every render
- Expensive computation in render body instead of `useMemo`
- Context value recreated each render, re-rendering every consumer
- Large lists rendered without virtualization
- State placed higher in the tree than it needs to be, widening the re-render blast radius

### 4. Memory
- Listeners, intervals, timeouts, subscriptions, and observers added without cleanup
- Closures capturing large objects that then outlive them
- Caches and maps that only ever grow — no eviction, no bound
- Whole files or result sets loaded into memory where a stream would do

### 5. Blocking and startup
- Synchronous filesystem or crypto calls on a request path
- CPU-bound work on the event loop or main thread
- Heavy modules imported eagerly where a dynamic import would defer them
- Newly added dependency that is large relative to what it provides — name the lighter alternative
```

This agent's `## Purpose` must name its lens as **Performance**, since the injected contract tells it to head its report `## Performance Review`. Add an `## Additional Rules` section:

```markdown
- Quantify where you can: "O(n²) over `orders`, ~500 rows in production" beats "this is slow". If you cannot estimate the scale, cap the finding at Medium.
```

- [ ] **Step 4: Create `pr-test-coverage.md`**

Frontmatter:

```yaml
---
name: pr-test-coverage
description: Reviews a pull request for test coverage gaps, missing edge cases, weak assertions, and tests that do not actually verify the behavior they claim to.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: if this merges and later breaks, will a test catch it? You judge both whether tests exist for the changed paths and whether they would actually fail on a regression.*

`## What You Analyze`:

```markdown
### 1. Coverage of the change
- New functions, endpoints, components, or branches with no test touching them
- Modified behavior whose existing tests were not updated — a test asserting the old contract that still passes is worse than no test
- Bug fixes with no regression test reproducing the original bug
- Error paths and failure modes: only the happy path exercised

### 2. Edge cases missing
- Empty collection, single element, and the boundary just past a limit
- Null, undefined, empty string, and zero where zero is meaningful
- Duplicate entries, unicode, and very long input
- Concurrent or repeated invocation where the code has state
- Timezone and DST boundaries in date logic

### 3. Assertion quality
- Asserting only that a call did not throw
- Asserting truthiness where a specific value is what matters (`expect(x).toBeTruthy()` on a value that should be exactly `3`)
- Snapshot tests standing in for behavioral assertions
- Tests with no assertion at all
- Mocks asserted against instead of behavior — verifying the mock was called, not that the outcome is correct

### 4. Test integrity
- Over-mocking: the system under test is mocked out and the test verifies the mock
- Shared mutable state between tests; order dependence
- Non-determinism: real clocks, real network, real random, unseeded fixtures
- Tests that would pass against an empty implementation
- Skipped, `.only`, or commented-out tests introduced by the diff

### 5. Specifications for the gaps
For every meaningful gap you find, write the missing test as a specification:
name, arrange, act, expected assertion. Use the test framework already present in the
supplied files. Prose descriptions of missing tests are not acceptable output.
```

This agent's `## Purpose` must name its lens as **Test Coverage**, since the injected contract tells it to head its report `## Test Coverage Review`. Add an `## Additional Rules` section:

```markdown
- If the PR changes no logic — docs, formatting, config only — say so and report no findings rather than demanding tests.
- Missing tests for new business-critical logic is High. Missing tests for a trivial accessor is Low. Judge by what breaking it would cost.
```

- [ ] **Step 5: Create `pr-integration.md`**

Frontmatter:

```yaml
---
name: pr-integration
description: Reviews a pull request for structural integration problems — half-completed refactors, orphaned references, dead code left behind, breaking changes to consumed contracts, and missing migrations or configuration.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: did this change fully land? Code can be logically correct and still be half-applied. You look for the parts of the change that were started and not finished, and for what breaks outside the diff.*

`## What You Analyze`:

```markdown
### 1. Half-completed refactors
- A symbol renamed in some call sites but not others
- A new abstraction introduced while the old path it replaces still exists and is still used
- A signature changed with callers left on the old shape
- Both the old and new implementation present, with no indication which is authoritative
- A migration started across files and applied to only some of them

### 2. Orphaned and dead
- Imports of symbols that no longer exist, or that exist but are now unused
- Exported symbols no longer referenced anywhere in the supplied context
- Files that appear superseded but were not deleted
- Feature flags, TODOs, or commented-out blocks introduced by this diff
- Config keys, env vars, or translation keys referenced in code but not added, or added and never referenced

### 3. Breaking changes to consumers
- Public API, exported types, or component props changed or removed
- Response shape or status codes changed on an existing endpoint
- A required field added to an existing input type with no default
- Enum member or constant removed
- Event, message, or queue payload shape changed without a version bump
- Peer or downstream packages relying on the changed surface

### 4. Companion changes that should be here and are not
- Schema change with no migration file
- New env var with no `.env.example` entry and no documentation
- New dependency in code with no manifest entry
- New route or command with no registration in its index, router, or module
- Behavior change contradicting documentation in the supplied files
- New plugin agent or skill with no registry entry, where the project has a registry

### 5. Consistency with the codebase
- New file placed outside the established directory convention
- Naming that departs from surrounding conventions
- A pattern reimplemented where an existing shared utility in the supplied context already does it
```

This agent's `## Purpose` must name its lens as **Integration**, since the injected contract tells it to head its report `## Integration Review`. Add an `## Additional Rules` section:

```markdown
- You only see the supplied files. Before reporting an orphaned or missing reference, check whether the evidence is actually in your context — if a symbol might be referenced in a file you were not given, mark it "needs verification" and cap at Medium.
```

- [ ] **Step 6: Verify frontmatter across the five files**

```bash
cd /home/ronmrcdo/Projects/vibe/claude-plugins
for f in plugins/code-reviewer/agents/pr-*.md; do
  echo "== $f"; sed -n '1,7p' "$f"; done
```

Expected: five files, each opening with `---`, each declaring `name:` matching its filename, `model: sonnet`, and `tools: Read, Grep, Glob`.

- [ ] **Step 7: Live run against a backend-only PR**

Restart Claude Code so the new agents register, then invoke the skill with a real backend PR URL.

Expected:
- Exactly 5 agents dispatch — no frontend, a11y, data-layer, or infra
- Each returns a report in the shared format
- The unified report renders with a verdict, and the verdict matches the severity counts in the Summary table
- The closing line "This is a recommendation only — nothing was posted to the PR." is present

Then confirm nothing was mutated:

```bash
git status --porcelain && git worktree list
```

Expected: no unexpected working-tree changes, and exactly one worktree.

---

## Phase 3: The four stack-conditional agents

**Files:**
- Create: `plugins/code-reviewer/agents/pr-frontend.md`
- Create: `plugins/code-reviewer/agents/pr-accessibility.md`
- Create: `plugins/code-reviewer/agents/pr-data-layer.md`
- Create: `plugins/code-reviewer/agents/pr-infra.md`

**Consumes:** the Input Contract block and Shared Agent Report Block used in Phase 2.
**Produces:** subagent types `code-reviewer:pr-frontend`, `code-reviewer:pr-accessibility`, `code-reviewer:pr-data-layer`, `code-reviewer:pr-infra`.

Same file shape as Phase 2: frontmatter, `# Title`, `## Purpose`, `## What You Analyze`, and
`## Additional Rules`. No Input Contract, Report Format, or Rules section — those are inlined
from `references/agent-report-contract.md` at dispatch time.

- [ ] **Step 1: Create `pr-frontend.md`**

Frontmatter:

```yaml
---
name: pr-frontend
description: Reviews frontend changes in a pull request for component design, state management, effect correctness, rendering behavior, and bundle impact. Dispatched only when frontend files or dependencies are detected.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: is this frontend code well-built? Component boundaries, state placement, effect correctness, data fetching, and what it costs to ship. Accessibility is a separate agent's job — do not duplicate it.*

`## What You Analyze`:

```markdown
### 1. Component design
- A component doing several unrelated jobs — extract by responsibility, not by line count
- Boolean prop proliferation (`isCompact`, `isInline`, `hasIcon`) where composition would be clearer
- Prop drilling more than two levels deep
- Logic duplicated across components that belongs in a shared hook or utility
- Presentation and data fetching fused where they could be separated

### 2. State
- Derived values held in state instead of computed during render
- State duplicated from props, then drifting out of sync
- State living above the components that use it, widening re-render scope
- Server data held in local state where the project already uses a data-fetching library
- Complex related state as separate `useState` calls where a reducer would keep transitions valid

### 3. Effects
- Effects doing work that belongs in an event handler
- Missing, incomplete, or deliberately suppressed dependency arrays
- Missing cleanup: subscriptions, listeners, timers, in-flight requests
- Effects that set state derivable during render, causing a second render pass
- Effect chains where one effect's state update triggers the next

### 4. Data fetching and boundaries
- Waterfalls: dependent requests that could run in parallel
- No loading, error, or empty state for an async surface
- Missing error boundary around a subtree that can throw
- Client component doing work that belongs on the server (React Server Components / Next.js App Router)
- `'use client'` placed higher in the tree than necessary, pulling children into the client bundle

### 5. Bundle and loading
- Heavy dependency imported eagerly for a rarely-visited surface
- Whole-library imports where the library supports named or per-path imports
- Route or modal not code-split where the project splits comparably heavy surfaces
- Large assets imported into the bundle rather than served
- New dependency duplicating something already in the manifest

### 6. Framework idioms
- React: keys from array index on a reorderable list; state mutated rather than replaced; conditional hook calls
- Next.js: client-side fetch where a server component would do; missing `next/image` where the project uses it; unnecessary `'use client'`
- Vue: reactivity lost by destructuring a reactive object; `v-if` and `v-for` on the same element
- Svelte: reactive statements with side effects; stores subscribed without cleanup
```

This agent's `## Purpose` must name its lens as **Frontend**, since the injected contract tells it to head its report `## Frontend Review`. Add an `## Additional Rules` section:

```markdown
- Accessibility belongs to `pr-accessibility`. Do not report ARIA, contrast, focus, or semantic-HTML findings here.
```

- [ ] **Step 2: Create `pr-accessibility.md`**

Frontmatter:

```yaml
---
name: pr-accessibility
description: Reviews frontend changes in a pull request against WCAG 2.1 Level AA, ARIA authoring practices, semantic HTML, keyboard operability, and screen reader behavior. Dispatched only when frontend files or dependencies are detected.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: can everyone use this? You check the changed markup and interaction against WCAG 2.1 Level AA and the ARIA Authoring Practices. This is conformance work — cite the success criterion.*

`## What You Analyze`:

```markdown
### 1. Semantic structure
- `<div>` or `<span>` carrying click handlers where `<button>` or `<a>` is correct
- Heading levels skipped, or headings used for visual size
- Lists, tables, and forms built from generic elements
- Landmarks missing or duplicated without distinguishing labels
- Data tables without `<th>`, `scope`, or a caption
- `<a>` used for actions, `<button>` used for navigation

### 2. Keyboard operability (WCAG 2.1.1, 2.1.2, 2.4.3, 2.4.7)
- Interactive elements unreachable by Tab, or reachable but not activatable by Enter/Space
- Positive `tabIndex` values distorting tab order
- Focus not moved into a dialog, drawer, or menu when it opens
- Focus not restored to the trigger on close
- No focus trap in a modal dialog
- Focus indicator removed (`outline: none`) with no visible replacement
- Keyboard trap: focus enters a region and cannot leave

### 3. Names, roles, and values (WCAG 4.1.2)
- Icon-only buttons with no accessible name
- Form inputs without an associated `<label>`, `aria-label`, or `aria-labelledby`
- Placeholder used as the only label
- `role` applied without the states and properties that role requires
- `aria-expanded`, `aria-selected`, `aria-checked` not updated when state changes
- `aria-hidden="true"` on an element that contains focusable content
- Redundant or conflicting ARIA on elements with native semantics

### 4. Images and media (WCAG 1.1.1, 1.2.x)
- `<img>` without `alt`; decorative images without `alt=""`
- Alt text restating the filename or beginning "image of"
- Informative SVG without `role="img"` and a `<title>`
- Video without captions; audio without a transcript
- Media that autoplays with sound

### 5. Color and visual (WCAG 1.4.3, 1.4.4, 1.4.11, 1.4.10)
- Text contrast below 4.5:1, or below 3:1 for large text — compute it from the supplied values
- UI component and focus indicator contrast below 3:1
- Color as the sole carrier of meaning — error states, status, required fields
- Layout that breaks at 200% zoom or at 320px width
- Text in images instead of real text

### 6. Forms and errors (WCAG 3.3.1, 3.3.2, 3.3.3)
- Validation errors not programmatically associated with their field (`aria-describedby`, `aria-invalid`)
- Errors announced only visually — no live region
- Required fields indicated only by color or a bare asterisk
- Error text that does not say how to fix the problem
- Inputs missing `autocomplete` where WCAG 1.3.5 applies

### 7. Dynamic content (WCAG 4.1.3)
- Content appearing or updating with no live region announcement
- `aria-live` on a container that mounts at the same time as its content
- Toasts and alerts that vanish before they can be read
- Route changes that do not move or announce focus
- Motion and animation with no `prefers-reduced-motion` respect (WCAG 2.3.3)
```

This agent's `## Purpose` must name its lens as **Accessibility**, since the injected contract tells it to head its report `## Accessibility Review`. Add an `## Additional Rules` section:

```markdown
- Cite the WCAG 2.1 success criterion number and level for every finding (e.g. "1.4.3 Contrast (Minimum), AA").
- A failure of a Level A or AA criterion on an interactive control is High. A Level AAA suggestion is Low.
- Do not report component performance or bundle concerns — those belong to `pr-frontend`.
```

- [ ] **Step 3: Create `pr-data-layer.md`**

Frontmatter:

```yaml
---
name: pr-data-layer
description: Reviews database changes in a pull request — query construction, indexing, schema and migration safety, transaction boundaries, and ORM usage. Dispatched only when SQL, migrations, or ORM dependencies are detected.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: is the data layer sound? Query correctness and cost, schema evolution safety, and transaction boundaries. Migrations are the highest-stakes part of most PRs because they are hard to undo.*

`## What You Analyze`:

```markdown
### 1. Migration safety
- A migration that locks a large table: adding a non-null column with a default, adding an index without `CONCURRENTLY`, changing a column type in place
- Destructive operations — dropped columns or tables — shipped in the same release as the code that stops using them, rather than after it
- No down migration, or a down migration that loses data
- Data backfill inside a schema migration, unbounded and unbatched
- A migration assuming an empty or small table
- Renames that break the running old version during a rolling deploy

### 2. Schema design
- A new foreign key column with no index
- Missing `NOT NULL` where the application always supplies a value; missing unique constraints where uniqueness is assumed
- Wrong type: string for money, `timestamp` without timezone, integer for a value that will exceed its range
- Denormalization introduced without a stated reason
- No cascade or restrict behavior specified on a new foreign key

### 3. Query construction and cost
- Filtering, joining, or sorting on unindexed columns
- Index added that duplicates an existing index's prefix
- `SELECT *` where specific columns are used
- Queries in a loop — N+1 — where a single query with `IN` or a join would do
- Missing eager loading where relations are accessed afterward
- Unbounded result sets: no `LIMIT`, no pagination
- `OFFSET`-based pagination on a large table where keyset pagination is warranted
- Leading-wildcard `LIKE '%…'` on a large table

### 4. Transactions and consistency
- Multi-statement writes that must be atomic and are not in a transaction
- Network calls, queue publishes, or emails inside a transaction, extending lock duration
- Read-modify-write without a lock or optimistic concurrency check
- Transaction scope wider than needed, holding locks across slow work
- Retry logic on a non-idempotent operation

### 5. ORM usage
- Raw queries with string interpolation instead of parameter binding
- Lazy loading triggered inside a serialization loop
- `save()` in a loop rather than a bulk operation
- Model-level hooks introducing hidden queries
- Connection pool exhaustion: connections acquired and not released, or pool size unchanged after adding a hot path
```

This agent's `## Purpose` must name its lens as **Data Layer**, since the injected contract tells it to head its report `## Data Layer Review`. Add an `## Additional Rules` section:

```markdown
- Rate migration risk by table size in production where you can infer it; if you cannot, say so and cap at Medium.
- A destructive or table-locking migration with no stated rollout plan is High at minimum.
```

- [ ] **Step 4: Create `pr-infra.md`**

Frontmatter:

```yaml
---
name: pr-infra
description: Reviews infrastructure changes in a pull request — Dockerfiles, CI workflows, Terraform, and Kubernetes manifests — for security, reproducibility, and correctness. Dispatched only when infrastructure files are detected.
model: sonnet
tools: Read, Grep, Glob
---
```

Purpose: *You hold one lens: is this infrastructure change safe and reproducible? Infra defects fail at deploy time or leak credentials, so secrets handling and permission scope get the most weight.*

`## What You Analyze`:

```markdown
### 1. Secrets and credentials
- Credentials, tokens, or keys literal in a Dockerfile, workflow, manifest, or `.tf` file — always Critical
- Secrets passed as build args or `ENV`, which persist in image layers
- Secrets echoed into logs, or a workflow step printing its environment
- Terraform state or `.tfvars` containing secrets committed to the repo
- Kubernetes `Secret` with base64 content committed as plaintext-equivalent

### 2. CI workflow security
- `pull_request_target` combined with a checkout of the PR head — arbitrary code with write-scoped secrets
- Actions pinned to a mutable tag or branch rather than a commit SHA
- `permissions:` unset or `write-all` where read is sufficient
- Untrusted input (`github.event.issue.title`, PR body) interpolated into a `run:` block — script injection
- Self-hosted runners on a public repository's PR triggers

### 3. Container images
- Base image with a mutable or absent tag (`latest`) instead of a digest or pinned version
- Running as root — no `USER` directive
- Package installs without a version pin, or without cleaning the package cache in the same layer
- Build context copying the whole repo — missing or incomplete `.dockerignore`
- Secrets or dev dependencies present in the final stage where a multi-stage build would drop them
- No `HEALTHCHECK` where the orchestrator relies on one

### 4. Kubernetes and Terraform
- Containers without resource requests and limits
- No liveness or readiness probe on a long-running service
- `hostNetwork`, `privileged`, or `allowPrivilegeEscalation` enabled
- Overly broad IAM: wildcard actions or resources in a policy
- Security group or firewall rule open to `0.0.0.0/0` on a non-public port
- Storage or database resources without encryption at rest, backups, or deletion protection
- Terraform resources that force replacement of stateful infrastructure on apply

### 5. Reproducibility and rollout
- Non-deterministic builds: unpinned dependencies, clock- or network-dependent build steps
- Deployment strategy that drops traffic — no rolling update, no `maxUnavailable`
- Environment-specific values hardcoded rather than parameterized
- A change to one environment's config with no equivalent in the others, where the repo keeps them in parallel
```

This agent's `## Purpose` must name its lens as **Infrastructure**, since the injected contract tells it to head its report `## Infrastructure Review`. Add an `## Additional Rules` section:

```markdown
- Any secret material present in the diff is Critical, regardless of the environment it targets.
- Distinguish "insecure" from "not hardened": an open security group on a public load balancer is expected; on a database subnet it is Critical.
```

- [ ] **Step 5: Verify all nine agent files**

```bash
cd /home/ronmrcdo/Projects/vibe/claude-plugins
ls plugins/code-reviewer/agents/pr-*.md | wc -l
grep -L "tools: Read, Grep, Glob" plugins/code-reviewer/agents/pr-*.md
grep -L "model: sonnet" plugins/code-reviewer/agents/pr-*.md
```

Expected: count is `9`; both `grep -L` calls print nothing (every file declares both keys).

- [ ] **Step 6: Live run against a full-stack PR**

Restart Claude Code so the four new agents register. Invoke the skill with a PR that touches frontend, backend, database, and CI.

Expected:
- 9 agents dispatch in one parallel batch
- `pr-frontend` and `pr-accessibility` report on disjoint concerns — no duplicated a11y findings in the frontend section
- Deduplication works: an issue both `pr-security` and `pr-correctness` flag appears once, at the higher severity
- The verdict matches the Summary table counts

Then run the same skill against a docs-only PR. Expected: 5 agents dispatch, `pr-test-coverage` reports no findings rather than demanding tests, and the verdict is APPROVE.

---

## Phase 4: Retire `/review-pr` and update the registry

**Files:**
- Delete: `plugins/commit-commands/commands/review-pr.md`
- Modify: `README.md`
- Modify: `.claude-plugin/marketplace.json`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Delete the old command**

```bash
cd /home/ronmrcdo/Projects/vibe/claude-plugins
rm plugins/commit-commands/commands/review-pr.md
```

- [ ] **Step 2: Update `marketplace.json`**

In the `commit-commands` entry, replace the `description` value:

```
"Workflow for git including commit, push, pr and a specialized command to review PR"
```

with:

```
"Workflow for git including commit, push, PR creation, branch cleanup, and daily standup summaries"
```

In the `code-reviewer` entry, replace the `description` value:

```
"Collection of agents specialized in reviewing the code."
```

with:

```
"Multi-agent code review for local changes and GitHub pull requests"
```

and add `"pull-request"` and `"github"` to that entry's `keywords` array.

- [ ] **Step 3: Update `README.md`**

In the **commit-commands** table, delete the row:

```
| `/review-pr <PR_NUMBER>` | Review a GitHub PR for security, anti-patterns, and best practices |
```

Confirm the remaining rows cover `/commit-push-pr`, `/commit-push`, `/clean-branches`, and
`/daily-standup` — the table is currently missing the latter two.

In the **code-reviewer** section, add below the existing command table:

```markdown
| Skill | Description |
|---|---|
| `review-github-pr` | Paste a GitHub PR URL for a stack-aware multi-agent review and an approve / request-changes recommendation |

`review-github-pr` fires when you paste a GitHub pull request URL — no slash command needed. It
reads the PR entirely through `gh` (no clone, fetch, or worktree), detects the tech stack from
changed files and manifest dependencies, and dispatches 5–9 specialized agents in parallel:

**Always:** `pr-correctness`, `pr-security`, `pr-performance`, `pr-test-coverage`, `pr-integration`
**Stack-conditional:** `pr-frontend`, `pr-accessibility` (frontend), `pr-data-layer` (SQL/ORM), `pr-infra` (Docker/CI/Terraform/k8s)

The verdict is deterministic — any Critical or High finding means Request Changes. The review is
read-only: nothing is ever posted to the PR.
```

- [ ] **Step 4: Update `CLAUDE.md`**

In the repository structure block, replace the `commit-commands` line and the `code-reviewer`
block so they read:

```
plugins/
  commit-commands/commands/        # Git workflow commands (commit-push-pr, commit-push, clean-branches, daily-standup)
  code-reviewer/
    commands/review-unstaged.md    # Orchestrator that spawns 7 parallel review agents
    skills/review-github-pr/       # PR-URL-triggered review: gh-only fetch, stack-aware dispatch, verdict
    agents/                        # 16 agents: 7 staged-changes reviewers + 9 pr-* PR reviewers
```

Then add this subsection after the existing **Multi-Agent Code Review Pattern** section:

```markdown
### GitHub PR Review Pattern

`review-github-pr` is a skill, not a command, so pasting a PR URL is enough to trigger it. It:

1. Parses `owner`/`repo`/`number` from the URL path — never infers the repo from the working directory
2. Fetches metadata, patch, and full file contents at the head SHA entirely through `gh` — no git, no worktree, no local mutation
3. Detects the stack by unioning changed-file extensions with manifest dependencies
4. Dispatches 5–9 `pr-*` agents in one parallel batch, addressed by subagent type
5. Compiles a deduplicated report and applies a deterministic verdict: any Critical or High finding means Request Changes

The review is read-only in both directions — it never posts to the PR and never writes to disk.
The `pr-*` agents declare `tools: Read, Grep, Glob` because dispatched agents otherwise inherit
write tools regardless of the dispatching skill's `allowed-tools`.
```

- [ ] **Step 5: Verify the registry and the removal**

```bash
cd /home/ronmrcdo/Projects/vibe/claude-plugins
jq -e . .claude-plugin/marketplace.json > /dev/null && echo "marketplace.json parses"
jq -r '.plugins[].source' .claude-plugin/marketplace.json | while read -r p; do
  [ -d "$p" ] && echo "OK  $p" || echo "MISSING  $p"; done
grep -rn "review-pr" --include=*.md --include=*.json . | grep -v docs/superpowers
```

Expected: `marketplace.json` parses; all three plugin sources resolve; the final `grep` returns
**no hits** — `review-github-pr` does not match `review-pr` as a whole word, and the only
remaining mentions are in the spec and this plan, both excluded.

Note: if the grep does surface `review-github-pr` lines, that is a substring match, not a leak —
confirm by eye that no line refers to the deleted `/review-pr` command.

- [ ] **Step 6: Full end-to-end verification**

Restart Claude Code so the deletion and the new files both take effect.

1. Confirm `/review-pr` is gone — it should not appear in the command list.
2. Paste a GitHub PR URL with no other instruction. Expected: `review-github-pr` fires on the
   paste alone, without a slash command.
3. Confirm the report includes the Coverage notes section and the closing line
   "This is a recommendation only — nothing was posted to the PR."
4. Confirm the read-only guarantee held:

```bash
git status --porcelain && git worktree list && git branch --list 'pr-review-*'
```

Expected: no unexpected working-tree changes, exactly one worktree, no `pr-review-*` branches.

5. Confirm nothing was posted:

```bash
gh pr view <number> --repo <owner>/<repo> --json reviews,comments \
  --jq '{reviews: (.reviews|length), comments: (.comments|length)}'
```

Expected: counts unchanged from before the run.
