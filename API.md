# API — MicroBiz Flow

Version: 0.1 · Base path `/api/v1` · Last updated: 2026-11-18

Endpoints are implemented milestone by milestone; `DEVELOPMENT_PLAN.md` says which milestone owns
which group. This document is the contract — if the code differs, the code is wrong or this document
needs updating in the same change.

---

## 1. Conventions

### Response envelope

Success:
```json
{ "data": { "id": "…" }, "meta": { "requestId": "018f…" } }
```

Collections:
```json
{
  "data": [ { "id": "…" } ],
  "meta": { "requestId": "018f…", "page": 1, "pageSize": 25, "total": 137 }
}
```

Error:
```json
{
  "error": {
    "code": "PAYMENT_EXCEEDS_BALANCE",
    "message": "Payment of RM 500.00 exceeds the outstanding balance of RM 320.50.",
    "details": [ { "path": "amountSen", "message": "Maximum is 32050" } ],
    "requestId": "018f…"
  }
}
```

- `code` is machine-readable and stable; the UI switches on it. `message` is human-readable English
  and safe to show. `details` carries field-level problems from Zod.
- `requestId` appears in every response and in the matching log line, so a user's screenshot is
  traceable.

### Status codes

| Code | Used for |
|---|---|
| 200 | successful read or update |
| 201 | resource created (with `Location`) |
| 204 | successful delete of a draft |
| 400 | malformed request (unparseable JSON, bad content type) |
| 401 | no valid session |
| 403 | authenticated but lacking the permission |
| 404 | not found **or deliberately hidden** — cross-tenant and unassigned-job access return 404 |
| 409 | conflict or business rule violation (`code` distinguishes them) |
| 422 | request shape valid but values invalid (Zod) |
| 429 | rate limited, with `Retry-After` |
| 500 | unexpected; generic message, logged with `requestId` |

404-not-403 for cross-tenant and technician-isolation cases is deliberate: a 403 confirms the record
exists (FR-A6).

### Error codes

`VALIDATION_FAILED`, `UNAUTHENTICATED`, `SESSION_EXPIRED`, `FORBIDDEN`, `NOT_FOUND`,
`RATE_LIMITED`, `EMAIL_ALREADY_REGISTERED`, `INVALID_CREDENTIALS`, `INVITE_EXPIRED`,
`TOKEN_INVALID`, `TOKEN_EXPIRED`, `TOKEN_ALREADY_USED`, `ILLEGAL_STATE_TRANSITION`,
`QUOTATION_ALREADY_APPROVED`, `QUOTATION_EXPIRED`, `JOB_INCOMPLETE_REPORTS`,
`REQUIRED_READINGS_MISSING`, `ADDITIONAL_WORK_NOT_APPROVED`, `INVOICE_NOT_EDITABLE`,
`PAYMENT_EXCEEDS_BALANCE`, `INVOICE_HAS_PAYMENTS`, `LAST_OWNER_CANNOT_BE_REMOVED`,
`SST_NOT_REGISTERED`, `FILE_TYPE_NOT_ALLOWED`, `FILE_TOO_LARGE`, `IDEMPOTENCY_KEY_REUSED`.

### Pagination, filtering, sorting

`?page=1&pageSize=25` (max 100), `?q=` for search, `?sort=-createdAt` (leading `-` = descending),
plus documented per-resource filters. Offset pagination is deliberate for v0.1 — these are
small datasets with page-number UIs. Revisit only with measurements.

### Money and dates on the wire

- Money is **always** an integer in sen with a `Sen` suffix: `"totalSen": 123450` means RM 1,234.50.
  The API never sends a formatted currency string and never sends a decimal.
- Timestamps are ISO 8601 UTC with `Z`. Calendar dates are `"YYYY-MM-DD"`.
- Phones are returned E.164 (`"+60123456789"`); input accepts local formats and is normalized.

### Idempotency

`POST /payments` and `POST /files/intents` accept an `Idempotency-Key` header. A repeat with the same
key and the same body returns the original result; the same key with a different body returns 409
`IDEMPOTENCY_KEY_REUSED`. Phones double-tap and signal drops cause retries — a duplicated payment is
a real-money bug.

### Auth transport

Session cookie: `httpOnly`, `Secure`, `SameSite=Lax`, path `/`. Mutating requests require the CSRF
token from `GET /auth/csrf` in an `X-CSRF-Token` header and a matching `Origin`. All browser requests
are sent with credentials. See `ARCHITECTURE.md` AD-4 and Decision D1 for the domain requirement.

