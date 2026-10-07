# System Design Summary

## Repo Structure

- `apps/api`: Express API for tenant-scoped product workflows
- `apps/web`: React frontend for support agents, managers, and admins
- `apps/worker`: background outbox consumer
- `packages/domain`: framework-independent business rules
- `packages/shared`: validation schemas and DTO types
- `prisma`: PostgreSQL schema and seed data
- `docs`: architecture plan, journal, feature plans, backlog, and walkthroughs
- `tests`: domain-level verification

## Major Module Connections

The frontend calls API endpoints through `apps/web/src/lib/api.ts`. The API resolves a demo actor in auth middleware, then route modules call service functions. Services use Prisma to read/write tenant-scoped data and domain helpers to enforce ticket rules. Important writes create audit logs and outbox events. The worker processes pending outbox events.

## Key Workflows

- Ticket queue: filter tickets, select a ticket, inspect customer/company context.
- Ticket update: validate input, enforce transition and assignment rules, update record, audit, emit outbox event.
- Commenting: create internal notes or public replies; public replies set first response time and emit notification events.
- Email ingestion: receive inbound email, upsert customer, create ticket, store email, emit integration event.
- Audit review: managers/admins inspect recent actions.

## Design Decisions

- Modular monolith over microservices to keep learning focused on boundaries inside one codebase.
- Domain package for reusable business rules.
- Zod schemas for API input validation.
- Prisma schema with tenant IDs on operational tables.
- Outbox pattern from the first version because notifications and integrations are realistic support-desk concerns.
- Dense frontend UI because internal tools are used repeatedly by trained users.

## Tradeoffs

- Header-based demo auth is intentionally temporary.
- Polling worker is simpler than a real queue.
- Search is database filtering, not full-text search.
- Tests currently focus on domain rules instead of database-backed integration.
- The frontend is a single app component to keep the first walkthrough approachable; it should be split as more features land.

## Improvements

Next good engineering exercises:

- Replace demo auth with a real provider.
- Add integration tests with a test database.
- Add saved views and automation rules.
- Add file attachments and object storage.
- Add webhooks with retry policies.
- Add cursor pagination and full-text search.
- Split the main React app into workflow-specific components.
- Add structured logging, metrics, and trace IDs.

## Workflows step by step (checked against this snapshot)

Line numbers refer to this snapshot. Nothing here was run.

### Every request

1. `apps/api/src/app.ts:13-14` CORS limited to `WEB_ORIGIN` (default `http://localhost:5173`), JSON bodies.
2. `:16` `/api/health` is mounted **before** auth; it runs `SELECT 1` (`apps/api/src/routes/health.ts:7-13`).
3. `:17` `authMiddleware` for everything else under `/api`: tenant slug and email from headers (defaults `acme-support` / `agent@acme.test`), one query for an active user in that tenant, 401 if none (`apps/api/src/middleware/auth.ts:19-41`).
4. `:18-21` routers; `:22` `errorMiddleware`: `ZodError` → 400 with flattened issues, `HttpError` → its status, anything else → 500 with the raw message (`apps/api/src/middleware/errors.ts:21-34`).

### Ticket queue — `GET /api/tickets`

`ticketQuerySchema` (`packages/shared/src/index.ts:10-16`) coerces `page` (default 1) and `pageSize` (default 25, max 100) and accepts `search`, `status`, `priority`, `assignedUserId`, `tag`. `listTickets` (`apps/api/src/modules/tickets/ticketService.ts:37-71`) always filters by `tenantId`, searches subject, description and customer name and email case-insensitively, sorts by priority then `updatedAt`, and adds `slaState` to each row with `getSlaState` (`packages/domain/src/index.ts:56-68`).

### Ticket update — `PATCH /api/tickets/:id`

1. `updateTicketSchema` (`packages/shared/src/index.ts:26-31`): status, priority, `assignedUserId` (nullable) and tags, all optional.
2. Load the ticket scoped to the tenant (`ticketService.ts:144-149`).
3. If a status is sent, `assertTicketTransition` (`:151-153`); if an assignee is sent, `canAssignTicket` and a tenant-scoped, active assignee (`:155-165`).
4. One transaction: update (resetting `slaDueAt` from *now* when the priority changes, setting `resolvedAt` when resolving), audit log with `before`/`after`, outbox event `ticket.updated` (`:167-204`).

