# Security

Clinical data demands boring, verifiable security. The model, in layers:

## Tenant isolation

- Every tenant-owned table carries `laboratoryId`, and every service query filters by it. This is a hard code review checklist item, not a convention.
- Branch is a second explicit scoping axis inside a laboratory.
- Platform-scoped roles (super admin, global catalog editor) are never assignable from a lab context; the service layer enforces it.

## Authorization

- A single composed guard runs the full chain in order: public bypass, JWT decode, subscription / trial / email-verification gate, CASL policy evaluation against `@RequirePermission()` metadata.
- Permissions travel in the JWT and are validated against code-defined registries; there are no permission tables to drift.
- The frontend renders navigation and actions from the same claims the backend enforces, so the UI never offers what the API would reject.

## Public results access

Patients open results from a QR code, without accounts. The design (full record in ADR-005):

- A signed JWT carries `{ orderId, scope: 'public:read', exp }`. No order id ever appears in a URL, so enumeration is impossible.
- A stored token hash enables revocation; expiry is configurable per laboratory.
- The scope is read-only and public endpoints are rate limited.

## Signup

Verify-first registration: submitting the form creates a pending registration, not an account. The real laboratory and user are created only when the email OTP is confirmed, so unverified signups leave zero rows behind. Disposable email domains are blocked.

## Data safety

- Migrations are additive only; destructive operations against databases are policy-forbidden.
- Soft deletes everywhere (`deletedAt`), no hard deletes outside explicit cascades.
- Scripted, dated production backups with a disciplined prod-to-dev restore flow.
- Hardened HTTP security headers; the XSS surface is minimal by construction (React escaping, no `dangerouslySetInnerHTML` in user-data paths).
