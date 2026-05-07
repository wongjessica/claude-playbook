# CLAUDE.md for a Polyglot Monorepo

## Project shape

- One repo, multiple stacks
- Common shape: TypeScript frontend(s), Python or Go backend(s), shared packages, infrastructure-as-code, sometimes mobile
- Managed with Turborepo, Nx, pnpm workspaces, or just well-disciplined folders
- Different teams own different subtrees but everyone touches `packages/` shared code

This is the shape that breaks naive CLAUDE.md the hardest, because conventions vary by subtree.

## The CLAUDE.md (root level)

```markdown
# Project Context

This is the monorepo for [Company]. It contains:

- `apps/web` - the customer-facing Next.js app
- `apps/admin` - internal admin tool, also Next.js
- `apps/api` - the main backend service, Go
- `apps/worker` - background jobs, Python
- `apps/mobile` - React Native iOS/Android app
- `packages/` - shared TypeScript packages (UI components, types, utilities)
- `infra/` - Terraform for cloud infra
- `docs/` - architecture docs, runbooks, ADRs

Each app and package has its own `CLAUDE.md` with stack-specific rules. **Read the relevant one before starting work in that subtree.** This root file only covers cross-cutting concerns.

## Cross-cutting rules

### Code that crosses subtrees

- Shared types live in `packages/types`. If you find yourself copying a type between two apps, move it there.
- The TypeScript apps and the Go API exchange data via OpenAPI specs in `packages/api-spec`. Generate clients with `pnpm gen:api-client`. Don't hand-write fetch calls against the API.
- The Python worker reads from the same Postgres database as the Go API. Schema migrations are owned by the Go service. Never write migrations from the worker side.

### Branch and commit conventions

- Branch names: `<type>/<scope>/<short-desc>`, e.g. `feat/web/onboarding-flow`
- Conventional commits: `<type>(<scope>): <subject>`, e.g. `fix(api): retry logic for stripe webhooks`
- Scope must match a top-level app or package name (`web`, `admin`, `api`, `worker`, `mobile`, or a package name)
- One PR = one scope when possible. Cross-cutting changes are PRs labeled `cross-cutting` and require an extra reviewer.

### Tooling

- `pnpm` for Node-side work (do not use npm or yarn, lockfile differs)
- `go` 1.22+ for the API
- `uv` for Python (do not use pip directly, do not use poetry)
- `terraform` for infra, version pinned in `infra/.terraform-version`

### Running things

- All apps support `pnpm dev` from their directory
- Full stack locally: `make dev` from root (spins up API, worker, web with shared Postgres)
- Tests: `pnpm test` in any TypeScript directory, `make test` in Go directories, `uv run pytest` in Python directories
- The root has `make test-all` that runs everything; expect ~5 minutes

### Where to put new things

- New shared logic → does it have UI? `packages/ui`. Pure logic? `packages/utils`. Types only? `packages/types`.
- New API endpoint → `apps/api/internal/handlers/<domain>.go`, register in `apps/api/cmd/server/routes.go`
- New background job → `apps/worker/jobs/<job_name>.py`, register in `apps/worker/jobs/__init__.py`
- New frontend page → `apps/web/app/<route>/page.tsx` (App Router conventions, see `apps/web/CLAUDE.md`)
- New mobile screen → `apps/mobile/src/screens/<ScreenName>/`

## Critical: read the subtree CLAUDE.md

Before working in any subtree, read the `CLAUDE.md` there:

- `apps/web/CLAUDE.md` - Next.js conventions
- `apps/api/CLAUDE.md` - Go service conventions
- `apps/worker/CLAUDE.md` - Python worker conventions
- `apps/mobile/CLAUDE.md` - React Native conventions
- `packages/ui/CLAUDE.md` - design system rules

## Architecture decisions

Significant decisions live in `docs/adr/` as numbered ADRs. When making a non-trivial design choice that crosses subtrees, write an ADR or reference an existing one.

## What not to do

- Do not add a new top-level directory at the root. Everything fits in `apps/`, `packages/`, `infra/`, or `docs/`. If you think you need another, propose an ADR first.
- Do not introduce a new language or major dependency without an ADR.
- Do not modify CI workflows in `.github/workflows/` without testing in a fork; broken CI blocks everyone.
- Do not commit generated files (the OpenAPI client outputs, Prisma client, Go vendor, etc.). They're regenerated in CI.
- Do not duplicate logic between apps. If you're tempted, the answer is a shared package.
```

