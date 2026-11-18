# DATABASE — MicroBiz Flow

Version: 0.1 · PostgreSQL 16 · Prisma · Last updated: 2026-11-18

This is the intended schema for v0.1. Tables are introduced milestone by milestone, not all at once;
`DEVELOPMENT_PLAN.md` says which milestone owns which table. Where this document and
`packages/db/prisma/schema.prisma` disagree, the schema is the truth and this document is a bug.

---

## 1. Conventions

| Concern | Rule |
|---|---|
| Naming | `snake_case` tables and columns, plural table names, `camelCase` in TypeScript via Prisma `@map` |
| Primary keys | `uuid` — `gen_random_uuid()` (pgcrypto); application-side `crypto.randomUUID()` for created rows |
| Tenancy | `business_id uuid NOT NULL` on every business-owned table, first column of its main index |
| Money | `integer`, column name suffixed `_sen`. Never `numeric`, never `double precision` |
| Quantities | `numeric(10,2)` — quantities can be fractional (1.5 hours), money cannot |
| Timestamps | `timestamptz`, UTC. `created_at` default `now()`, `updated_at` maintained by Prisma |
| Calendar days | `date` where the value means a day in Malaysia, not an instant |
| Enums | Postgres enums, `SCREAMING_SNAKE` values, mirrored as Zod unions in `packages/shared` |
| Deletes | Soft (`is_active`, `archived_at`) for master data; **never** for financial documents |
| Text | `text`, with length limits enforced in Zod rather than `varchar(n)` |

### Tenant-owned tables

Every table below except `users`' global email index, `sessions`, and `businesses` itself is
tenant-owned and appears in the `TENANT_OWNED` list consumed by `tenantClient()` (see
`ARCHITECTURE.md` §4). Adding a table means adding it to that list or deliberately excluding it.

## 2. Entity relationships

```
businesses ─┬─< users ─────────< sessions
            ├─< customers ─────< properties ──< equipment ──< equipment_photos
            │                        │              │
            │                        │              └──< (referenced by lines, jobs, reports)
            ├─< service_items
            ├─< quotations ────< quotation_lines
            │       │
            │       └──< approval_tokens
            ├─< jobs ─┬─< job_assignments  (→ users)
            │         ├─< job_equipment    (→ equipment)
            │         ├─< job_lines
            │         ├─< additional_work_requests ──< additional_work_lines
            │         └─< service_reports ─┬─< inspection_readings
            │                              └─< report_photos
            ├─< invoices ─┬─< invoice_lines
            │             ├─< payments
            │             └─< credit_notes ──< credit_note_lines
            ├─< files
            ├─< number_sequences
            └─< audit_events   (append-only)
```

## 3. Tables

### 3.1 businesses

The tenant root.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| name | text | trading name, appears on documents |
| slug | text unique | reserved for future per-business URLs |
| registration_no | text null | SSM number |
| tin | text null | LHDN tax identification number — printed on invoices so corporate customers can self-bill |
| msic_code | text null | industry code; carried for future MyInvois compatibility |
| sst_registered | boolean default false | **opt-in**; most pilots are under the RM500k threshold |
| sst_reg_no | text null | required when `sst_registered` |
| sst_rate_basis_points | integer default 800 | 800 = 8%; stored so a rate change does not rewrite history |
| phone, email | text | contact on documents |
| address_line1/2, city, state, postcode | text / `my_state` enum | postcode exactly 5 digits (Zod) |
| logo_file_id | uuid null → files | letterhead |
| timezone | text default `Asia/Kuala_Lumpur` | display timezone |
| currency | text default `MYR` | single currency in v0.1; column exists to avoid a later table rewrite |
| invoice_terms, quotation_terms | text null | default document footers |
| payment_instructions | text null | bank account / DuitNow details printed on invoices |
| created_at, updated_at | timestamptz | |

Check constraint: `sst_registered = false OR sst_reg_no IS NOT NULL`.

### 3.2 users

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id | uuid → businesses | one business per user in v0.1 (AD-5) |
| email | text, **unique globally** | lowercased on write |
| password_hash | text null | null until an invite is accepted |
| name, phone | text | |
| role | `user_role` enum | `OWNER`, `ADMIN`, `TECHNICIAN` |
| is_active | boolean default true | deactivation revokes sessions (FR-A7) |
| invited_by | uuid null → users | |
| invite_token_hash | text null | SHA-256; see §5 |
| invite_expires_at | timestamptz null | |
| last_login_at | timestamptz null | |
| created_at, updated_at | timestamptz | |

