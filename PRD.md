# PRD — MicroBiz Flow

Version: 0.1 (MVP definition) · Last updated: 2026-11-18 · Status: **awaiting pilot validation**

Items marked **[ASSUMPTION]** are unvalidated and must be confirmed with pilot businesses. They are
listed together in `DEVELOPMENT_PLAN.md` so they can be tested rather than quietly inherited.

---

## 1. Problem

A typical Malaysian air-conditioning service business runs 2–10 technicians and coordinates
everything through WhatsApp, a notebook and a spreadsheet. The observable symptoms:

- **Job details live in chat.** Which unit, which floor, what the customer complained about, what
  was quoted — scattered across threads. The technician phones the office from the site.
- **No equipment memory.** Nobody knows that the cassette unit in Meeting Room 2 was gassed twice in
  eight months, so a recurring leak is re-diagnosed every visit and sold as a one-off repair.
- **Quotations are PDFs sent and forgotten.** Approval is "ok boss" in chat. There is no record of
  what the customer actually agreed to, so disputes are settled by scrolling.
- **Revenue leaks at the edges.** Additional work done on site is not captured, job sheets are lost,
  invoices go out late, and nobody knows who owes what without rebuilding the spreadsheet.
- **Maintenance contracts are run from memory.** Scheduled servicing slips, customers churn.

The cost is not primarily lost sales. It is unbilled work, repeat diagnosis, and the owner being the
single point of coordination for every job.

## 2. Users

| Role | Who they are | Primary device | What they need |
|---|---|---|---|
| **Business Owner** | Founder, often still on the tools | Phone, occasional laptop | Know what is scheduled, what is unpaid, what each technician did |
| **Administrator** | Office admin or the owner's spouse/family member | Laptop | Create quotations, schedule jobs, chase payment, answer customer calls with facts |
| **Technician** | 20–45, Malay/English, phone-literate but not software-literate | Android phone, often cheap, often poor signal | See today's jobs, find the site, know which units to service, record what was done, get out |
| **Customer** | Homeowner, condo manager, shop or small office manager | Phone, WhatsApp | Approve a quotation without friction, see what was done, know what they owe |

**[ASSUMPTION]** Technicians will fill a structured digital job sheet if it takes under three minutes
per unit. If it takes longer they will revert to paper or skip fields, and the equipment history —
the whole differentiator — degrades into empty records.

**[ASSUMPTION]** Customers will approve a quotation via a tokenized web link sent through WhatsApp,
without an account. If they insist on replying "ok" in chat instead, approval capture fails and the
quotation audit trail is worthless.

## 3. The core loop

```
Customer ─▶ Property ─▶ Equipment (each AC unit)
                            │
                   Quotation ─▶ (customer approval link)
                            │
                          Job ─▶ Technician assignment
                            │
                  Service Report + Inspection (per unit)
                            │
                      Invoice ─▶ Payment ─▶ Audit trail
                            │
                   Maintenance schedule  [v0.2]
```

Equipment is the spine, not an attribute of a job. A job references the units it touches; a unit
accumulates every report, reading, photo and warranty fact across its life, including across changes
of ownership of the property.

## 4. Scope

### In scope for v0.1

| # | Module | What it must do |
|---|---|---|
| 1 | **Auth, tenancy, RBAC** | Business registration, owner account, invite admins and technicians, email + password login, session management, server-enforced role permissions, complete tenant isolation |
| 2 | **Customers** | Individual and company customers, contacts, phones, billing address, notes, search, soft deactivate |
| 3 | **Properties** | Multiple properties per customer, Malaysian address, access notes (gate code, which lift, who to ask for) |
| 4 | **Equipment** | AC units per property with asset tag, location label, type, brand, model, serial, capacity, refrigerant, install date, warranty expiry, condition, status, photos, and a full service history timeline |
| 5 | **Service catalogue** | Reusable service and part items with unit, price in sen, SST taxability, active flag |
| 6 | **Quotations + approval** | Draft from catalogue and equipment, line items, discount, optional SST, validity date, PDF-equivalent public view, tokenized approval/rejection link, versioning with revision history on approved quotes |
| 7 | **Jobs + scheduling** | Create standalone or from an approved quotation, job type, priority, date and time slot, assign technicians, link the specific units covered, day/week schedule view, status lifecycle |
| 8 | **Technician workspace** | Mobile-first: today's jobs, job detail with address and access notes, per-unit checklist, start/complete, photo capture, additional-work request, offline-tolerant drafts |
| 9 | **Service reports + inspections** | One report per unit per visit: findings, work done, readings, recommendations, next service due, before/after photos, customer signature and name |
| 10 | **Invoices + manual payments** | Invoice from a job or a quotation, immutable once issued, void and credit note, record cash/bank/DuitNow/cheque payments, balance tracking, overpayment blocked, full financial audit trail |

