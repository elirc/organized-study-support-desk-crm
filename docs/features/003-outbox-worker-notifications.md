# Feature Plan: Outbox Worker And Notifications

## Goal

Introduce a reliable boundary for asynchronous work such as customer notifications, webhooks, and integrations.

## Impacted Areas

- `prisma/schema.prisma`
- `apps/api/src/modules/tickets/ticketService.ts`
- `apps/api/src/modules/notifications/notificationRoutes.ts`
- `apps/worker/src/worker.ts`

## Implementation Approach

Create `OutboxEvent` rows in the same transaction as the domain change. Let a separate worker process poll pending events and mark them processed or failed.

## Design Considerations

An outbox avoids the classic bug where the database write succeeds but the external notification fails before the system records that it needs retrying.

## Build Journal

### What Was Built

Ticket creation, ticket updates, public replies, and inbound email ingestion create outbox events. The worker polls pending events, logs the payload, and updates processing state.

### How It Connects

The worker depends on the same Prisma client as the API but runs as a separate process. That mirrors how many modular monoliths split web and background runtimes while sharing the same codebase and database.

### Lessons For A Growing Developer

Asynchronous side effects deserve explicit state. A durable event table is easier to inspect and retry than a hidden in-memory promise chain.

### Tradeoffs And Future Improvements

The current worker uses polling and console logging. Future versions should add exponential backoff, dead-letter queues, idempotency keys, and real providers.

## As built (checked against this snapshot)

- `OutboxEvent` (`prisma/schema.prisma`, model at line 179) with `status` (`OutboxStatus` enum), `attempts`, `lastError`, `processedAt`; indexed on `(status, createdAt)` for the poller.
- Producers: `ticket.created` (`ticketService.ts:131-137`), `ticket.updated` (`:195-201`), `notification.customer_reply` for public replies (`:245-252`), `ticket.email_ingested` (`notificationRoutes.ts:66-72`). Each is written in the same transaction as its change.
- Consumer: `apps/worker/src/worker.ts:6-44`, importing the API's Prisma client directly (`:2`).

**Gaps.** The build journal says the worker updates "processing state" — it does, but `FAILED` is terminal: nothing retries it, and `attempts` is counted but never compared to a limit. There is no claim step, so two workers process the same events twice. If a pass throws before the per-event `try`, the polling loop ends. The "delivery" is a `console.log`.

## Exercises

1. **Goal:** retry `FAILED` events up to five attempts with a delay (add `availableAt` to the model and select `status in (PENDING, FAILED) and attempts < 5 and availableAt <= now`). **Check:** an event whose handler throws twice and then succeeds ends `PROCESSED` with `attempts` 3.
2. **Goal:** claim events atomically with `UPDATE ... SET status = 'PROCESSING' WHERE id IN (SELECT ... FOR UPDATE SKIP LOCKED LIMIT 10) RETURNING *` via `prisma.$queryRaw`. **Check:** with two worker processes running, each event's `attempts` ends at 1.