Indexes: `unique(email)`, `(business_id, role)`, `(business_id, is_active)`.
Constraint: a business must always retain at least one active `OWNER` — enforced in the service layer
with a test, because it is a cross-row rule.

`CUSTOMER` is deliberately **not** a role in v0.1: the customer portal is deferred and quotation
approval uses tokens, not accounts. Adding it later is additive.

### 3.3 sessions

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | the opaque value in the cookie is a separate random token, not this id |
| user_id | uuid → users | |
| token_hash | text unique | SHA-256 of the cookie token |
| expires_at | timestamptz | |
| revoked_at | timestamptz null | |
| ip, user_agent | text null | shown in a "your sessions" view |
| created_at, last_seen_at | timestamptz | |

Index: `(user_id, revoked_at)`, `token_hash`.

### 3.4 customers

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id | uuid → businesses | |
| type | `customer_type` enum | `INDIVIDUAL`, `COMPANY` |
| name | text | person or company name |
| company_reg_no, tin | text null | for company customers |
| phone, alt_phone | text null | E.164 normalized |
| email | text null | |
| billing_address_line1/2, billing_city, billing_state, billing_postcode | | may differ from any property |
| notes | text null | |
| is_active | boolean default true | |
| created_at, updated_at | timestamptz | |

Indexes: `(business_id, is_active)`, `(business_id, phone)`, and a trigram index on `name` for
search. Search by phone matches on the normalized form (FR-C2).

### 3.5 properties

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, customer_id | uuid | |
| label | text | "Home", "Shop lot Taman Desa", "Level 3 office" |
| address_line1/2, city, state, postcode | | |
| latitude, longitude | numeric null | optional, powers maps deep links |
| property_type | `property_type` enum | `RESIDENTIAL`, `SHOP`, `OFFICE`, `INDUSTRIAL`, `OTHER` |
| access_notes | text null | gate code, which lift, who to ask for — shown prominently to technicians (FR-C4) |
| created_at, updated_at | timestamptz | |

Index: `(business_id, customer_id)`.

Customer transfer (FR-C3) updates `customer_id` and writes an `audit_events` row; equipment and its
history stay attached to the property.

### 3.6 equipment

The spine of the product.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, property_id | uuid | |
| asset_tag | text | system-generated, unique per property, printable (FR-E2) |
| location_label | text | "Master bedroom", "Front-left cassette" — how a human finds it |
| type | `equipment_type` enum | `WALL_SPLIT`, `CASSETTE`, `CEILING_EXPOSED`, `DUCTED`, `WINDOW`, `PORTABLE`, `VRF_INDOOR`, `OUTDOOR_CONDENSER`, `OTHER` |
| brand, model, serial_no | text null | **all optional** — nameplates are often unreadable (FR-E1) |
| capacity_btu | integer null | 9000, 12000, 18000… |
| refrigerant | `refrigerant_type` enum null | `R22`, `R32`, `R410A`, `R290`, `OTHER` |
| is_inverter | boolean null | |
| installed_on | date null | |
| warranty_expires_on | date null | surfaced in quotations and jobs (FR-E6) |
| condition | `equipment_condition` enum null | `GOOD`, `FAIR`, `POOR`, `FAULTY` — last assessed condition |
| status | `equipment_status` enum | `ACTIVE`, `REMOVED`, `REPLACED` |
| replaced_by_equipment_id | uuid null → equipment | preserves both histories (FR-E4) |
| notes | text null | |
| created_at, updated_at | timestamptz | |

Indexes: `unique(property_id, asset_tag)`, `(business_id, property_id, status)`,
`(business_id, serial_no)` for lookup when a serial is readable.

Relocation (FR-E5) changes `property_id` and writes an audit event; the asset tag is regenerated for
the new property and the old tag recorded in the audit row.

The timeline in FR-E3 is a query, not a table: service reports, inspection readings, job links,
quotation lines and audit events for that equipment id, merged and sorted. Materialize it only if
measurement shows a problem.

### 3.7 equipment_photos

`id, business_id, equipment_id, file_id → files, kind (NAMEPLATE | UNIT | LOCATION | OTHER), caption,
created_at`.

