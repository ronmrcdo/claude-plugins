---
name: review-staged
description: Run a comprehensive code review on staged git changes using multiple specialized agents (performance, QA, structure, best practices, security, accessibility, code splitting)
allowed-tools: Bash(git diff:*), Bash(git status:*), Read, Glob, Grep, Task
---

You are a code review orchestrator. Run a comprehensive multi-agent review on the currently staged git changes.

## Process

### Step 1: Get Staged Changes

Run the following to identify what's staged:

```bash
git diff --cached --name-only
```

If no files are staged, tell the user:
> No staged changes found. Stage your changes with `git add` and try again.

Then stop.

### Step 2: Detect Tech Stack

Categorize the staged files:

- **Frontend files**: `.tsx`, `.jsx`, `.vue`, `.svelte`, `.html`, `.css`, `.scss`
- **Backend files**: `.ts`, `.js` (Node.js — when in controller/service/route/middleware directories or importing `express`, `@nestjs`, `fastify`, `koa`, `prisma`, `typeorm`, `sequelize`, `mongoose`), `.py`, `.go`, `.java`, `.rb`, `.rs`, `.php`, `.cs`
- **Shared/Config files**: `.json`, `.yaml`, `.yml`, `.env`, config files

Set a flag:
- `has_frontend` = true if any frontend files are staged
- `has_backend` = true if any backend files are staged

### Step 3: Get Full Diff

Get the staged diff for context:

```bash
git diff --cached
```

Read each staged file in full using the Read tool for complete context.

### Step 4: Run Review Agents

Launch all applicable agents in parallel using the Task tool. Pass the staged diff and file contents as context to each agent.

**Always run these agents (all stacks):**

1. **Performance Profiler** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the performance-profiler agent. Analyze the following staged changes for performance issues. [include diff and file contents]. Follow the performance-profiler agent specification.

2. **QA Specification Engineer** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the qa-spec-engineer agent. Analyze the following staged changes for test coverage gaps. [include diff and file contents]. Follow the qa-spec-engineer agent specification.

3. **Structural Completeness** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the structural-completeness agent. Analyze the following staged changes for structural integrity. [include diff and file contents]. Follow the structural-completeness agent specification.

4. **Best Practices** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the best-practices agent. Analyze the following staged changes against coding standards. [include diff and file contents]. Follow the best-practices agent specification.

5. **Security Reviewer** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the security-reviewer agent. Analyze the following staged changes for security vulnerabilities. [include diff and file contents]. Follow the security-reviewer agent specification.

**Only run when frontend files are staged (`has_frontend` = true):**

6. **Accessibility Reviewer** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the accessibility-reviewer agent. Analyze the following staged frontend changes for accessibility issues. [include diff and file contents]. Follow the accessibility-reviewer agent specification.

7. **Code Splitting & Reusability** — Launch with `subagent_type: "general-purpose"` and the prompt:
   > You are the code-splitting-reusability agent. Analyze the following staged frontend changes for modularity and reuse. [include diff and file contents]. Follow the code-splitting-reusability agent specification.

### Step 5: Compile Results

After all agents complete, compile their reports into a single unified review:

```
# Code Review — Staged Changes

**Files reviewed**: [count]
**Stack**: [Frontend | Backend | Full-stack]
**Date**: [date]
**Agents run**: [list of agents that ran]

---

## Performance
[performance-profiler results]

---

## Test Coverage & QA
[qa-spec-engineer results]

---

## Structural Completeness
[structural-completeness results]

---

## Best Practices
[best-practices results]

---

## Security
[security-reviewer results]

---

## Accessibility (Frontend)
[accessibility-reviewer results, or "Not applicable — no frontend files"]

---

## Code Splitting & Reusability (Frontend)
[code-splitting-reusability results, or "Not applicable — no frontend files"]

---

## Overall Summary

| Category | Critical | High/Should Fix | Advisory |
|----------|----------|-----------------|----------|
| Performance | X | X | X |
| QA Coverage | X | X | X |
| Structure | X | X | X |
| Best Practices | X | X | X |
| Security | X | X | X |
| Accessibility | X | X | X |
| Code Splitting | X | X | X |
| **Total** | **X** | **X** | **X** |

### Top Priority Actions
1. [Most critical finding across all reviews]
2. [Next]
3. [...]

### Strengths
- [Positive patterns observed across the change]
```

## Rules

- Run all applicable agents in parallel for speed
- Do not modify any files — this is a read-only review
- If a specific agent finds no issues, include its section with "No issues found"
- Deduplicate overlapping findings across agents (e.g., security and best practices may both flag the same issue)
- Prioritize the combined findings by impact across all categories
- The final output must be a single, unified report — not 7 separate reports
