# Architecture Plan

## Goal

Build a realistic internal support desk / CRM platform as a modular monolith. The learning goal is to practice how a growing product keeps domain rules, API workflows, data modeling, frontend screens, background processing, and documentation connected without becoming a pile of unrelated files.

## Chosen Stack

- TypeScript across the repo
- React + Vite for the frontend
- Express for the backend API
- Prisma with PostgreSQL for persistence
- A lightweight worker process for outbox/background jobs
- Vitest for domain and service-level tests
- Docker Compose for local infrastructure
- GitHub Actions for CI-shaped verification

## System Shape

The repo is intentionally one deployable codebase with several runtime entry points:

- `apps/api`: HTTP API, auth middleware, route modules, orchestration services
- `apps/web`: dense internal-tool UI for queues, tickets, customers, and audit logs
- `apps/worker`: background consumer for outbox events
- `packages/domain`: reusable business rules with no database dependency
- `packages/shared`: shared validation schemas and DTO types
- `prisma`: database schema and seed data
- `docs`: architecture, feature plans, walkthroughs, and journals

This is a modular monolith because most enterprise learning happens inside the boundaries: where rules live, how transactions are shaped, how tenant isolation is enforced, and how workflows evolve without splitting into services prematurely.

## Initial Domain Model

Core entities:

- Tenant: the isolation boundary for all product data
- User: internal staff member with role-based permissions
- Company: account-level customer organization
- Customer: support contact
- Ticket: workflow object with status, priority, tags, SLA due date, and assignment
- TicketComment: public replies and internal notes
- EmailMessage: inbound/outbound email records
- AuditLog: append-only record of important actions
- OutboxEvent: transactional event queue for worker processing

## Initial Workflows

1. Agents browse and filter tickets.
2. Agents inspect customer/company context.
3. Agents update ticket status, priority, and assignment.
4. Agents add public replies or internal notes.
5. Public replies enqueue notification events.
6. Inbound email creates a customer if needed, creates a ticket, stores the message, and emits an outbox event.
7. Managers and admins review audit logs.

## Design Principles

- Backend authorization is the source of truth.
- Tenant isolation is applied in every query that touches customer or ticket data.
- Domain rules live outside Express route handlers.
- Route handlers validate input before calling services.
- Transactions create the primary change, audit log, and outbox event together.
- Frontend role checks are convenience only; they do not replace API authorization.
- Documentation explains the why, not just the what.

## Deliberate Simplifications

- Authentication uses seeded demo users via request headers. This keeps the first version focused on authorization, tenant isolation, and workflow boundaries. A production auth provider is a backlog feature.
- The worker logs notifications instead of integrating with a real email provider. The outbox boundary is still present, so the future integration point is realistic.
- Search uses database filtering. Full-text indexing is deferred until the product has enough search behavior to justify the added operational complexity.

## Plan versus code (checked against this snapshot)

| Plan | In the code |
|---|---|
| Express API, React + Vite, Prisma + PostgreSQL, worker, Vitest, Docker Compose, GitHub Actions | All present. Compose also starts Redis, which nothing uses. |
| `packages/domain`, `packages/shared` | Present as source folders imported through TypeScript path aliases, not as separate npm packages. |
| Entities: Tenant, User, Company, Customer, Ticket, TicketComment, EmailMessage, AuditLog, OutboxEvent | Exactly these nine models (`prisma/schema.prisma`). |
| Workflow 1-7 | All seven exist as routes: list and filter tickets, customer and company lists, ticket update, comments, public-reply events, inbound email, audit logs (managers and admins only). |
| "Backend authorization is the source of truth" | Partly: tenant scoping is consistent, but role checks exist only for status transitions, assignment, internal notes and the audit log. Viewers can create tickets, change priority and tags, and send public replies. |
| "Tenant isolation is applied in every query" | Yes for reads and the lookups before writes. Updates then use `where: { id }` after a tenant-scoped lookup (`apps/api/src/modules/tickets/ticketService.ts:144-149`, `:168-169`), which is safe as long as the lookup stays. |
| "Route handlers validate input before calling services" | Yes for tickets, comments and inbound email (Zod). The customer search string is not validated beyond its type. |
| "Transactions create the primary change, audit log, and outbox event together" | Yes for ticket create, update and comment. Inbound email upserts the customer outside the transaction and writes no audit log. |

## Exercise

**Goal:** turn "Backend authorization is the source of truth" into a table: for each route, which roles may call it today and which should.

**Check:** your table has at least these differences to fix: viewers can `POST /api/tickets`, `PATCH /api/tickets/:id` (priority, tags), `POST /api/tickets/:id/comments` with `PUBLIC`, and `POST /api/notifications/inbound-email`; and every role can `GET /api/admin/users`.