### 3.8 service_items (catalogue)

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id | uuid | |
| code | text | unique per business |
| name, description | text | |
| kind | `service_item_kind` enum | `SERVICE`, `PART`, `LABOUR`, `OTHER` |
| unit | `unit_of_measure` enum | `JOB`, `UNIT`, `HOUR`, `PIECE` |
| unit_price_sen | integer | |
| is_sst_taxable | boolean default true | per-item taxability (FR-S1) |
| is_active | boolean default true | |
| created_at, updated_at | timestamptz | |

Index: `unique(business_id, code)`.

### 3.9 quotations

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, customer_id, property_id | uuid | |
| quote_no | text | `QT-2026-0001`, gapless per business |
| version | integer default 1 | |
| parent_quotation_id | uuid null → quotations | previous version (FR-Q6) |
| status | `quotation_status` enum | `DRAFT`, `SENT`, `APPROVED`, `REJECTED`, `EXPIRED`, `SUPERSEDED`, `CANCELLED` |
| subtotal_sen, discount_sen, sst_sen, total_sen | integer | all derived, all stored |
| sst_rate_basis_points | integer null | rate snapshot at issue |
| valid_until | date | |
| notes, terms | text null | |
| created_by | uuid → users | |
| sent_at | timestamptz null | |
| approved_at | timestamptz null | |
| approved_by_name | text null | typed by the customer on the public page |
| approval_ip, approval_user_agent | text null | approval evidence (FR-Q5) |
| rejected_at, rejection_reason | | |
| created_at, updated_at | timestamptz | |

Indexes: `unique(business_id, quote_no, version)`, `(business_id, status)`,
`(business_id, customer_id)`.

Immutability: once `status = APPROVED`, the row and its lines are frozen. The service layer rejects
mutation; a revision inserts a new quotation with `version = n+1`, `parent_quotation_id` set, and
flips the parent to `SUPERSEDED`.

### 3.10 quotation_lines

`id, quotation_id, business_id, line_no, service_item_id null, equipment_id null, description (text,
snapshotted), quantity numeric(10,2), unit, unit_price_sen, is_sst_taxable (snapshotted),
line_subtotal_sen, created_at`.

`service_item_id` is a reference for reporting only — `description`, `unit_price_sen` and
`is_sst_taxable` are **snapshots** so catalogue edits never alter an existing document (FR-S2).

### 3.11 approval_tokens

One mechanism for both public approval surfaces (§5).

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id | uuid | |
| subject_type | `approval_subject` enum | `QUOTATION`, `ADDITIONAL_WORK` |
| subject_id | uuid | |
| token_hash | text unique | SHA-256 of a 32-byte random value |
| expires_at | timestamptz | min(document validity, created + 14 days) |
| used_at | timestamptz null | set on approve or reject — single decision |
| revoked_at | timestamptz null | superseding a document revokes its tokens |
| created_by | uuid → users | |
| created_at | timestamptz | |

Index: `token_hash`, `(subject_type, subject_id)`.

### 3.12 jobs

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, customer_id, property_id | uuid | |
| quotation_id | uuid null → quotations | source, when converted |
| job_no | text | `JOB-2026-0001` |
| title | text | |
| type | `job_type` enum | `INSTALLATION`, `SERVICE`, `REPAIR`, `INSPECTION`, `MAINTENANCE` |
| priority | `job_priority` enum | `LOW`, `NORMAL`, `HIGH`, `URGENT` |
| status | `job_status` enum | see §4.2 |
| scheduled_date | date null | a day in Malaysia |
| scheduled_slot | `time_slot` enum null | `MORNING`, `AFTERNOON`, `EVENING`, `SPECIFIC` (FR-J2) |
| scheduled_start_at, scheduled_end_at | timestamptz null | used when slot is `SPECIFIC` |
| actual_start_at, actual_end_at | timestamptz null | set by the technician |
| instructions | text null | office → technician |
| completion_notes | text null | |
| customer_signature_file_id | uuid null → files | |
| signed_by_name | text null | |
| cancelled_at, cancellation_reason | | |
| created_by | uuid → users | |
| created_at, updated_at | timestamptz | |

Indexes: `unique(business_id, job_no)`, `(business_id, scheduled_date, scheduled_slot)`,
`(business_id, status)`, `(business_id, customer_id)`.

