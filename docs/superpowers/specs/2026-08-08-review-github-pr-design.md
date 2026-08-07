# Design — `review-github-pr` skill

**Date:** 2026-08-08
**Status:** Approved for planning
**Plugin:** `code-reviewer`

## Problem

Reviewing a GitHub PR today means running `/review-pr <number>`, which fetches the PR
head into a throwaway git worktree and delegates to `superpowers:requesting-code-review`.
That approach has three costs:

1. It mutates the local repo — `git fetch`, `git worktree add`, a temporary branch — and
   depends on a cleanup step that must run on every exit path, including cancellation.
2. It only works for PRs in the repo you are standing in.
3. It runs one generalist reviewer, so the depth of the review does not adapt to what the
   PR actually touches.

We want to paste a PR URL and get a multi-agent review back, with a merge recommendation,
without touching the local working tree at all.

## Goals

- Paste a GitHub PR URL; get a unified review report and an Approve / Request Changes call.
- Work on any PR the user can see with `gh`, regardless of the current directory.
- Dispatch review agents selected by the PR's detected tech stack.
- Never mutate anything: not the local repo, not the PR.

## Non-goals

- Posting the review to GitHub. The skill never comments, approves, requests changes, merges,
  or edits the PR. Output goes to the user only.
- Reviewing local uncommitted work — `/review-unstaged` already covers that and is unchanged.
- Reviewing a PR by bare number. The URL is the input; supporting bare numbers would
  reintroduce the repo-relative coupling this design removes.

## Constraints

**No `git` whatsoever.** No `fetch`, `pull`, `checkout`, `worktree`, or `clone`. All PR data
arrives over the GitHub API via `gh`. This is the constraint that shapes the whole design:
reviewers never get a working tree, so every byte they see must be fetched explicitly.

**Read-only.** Enumerated in "Read-only guarantee" below.

---

## Packaging

A skill, not a command, for two reasons:

- **Trigger.** The requirement is that pasting a URL is enough. A slash command only fires when
  typed; a skill's description matches the paste itself.
- **Structure.** Nine agent specs plus a report template plus a detection table is too much for
  one flat command file. A skill is a directory and can hold reference files loaded on demand.

```
plugins/code-reviewer/
  skills/review-github-pr/
    SKILL.md                        # orchestrator
    references/stack-detection.md   # extension + dependency → agent mapping
    references/severity-rubric.md   # shared severity definitions (injected into agents)
    references/report-format.md     # unified report template + verdict rule
  agents/
    pr-correctness.md               # 5 always-on
    pr-security.md
    pr-performance.md
    pr-test-coverage.md
    pr-integration.md
    pr-frontend.md                  # 4 stack-conditional
    pr-accessibility.md
    pr-data-layer.md
    pr-infra.md
```

No companion slash command. A second entry point to the same logic is duplication, and the
skill remains directly invocable by name (`code-reviewer:review-github-pr`) when the user
wants to force it.

### SKILL.md frontmatter

```yaml
name: review-github-pr
description: >
  Review a GitHub pull request from its URL using stack-aware parallel agents and return an
  approve / request-changes recommendation. Use when the user pastes a GitHub pull request
  URL, or asks to review, check, or assess a PR. Read-only — never comments on, approves, or
  modifies the PR, and never fetches or checks out code locally.
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api:*), Read, Glob, Grep, Task
model: sonnet
```

---

## Data flow

### Step 1 — Parse the URL

Extract `owner`, `repo`, `number` by matching the URL path against:

```
/(?<owner>[^/]+)/(?<repo>[^/]+)/pull/(?<number>\d+)
```

Path-only matching means trailing segments (`/files`, `/commits`, `/checks`), fragments
(`#discussion_r123`), and query strings are all tolerated, and GitHub Enterprise hosts work
unchanged since `gh` resolves the host from its own config.

