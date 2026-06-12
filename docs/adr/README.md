# Architecture Decision Records (summaries)

The private codebase is governed by 11 ADRs. Each records the decision, the rationale, the trade-offs accepted and a review trigger that defines when to revisit. These are condensed versions.

A recurring theme: **pragmatism over ceremony**. Several ADRs deliberately reject patterns (repositories, rich domain entities, unit-of-work) that would be premature at this scale, but each rejection is written down with the conditions under which it should be reconsidered.

---

## ADR-001 · Direct Prisma usage in services

Services use Prisma directly, no repository abstraction. Prisma already provides a clean, type-safe API; a repository layer would be boilerplate at this scale. Accepted trade-off: tighter ORM coupling. Revisit if multiple databases or a larger team appear.

## ADR-002 · Types-only domain layer

The domain layer contains types and interfaces, not rich domain entities. Business logic lives in services; DTO validation via `class-validator` is sufficient. Revisit if complex invariants emerge or the same validation duplicates 3+ times.

## ADR-003 · Prisma transactions in services

Explicit `prisma.$transaction()` calls in services, no Unit of Work abstraction. Simple and visible at the call site.

## ADR-004 · Query-parameter modals (deferred)

Modal state via URL query params instead of a global store. Deferred to post-launch; the current pattern is sufficient.

## ADR-005 · Public results access tokens

Passwordless patient access to results via QR. A signed JWT carries `{ orderId, scope: 'public:read', exp }`; a stored hash enables revocation; expiry is configurable per laboratory. No order id ever appears in the URL (prevents enumeration), the scope is read-only, and public endpoints are rate limited.

## ADR-006 · Exam personalization: metadata, forking, shadowing

The 3-tier strategy that keeps one global catalog serving thousands of per-lab variants:

1. **Metadata overrides** in pivot tables for commercial data (name, price, turnaround). No structural duplication.
2. **Implicit forking**: the exam tree is auto-cloned only when a lab mutates structure, never on metadata edits.
3. **Rule shadowing**: lab-scoped reference range rules override global ones without duplicating analytes.

Constraint that keeps it testable: the range resolver stays a pure filter pipeline; merge logic lives in the engine service.

## ADR-007 · PBAC (Plan-Based Access Control)

Authorization is a 3-tier chain: Plan → Claims → Quotas, with CASL as policy engine. A single composed `APP_GUARD` runs the ordered stages: public-route check, JWT decode, subscription/trial/email gate for tenant users, CASL policy check. Platform-scope roles are not tenants, so tenant gating does not apply to them; their claims go through the same policy evaluation. The JWT carries `plan` and `permissions[]`, so authorization needs zero DB hits per request. Claims are raw strings validated against registries defined in backend code, rather than permission tables in the database (nothing to drift). Enforcement is server-side on every request: `@RequirePermission()` guards every protected route, and the frontend's permission-driven UI is a mirror for UX, never the check itself.

This ADR also encodes a hard-won lesson: an earlier standalone subscription guard silently no-opped because it ran before the JWT guard populated `request.user`. Any check depending on the authenticated user must run after JWT decoding inside the same guard.

## ADR-008 · Catalog classification per NOM-007-SSA3-2011

The exam catalog uses the category taxonomy of the official Mexican clinical lab regulation (NOM-007-SSA3-2011) as the authoritative standard, cross-mapped to LOINC classes. This mirrors how labs actually declare service areas to the regulator (COFEPRIS) and matches the taxonomy of major Mexican private labs. One operative constraint: PCR / viral load exams belong to Molecular Biology, not Microbiology, because technique, equipment and regulation all separate them.

## ADR-009 · Frontend: feature-first structure + decomposition

Code is organized by feature domain (`features/<domain>/`), not by type; only genuinely shared code lives at the root. Hard rules: pages are coordinators only (~150 lines max), 500-line hard limit per component, modals always in separate files, no barrel files, tab contents are components. Data flow: TanStack Query owns all server state, mutations always invalidate, Server Components by default.

## ADR-010 · Backend: Clean Architecture Light

Every module follows `application/` (services, all business logic) + `http/` (controllers, DTOs) + `domain/types/`. Controllers are thin: parse, delegate, respond. DTOs validate every field. Soft deletes everywhere, every query filters by `laboratoryId`, user-facing errors in Spanish, code in English.

## ADR-011 · Permission-driven UI

The frontend derives navigation and available actions from the same claims the backend enforces, so the UI never shows actions the API would reject.