### Comment — `POST /api/tickets/:id/comments`

Viewers may not write internal notes (`ticketService.ts:208-210`). One transaction creates the comment, sets `firstResponseAt` on the first public reply, writes an audit row, and — for public replies only — an outbox event `notification.customer_reply` (`:218-256`).

### Inbound email — `POST /api/notifications/inbound-email`

Validates `externalId`, `fromEmail`, `subject`, `body` (`apps/api/src/modules/notifications/notificationRoutes.ts:7-12`), upserts the customer by `(tenantId, email)` **outside** the transaction (`:22-30`), then in one transaction creates a `NORMAL` ticket tagged `email`, the `EmailMessage`, and an outbox event `ticket.email_ingested` (`:32-75`). It writes no audit log, and it is called with the caller's demo headers, so the "email provider" is any signed-in user.

### Outbox worker

Every `WORKER_POLL_INTERVAL_MS` (default 4000 ms), take 10 `PENDING` events oldest first, mark each `PROCESSING` with `attempts + 1`, log it, mark it `PROCESSED`; on error mark it `FAILED` with `lastError` (`apps/worker/src/worker.ts:6-34`).

## Review findings

1. **Internal notes leak to viewers through write responses.** `ticketInclude` (`ticketService.ts:13-20`) includes every comment; `getTicket` overrides it with a visibility filter (`:76-83`), but `createTicket` and `updateTicket` return `ticketInclude` as-is (`:117`, `:181`). A viewer may patch priority or tags (only status changes and assignment are checked), so `PATCH /api/tickets/:id` with `{"tags": ["x"]}` returns internal notes to a viewer.
2. **Viewers can write.** Nothing stops a viewer from creating tickets, changing priority or tags, posting public replies to customers, or ingesting email. `requireRole` is used only on the audit log route (`apps/api/src/modules/admin/adminRoutes.ts:22`).
3. **Invalid transitions are 500s**, because the domain throws a plain `Error`.
4. **Ticket numbering races** (see README); so does `firstResponseAt` (read before the transaction at `ticketService.ts:212-215`).
5. **Reopening keeps `resolvedAt`**, and moving out of `PENDING` keeps the original `slaDueAt`, so time spent paused still counts against the SLA.
6. **The worker has no claim step, no retry and no restart on failure** (see README). Two worker processes would both pick the same events.
7. **Unused domain helpers.** `canViewInternalNote` and `canManageUsers` (`packages/domain/src/index.ts:70-76`) are not used; the visibility rule is re-implemented inline in `visibleCommentWhere` (`ticketService.ts:22-24`).

## Exercises

Each has a **Goal** and a **Check** you can observe.

1. **Reproduce finding 1.**
   *Goal:* with the seed loaded, call `PATCH /api/tickets/<a seeded ticket id>` with headers `x-user-email: viewer@acme.test` and body `{"tags":["billing"]}`.
   *Check:* the response's `comments` array contains a comment with `"visibility": "INTERNAL"`, while `GET` for the same ticket as the viewer does not. Then return `getTicket(auth, id)` from `updateTicket` instead of the raw update result and confirm the internal note disappears from the PATCH response.
2. **Make transitions a 409.**
   *Goal:* throw `new HttpError(409, ...)` (or map a domain error class) for invalid transitions.
   *Check:* as the agent, `PATCH` a `NEW` ticket to `CLOSED` returns 409 with "Cannot transition ticket from NEW to CLOSED".
3. **Keep the worker alive.**
   *Goal:* wrap `processOutboxBatch` in `try/catch` inside `loop` and always schedule the next pass.
   *Check:* stop PostgreSQL while the worker runs, start it again, create a ticket: its `ticket.created` event still reaches `PROCESSED`.
4. **Pause the SLA properly.**
   *Goal:* store the remaining SLA time when a ticket enters `PENDING` and recompute `slaDueAt` when it leaves.
   *Check:* an `URGENT` ticket set to `PENDING` for 30 minutes and then back to `OPEN` has a `slaDueAt` 30 minutes later than before.
