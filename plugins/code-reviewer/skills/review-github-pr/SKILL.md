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
