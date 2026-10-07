# Important File Walkthrough

## `prisma/schema.prisma`

This is the product data model. Read it first to understand the nouns in the system: tenants, users, companies, customers, tickets, comments, email messages, audit logs, and outbox events. Notice that tenant isolation is not implied; it is represented by `tenantId` on operational tables.

Important sections:

- Enums define workflow vocabulary.
- `Ticket` connects customer, assignee, comments, email messages, SLA fields, tags, and status.
- `AuditLog` and `OutboxEvent` show operational concerns that are not directly visible in the UI but matter for maintenance.

Senior concept: schema design is product design. A hard-to-query schema becomes a hard-to-change product.

## `packages/domain/src/index.ts`

This file holds business rules that should survive framework changes. Ticket transitions, SLA due-date math, SLA state, and basic permission helpers live here.

Read this file before changing ticket behavior. If a rule is important enough to affect multiple modules, it probably belongs here or in a nearby domain helper.

Senior concept: keeping domain rules framework-free makes them easy to test and harder to accidentally bypass.

## `packages/shared/src/index.ts`

This file defines Zod schemas for API inputs. The schemas produce TypeScript types, so validation and DTO typing stay connected.

Senior concept: validation belongs at boundaries. The API should convert unknown request data into known application data before service logic runs.

## `apps/api/src/middleware/auth.ts`

This middleware creates the request auth context. In this learning version it uses demo headers, but the rest of the app treats the result like a real authenticated actor.

Important habit: downstream modules should use `req.auth.tenantId`; they should not rediscover tenancy on their own.

## `apps/api/src/modules/tickets/ticketService.ts`

This is the main orchestration file. It lists tickets, reads ticket details, creates tickets, updates tickets, and adds comments.

Important sections:

- `listTickets` builds the query shape from validated filters.
- `createTicket` uses a transaction to create the ticket, audit log, and outbox event.
- `updateTicket` checks domain transitions and assignment permissions before writing.
- `addTicketComment` distinguishes public replies from internal notes and emits notification events for public replies.

Senior concept: orchestration code coordinates rules, persistence, and side effects. It should be explicit and boring.

## `apps/api/src/modules/notifications/notificationRoutes.ts`

This route simulates inbound email ingestion. It upserts a customer, creates a ticket, stores the email message, and emits an outbox event.

Senior concept: integrations should preserve source data. Storing the inbound email separately from the ticket gives future debugging and reprocessing options.

## `apps/worker/src/worker.ts`

The worker consumes outbox events. It is intentionally small, but it demonstrates a separate runtime reading durable background work from the database.

Senior concept: background jobs are part of the system design, not an afterthought. They need state, retries, and observability as the system grows.

## `apps/web/src/App.tsx`

This is the main frontend workflow surface. It fetches queue data, selects tickets, updates status/priority/assignment, adds comments, creates tickets, and switches to customer/audit views.

Read it by following user workflows rather than top-to-bottom:

- `refresh`
- `selectTicket`
- `updateSelectedTicket`
- `submitComment`
- `submitTicket`
- `simulateInboundEmail`

Senior concept: frontend state should model workflows. A dense internal UI should optimize repeated operational tasks, not present a decorative landing page.

## `apps/web/src/lib/api.ts`

This file is the frontend's API boundary. It centralizes request headers, response types, and endpoint functions.

Senior concept: isolating API calls keeps components focused on workflow and rendering. It also gives you one place to evolve auth headers or error handling.

## `tests/domain/ticket-rules.test.ts`

These tests protect the highest-value pure rules: transitions, SLA state, and authorization helpers.

Senior concept: start tests where bugs are expensive and setup is cheap. Domain tests are fast, stable, and clarify expected behavior.


## Line references and review notes (2026-10 docs pass)

Line numbers refer to this snapshot. Nothing was run.

- **`prisma/schema.prisma`** — enums at the top (`Role`, `TicketStatus`, `TicketPriority`, `CommentVisibility`, `OutboxStatus`, lines 10-42); `Ticket` (`:103-130`) with unique `(tenantId, number)` and a GIN index on `tags`; `EmailMessage` unique on `(tenantId, externalId)` (`:160`). Review note: the order of `TicketPriority` values decides how `orderBy: { priority: "desc" }` sorts.
- **`packages/domain/src/index.ts`** — transitions (`:24-43`): unchanged status is always allowed, viewers can change nothing, agents may close only from `RESOLVED` or `PENDING`. SLA (`:45-68`). Helpers (`:70-80`): `canViewInternalNote` and `canManageUsers` are unused. Review note: `assertTicketTransition` throws a plain `Error`, which becomes a 500.
- **`packages/shared/src/index.ts`** — four Zod schemas and their inferred types (`:5-41`).
- **`apps/api/src/middleware/auth.ts`** — default headers (`:20-21`) make an anonymous request the agent; `requireRole` (`:43-57`) is used on one route.
- **`apps/api/src/modules/tickets/ticketService.ts`** — `listTickets` (`:37-71`), `getTicket` (`:73-88`), `createTicket` (`:90-141`), `updateTicket` (`:143-205`), `addTicketComment` (`:207-257`). Review notes: write responses use `ticketInclude` with all comments (`:13-20`); ticket numbers are max + 1 (`:100-110`).
- **`apps/api/src/modules/notifications/notificationRoutes.ts`** — inbound email (`:16-79`); the customer upsert is outside the transaction and there is no audit log.
- **`apps/worker/src/worker.ts`** — batch (`:6-34`), loop (`:36-44`); one thrown pass ends the loop.
- **`apps/web/src/App.tsx`** — `refresh` (`:58-78`), `selectTicket` (`:84-86`), `updateSelectedTicket` (`:88-93`), `submitComment` (`:95-101`), `submitTicket` (`:103-116`), `loadAudit` (`:118-126`), `simulateInboundEmail` (`:128-138`). Review note: only `refresh` and `loadAudit` catch errors.
- **`apps/web/src/lib/api.ts`** — one `request` helper sends `/api…` with the two demo headers and throws the server's `error` message on failure.
- **`tests/domain/ticket-rules.test.ts`** — seven tests: agent close from `RESOLVED`, viewer blocked, invalid transition throws, urgent SLA is two hours, at-risk, pending pauses, operational roles can assign.

## Exercises

Each has a **Goal** and a **Check** you can observe.

1. **Follow a public reply.**
   *Goal:* as the agent, post a `PUBLIC` comment on a ticket with no earlier public reply, then look at the database.
   *Check:* the ticket now has `firstResponseAt`, there is a `ticket.public_reply_added` audit row and a `notification.customer_reply` outbox event, and within a few seconds the worker marks that event `PROCESSED`.
2. **Switch to the viewer in the UI.**
   *Goal:* choose `viewer@acme.test` in the user switcher and open a ticket that has an internal note.
   *Check:* the note is hidden; now change the ticket's priority in the UI and check the network response of the `PATCH` request — the internal note is in it (finding 1 in `docs/architecture/final-system-design.md`).
3. **Test the boundary you just found.**
   *Goal:* write a domain or API test that captures "viewers never receive internal comments from any ticket endpoint".
   *Check:* the test fails on the current code and passes after the fix.
