# CLAUDE.md for a Next.js Full-Stack App

## Project shape

- Next.js 14+ with App Router (not Pages Router)
- TypeScript everywhere
- Prisma as the ORM, Postgres as the database
- Server Actions for mutations, RSC for reads
- Tailwind for styling, shadcn/ui for component primitives
- Auth.js (formerly NextAuth) for auth
- Vercel for hosting (or similar)

This shape covers a huge fraction of modern full-stack apps: SaaS dashboards, internal tools, content platforms.

## The CLAUDE.md

```markdown
# Project Context

This is the web application for [Product]. Single Next.js app, no separate backend service. The Next.js server handles auth, data access via Prisma, and serves React Server Components plus a small amount of client-side interactivity.

## Stack

- Next.js 14 (App Router only, never Pages Router)
- TypeScript, strict mode on, no `any` allowed unless commented `// any: <reason>`
- React Server Components by default, Client Components only when necessary (state, effects, browser APIs)
- Prisma ORM, Postgres
- Server Actions for mutations
- Auth.js for auth, with the credentials provider plus Google OAuth
- Tailwind CSS for all styling
- shadcn/ui components in `components/ui/` (don't edit these unless adding a variant)
- Zod for runtime validation at trust boundaries

## Conventions

- File naming: `kebab-case.tsx` for files, `PascalCase` for component exports
- Server Components are the default. A file is a Client Component only if it has `"use client"` at the top, and that should be rare.
- Imports: absolute via `@/` alias, never relative across more than one directory
- Async functions for everything that touches data, even when it could be sync, for forward compatibility
- Prefer `function` declarations for components, arrow functions for handlers/utilities

## Data access rules

- All database access goes through `lib/db.ts`, which exports the Prisma client
- All write operations happen in Server Actions (`app/actions/<domain>.ts`)
- All Server Actions must validate input with Zod before touching the DB
- Never call Prisma from a Client Component. If you need data in a client component, pass it down from a Server Component or fetch via a route handler
- Never expose Prisma error messages to the client. Catch and translate.

## Auth and authorization

- `auth()` from `lib/auth.ts` returns the current session in any Server Component or Server Action
- Authorization checks happen in Server Actions, never trust the client to gate. Pattern:
  ```ts
  const session = await auth()
  if (!session?.user) throw new Error("Unauthorized")
  if (!canEdit(session.user, resource)) throw new Error("Forbidden")
  ```
- `canEdit`, `canRead`, etc. live in `lib/permissions.ts`. Add to that file rather than scattering checks.

## Forms and mutations

- Use `<form action={serverAction}>` with Server Actions for forms
- For optimistic updates, use `useOptimistic` (React 19+)
- Don't reach for `react-hook-form` unless the form is genuinely complex (multi-step, dynamic fields). The native form + Server Action pattern handles 90% of cases with less code.

## Styling

- Tailwind utility classes, no CSS modules, no styled-components
- Color palette via CSS variables defined in `app/globals.css` (don't hardcode hex values)
- Spacing follows Tailwind's default scale, prefer multiples of 4
- For repeated component patterns, extract a component, not a `@apply` rule

## Testing

- Vitest for unit tests, Playwright for end-to-end
- Run unit tests: `pnpm test`
- Run E2E: `pnpm test:e2e` (spins up the app, requires a test database)
- Test files: `*.test.ts` colocated with source for unit, `e2e/` directory for Playwright
- We don't aim for high coverage. We test: pure logic in `lib/`, complex Server Actions, critical user flows in E2E.

## Common gotchas

- `useEffect` is a code smell in this codebase. If you're reaching for it, ask whether the data should come from the server instead.
- Server Actions can't return non-serializable data. No Date objects (use ISO strings), no class instances, no functions.
- Prisma's `findUnique` returns null for missing records, `findUniqueOrThrow` throws. Pick deliberately.
- Form actions need to be top-level functions or imported from a `'use server'` file. Inline arrow functions in JSX won't work.
- `revalidatePath` and `revalidateTag` after mutations, otherwise stale data shows.

## How to run things

- Dev: `pnpm dev` (runs on :3000)
- Database: `pnpm db:push` (dev), `pnpm db:migrate` (production migrations)
- Type check: `pnpm typecheck`
- Lint: `pnpm lint`
- Build (production check): `pnpm build`

## What not to do

- Do not add a separate API server. Server Actions and Route Handlers cover what we need.
- Do not use `getServerSideProps` or `getStaticProps`. Those are Pages Router. App Router uses async Server Components.
- Do not install state management libraries (Redux, Zustand, Jotai) without discussion. Server-derived state plus URL state plus a small amount of `useState` covers most needs.
- Do not bypass Prisma with raw SQL unless there's a measured perf reason. If you do, leave a comment explaining why.
- Do not commit `.env.local`. Secrets go in Vercel's environment config. Local-only `.env.local` is gitignored.
```

## Decisions and reasoning

**The split between "Conventions" and "Data access rules" is intentional.** Conventions are style. Data access rules are correctness. Mixing them dilutes both.

**The auth section gives a code pattern.** This breaks the "no examples" guideline elsewhere because auth is the single biggest source of correctness bugs in app code, and the pattern is small enough that showing it once prevents many wrong attempts.

**The "Forms and mutations" section is opinionated against `react-hook-form` for simple cases.** This is the kind of opinion you should bake into CLAUDE.md only if you've decided as a team. Otherwise Claude will reach for the popular library and you'll be undoing it forever.

**The "Common gotchas" section captures App Router landmines specifically.** These are the things that trip up developers (and Claude) who learned Next.js in the Pages Router era. They're real, recurring, and worth documenting.

**`useEffect` is called out as a code smell explicitly.** This is the most-misused hook in modern React. If you let Claude write whatever it wants, you'll get unnecessary `useEffect`s for things that should be server-derived. Stating the preference inverts the default.

## What's deliberately not included

- **A list of all environment variables.** Lives in `.env.example`. CLAUDE.md just says "Vercel env config" and points at the file.
- **The Prisma schema.** Claude reads it on demand. Pre-loading it bloats context.
- **Component documentation.** shadcn components are well-documented externally; ours aren't standardized enough to be worth listing.
- **Routing details.** Next.js App Router is conventional enough that Claude knows it. We only call out the ones that surprise people.
- **A "what we're moving toward" section.** Tempting, but adds noise. If we're migrating off something, that goes in a migration doc and is referenced when relevant.

## How to adapt this to your project

- If you're using Pages Router still: this entire doc is wrong for you. Don't try to mix.
- If you're on a separate backend (e.g. Next.js frontend + Express/FastAPI/Go backend): replace the "Data access rules" and "Server Actions" sections with the integration pattern between front and back.
- If you're not using Prisma: replace ORM-specific notes with your equivalent (Drizzle, Kysely, raw SQL).
- The "Common gotchas" section is the one to grow over time. Yours will look different from this list within a month of real use.