If no match: stop and print usage —
`Paste a GitHub PR URL, e.g. https://github.com/owner/repo/pull/123`. Do not guess a repo
from the current directory.

### Step 2 — Fetch metadata

```bash
gh pr view <number> --repo <owner>/<repo> --json \
  number,title,body,author,state,isDraft,baseRefName,headRefName,headRefOid,\
additions,deletions,changedFiles,files,labels
```

On failure (PR not found, `gh` not authenticated, no access), stop and surface the error
verbatim. Nothing has been created, so there is nothing to unwind.

`state` and `isDraft` are informational, never fatal. A draft, merged, or closed PR is still
reviewed; the state is stated at the top of the report and, for merged/closed PRs, the verdict
is labelled advisory.

### Step 3 — Fetch the diff

```bash
gh pr diff <number> --repo <owner>/<repo> --patch
```

### Step 4 — Fetch full file contents at the head SHA

The diff alone shows changed hunks with three lines of context, which is too little to judge
whether a function is correct. For each changed file, fetch the whole file as it exists in the PR:

```bash
gh api "repos/<owner>/<repo>/contents/<url-encoded-path>?ref=<headRefOid>" \
  -H "Accept: application/vnd.github.raw"
```

**Skipped — diff only, no content fetch:**

- Deleted files (no content at head).
- Lockfiles: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb`, `Cargo.lock`,
  `poetry.lock`, `Gemfile.lock`, `composer.lock`, `go.sum`.
- Generated or vendored: `dist/`, `build/`, `vendor/`, `node_modules/`, `*.min.js`,
  `*.min.css`, `*.map`, `*.snap`, `*.pb.go`, `*_pb2.py`.
- Binaries: images, fonts, archives, PDFs.
- Files over 500 KB.

For renamed files, fetch at the new path. If a fetch fails for any reason — including the
GitHub 1 MB limit on the contents endpoint — fall back to diff-only for that file and record it.

**Size guard.** Above 60 changed files or 3000 changed lines, full-content fetching is limited
to the highest-signal files, ranked by changed-line count with application source ahead of
tests and config. The report then names every file that got diff-only treatment. Silent
truncation is the failure mode to avoid here: a report that quietly reviewed 40 of 120 files
reads exactly like one that reviewed all 120.

### Step 5 — Detect the tech stack

Two independent signals, unioned:

- **Changed-file extensions**, from the metadata `files` list.
- **Manifest dependencies**, fetched the same way as Step 4 at the head SHA:
  `package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `Cargo.toml`,
  `composer.json`, `Gemfile`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `*.csproj`,
  `Package.swift`, `pubspec.yaml`.

Manifest fetches are best-effort — a 404 means the project is not that stack, not an error.
Dependencies matter because extensions lie: a `.ts` file is backend or frontend depending on
whether the project depends on `next` or `@nestjs/core`.

The full mapping lives in `references/stack-detection.md`.

### Step 6 — Dispatch agents

All applicable agents dispatch in a single parallel batch via the Task tool, addressed by their
real subagent type (`code-reviewer:pr-security`, etc.) so each loads its own specification
rather than being told to impersonate one.

Every agent receives the same payload:

- PR metadata: number, title, author, base ← head, state, additions/deletions/changed files.
- The PR body, as the statement of intent to judge the diff against. If the body is empty, say
  so and have the agent judge against the changes' apparent intent.
- The full patch.
- Full contents of every fetched file.
- The detected stack.
- The list of diff-only files, so agents can scope confidence accordingly.
- The severity rubric from `references/severity-rubric.md`, injected inline.

The rubric is injected rather than referenced by path because the verdict is computed from
severities. If two agents apply different thresholds, the verdict is meaningless.

### Step 7 — Compile and rule

Deduplicate overlapping findings across agents, then emit one unified report and the verdict.

---

## Agent roster

### Always dispatched (5)

