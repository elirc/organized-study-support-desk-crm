# Feature Plan: Initial Operational Core

## Goal

Create the first usable version of the support desk: tenant-scoped customer data, ticket queue, ticket detail workflow, internal/public comments, SLA tracking, email ingestion, outbox worker, audit logs, seed data, and frontend screens.

## Impacted Areas

- `prisma/schema.prisma`
- `prisma/seed.ts`
- `packages/domain`
- `packages/shared`
- `apps/api`
- `apps/web`
- `apps/worker`
- `tests/domain`

## Implementation Approach

1. Model the core support entities in Prisma.
2. Put workflow rules in `packages/domain`.
3. Put validation schemas in `packages/shared`.
4. Build thin Express routes that call service functions.
5. Use transactions for ticket changes, audit logs, and outbox events.
6. Build a dense React workspace around the queue and ticket detail.
7. Add tests for domain rules before database integration tests.

## Design Considerations

The system should feel production-shaped without pretending to be production-complete. Header-based demo auth is acceptable for the first slice because it keeps attention on tenant isolation and authorization patterns. The outbox worker is intentionally simple, but the table and processing lifecycle are realistic.

## Build Journal

### What Was Built

The repo now has a TypeScript monorepo-style layout with an Express API, React/Vite frontend, Prisma/PostgreSQL schema, worker process, domain package, shared validation schemas, tests, Docker Compose, CI, and documentation.

### How It Connects

The frontend calls `/api` endpoints with tenant/user headers. The auth middleware resolves the current actor. Ticket services enforce domain transitions, write audit logs, and enqueue outbox events. The worker polls outbox events and marks them processed.

### Lessons For A Growing Developer

Business rules should be easy to find and test. A route handler should not be the only place where a workflow rule exists. Also, important side effects should be transactionally connected to the write that caused them, which is why audit logs and outbox events are created with ticket updates.

### Tradeoffs And Future Improvements

The first version does not include real login, refresh tokens, full-text search, real email delivery, or a queueing library. Those are good future features once the basic module boundaries are stable.

## As built (checked against this snapshot)

| Plan step | Code |
|---|---|
| 1. Model the entities | `prisma/schema.prisma`: 9 models, 5 enums; unique `(tenantId, number)` on tickets, `(tenantId, email)` on users and customers, `(tenantId, externalId)` on email messages; GIN index on `Ticket.tags` |
| 2. Rules in `packages/domain` | `packages/domain/src/index.ts`: transitions (`:24-43`), SLA (`:45-68`), permission helpers (`:70-80`) |
| 3. Validation in `packages/shared` | `packages/shared/src/index.ts`: query, create, update and comment schemas |
| 4. Thin routes over services | `apps/api/src/modules/tickets/ticketRoutes.ts` → `ticketService.ts` |
| 5. Transactions with audit and outbox | `ticketService.ts:99-140`, `:167-204`, `:218-256` |
| 6. Dense React workspace | `apps/web/src/App.tsx` (one component, about 400 lines) |
| 7. Domain tests | `tests/domain/ticket-rules.test.ts`: seven tests |

What the build journal does not mention: the web handlers for status, priority and assignment changes (`App.tsx:88-93`) and comments (`:95-101`) have no `try/catch`, so a rejected change (for example an invalid transition) fails silently in the UI; and `refresh` (`:58-78`) only auto-selects the first ticket when nothing is selected yet.

## Exercise

**Goal:** add one API integration test that creates a ticket through `createApp()` with `supertest` against a migrated, seeded test database.

**Check:** the test asserts a 201, a `number` above 1000, and one new `ticket.created` row in `OutboxEvent`; it fails if you remove the `outboxEvent.create` call in `createTicket`.
