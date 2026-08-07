# Claude Code Plugins

A comprehensive collection of Claude Code plugins for git workflows, multi-agent code review, and accessibility compliance auditing.

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and configured
- Git installed (for commit-commands)
- [GitHub CLI (`gh`)](https://cli.github.com/) installed (for PR-related commands)

## Installation

### 1. Add the Plugin Marketplace

```bash
# Add from GitHub
/plugin marketplace add ronmrcdo/claude-plugins

# Or add locally for development
/plugin marketplace add /path/to/claude-plugins
```

### 2. Install Plugins

```bash
# Git workflows (commit, push, PR, branch cleanup)
/plugin install commit-commands

# Multi-agent code reviewer (7 specialized review agents)
/plugin install code-reviewer

# Accessibility compliance auditor (WCAG 2.1 Level AA)
/plugin install accessibility-compliance
```

## Plugins

### commit-commands

Git workflow automation with conventional commit enforcement.

| Command | Description |
|---|---|
| `/commit-push-pr` | Commit staged changes with conventional commits, push, and create a PR |
| `/commit-push` | Commit and push changes without creating a PR |
| `/clean-branches` | Delete local branches that are merged or have lost their remote tracking |
| `/daily-standup` | Generate a daily standup message from your GitHub PR activity |

**Commit format:** `type(scope): subject` where type is one of `feat`, `fix`, `refactor`, `docs`, `chore`, `test`, `style`, `perf`.

---

### code-reviewer

Multi-agent code review system that runs **7 specialized agents in parallel** against your unstaged changes.

| Command | Description |
|---|---|
| `/review-unstaged` | Run a comprehensive review on all unstaged git changes |

Agents launched on every review:

| Agent | Focus Area |
|---|---|
| Performance Profiler | Runtime inefficiencies, memory issues, algorithmic complexity |
| QA Spec Engineer | Test coverage gaps, missing edge cases, assertion quality |
| Structural Completeness | Import consistency, dead code, wiring & registration |
| Best Practices | SOLID principles, naming conventions, clean code patterns |
| Security Reviewer | OWASP Top 10, injection attacks, auth flaws, data exposure |

Additional agents launched when frontend files are detected (`.tsx`, `.jsx`, `.vue`, `.svelte`, `.html`):

| Agent | Focus Area |
|---|---|
| Accessibility Reviewer | WCAG 2.1 compliance, ARIA correctness, keyboard navigation |
| Code Splitting & Reusability | Component extraction, lazy loading, bundle optimization |

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

---

### accessibility-compliance

Standalone accessibility auditing against WCAG 2.1 Level AA.

| Command | Description |
|---|---|
| `/a11y-audit` | Audit specified files or components for accessibility violations |

Checks performed:
- Semantic HTML structure and heading hierarchy
- ARIA roles, states, and properties
- Keyboard accessibility and focus management
- Form labels, error messages, and validation
- Image alt attributes and media accessibility
- Framework-specific patterns (React, Vue, SPA)

## License

MIT
