# Architecture

Lab2Next is a pnpm monorepo with three deployables: a NestJS API, a Next.js application and a marketing landing. One PostgreSQL database, tenant-partitioned by `laboratoryId`.

## System overview

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TB
    subgraph Edge
        APP[Next.js 16 app<br/>app.lab2next.com]
        PORTAL[Public results portal<br/>token-gated routes]
        LAND[Landing<br/>lab2next.com]
    end

    subgraph API [NestJS 11 API · three layers per module]
        subgraph HTTPL [http/ layer]
            GUARD[UnifiedAuthGuard<br/>single APP_GUARD]
            CTRL[Thin controllers + DTOs<br/>orders · patients · catalog · billing · reports · permissions]
        end
        subgraph APPL [application/ layer]
            SVC[Services<br/>business logic + transactions]
            ENGINE[Exam engine<br/>range resolver · calculated fields · flagging]
            PDF[PDF report generator]
        end
        subgraph DOML [domain/ + data]
            TYPES[Domain types<br/>pure, no logic]
            PRISMA[Prisma<br/>tenant-scoped queries]
        end
    end

    subgraph Data
        PG[(PostgreSQL)]
    end

    subgraph External
        STRIPE[Stripe]
        MAIL[Transactional email]
        WA[WhatsApp share links]
    end

    APP --> GUARD
    PORTAL --> GUARD
    GUARD --> CTRL --> SVC
    SVC --> ENGINE
    SVC --> PDF
    SVC -.-> TYPES
    SVC --> PRISMA --> PG
    SVC --> STRIPE
    SVC --> MAIL
    APP -.-> WA
```

## Multi-tenancy

Single database, shared schema, strict row-level discipline:

- Every tenant-owned table carries `laboratoryId`. Every service query filters by it; code review enforces this as a hard checklist item.
- Branch (`sucursal`) is a second, explicit scoping axis inside a laboratory: orders, appointments, capacity and price lists are branch-aware.
- The global exam catalog is the one deliberate exception: it is shared, and per-lab customization happens through overrides and forks (see below), never by mutating global rows.

## Authorization: PBAC in a single guard

A 3-tier chain, Plan → Claims → Quotas, evaluated by one composed `APP_GUARD`:

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart LR
    R[Request] --> P{"@Public()?"}
    P -- yes --> OK[Allow]
    P -- no --> J[Decode JWT<br/>header or cookie]
    J -- invalid --> D401[401]
    J --> SUB[Subscription / trial /<br/>email-verification gate<br/>tenant scope]
    SUB -- blocked --> D403a[403]
    SUB --> CASL["CASL policy check<br/>JWT permissions[] vs<br/>@RequirePermission()"]
    CASL -- deny --> D403b[403]
    CASL -- allow --> OK
```

Platform-scope roles (the super admin operating the platform itself) are not tenants, so the tenant subscription gate does not apply to them; their access is still resolved by the same policy evaluation, against platform-scope claims that are never assignable from a lab context.

Key properties:

- **Zero DB hits per request**: the JWT carries `plan` and `permissions[]`; policies evaluate against the token, not the database.
- **One guard, ordered stages**: an earlier iteration registered the subscription check as an independent `APP_GUARD` and it silently no-opped because `request.user` did not exist yet. The fix (and the lesson) became the unified guard: any check that depends on `request.user` must run after JWT decoding inside the same guard chain.
- **Permission-driven UI, backend-enforced**: the frontend renders navigation and actions from the same claims so users never see dead-end buttons, but it is only a mirror. Every protected route is enforced server-side by the guard chain; stripping the UI would change nothing about what the API allows.

## The exam engine

The hardest design problem in the product: one curated global catalog, thousands of per-lab customizations, no duplication explosion.

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TB
    GC[Global catalog<br/>exams · sections · analytes · range rules]
    M[Tier 1 · Metadata overrides<br/>per-lab pivot tables: name, price, turnaround]
    F[Tier 2 · Implicit fork<br/>exam tree cloned only on structural mutation]
    S[Tier 3 · Rule shadowing<br/>lab-scoped range rules override global ones]
    RES[ReferenceRangeResolver<br/>pure filter pipeline]
    OUT[Effective exam for one lab]

    GC --> M --> OUT
    GC --> F --> OUT
    GC --> S --> RES --> OUT
```

- Metadata edits never fork. Forking is triggered exclusively by structural mutation, so the common case (rename, reprice) is a single pivot row.
- Reference range rules support multi-axis conditions (age, sex, physiological phase) and resolve through a pure pipeline; merge logic lives in the engine service, keeping the resolver testable in isolation.
- Result capture evaluates each value against the resolved ranges and flags H/L automatically; calculated analytes derive from sibling values.

## Public results access

Patients get results without accounts or apps:

- A signed JWT (`orderId`, read-only scope, expiry) is encoded into a QR / shareable link.
- No order id in the URL, so enumeration is impossible; a stored token hash enables revocation; expiry is configurable per laboratory.
- Public endpoints are rate limited.

## Order lifecycle and state

```mermaid
%%{init: {'theme': 'neutral'}}%%
stateDiagram-v2
    [*] --> Pending: order created
    Pending --> InProcess: samples collected
    InProcess --> Captured: results entered
    Captured --> Validated: chemist validates
    Validated --> Delivered: PDF + portal/WhatsApp
    Delivered --> [*]
```

Payment status tracks in parallel with operational status, so reception can collect before, during or after processing.

## Backend layout: Clean Architecture Light

Deliberately pragmatic. Full rules in the [ADR summaries](adr/README.md), the short version:

- Controllers are thin: parse request, call service, return response.
- All business logic lives in `application/` services that use Prisma directly (no repository layer: a documented, revisitable decision).
- Domain layer is types only. Transactions are explicit `prisma.$transaction()` calls in services.
- Soft deletes everywhere, additive-only migrations.

## Frontend layout: feature-first

- Code lives in `features/<domain>/`, shared code must earn its place at the root.
- Pages are coordinators only (~150 lines max), tabs and modals are separate files, hard 500-line limit per component.
- TanStack Query owns all server state; mutations always invalidate.
- Server Components by default, `"use client"` only where interactivity requires it.

## Quality and safety nets

- Playwright E2E harness drives the real app (register, verify, orders, results) with video recording.
- Validation discipline: field rules are defined symmetrically on the frontend (trim/normalize) and backend (DTO decorators), with the invariant that backend caps are always >= frontend caps.
- Production backups and dev refresh are scripted and dated; destructive operations against databases are policy-forbidden.
