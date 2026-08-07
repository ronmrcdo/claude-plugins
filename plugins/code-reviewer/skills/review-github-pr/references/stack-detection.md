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
