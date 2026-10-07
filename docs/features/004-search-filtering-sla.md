# Feature Plan: Search, Filtering, And SLA

## Goal

Give agents practical queue controls and make SLA state visible in both API responses and the frontend.

## Impacted Areas

- `packages/domain/src/index.ts`
- `packages/shared/src/index.ts`
- `apps/api/src/modules/tickets/ticketService.ts`
- `apps/web/src/App.tsx`

## Implementation Approach

Use shared query validation for filters, Prisma queries for search and tag/status/priority filters, and a domain helper to calculate SLA state from ticket snapshots.

## Design Considerations

Queue behavior should start simple but stay extensible. The current filtering approach can later evolve into saved views or full-text search without changing the basic API shape.

## Build Journal

### What Was Built

The ticket list supports search, status, priority, assignment, tag, page, and page size parameters at the API layer. The frontend exposes search, status, and priority controls. SLA state is computed consistently by the domain helper.

### How It Connects

The shared schema validates query parameters before they reach the service. The service queries Prisma and enriches ticket responses with `slaState`. The frontend renders badges from that API state.

### Lessons For A Growing Developer

Filtering is a contract, not just UI state. Good systems validate and name the query shape so future screens and API consumers can reuse it.

### Tradeoffs And Future Improvements

Database `contains` search is adequate for a first version. A larger system should add full-text indexes, saved filters, stable sorting, and cursor pagination.

## As built (checked against this snapshot)

- Query contract: `ticketQuerySchema` (`packages/shared/src/index.ts:10-16`) — `page`, `pageSize` (max 100), `search`, `status`, `priority`, `assignedUserId`, `tag`.
- Query: `listTickets` (`apps/api/src/modules/tickets/ticketService.ts:37-71`) — tenant filter plus each optional filter; `tags: { has: tag }` uses the GIN index on `Ticket.tags`; search is four `contains … insensitive` clauses; order `priority desc, updatedAt desc`; returns `{ items, total, page, pageSize }`.
- SLA: due date = creation + 72 h / 24 h / 8 h / 2 h for `LOW` / `NORMAL` / `HIGH` / `URGENT` (`packages/domain/src/index.ts:45-54`); state `PAUSED` for pending, resolved and closed tickets, `MET` with no due date, `BREACHED` when past due, `AT_RISK` within one hour (`:56-68`).
- Frontend: search, status and priority controls feed `query` into `refresh` (`apps/web/src/App.tsx:34-36`, `:80-82`).

**Gaps.** `priority desc` sorts by the enum's declared order in PostgreSQL, so the result depends on the order in `prisma/schema.prisma` (`LOW, NORMAL, HIGH, URGENT` sorts `URGENT` first only because it is declared last). `contains` search cannot use an index. Offset pagination shifts when tickets are updated between pages. The SLA does not pause in a way that extends the due date.

## Exercises

1. **Goal:** add a test for `getSlaState` exactly at the boundaries: 60 minutes left and 1 ms past due. **Check:** the first returns `AT_RISK`, the second `BREACHED`.
2. **Goal:** add a `sort` parameter (`priority`, `slaDueAt`, `updatedAt`) to the schema and the query. **Check:** `?sort=slaDueAt` returns the ticket closest to breach first.