Check: `scheduled_slot = 'SPECIFIC'` requires both `scheduled_start_at` and `scheduled_end_at`.

### 3.13 job_assignments

`id, business_id, job_id, user_id → users, role (LEAD | HELPER), assigned_by, assigned_at,
unassigned_at null`.

Index: `unique(job_id, user_id) where unassigned_at is null`, and `(user_id, job_id)` — this index
drives the technician's job list and the isolation predicate, so it matters.

### 3.14 job_equipment

`id, business_id, job_id, equipment_id, created_at`. `unique(job_id, equipment_id)`.

This is the set FR-J4 generates the technician checklist from, and the set FR-T3 requires a submitted
report for.

### 3.15 job_lines

Planned work, carried from the quotation or added by the office. Same shape as `quotation_lines`
(snapshotted description, price, taxability) plus `source` (`QUOTATION`, `MANUAL`,
`ADDITIONAL_WORK`) and `additional_work_request_id null`. These lines are what the invoice is built
from.

### 3.16 additional_work_requests

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, job_id | uuid | |
| description | text | what and why, written by the technician |
| amount_sen | integer | sum of its lines |
| status | `approval_status` enum | `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED` |
| requested_by | uuid → users | the technician |
| requested_at | timestamptz | |
| approval_method | `approval_method` enum null | `CUSTOMER_LINK`, `RECORDED_VERBAL` |
| decided_at | timestamptz null | |
| decided_by_name | text null | customer's typed name, or the staff member who recorded a verbal approval |
| decided_by_user_id | uuid null → users | set only for `RECORDED_VERBAL` |
| decision_ip, decision_user_agent | text null | |
| rejection_reason | text null | |

`additional_work_lines` mirrors `quotation_lines`.

FR-T6 / invariant 11: a job line with `source = ADDITIONAL_WORK` is only invoiceable when its request
is `APPROVED`. Enforced in the invoicing service with a test, and in the UI by hiding unapproved
lines from the invoice builder.

### 3.17 service_reports

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, job_id, equipment_id | uuid | |
| technician_id | uuid → users | |
| performed_at | timestamptz | |
| findings, work_done, recommendations | text | |
| condition_rating | `equipment_condition` enum | written back to `equipment.condition` on submit |
| next_service_due_on | date null | seeds v0.2 maintenance (FR-R4) |
| requires_followup | boolean default false | |
| status | `report_status` enum | `DRAFT`, `SUBMITTED` |
| submitted_at | timestamptz null | |
| amends_report_id | uuid null → service_reports | corrections are amendments (FR-R5) |
| created_at, updated_at | timestamptz | |

Index: `(business_id, equipment_id, performed_at desc)` — the equipment timeline query.
`unique(job_id, equipment_id) where amends_report_id is null` — one report per unit per visit.

### 3.18 inspection_readings

Readings are rows, not columns, so new measurements do not require a migration — but the **required
set per job type** is a server-side rule, not a loose JSON blob.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, service_report_id | uuid | |
| key | `reading_key` enum | `SUPPLY_AIR_TEMP_C`, `RETURN_AIR_TEMP_C`, `AMBIENT_TEMP_C`, `SUCTION_PRESSURE_PSI`, `DISCHARGE_PRESSURE_PSI`, `VOLTAGE_V`, `RUNNING_CURRENT_A`, `FILTER_CONDITION`, `DRAINAGE_OK`, `GAS_TOPPED_UP`, `COIL_CLEANED`, `NOISE_LEVEL` |
| value_numeric | numeric(10,2) null | for measurements |
| value_text | text null | for enumerated observations |
| value_boolean | boolean null | for checks |
| unit | text null | `°C`, `psi`, `V`, `A` |
| created_at | timestamptz | |

`unique(service_report_id, key)`. Exactly one of the three value columns is non-null (check
constraint). FR-R2's required set lives in `packages/shared` as a `requiredReadings(jobType)` map, so
the server, the form and the tests read the same definition.

### 3.19 report_photos

`id, business_id, service_report_id, file_id → files, kind (BEFORE | AFTER | ISSUE | NAMEPLATE |
OTHER), caption, created_at`.