## The subtree CLAUDE.md (example: `apps/api/CLAUDE.md`)

```markdown
# apps/api - Go API service

Read the root `/CLAUDE.md` first for repo-wide rules. This file covers Go-specific conventions.

## Stack

- Go 1.22
- chi router
- sqlx for database access (we don't use an ORM)
- pgx as the Postgres driver
- testify for assertions, dockertest for integration tests

## Conventions

- Package layout follows the standard Go project layout: `cmd/`, `internal/`, `pkg/`
- `internal/` for everything not meant to be imported by other Go modules (which is most things in a monorepo)
- Errors: wrap with `fmt.Errorf("doing X: %w", err)`. Don't drop errors. Don't return `errors.New("...")` from public functions, return typed errors.
- Logging: `slog` from stdlib, structured, never `log.Println`
- Context: every function that does I/O takes `ctx context.Context` as the first argument

## Testing

- Unit tests in same package, `_test.go` suffix
- Integration tests in `internal/<package>/integration/`, build tag `integration`
- Run unit: `make test`
- Run integration: `make test-integration` (requires Docker)

## What not to do

- Do not add an ORM. We tried gorm, removed it.
- Do not panic except in `main()` for fatal startup errors. Return errors.
- Do not use `context.TODO()` outside of tests. If you don't have a context, get one.
```

## Decisions and reasoning

**Two-tier CLAUDE.md is the key insight for monorepos.** The root has cross-cutting rules. The subtrees have stack-specific rules. Trying to put everything in the root makes it unreadable; trying to put everything in subtrees means cross-cutting things get inconsistent.

**The root `CLAUDE.md` explicitly tells Claude to read the subtree one.** This is critical. Without that pointer, Claude will work from the root config alone and miss subtree conventions.

**The "Where to put new things" section is the most-used part of the doc.** Monorepos are structured for placement, not for discovery. New code should land in a predictable spot. This section makes the placement decisions explicit.

**Branch and commit conventions get their own section.** In a monorepo with multiple teams, these matter. The `scope` discipline (matching subtree name) lets CI route reviews and lets people scan the changelog.

**The ADR pointer is in the architecture section, not the rules section.** ADRs are reference material. Linking to them keeps Claude grounded in actual past decisions instead of inventing new ones.

## What's deliberately not included in the root

- **Stack-specific rules.** Those are in subtree CLAUDE.md. Putting them at the root would mean re-reading Go conventions when working on TypeScript.
- **A list of every package.** The directory listing answers that. Listing it in CLAUDE.md doubles maintenance.
- **Test commands per app.** Each subtree's CLAUDE.md handles that. Root says "look in the subtree."
- **Detailed architecture.** That's what `docs/adr/` is for. The root just points there.

## How to adapt this to your project

- If your monorepo has 2-3 apps, you can probably skip the per-subtree CLAUDE.md and put everything in the root. The two-tier approach pays off around 4+ subtrees.
- The "scope" discipline in branches/commits assumes a Conventional Commits setup. Drop it if your team doesn't use that.
- Replace the example apps (web, admin, api, worker, mobile) with whatever you actually have. Don't keep the placeholder list.
- The most important section to customize is "Where to put new things." Yours will be very different.

## A note on context bloat

In a large monorepo, even the root CLAUDE.md plus a relevant subtree one is a lot of tokens. If you find Claude losing track of things, consider:

- Splitting the root file into "rules" and "reference" sections. CLAUDE.md only loads rules.
- Letting subtree CLAUDE.md fully shadow the root for some sections (e.g. "What not to do" can be subtree-specific).
- Using `@import` patterns if your tooling supports them.

The principle is the same: load only what's load-bearing for the current task.
