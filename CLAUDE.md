# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code plugin marketplace** — a specification-driven collection of plugins that provide git workflow automation, multi-agent code review, and accessibility compliance auditing. There is no compiled source code; all behavior is defined in markdown specification files and a single JSON config.

## Repository Structure

```
.claude-plugin/marketplace.json    # Central plugin registry (3 plugins)
plugins/
  commit-commands/commands/        # Git workflow commands (commit-push-pr, commit-push, clean-branches, daily-standup)
  code-reviewer/
    commands/review-unstaged.md    # Orchestrator that spawns 7 parallel review agents
    skills/review-github-pr/       # PR-URL-triggered review: gh-only fetch, stack-aware dispatch, verdict
    agents/                        # 16 agents: 7 staged-changes reviewers + 9 pr-* PR reviewers
  a11y-compliance/
    commands/a11y-audit.md         # Standalone accessibility audit command
    agents/a11y-auditor.md         # WCAG 2.1/2.2 auditor agent
```

## Architecture

### Plugin System
- **marketplace.json** registers plugins with metadata; `pluginRoot` is `./plugins`
- Each plugin has a `commands/` directory (user-invocable skills) and optionally an `agents/` directory
- Commands and agents are markdown files with YAML front matter defining `name`, `description`, `allowed-tools`, and `model`

### Multi-Agent Code Review Pattern
`/review-unstaged` is the orchestrator command. It:
1. Detects tech stack from unstaged file extensions
2. Spawns 5 core agents in parallel (performance, qa, structure, best-practices, security)
3. Conditionally spawns 2 frontend agents (a11y, code-splitting) when `.tsx`, `.jsx`, `.vue`, `.svelte`, or `.html` files are detected
4. Compiles and deduplicates results into a unified report

All review agents use `model: sonnet` and follow a consistent output format: scope, severity-rated findings (Critical/High/Medium/Low), summary table with `file:line` references, and priority actions.

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

### Tool Scoping Convention
Commands declare minimal tool permissions in front matter:
- `Bash(git:*)`, `Bash(gh:*)` — scoped shell access
- `Read`, `Glob`, `Grep` — read-only file access
- `Task` — agent spawning (orchestrator commands only)

### File Conventions
- **Naming:** kebab-case for all files and directories
- **Commit format:** Conventional Commits — `type(scope): subject` with types: `feat`, `fix`, `refactor`, `docs`, `chore`, `test`, `style`, `perf`
- **Protected branches:** `main`, `master`, `beta`, `staging`, `feat/2.0` (in clean-branches)

## Adding New Content

- **New agent:** Create a `.md` file in the plugin's `agents/` directory with front matter (`name`, `description`, `model`)
- **New command:** Create a `.md` file in the plugin's `commands/` directory with front matter (`name`, `description`, `allowed-tools`)
- **New plugin:** Add a directory under `plugins/`, then register it in `marketplace.json`

## Prerequisites

- Claude Code CLI
- Git and GitHub CLI (`gh`) for commit-commands plugin