### Explicitly out of scope for v0.1

Deferred to v0.2 or later. Do not build "a little bit" of these.

- Maintenance plans, contracts and recurring schedule generation
- Redis + BullMQ background jobs; all v0.1 work is request-scoped
- Automated notifications (email, WhatsApp Business API, SMS, push). v0.1 produces **shareable links
  and copy-ready WhatsApp text** that a human sends
- Full customer portal with login and history
- Analytics dashboards and reporting beyond simple counts on the home screen
- Payment gateways, online card payment, DuitNow API reconciliation
- LHDN MyInvois submission
- Inventory, stock levels, purchasing, supplier management
- Technician payroll, commission and timesheets
- Bahasa Melayu or Chinese UI
- Native mobile apps
- Multi-currency

### Out of scope permanently (for now)

Full accounting (general ledger, chart of accounts, tax filing). The product hands clean data to an
accountant; it does not become one.

## 5. Functional requirements

Numbering is stable; tests and the development plan reference these IDs.

### 5.1 Authentication and tenancy

- **FR-A1** A visitor can register a business and become its Owner in one flow: business name, own
  name, email, password, phone.
- **FR-A2** An Owner or Administrator can invite a user by email with a role. The invite is a
  tokenized, expiring link; the invitee sets their own password. No admin-set passwords.