## 2. Permissions

One matrix in `packages/shared/rbac.ts`, consumed by `requirePermission()` on the server and for
conditional rendering on the client. The server is the control (invariant 2).

| Permission | Owner | Admin | Technician |
|---|:--:|:--:|:--:|
| `business:read` / `business:update` | ✓ / ✓ | ✓ / ✓ | – / – |
| `user:invite` / `user:update` / `user:deactivate` | ✓ | ✓ | – |
| `user:read` | ✓ | ✓ | own record |
| `customer:*` | ✓ | ✓ | read, only via assigned jobs |
| `property:*` | ✓ | ✓ | read, only via assigned jobs |
| `equipment:create` / `equipment:update` | ✓ | ✓ | ✓ (on assigned jobs — technicians find undocumented units on site) |
| `equipment:read` | ✓ | ✓ | only via assigned jobs |
| `catalogue:*` | ✓ | ✓ | read |
| `quotation:*` | ✓ | ✓ | – |
| `job:create` / `job:update` / `job:assign` / `job:cancel` | ✓ | ✓ | – |
| `job:read` | ✓ | ✓ | assigned only |
| `job:execute` (start, complete, report) | ✓ | ✓ | assigned only |
| `report:create` / `report:submit` | ✓ | ✓ | assigned only |
| `additional_work:request` | ✓ | ✓ | ✓ |
| `additional_work:decide` (record verbal approval) | ✓ | ✓ | – |
| `invoice:create` / `invoice:issue` / `invoice:void` | ✓ | ✓ | – |
| `invoice:read` | ✓ | ✓ | – |
| `payment:record` | ✓ | ✓ | – |
| `payment:reverse` | ✓ | – | – |
| `credit_note:issue` | ✓ | – | – |
| `audit:read` | ✓ | ✓ (non-financial) | – |

Two deliberate choices, both worth challenging with pilots:
`payment:reverse` and `credit_note:issue` are Owner-only because they are the two ways money
disappears from the record. Technicians can **create** equipment because the single most common field
reality is finding units nobody documented.

## 3. Endpoints

`A` = Owner/Admin, `T` = Technician (scoped to assigned jobs), `P` = public (token).

### 3.1 Auth and session

| Method | Path | Who | Notes |
|---|---|---|---|
| POST | `/auth/register` | – | Create business + Owner. Rate limited. Returns session cookie |
| POST | `/auth/login` | – | Email + password. Rate limited per email and IP |
| POST | `/auth/logout` | any | Revokes the current session |
| GET | `/auth/me` | any | Current user, role, business summary, permission list |
| GET | `/auth/csrf` | any | Issues the CSRF token |
| GET | `/auth/sessions` | any | Own active sessions |
| DELETE | `/auth/sessions/:id` | any | Revoke one of own sessions |
| POST | `/auth/password` | any | Change own password (requires current) |
| POST | `/auth/password/forgot` | – | Always 204, never reveals whether the email exists |
| POST | `/auth/password/reset` | – | Token + new password |

### 3.2 Business and users

| Method | Path | Who | Notes |
|---|---|---|---|
| GET | `/business` | A | Own business profile |
| PATCH | `/business` | A | Profile, letterhead, terms, payment instructions, SST settings |
| GET | `/users` | A | List members |
| POST | `/users/invites` | A | `{ email, name, role }` → emailed/shareable invite link |
| POST | `/users/invites/accept` | – | `{ token, password, name }` |
| PATCH | `/users/:id` | A | Name, phone, role. Last active Owner cannot be demoted |
| POST | `/users/:id/deactivate` | A | Revokes all their sessions |
| POST | `/users/:id/sessions/revoke` | A | Lost-phone action |

### 3.3 Customers and properties

| Method | Path | Who | Notes |
|---|---|---|---|
| GET | `/customers` | A | `?q=` matches name, normalized phone, property address |
| POST | `/customers` | A | |
| GET | `/customers/:id` | A / T* | T only when reachable from an assigned job |
| PATCH | `/customers/:id` | A | |
| POST | `/customers/:id/deactivate` | A | |
| GET | `/customers/:id/properties` | A / T* | |
| POST | `/properties` | A | |
| GET | `/properties/:id` | A / T* | Includes equipment and access notes |
| PATCH | `/properties/:id` | A | |
| POST | `/properties/:id/transfer` | A | `{ customerId }`; audited; equipment history stays |

### 3.4 Equipment