### 3.20 invoices

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, customer_id | uuid | |
| job_id, quotation_id | uuid null | provenance |
| invoice_no | text | assigned at issue, never at draft creation |
| status | `invoice_status` enum | `DRAFT`, `ISSUED`, `PARTIALLY_PAID`, `PAID`, `VOID` |
| issue_date, due_date | date | |
| subtotal_sen, discount_sen, sst_sen, total_sen | integer | |
| sst_rate_basis_points | integer null | snapshot |
| amount_paid_sen | integer default 0 | maintained in the payment transaction |
| business_tin, business_sst_reg_no | text null | **snapshotted** onto the document (FR-I10) |
| billing_name, billing_address_snapshot | text | the customer may move later |
| terms, notes | text null | |
| issued_at, issued_by | | |
| voided_at, void_reason, voided_by | | |
| created_by | uuid → users | |
| created_at, updated_at | timestamptz | |

Indexes: `unique(business_id, invoice_no)`, `(business_id, status, due_date)`,
`(business_id, customer_id)`.

Balance is derived: `total_sen - amount_paid_sen - credited_sen`. It is **not** a stored column, to
avoid two sources of truth; the outstanding list computes it in SQL.

Immutability: `status != DRAFT` means the row and lines are frozen except for
`amount_paid_sen`/status transitions driven by payments, and the void fields. Enforced in the service
layer; a database trigger is a candidate hardening step.

`invoice_lines` mirrors `quotation_lines` with `job_line_id null` provenance.

### 3.21 payments

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id, invoice_id | uuid | |
| amount_sen | integer | positive only |
| method | `payment_method` enum | `CASH`, `BANK_TRANSFER`, `DUITNOW_QR`, `CHEQUE`, `OTHER` |
| reference | text null | transaction ref, cheque number |
| paid_at | timestamptz | when the money moved, not when it was keyed in |
| cash_rounding_sen | integer default 0 | 5-sen rounding on cash (FR-I7); never alters `total_sen` |
| notes | text null | |
| recorded_by | uuid → users | |
| reversed_at, reversal_reason, reversed_by | | reversal is an event, not a delete (FR-I6) |
| idempotency_key | text null | `unique(business_id, idempotency_key)` — a double-tap on a phone must not double-record |
| created_at | timestamptz | |

Index: `(business_id, invoice_id)`, `(business_id, paid_at)`.

Recording a payment, in one transaction: `SELECT … FOR UPDATE` the invoice → recompute paid and
credited totals → reject if `amount_sen > balance` with `PAYMENT_EXCEEDS_BALANCE` → insert payment →
update `amount_paid_sen` and status → insert `audit_events`. The lock is what makes invariant 7 hold
under two admins keying the same cheque at once.

### 3.22 credit_notes

`id, business_id, invoice_id, credit_no, reason (text, required), subtotal_sen, sst_sen, total_sen,
status (DRAFT | ISSUED), issued_at, issued_by, created_by, created_at`, plus `credit_note_lines`.

The only route to reducing what a customer owes on an issued invoice, and the prerequisite for
accepting an overpayment.

### 3.23 files

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id | uuid | |
| storage_key | text unique | `{businessId}/{subjectType}/{subjectId}/{uuid}.{ext}` |
| mime_type | text | allowlisted |
| size_bytes | integer | policy-limited per subject type |
| status | `file_status` enum | `PENDING`, `READY`, `FAILED` |
| subject_type, subject_id | | what it belongs to |
| uploaded_by | uuid → users null | null for customer signature on a public page |
| created_at, confirmed_at | timestamptz | |

### 3.24 number_sequences

`business_id, kind (QUOTATION | JOB | INVOICE | CREDIT_NOTE), period (text, e.g. '2026'), next_value
integer`. Primary key `(business_id, kind, period)`.

Allocation is `UPDATE number_sequences SET next_value = next_value + 1 WHERE … RETURNING next_value`
inside the issuing transaction — gapless and race-free (ARCHITECTURE §8).

### 3.25 audit_events

Append-only. No updates, no deletes, ever.

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| business_id | uuid | |
| actor_type | `actor_type` enum | `USER`, `CUSTOMER_PUBLIC`, `SYSTEM` |
| actor_user_id | uuid null → users | |
| actor_label | text null | customer's typed name for public actions |
| entity_type, entity_id | text, uuid | |
| action | text | `invoice.issued`, `payment.recorded`, `payment.reversed`, `quotation.approved`, `equipment.relocated`, `user.deactivated` |
| amount_sen | integer null | present for financial events, enables a money-only audit view |
| before, after | jsonb null | changed fields only, not whole rows |
| request_id | text null | ties the event to a log line |
| ip, user_agent | text null | |
| occurred_at | timestamptz default now() | |