- **FR-A3** Login is email plus password. Failed attempts are rate-limited per email and per IP.
- **FR-A4** A session can be ended by the user, and all sessions can be ended by an Owner for any
  user in the business (used when a technician's phone is lost).
- **FR-A5** No API response ever contains a record belonging to another business. This is a test
  requirement, not a hope: the integration suite asserts cross-tenant reads return 404.
- **FR-A6** A technician who requests a job they are not assigned to gets 404 (not 403 — a technician
  should not learn that the job exists).
- **FR-A7** Deactivating a user immediately invalidates their sessions.

### 5.2 Customers and properties

- **FR-C1** Create, edit, deactivate customers of type individual or company. Company customers
  carry a registration number and TIN for invoicing.
- **FR-C2** Search customers by name, phone or property address, with the normalized phone matching
  `012-345 6789`, `0123456789` and `+60123456789` identically.
- **FR-C3** A customer has zero or more properties; a property belongs to exactly one customer at a
  time. Transferring a property to another customer is recorded, and the equipment history stays
  with the property.
- **FR-C4** A property carries an access notes field that is shown prominently to technicians.

### 5.3 Equipment

- **FR-E1** Register an AC unit against a property with a human-meaningful location label ("Master
  bedroom", "Shop lot front-left cassette"). Serial and model are optional — nameplates are often
  unreadable or painted over.
- **FR-E2** Each unit gets a per-property asset tag, generated by the system and printable, so a
  technician can identify a unit without a legible nameplate. **[ASSUMPTION]** Businesses will
  actually label units with a sticker or marker on first visit.
- **FR-E3** A unit's detail view shows a reverse-chronological timeline: every service report,
  reading, photo, quotation line, job and warranty fact.
- **FR-E4** A unit can be marked replaced, with a link to the unit that replaced it, preserving the
  history of both.
- **FR-E5** A unit can be moved to another property within the same customer (units get relocated),
  and the move is recorded on the timeline.
- **FR-E6** Warranty expiry is visible wherever the unit appears in a quotation or job.

### 5.4 Service catalogue

- **FR-S1** Items have a code, name, description, unit of measure (job, unit, hour, piece), price in
  sen, and an SST-taxable flag.
- **FR-S2** A quotation or invoice line created from an item **snapshots** its description and price.
  Later catalogue edits never change an existing document.

### 5.5 Quotations and approval

- **FR-Q1** Build a quotation for a customer and property, with lines that may reference specific
  equipment and catalogue items, plus free-text lines.
- **FR-Q2** Totals: line subtotals, document-level discount, SST applied only when the business is
  SST-registered and only to taxable lines, then grand total. All arithmetic in sen, rounded half-up
  per line.
- **FR-Q3** A quotation has a validity date and expires automatically for approval purposes.
- **FR-Q4** Sending a quotation generates a public link with a high-entropy, expiring token, and
  copy-ready WhatsApp message text. The link requires no login.
- **FR-Q5** The public view shows the business letterhead, lines, equipment covered, totals, terms,
  and Approve / Reject actions. Approval captures a typed name, timestamp, IP and user agent.
- **FR-Q6** Approving locks the quotation. Editing an approved quotation creates version *n+1* in
  draft and marks version *n* superseded; the approved snapshot remains viewable with its approval
  evidence.
- **FR-Q7** Rejection captures an optional reason and is visible to staff.
- **FR-Q8** A quotation can be converted to a job, carrying lines and equipment references across.

### 5.6 Jobs and scheduling

- **FR-J1** Create a job standalone or from an approved quotation. Type: install, service, repair,
  inspection, maintenance.
- **FR-J2** Scheduling is a date plus a **time slot** (morning, afternoon, evening) or an explicit
  start/end time. **[ASSUMPTION]** These businesses book in half-day windows, not appointments, and a
  slot-based scheduler fits reality better than a calendar grid.
- **FR-J3** Assign one or more technicians, with one marked lead.
- **FR-J4** A job lists the specific equipment it covers; the technician's checklist is generated per
  unit from that list.
- **FR-J5** Status lifecycle: `DRAFT → SCHEDULED → ASSIGNED → EN_ROUTE → IN_PROGRESS →
  AWAITING_APPROVAL → COMPLETED`, plus `CANCELLED` from any pre-completion state. Illegal transitions
  are rejected by the server with a specific error.
- **FR-J6** A day view and a week view show jobs by technician, with unassigned jobs visible.
- **FR-J7** Rescheduling and reassigning are recorded with who and when.

### 5.7 Technician workspace

- **FR-T1** A technician's home screen is today's assigned jobs, in slot order, with customer name,
  address, unit count and status — readable at a glance in sunlight.
- **FR-T2** Tapping a job shows address with a maps deep link, access notes, customer phone with a
  call and a WhatsApp action, the work scope, and the unit checklist.
- **FR-T3** Start job records a timestamp. Complete job is blocked until every covered unit has a
  submitted service report with its required fields.
- **FR-T4** Per unit, the technician records readings, condition, work done, findings,
  recommendations and photos.
- **FR-T5** Photo capture resizes on device before upload to respect mobile data, uploads directly to
  private storage via a presigned URL, and retries on failure.
- **FR-T6** Additional work: the technician adds proposed lines, submits them, and the customer
  approves via a tokenized link (or an Administrator records a verbal approval with their name
  attached). Nothing extra is invoiceable without one of those two records.
- **FR-T7** A partially filled job sheet survives app reload, signal loss and phone restart.
  **[ASSUMPTION]** Device-local drafts plus retry are sufficient; full offline sync is not needed
  for v0.1.
- **FR-T8** Customer signature is captured on the device at job completion, together with the signing
  person's name.

### 5.8 Service reports and inspections

- **FR-R1** One report per unit per visit, linked to job, unit and technician.
- **FR-R2** Required inspection fields depend on job type. For a standard service the minimum set is:
  supply and return air temperature, operating condition, filter condition, drainage check, and an
  overall condition rating. The server enforces the required set.
  **[ASSUMPTION]** This is the right minimum set — needs a technician to confirm what they actually
  measure versus what sounds thorough on paper.
- **FR-R3** Numeric readings are stored as numbers with units so they can be trended per unit later.
- **FR-R4** A report sets the recommended next service date for that unit, seeding v0.2 maintenance.
- **FR-R5** A submitted report is immutable; corrections are an amendment that references the
  original.
- **FR-R6** A report is renderable as a customer-facing document with photos.

### 5.9 Invoices and payments

- **FR-I1** Create an invoice from a completed job (including approved additional work) or from an
  approved quotation, or standalone.
- **FR-I2** Issuing assigns an immutable, gapless per-business invoice number and freezes the
  document. Draft invoices are editable; issued ones are not.
- **FR-I3** Statuses: `DRAFT`, `ISSUED`, `PARTIALLY_PAID`, `PAID`, `VOID`. Void requires a reason and
  is only possible before any payment is recorded; after payment, use a credit note.
- **FR-I4** Record a payment with amount, method (cash, bank transfer, DuitNow QR, cheque, other),
  reference, paid date, and notes. Several payments per invoice.
- **FR-I5** A payment that would push total payments above the outstanding balance is rejected with a
  clear message naming the balance. Overpayment is handled by issuing a credit note first.
- **FR-I6** A recorded payment can be reversed with a reason; reversal is a new append-only event,
  never a deletion.
- **FR-I7** Cash payments record a separate 5-sen rounding adjustment; the invoice total is untouched.
- **FR-I8** Every financial mutation — issue, void, payment, reversal, credit note, discount change
  on a draft — writes an audit event with actor, timestamp, before/after and request id, in the same
  transaction as the change.
- **FR-I9** An outstanding list shows what is owed, by customer, with ageing buckets.
- **FR-I10** Invoices carry the business TIN and SST registration number when present, so corporate
  customers can self-bill under the e-Invoice rules.

## 6. Non-functional requirements

- **NFR-1 Performance.** Technician job list and job detail respond in under 500ms at p95 on a
  warm database with 50k equipment rows. The technician flow works acceptably on 3G.
- **NFR-2 Mobile.** Every technician screen is usable at 360px width and tested there. Touch targets
  at least 44px.
- **NFR-3 Accessibility.** Keyboard-navigable, labelled form controls, visible focus, WCAG AA
  contrast, errors announced not just coloured. No reliance on hover.
- **NFR-4 Security.** Tenant isolation tested, server-side authorization on every route,
  rate-limited auth, hashed public tokens, private file storage with short-lived presigned URLs,
  no secrets in the repo, dependency audit in CI.
- **NFR-5 Data integrity.** Money in integer sen, financial changes auditable, no destructive edit of
  issued documents, foreign keys and uniqueness enforced in the database, not only in code.
- **NFR-6 Observability.** Structured request logs with a request id surfaced to the user; errors
  carry enough context to diagnose without reproducing.
- **NFR-7 Testing.** Every invariant in `CLAUDE.md` has at least one test. Financial and tenancy code
  is integration-tested against a real Postgres, not mocks.
- **NFR-8 Backups.** Daily automated database backup with a documented, rehearsed restore procedure
  before the first pilot loads real data.

## 7. Success metrics for the pilot

Measured per pilot business over the first 60 days:

| Metric | Target | Why it matters |
|---|---|---|
| Jobs with a complete digital service report | > 80% | If technicians do not fill reports, the differentiator does not exist |
| Median time to fill a report per unit | < 3 min | The adoption cliff |
| Quotations approved via link (vs chat) | > 60% | Validates the approval mechanism |
| Additional work captured and invoiced | measurable increase vs baseline | The clearest money argument |
| Days from job completion to invoice issued | < 2 (from a baseline owners report in weeks) | The second money argument |
| Units with more than one service record after 60 days | > 30% | Proves equipment history accumulates |
| Weekly active technicians | 100% of the pilot's technicians | Partial adoption means paper runs in parallel and data is unreliable |

## 8. Key product risks

1. **Technician adoption is the whole product.** Every field added to a job sheet costs adoption.
   Required fields must be defended one at a time, not accumulated.
2. **WhatsApp is the real interface.** Customers will not install anything or create accounts.
   Anything customer-facing must work as a link pasted into a chat.
3. **Equipment data entry is the cold-start problem.** A new business must inventory existing units
   before the history has value. Onboarding must make the first property's units fast to enter, and
   allow progressive capture during normal visits rather than demanding a full audit up front.
4. **Connectivity.** Plant rooms and basements have no signal. A lost job sheet destroys trust
   permanently.
5. **Price sensitivity.** These businesses compare against free (WhatsApp) and cheap. The product
   must show money recovered, not features.
