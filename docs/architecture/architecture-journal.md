# Architecture Journal

## 2026-05-04 - Initial System Spine

The first implementation creates the product spine rather than a toy screen. The important choice is that ticket workflow rules live in `packages/domain`, not inside React components or Express route callbacks. A junior developer should notice that this makes the rules testable without a web server or database.

The API is organized by product module: tickets, customers, notifications, and admin. The ticket module owns the highest-risk workflow because ticket updates touch status transitions, assignment, SLA, audit logging, and outbox events. That is why the route file is thin and the service file carries orchestration.

Tenant isolation is represented by `tenantId` on operational tables. The current auth layer resolves a demo user from `x-tenant-slug` and `x-user-email`, then every module uses `req.auth.tenantId`. This is not production authentication, but it teaches the correct data-access habit: queries should be scoped by tenant as close to the persistence layer as possible.

The outbox table was added early because notifications and integrations are common places where enterprise apps become unreliable. Creating the ticket and event in one transaction means the worker can be retried independently without losing the fact that something happened.

The frontend is intentionally dense and workflow-oriented. It is not a marketing page. The first screen is the working queue because support agents live in queues, filters, details, status changes, and replies.

## 2026-05-04 - Verification Discipline

The first tests cover ticket transition rules, SLA state, and authorization helpers. These tests do not prove the whole app works, but they protect the rules most likely to create product bugs when future features arrive.

CI runs typecheck, tests, and build. Database-backed integration tests are deferred until migrations are stabilized, because the first local setup should remain approachable.

## 2026-10 - Snapshot review

The two entries above are the author's record; the repository is a single-commit snapshot. Checked against the code:

- **"Ticket workflow rules live in `packages/domain`"** — transitions, SLA math and SLA state do (`packages/domain/src/index.ts`). The visibility rule for internal notes is duplicated inline in the service (`visibleCommentWhere`), and the domain's `canViewInternalNote` is unused.
- **"The route file is thin and the service file carries orchestration"** — true: `ticketRoutes.ts` only parses input and calls the service. The other modules (customers, notifications, admin) query Prisma directly from their routes; that is fine at their size, but it is not the same pattern.
- **"Every module uses `req.auth.tenantId`"** — true for every query in this snapshot.
- **"Creating the ticket and event in one transaction"** — true for create, update and comments. The worker side is the weak half: no claim step, no retry of `FAILED` events, and the loop stops if a pass throws.
- **"CI runs typecheck, tests, and build"** — the workflow does (`.github/workflows/ci.yml`); whether it passed is not recorded in this snapshot. It has no database, so it cannot run integration tests even if some were added.

The lesson worth adding: **role checks drift unless each write route says who may call it.** Here only status changes, assignment and internal notes are checked, so read-only "viewers" can still change tickets and talk to customers.

## Exercise

**Goal:** write the next journal entry for a decision you will make: where role checks for writes live (route middleware with `requireRole`, or service functions with a domain helper such as `canEditTicket(role)`).

**Check:** after implementing it, every write route rejects `viewer@acme.test` with 403, and a domain test covers the new helper.