| Method | Path | Who | Notes |
|---|---|---|---|
| GET | `/equipment` | A | `?propertyId=&customerId=&status=&q=` (q matches serial, model, asset tag) |
| POST | `/equipment` | A / T* | Asset tag generated server-side |
| GET | `/equipment/:id` | A / T* | Unit detail with warranty status |
| PATCH | `/equipment/:id` | A / T* | |
| GET | `/equipment/:id/timeline` | A / T* | Merged reports, readings, jobs, quotation lines, audit — paginated (FR-E3) |
| POST | `/equipment/:id/replace` | A | `{ replacementEquipment }` — creates the new unit and links both |
| POST | `/equipment/:id/relocate` | A | `{ propertyId }`; audited; re-tagged |
| POST | `/equipment/:id/photos` | A / T* | Attach a confirmed file |

### 3.5 Service catalogue

`GET /catalogue`, `POST /catalogue`, `GET /catalogue/:id`, `PATCH /catalogue/:id`,
`POST /catalogue/:id/deactivate`. A — technicians get read-only access for additional-work lines.

### 3.6 Quotations

| Method | Path | Who | Notes |
|---|---|---|---|
| GET | `/quotations` | A | `?status=&customerId=&q=` |
| POST | `/quotations` | A | Draft with lines; totals computed server-side and returned |
| GET | `/quotations/:id` | A | Includes versions and approval evidence |
| PATCH | `/quotations/:id` | A | Drafts only; 409 `QUOTATION_ALREADY_APPROVED` otherwise |
| DELETE | `/quotations/:id` | A | Drafts only |
| POST | `/quotations/:id/send` | A | Assigns number, sets `SENT`, mints a token, returns `{ publicUrl, whatsappText }` |
| POST | `/quotations/:id/revise` | A | Creates version n+1 as draft, supersedes n (FR-Q6) |
| POST | `/quotations/:id/cancel` | A | |
| POST | `/quotations/:id/convert-to-job` | A | Returns the created job |
| GET | `/quotations/:id/versions` | A | Full revision history |

Client-sent totals are ignored. Totals are always recomputed from lines, business SST settings and
`roundSen`; a client that disagrees is a client bug, and a client that is trusted is a money bug.

### 3.7 Jobs and scheduling

| Method | Path | Who | Notes |
|---|---|---|---|
| GET | `/jobs` | A / T | `?date=&from=&to=&status=&technicianId=&customerId=&unassigned=true`. T sees only assigned jobs regardless of filters |
| POST | `/jobs` | A | Standalone or from a quotation |
| GET | `/jobs/:id` | A / T* | Full detail: property, access notes, equipment, lines, reports |
| PATCH | `/jobs/:id` | A | Title, type, priority, instructions, lines |
| POST | `/jobs/:id/schedule` | A | `{ scheduledDate, scheduledSlot }` or explicit start/end |
| POST | `/jobs/:id/assign` | A | `{ assignments: [{ userId, role }] }` — replaces the set, audited |
| POST | `/jobs/:id/equipment` | A | Set the units covered |
| POST | `/jobs/:id/cancel` | A | `{ reason }` |
| GET | `/schedule` | A | Day/week board: `?from=&to=` grouped by technician and slot, plus unassigned |
| GET | `/jobs/today` | T | The technician home screen (FR-T1) — one request, everything it renders |

### 3.8 Technician execution

| Method | Path | Who | Notes |
|---|---|---|---|
| POST | `/jobs/:id/en-route` | T* | |
| POST | `/jobs/:id/start` | T* | Records `actual_start_at` |
| POST | `/jobs/:id/complete` | T* | 409 `JOB_INCOMPLETE_REPORTS` listing units still missing a report |
| POST | `/jobs/:id/signature` | T* | `{ fileId, signedByName }` |
| GET | `/jobs/:id/checklist` | T* | Per-unit checklist with the required readings for this job type |

### 3.9 Service reports

| Method | Path | Who | Notes |
|---|---|---|---|
| POST | `/jobs/:jobId/reports` | T* | Creates or upserts the draft report for one unit |
| PATCH | `/reports/:id` | T* | Drafts only |
| POST | `/reports/:id/readings` | T* | Upsert readings by key |
| POST | `/reports/:id/photos` | T* | Attach confirmed files with `kind` |
| POST | `/reports/:id/submit` | T* | Validates the required set; 422 `REQUIRED_READINGS_MISSING` with the missing keys. Writes condition back to the unit |
| GET | `/reports/:id` | A / T* | |
| POST | `/reports/:id/amend` | A / T* | New report referencing the original (FR-R5) |
| GET | `/jobs/:jobId/reports` | A / T* | |

