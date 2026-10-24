# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev                            # Next.js dev server on :3000
npm run build
npm run lint                           # next lint (ESLint + prettier config)
npm test                               # vitest run, with TZ=America/Los_Angeles
npx vitest run lib/task-authz.test.ts  # single test file
npx vitest run -t "respects limit"     # single test by name
```

CI (`.github/workflows/ci.yml`) runs `npm ci`, build, lint, test on Node 20.

`npm test` sets `TZ=` inline (POSIX syntax) — on Windows PowerShell set `$env:TZ` first or run via Git Bash; date tests (`due-date`, `relative-time`) depend on that timezone.

## Architecture

Next.js 14 App Router + TypeScript + Tailwind, with `better-sqlite3` and no ORM. Path alias `@/` maps to the repo root.

- **Database** (`lib/db.ts`): an in-memory SQLite DB, created lazily on first `getDb()` call from `lib/schema.sql` and seeded by `lib/seed.ts`. It is cached on `globalThis.__taskflowDb` (survives dev hot reloads). Nothing persists; a server restart resets the data. `better-sqlite3` is listed in `serverComponentsExternalPackages` in `next.config.js`. Tests call `resetDbForTests()` to get a fresh DB per test file.
- **Auth** (`lib/auth.ts`): there is no real login. `getCurrentUser()` reads a user-id cookie (`AUTH_COOKIE` in `lib/constants.ts`) and falls back to user 1. `components/UserSwitcher.tsx` swaps between the seeded users to exercise ownership checks.
- **API routes** (`app/api/{projects,tasks}/...`): route handlers query `getDb()` directly with prepared statements. Each `route.ts` has a colocated `route.test.ts`. Task creation goes through `lib/rate-limiter.ts`, and task changes call `invalidateTaskCount` from `lib/summary-cache.ts` (used by the project summary route).
- **Pure logic in `lib/`**: small, separately tested helpers (`task-patch`, `task-authz`, `sort-tasks`, `project-stats`, `dashboard-tasks`, `due-date`, `relative-time`, `sanitize-feedback`). Routes and pages delegate to these, so unit tests target them rather than going through HTTP.
- **UI**: server-component pages in `app/` (`page.tsx`, `dashboard`, `projects/[id]`, `tasks/[id]`); interactive client components in `components/` with colocated RTL tests (jsdom, globals enabled, setup in `vitest.setup.ts`).

## Known state of the repo

This is a training repo with deliberately seeded bugs, so **some tests fail on purpose**. `docs/SESSION_1.md` lists the four in scope for Session 1; `ISSUES.md` lists more (comment pagination, dashboard sort order, etc.). Don't treat unrelated failing tests as regressions you caused, and don't fix out-of-scope ones unless asked. For example, `lib/task-authz.ts` `canDeleteTask` and the `limit - 1` page size in `app/api/tasks/route.ts` are seeded bugs.
