# Support Desk CRM

A modular-monolith learning project modeled after internal support and CRM tools. It includes a React frontend, Express API, Prisma/PostgreSQL schema, background worker, domain package, tests, and learning-oriented architecture documentation.

## Local Development

```bash
npm install
cp .env.example .env
docker compose up -d
npm run prisma:migrate
npm run prisma:seed
npm run dev
```

Frontend: http://localhost:5173  
API health: http://localhost:4000/api/health

**There is no `.env.example` in this snapshot**, so the `cp` step fails. Create `.env` in the repository root yourself. The database values match `docker-compose.yml`:

```env
DATABASE_URL=postgresql://support:support@localhost:5432/support_desk
PORT=4000
WEB_ORIGIN=http://localhost:5173
WORKER_POLL_INTERVAL_MS=4000
```

Only `DATABASE_URL` is required; the others are the defaults the code uses (`apps/api/src/server.ts:4`, `apps/api/src/app.ts:13`, `apps/worker/src/worker.ts:4`). `docker-compose.yml` also starts Redis, but no code in this snapshot uses it. There is no `prisma/migrations/` folder yet, so the first `npm run prisma:migrate` (which runs `prisma migrate dev`) creates the initial migration.

`npm run dev` starts three processes with `concurrently`: the API (`tsx watch apps/api/src/server.ts`), the Vite frontend (root `apps/web`, proxying `/api` to `http://localhost:4000`, `vite.config.ts`) and the worker (`tsx watch apps/worker/src/worker.ts`).

## Repository shape

This is a single npm package (one `package.json`), not npm workspaces. `packages/domain` and `packages/shared` are plain source folders that the API, worker, web app and tests import through TypeScript path aliases: `@support/domain` and `@support/shared` (`tsconfig.json` `paths`, mirrored in `vitest.config.ts`).

| Path | What it holds |
|---|---|
| `apps/api/src/app.ts` | Express app: CORS, JSON, health, auth middleware, four routers, error middleware |
| `apps/api/src/middleware/` | `auth.ts` (demo actor from headers, `requireRole`), `errors.ts` (`HttpError`, `asyncHandler`, error mapping) |
| `apps/api/src/modules/` | `tickets` (routes + service), `customers`, `notifications` (inbound email), `admin` |
| `apps/worker/src/worker.ts` | Outbox polling loop |
| `apps/web/src/` | One-screen React app (`App.tsx`) and its API client (`lib/api.ts`) |
| `packages/domain/src/index.ts` | Ticket transitions, SLA math and state, permission helpers |
| `packages/shared/src/index.ts` | Zod schemas for API input |
| `prisma/` | Schema (9 models) and seed |
| `tests/domain/ticket-rules.test.ts` | Seven domain tests |

## Who you are

There is no login. `authMiddleware` (`apps/api/src/middleware/auth.ts:19-41`) reads `x-tenant-slug` and `x-user-email` and loads that active user. **Missing headers default to `acme-support` and `agent@acme.test`**, so an anonymous request is an agent. The seed creates one tenant (`acme-support`) with four users: `admin@`, `manager@`, `agent@` and `viewer@acme.test`. The web app has a user switcher with those four emails (`apps/web/src/App.tsx:18`).

## Known gaps (verified in this snapshot)

- **Viewers can read internal notes.** `GET /api/tickets/:id` hides internal comments from viewers, but `PATCH /api/tickets/:id` returns the ticket with *all* comments (`apps/api/src/modules/tickets/ticketService.ts:13-20`, `:181`), and a viewer is allowed to patch priority or tags. Viewers can also post public replies to customers and create tickets.
- **Invalid status transitions return 500.** `assertTicketTransition` throws a plain `Error` (`packages/domain/src/index.ts:39-43`), which the error middleware maps to 500 (`apps/api/src/middleware/errors.ts:32-33`).
- **Ticket numbers race.** New numbers are "highest + 1" read inside the transaction (`ticketService.ts:100-110`, `apps/api/src/modules/notifications/notificationRoutes.ts:33-43`); two concurrent creates pick the same number and the second fails the `@@unique([tenantId, number])` index with a 500.
- **The worker can stop silently.** If one polling pass throws (for example the database is briefly unreachable), the loop's promise rejects before the next `setTimeout` is scheduled (`apps/worker/src/worker.ts:36-44`), so polling ends.
- **Failed outbox events are never retried**, and a crash between the two status updates leaves an event in `PROCESSING` forever.
- **Tests cover the domain package only** (seven tests). CI (`.github/workflows/ci.yml`) runs `prisma generate` and `verify` with no database.

## Learning Path

Start with `docs/architecture/architecture-plan.md`, then read `docs/architecture/architecture-journal.md`. After the initial build, use `docs/walkthrough/important-files.md` to learn how the main files fit together; it now includes line references, review notes and exercises. `docs/architecture/final-system-design.md` traces each workflow step by step.

## About this snapshot

This repository is a single-commit import. Statements in the docs about how the project was built (journal entries, CI results) are the author's record. Claims about code were re-checked against this snapshot during a 2026-10 docs pass.