### 3.10 Additional work

| Method | Path | Who | Notes |
|---|---|---|---|
| POST | `/jobs/:id/additional-work` | T* | `{ description, lines[] }` → `PENDING`, returns `{ publicUrl, whatsappText }` |
| GET | `/additional-work/:id` | A / T* | |
| POST | `/additional-work/:id/record-decision` | A | Staff records a verbal approval: `{ decision, decidedByName }`, attributed to the staff user |
| POST | `/additional-work/:id/cancel` | A / T* | |

### 3.11 Invoices

| Method | Path | Who | Notes |
|---|---|---|---|
| GET | `/invoices` | A | `?status=&customerId=&overdue=true&from=&to=` |
| POST | `/invoices` | A | From a job (`{ jobId }`), a quotation, or standalone lines. Rejects unapproved additional-work lines with `ADDITIONAL_WORK_NOT_APPROVED` |
| GET | `/invoices/:id` | A | Includes payments, credit notes and derived balance |
| PATCH | `/invoices/:id` | A | Drafts only; 409 `INVOICE_NOT_EDITABLE` otherwise |
| DELETE | `/invoices/:id` | A | Drafts only |
| POST | `/invoices/:id/issue` | A | Assigns the number, freezes the document, audits |
| POST | `/invoices/:id/void` | A | `{ reason }`; 409 `INVOICE_HAS_PAYMENTS` if any payment exists |
| GET | `/invoices/outstanding` | A | Ageing buckets by customer (FR-I9) |
| GET | `/invoices/:id/public-view` | A | Printable/shareable rendering payload |

### 3.12 Payments and credit notes

| Method | Path | Who | Notes |
|---|---|---|---|
| POST | `/invoices/:id/payments` | A | `Idempotency-Key`. `{ amountSen, method, reference, paidAt, cashRoundingSen, notes }`. Locks the invoice; 409 `PAYMENT_EXCEEDS_BALANCE` with the actual balance in `details` |
| GET | `/invoices/:id/payments` | A | |
| POST | `/payments/:id/reverse` | Owner | `{ reason }`; append-only reversal |
| POST | `/invoices/:id/credit-notes` | Owner | `{ reason, lines[] }` |
| POST | `/credit-notes/:id/issue` | Owner | |

### 3.13 Files

| Method | Path | Who | Notes |
|---|---|---|---|
| POST | `/files/intents` | any auth, P (signature) | `{ subjectType, subjectId, mimeType, sizeBytes }` → `{ fileId, uploadUrl, expiresAt }` |
| POST | `/files/:id/confirm` | same | Marks `READY` |
| GET | `/files/:id/url` | authorized readers | Short-lived presigned `GET` |

### 3.14 Public (token) endpoints

Mounted under `/api/v1/public`, no session middleware, rate limited per token and per IP.

| Method | Path | Notes |
|---|---|---|
| GET | `/public/quotations/:token` | Narrow DTO: letterhead, lines, equipment covered, totals, terms, validity. 410 on expired/used |
| POST | `/public/quotations/:token/approve` | `{ approvedByName }` — records name, time, IP, user agent |
| POST | `/public/quotations/:token/reject` | `{ reason? }` |
| GET | `/public/additional-work/:token` | Same pattern for on-site extra work |
| POST | `/public/additional-work/:token/approve` | `{ approvedByName }` |
| POST | `/public/additional-work/:token/reject` | `{ reason? }` |

These responses never include customer lists, other documents, internal user names, or ids beyond
what the page renders.

### 3.15 Audit

`GET /audit` (A) — `?entityType=&entityId=&from=&to=&financialOnly=true`. Owner sees everything;
Admin does not see financial reversals and credit notes they cannot perform.

### 3.16 Health

`GET /health` — liveness, no auth, no database query. `GET /health/ready` — checks database
connectivity, used by Render.

## 4. What this API deliberately does not do in v0.1

- No webhooks, no public API keys, no third-party integrations.
- No notification sending. `send` endpoints return `publicUrl` and `whatsappText` for a human to
  paste into WhatsApp; nothing is dispatched automatically.
- No PDF generation server-side. Documents are print-styled HTML pages; the browser makes the PDF.
  Revisit if pilots need attachable files rather than links.
- No bulk import. Equipment onboarding is manual in v0.1; CSV import is a v0.2 candidate once the
  real shape of a pilot's existing records is known.
- No GraphQL, no tRPC. REST plus shared Zod types gives end-to-end type safety without adding a layer.
