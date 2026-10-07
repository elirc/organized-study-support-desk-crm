# Feature Backlog

This backlog is intentionally product-shaped. Some items are implemented in the initial build, while later phases are designed to create realistic maintenance pressure.

## Phase 1 - Initial Operational Core

- Tenant-scoped data model
- Demo authentication context
- Role-based access control helpers
- Customers, companies, and contacts
- Ticket queue with search/filtering/pagination shape
- Ticket status, priority, assignment, and tags
- Internal notes vs public replies
- SLA due date and SLA state calculation
- Inbound email ingestion
- Outbox pattern and worker
- Audit logs
- Seed data
- Frontend queue, ticket detail, customer, and audit screens
- CI/typecheck/test/build scripts

## Phase 2 - Workflow Depth

- Saved views
- Automation rules
- Escalation policies
- Approval workflows for sensitive replies
- Team inbox ownership
- Collision indicators when multiple agents work the same ticket

## Phase 3 - Platform Capabilities

- Real authentication provider
- Fine-grained permissions
- Webhooks
- Public API contracts
- CSV import/export
- File uploads and object storage
- Knowledge base
- Customer portal

## Phase 4 - Operations and Insight

- Reporting dashboards
- Team performance analytics
- Observability and structured logging
- Health checks beyond database reachability
- Data retention controls
- Compliance exports
- Billing or usage tracking if the product becomes multi-tenant SaaS

## Build Status

The initial pass implements Phase 1. Later phases are deliberately documented as future expansion because they are valuable learning exercises that build on the same module boundaries.

## Phase 1 status (checked against this snapshot)

| Item | Status |
|---|---|
| Tenant-scoped data model | done: `tenantId` on every operational model (`prisma/schema.prisma`) |
| Demo authentication context | done: headers → user (`apps/api/src/middleware/auth.ts`) |
| RBAC helpers | partial: helpers exist, but writes are mostly unchecked (see `docs/architecture/final-system-design.md`, findings 1-2) |
| Customers, companies, contacts | read-only lists; no create/edit routes for customers or companies (customers are created by inbound email or the seed) |
| Ticket queue with search/filter/pagination | done |
| Status, priority, assignment, tags | done; invalid transitions return 500 |
| Internal notes vs public replies | done; internal notes leak to viewers in PATCH responses |
| SLA due date and state | done; pausing does not extend the due date |
| Inbound email ingestion | done; no audit log, no sender verification |
| Outbox and worker | done as a polling loop; no retry or claim |
| Audit logs | done for ticket create, update and comments |
| Seed data | done: one tenant, four users, customers and tickets with both comment types |
| Frontend screens | done: queue, ticket detail, customers, audit |
| CI scripts | done: typecheck, test, build |

Phase 2-4 items have no code yet.

## Exercise

**Goal:** pick the Phase 2 item "Collision indicators when multiple agents work the same ticket" and write its feature plan in the format of `001-initial-operational-core.md`.

**Check:** the plan names the data it needs (for example `Ticket.updatedAt` for optimistic concurrency, or a presence table), the files it touches (`ticketService.ts`, `packages/shared/src/index.ts`, `apps/web/src/App.tsx`), and a test that two conflicting updates do not silently overwrite each other.