Indexes: `(business_id, occurred_at desc)`, `(business_id, entity_type, entity_id, occurred_at desc)`,
partial index `(business_id, occurred_at desc) where amount_sen is not null` for the financial view.

Invariant 8: financial audit rows are written in the **same transaction** as the change they
describe. An audit trail written on a best-effort basis after commit is not an audit trail.

## 4. State machines

Illegal transitions are rejected by the server with a `BusinessRuleError` naming the current and
attempted state. Each machine has a unit-tested transition table in `packages/shared`.

### 4.1 Quotation

```
DRAFT ──▶ SENT ──┬──▶ APPROVED ──▶ SUPERSEDED   (when a revision is created)
  │              ├──▶ REJECTED
  │              └──▶ EXPIRED                   (valid_until passed, evaluated on read)
  └──▶ CANCELLED
```
`APPROVED` is terminal except for being superseded. A superseded version stays readable with its
approval evidence (FR-Q6).

### 4.2 Job

```
DRAFT ──▶ SCHEDULED ──▶ ASSIGNED ──▶ EN_ROUTE ──▶ IN_PROGRESS ──┬──▶ COMPLETED
                                                                 └──▶ AWAITING_APPROVAL ──▶ IN_PROGRESS
any pre-completion state ──▶ CANCELLED
```
`IN_PROGRESS → COMPLETED` is gated: every row in `job_equipment` needs a `SUBMITTED` service report
with its required readings present (FR-T3, invariant 10). `AWAITING_APPROVAL` is entered when an
additional-work request is pending.

### 4.3 Invoice

```
DRAFT ──▶ ISSUED ──┬──▶ PARTIALLY_PAID ──▶ PAID
                   ├──▶ PAID               (paid in full in one payment)
                   └──▶ VOID               (only while no payment exists)
DRAFT ──▶ (deleted)                        (drafts may be deleted; issued documents never)
```
After any payment, correction is a credit note, not a void (FR-I3).

### 4.4 Equipment

```
ACTIVE ──┬──▶ REMOVED
         └──▶ REPLACED   (requires replaced_by_equipment_id)
```

## 5. Token handling

Applies to `approval_tokens.token_hash`, `sessions.token_hash` and `users.invite_token_hash`.

```
raw = crypto.randomBytes(32).toString('base64url')   // 256 bits
store SHA-256(raw)                                    // never the raw value
return raw exactly once, in the response that created it
compare with timingSafeEqual on a hashed lookup
```

Never logged, never in an error message, never in an analytics event. Rotation on use where the
token is single-decision. Invariant 12.

## 6. Indexing policy

- Every foreign key gets an index — Postgres does not create one automatically.
- Every tenant-owned table's primary access index starts with `business_id`.
- List screens get a covering composite index matching their sort: jobs by
  `(business_id, scheduled_date, scheduled_slot)`, invoices by `(business_id, status, due_date)`,
  reports by `(business_id, equipment_id, performed_at desc)`.
- Customer name search uses `pg_trgm` + GIN. Enabled as an extension in the first migration.
- Add an index when a query plan justifies it, not speculatively — but the three above are justified
  by the screens in the PRD and ship with their tables.

## 7. Migration policy

- `prisma migrate dev` locally; the generated SQL is reviewed before commit, not trusted blindly.
- Committed migrations are immutable. A mistake is corrected by a new migration.
- Additive-first for anything deployed: add nullable column → backfill → enforce in a later
  migration. Never drop a column in the same release that stops writing it.
- Any migration that rewrites a large table is called out in the milestone summary with its expected
  lock behaviour. At pilot scale this is unlikely to matter, but the habit is cheap.
- Seeds: `packages/db/prisma/seed.ts` creates two businesses with overlapping-looking data
  specifically so tenancy bugs are visible during development. Seeds are clearly labelled demo data
  and never run against production.

## 8. Extensions required

`pgcrypto` (`gen_random_uuid`), `pg_trgm` (customer/address search). Both enabled in the first
migration; both available on Render Postgres.