| Agent | Lens |
|---|---|
| `pr-correctness` | Logic bugs, edge cases, regressions; whether the diff does what the PR body claims |
| `pr-security` | Injection, authn/authz, secrets, data exposure, OWASP-class issues in the changed code |
| `pr-performance` | Algorithmic complexity, N+1 queries, re-render storms, memory retention, blocking I/O |
| `pr-test-coverage` | Whether changed paths are tested and whether the assertions are meaningful |
| `pr-integration` | Half-landed refactors, orphaned references, breaking changes for consumers, config or migration left behind |

`pr-correctness` and `pr-integration` stay separate despite overlapping scope. A PR can be
logically correct and structurally half-landed, and a single agent holding both lenses reliably
reports the louder one only.

### Stack-conditional (4)

| Agent | Fires when |
|---|---|
| `pr-frontend` | `.tsx`, `.jsx`, `.vue`, `.svelte`, `.html`, `.css`, `.scss` changed, **or** `react`, `next`, `vue`, `svelte`, `@angular/core` in dependencies |
| `pr-accessibility` | Same trigger as `pr-frontend` |
| `pr-data-layer` | `.sql` or migration directories changed, **or** `prisma`, `typeorm`, `sequelize`, `mongoose`, `drizzle-orm`, `sqlalchemy`, `gorm` in dependencies |
| `pr-infra` | `Dockerfile`, `docker-compose.y*ml`, `*.tf`, k8s manifests, or `.github/workflows/*` changed |

Range: 5 agents on a pure backend PR, 9 on a full-stack PR touching the database and CI.

Accessibility is separate from `pr-frontend` because it is conformance work against an external
spec (WCAG 2.1 AA, ARIA authoring practices) rather than reasoning about how code executes.
Merged into a performance-and-structure agent, it consistently degrades to a token mention.

### Agent specification shape

Each agent file carries `name`, `description`, `model: sonnet` frontmatter, matching the
existing convention, plus `tools: Read, Grep, Glob` — agents default to every tool, including
write tools, so the read-only guarantee has to be re-asserted at the agent layer rather than
assumed from the skill's `allowed-tools`. The agents receive their entire input in the dispatch
prompt and have no working tree to inspect, so read tools are the most they can justify.

The file sections are:

1. **Purpose** — the one lens this agent holds.
2. **Input contract** — the payload described in Step 6, and an explicit note that no working
   tree exists: the agent cannot run code, cannot open files not supplied, and must not attempt to.
3. **What you analyze** — the checklist, stack-specific where relevant.
4. **Report format** — findings grouped by severity, each with `file:line`, the concrete failure
   it causes, and the fix.
5. **Rules** — cite `file:line` for every finding; give the fix, not a description of the fix;
   scope to the PR's changes rather than the whole codebase; mark anything unverifiable from the
   supplied context as "needs verification".

The nine new agents are prefixed `pr-` so they never collide with the seven existing
staged-changes agents, which stay exactly as they are and continue to serve `/review-unstaged`.
The plugin ends up shipping 16 agents.

---

## Severity rubric

Defined once in `references/severity-rubric.md` and injected into every agent dispatch.

| Severity | Meaning |
|---|---|
| **Critical** | Exploitable vulnerability, data loss or corruption, breaks production on merge, or silently wrong results on a common path |
| **High** | A bug real users will hit on a reachable path; missing authorization on a non-trivial surface; a breaking change to a consumed contract; no tests for new business-critical logic |
| **Medium** | Degraded behavior in edge cases, notable performance regression, weak error handling, meaningful test gap, WCAG AA violation on a non-critical control |
| **Low** | Style, naming, minor duplication, hardening suggestions, nits |

**Anti-inflation rules**, which matter because Critical and High are auto-blocking:

- If you cannot name a concrete trigger — specific inputs or state that produce the failure —
  cap the finding at Medium.
- If the finding cannot be verified from the supplied context, mark it "needs verification" and
  cap it at Medium.
- Rate on exploitability and blast radius, not on which category the issue belongs to.

