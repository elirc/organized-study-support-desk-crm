# Feature Plan: RBAC And Tenant Isolation

## Goal

Ensure that support data is scoped to a tenant and that role checks are enforced by the backend.

## Impacted Areas

- `apps/api/src/middleware/auth.ts`
- `apps/api/src/modules/tickets/ticketService.ts`
- `apps/api/src/modules/admin/adminRoutes.ts`
- `packages/domain/src/index.ts`
- `prisma/schema.prisma`

## Implementation Approach

Resolve the actor once in middleware, store a small auth context on the request, and require services to use `auth.tenantId` in database queries. Keep reusable permission helpers in the domain package so tests can protect them.

## Design Considerations

Frontend role checks can improve UX, but they are not security. The API must reject unauthorized actions even if a user bypasses the UI.

## Build Journal

### What Was Built

Demo auth resolves users by tenant slug and email. Ticket listing, detail, creation, updates, customer listing, and audit logs all scope data by tenant. Admin audit logs require manager or admin roles.

### How It Connects

The middleware produces `req.auth`, and service modules consume that context. The Prisma schema includes `tenantId` on the operational tables so isolation is represented directly in the model.

### Lessons For A Growing Developer

Authorization is not a single if statement. It is a habit across data modeling, query construction, service design, and tests.

### Tradeoffs And Future Improvements

The current role model is coarse. A future version should add permission grants, teams, queue membership, and tests that prove cross-tenant records cannot be accessed.

## As built (checked against this snapshot)

- Actor resolution: `authMiddleware` (`apps/api/src/middleware/auth.ts:19-41`) builds `req.auth = { tenantId, userId, role, email }`.
- Tenant scoping: every query in `ticketService.ts`, `customerRoutes.ts`, `adminRoutes.ts` and `notificationRoutes.ts` filters by `req.auth.tenantId` or looks the record up with it first.
- Roles checked: status transitions (`packages/domain/src/index.ts:32-37` — viewers can change nothing, agents may close only from `RESOLVED` or `PENDING`), assignment (`canAssignTicket`, admins, managers and agents), internal notes (viewers may not write them, `ticketService.ts:208-210`, and do not see them in `GET`, `:22-24`), audit logs (`requireRole(["ADMIN", "MANAGER"])`, `adminRoutes.ts:22`).

**Gaps.** The build journal says "admin audit logs require manager or admin roles" — true — but nothing else is role-gated: viewers can create tickets, change priority and tags, send public replies and ingest email, and every role can list users. Internal notes leak to viewers through the `PATCH` response. The "tests that prove cross-tenant records cannot be accessed" listed as future work still do not exist; the seed has only one tenant to test against.

## Exercises

1. **Goal:** add a second tenant to `prisma/seed.ts` with its own agent and ticket. **Check:** as that agent, `GET /api/tickets/<an acme-support ticket id>` returns 404.
2. **Goal:** add `requireRole(["ADMIN", "MANAGER", "AGENT"])` to ticket writes and inbound email. **Check:** the viewer gets 403 for `POST /api/tickets` and `PATCH /api/tickets/:id`, but still 200 for `GET /api/tickets`.