---

## Output

One unified report, deduplicated across agents — not nine reports stapled together.

```
# PR Review — #123: <title>
<owner>/<repo> · @author · base ← head · +420/−87 across 12 files · OPEN
Stack: Next.js + NestJS + Postgres          Agents dispatched: 8

## Verdict: REQUEST CHANGES
2 Critical, 1 High blocking.

### Blocking findings
| # | Sev | Category | file:line | Issue | Fix |

### Suggestions (non-blocking)
| # | Sev | Category | file:line | Suggestion |

### Findings by category
<one section per dispatched agent; "No issues found" when clean>

### Summary
| Category | Critical | High | Medium | Low |

### Strengths
<positive patterns worth keeping>

### Coverage notes
<files reviewed diff-only, agents that failed to return>
```

### Verdict rule

Stated in SKILL.md so the call is auditable rather than vibes:

```
Critical > 0 or High > 0   →  REQUEST CHANGES
only Medium / Low          →  APPROVE (with comments)
no findings                →  APPROVE
```

Every blocking finding appears in the Blocking table with its `file:line`, so the verdict always
traces to the specific findings that produced it.

**Edge cases:**

- If any agent fails or returns nothing, note it in Coverage notes and label the verdict
  **provisional**.
- If the size guard reduced any file to diff-only, note it and label the verdict **partial**.
- For a merged or closed PR, label the verdict **advisory**.

The verdict is a recommendation to the user. The skill does not act on it.

---

## Read-only guarantee

Enforced at two layers, because the skill's `allowed-tools` does not constrain the agents it
dispatches: the skill scopes itself, and each `pr-*` agent scopes itself via `tools:` frontmatter
(see "Agent specification shape").

The skill's `allowed-tools` excludes every write tool. Because `Bash(gh api:*)` can technically
issue writes, SKILL.md additionally forbids, by name:

- `gh pr review`, `gh pr comment`, `gh pr merge`, `gh pr close`, `gh pr edit`, `gh pr ready`
- Any `gh api` invocation with `-X POST`, `-X PATCH`, `-X PUT`, `-X DELETE`, `--method` with a
  non-GET verb, or `-f`/`--field` (which implies POST)
- Any `git` subcommand, in particular `fetch`, `pull`, `clone`, `checkout`, `worktree`
- Any file modification

The report is returned to the user in the terminal. Nothing is posted anywhere.

---

## Removing `/review-pr`

| File | Change |
|---|---|
| `plugins/commit-commands/commands/review-pr.md` | Delete |
| `README.md` | Drop the `/review-pr` row from the commit-commands table; add `review-github-pr` to the code-reviewer section |
| `.claude-plugin/marketplace.json` | `commit-commands` description no longer advertises PR review; `code-reviewer` description gains it |
| `CLAUDE.md` | Update the repo-structure block and the multi-agent section to cover the new skill and agents |

The `superpowers:requesting-code-review` delegation disappears with the file. Nothing else in
the repo references `/review-pr`.

---

## Verification

There is no build or test framework — the repo is markdown specs plus one JSON file — so
verification is structural plus one live run:

1. `jq . .claude-plugin/marketplace.json` parses, and every `source` path exists.
2. Every new agent and skill file has valid YAML frontmatter with the required keys, and every
   `pr-*` agent declares a read-only `tools:` list.
3. No file in the repo still references `/review-pr`
   (`grep -rn "review-pr" --include=*.md --include=*.json .` returns only intentional hits).
4. Live run against a real PR URL confirms: the URL parses, `gh` calls succeed, the correct
   agent set fires for the detected stack, the report renders, and the verdict matches the
   severity counts.
5. Live run against a frontend-only PR and a backend-only PR confirms conditional dispatch
   actually varies — 9 agents versus 5.
6. Confirm no local mutation: `git status` is unchanged and `git worktree list` shows only the
   main worktree after a run.

## Open questions

None.
