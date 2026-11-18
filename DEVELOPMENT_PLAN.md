# DEVELOPMENT PLAN — MicroBiz Flow

Last updated: 2026-11-18 · Target: **v0.1 MVP** · Status: **M0 complete, M1 awaiting approval**

Legend: `[ ]` not started · `[~]` in progress · `[x]` done · `[!]` blocked

---

## Status summary

| Milestone | Scope | Status |
|---|---|---|
| **M0** | Foundational documentation | `[x]` complete |
| **M1** | Monorepo, database, auth, tenancy, RBAC | `[ ]` **awaiting approval** |
| **M2** | App shell, navigation, design system | `[ ]` |
| **M3** | Customers and properties | `[ ]` |
| **M4** | Equipment and timeline | `[ ]` |
| **M5** | Service catalogue + money engine | `[ ]` |
| **M6** | Quotations | `[ ]` |
| **M7** | Public approval links | `[ ]` |
| **M8** | Jobs and scheduling | `[ ]` |
| **M9** | Technician workspace | `[ ]` |
| **M10** | Service reports and inspections | `[ ]` |
| **M11** | Additional work approval | `[ ]` |
| **M12** | Invoices | `[ ]` |
| **M13** | Payments, credit notes, audit view | `[ ]` |
| **M14** | Hardening, E2E, staging, pilot readiness | `[ ]` |

Each milestone ends with: `npm run verify` green, docs updated, and a summary of files changed, what
works, tests added, and known gaps. No milestone is "done" with a failing or skipped test.

---

## M0 — Foundational documentation `[x]`

- [x] `CLAUDE.md` — working agreement, invariants, conventions
- [x] `PRD.md` — problem, users, v0.1 scope, functional requirements, metrics, risks
- [x] `ARCHITECTURE.md` — topology, tenancy, auth, money/time, testing, deployment, numbered decisions
- [x] `DATABASE.md` — full v0.1 schema, state machines, indexing and migration policy
- [x] `API.md` — conventions, RBAC matrix, endpoint contracts
- [x] `DEVELOPMENT_PLAN.md` — this file
- [x] Git repository initialized on `main`, `.gitignore`
- [ ] Decisions D1–D6 resolved (see below) — **blocks M1**

## M1 — Monorepo, database, auth, tenancy, RBAC `[ ]` ← proposed next

The foundation every later milestone sits on. Nothing product-facing beyond logging in, deliberately.

**Scaffold**
- [ ] npm workspaces root: `apps/api`, `apps/web`, `packages/shared`, `packages/db`
- [ ] TypeScript project references, `strict` + `noUncheckedIndexedAccess`, shared tsconfig base
- [ ] ESLint + Prettier, including the rule banning base-Prisma imports outside `packages/db`
- [ ] `docker-compose.yml` with Postgres 16, `npm run db:up`
- [ ] `.env.example` complete; Zod-validated `env.ts` in both apps that exits on invalid config
- [ ] Top-level scripts from `CLAUDE.md`, including `npm run verify`
- [ ] GitHub Actions: install, typecheck, lint, test against a Postgres service container

**Database**
- [ ] Prisma initialized; `pgcrypto` and `pg_trgm` enabled in the first migration
- [ ] Tables: `businesses`, `users`, `sessions`, `number_sequences`, `audit_events`
- [ ] Enums: `user_role`, `my_state`, `actor_type`
- [ ] `tenantClient(businessId)` extension with the `TENANT_OWNED` registry
- [ ] Seed: two businesses with similar-looking data so tenancy bugs are visible

**Shared**
- [ ] Money helpers: `roundSen`, `senToDisplay`, `displayToSen`, with the RM 21.47m ceiling in Zod
- [ ] Date/timezone helpers: UTC storage, `Asia/Kuala_Lumpur` formatting, calendar-date handling
- [ ] Malaysian phone normalizer and postcode/state schemas
- [ ] RBAC matrix and `hasPermission()`
- [ ] Auth request/response schemas

**API**
- [ ] Express app: helmet, CORS allowlist, request id, pino request logging, JSON limits
- [ ] `AppError` taxonomy and the single error handler producing the `API.md` wire format
- [ ] Session middleware: cookie → session → user → `req.ctx` with the tenant-scoped client
- [ ] CSRF: double-submit + `Origin` check on mutations
- [ ] Rate limiter on auth routes
- [ ] `requirePermission()` middleware driven by the shared matrix
- [ ] Endpoints: `/auth/register`, `/auth/login`, `/auth/logout`, `/auth/me`, `/auth/csrf`,
      `/auth/sessions` (list, revoke), `/auth/password`, `/users` (list, invite, accept, update,
      deactivate, revoke sessions), `/business` (get, patch), `/health`, `/health/ready`
- [ ] Argon2id hashing with parameters in config
- [ ] Audit events for `business.created`, `user.invited`, `user.deactivated`, `session.revoked`

**Web**
- [ ] Next.js App Router, Tailwind, shadcn/ui initialized
- [ ] `apiFetch` helper handling cookies, CSRF and the error envelope — the single network edge,
      switchable between the subdomain and proxy transports (Decision D1)
- [ ] Register, login, logout, accept-invite pages with inline field errors
- [ ] Authenticated layout with a route guard and a placeholder dashboard
- [ ] Team management screen (list, invite, deactivate)
- [ ] Loading, empty and error states on the one list screen that exists

**Tests**
- [ ] Unit: money rounding (including the half-up boundary and the ceiling), phone normalization,
      timezone conversion across midnight MYT, RBAC matrix, token hashing
- [ ] Integration: full auth lifecycle; rate limiting; invite acceptance including expiry;
      deactivation revoking sessions; last-Owner protection
- [ ] Integration: `expectTenantIsolation` helper and its application to every endpoint in this
      milestone — business A cannot read, update or deactivate business B's users
- [ ] Integration: permission matrix asserted per route, including technician 404s

**Acceptance criteria**
1. A new business can register, log in, invite a technician, and the technician can accept and log in.
2. Cross-tenant access returns 404 in an automated test, not by inspection.
3. A route with no `requirePermission()` fails a test that enumerates registered routes.
4. Invalid env config stops the process at boot with a readable message.
5. `npm run verify` is green from a clean clone with only `npm install` and `npm run db:up`.

**Risks**
- Decision D1 (auth transport / domain) changes cookie configuration and the `apiFetch` shape. It is
  isolated to two files by design, but it should be settled before the work starts rather than after.
- The Prisma extension approach to tenancy needs care with nested writes; the isolation tests are
  what prove it, so they are written in this milestone and extended in every later one.

## M2 — App shell, navigation, design system `[ ]`
- [ ] Layout: desktop sidebar, mobile bottom navigation, role-aware menu
- [ ] shadcn/ui component set: forms, tables, dialogs, toasts, date and slot pickers, money input
- [ ] Reusable `DataTable` with loading skeletons, empty states, error retry, pagination
- [ ] Money and date display components (single place where sen becomes `RM 1,234.50`)
- [ ] Mobile-first technician shell at 360px with 44px touch targets
- [ ] Accessibility baseline: focus rings, labelled controls, announced errors, AA contrast

## M3 — Customers and properties `[ ]`
- [ ] Tables, migrations, enums (`customer_type`, `property_type`)
- [ ] CRUD endpoints, trigram search across name/phone/address
- [ ] Property transfer with audit
- [ ] UI: customer list with search, detail with properties, forms
- [ ] Tests: search normalization (three phone formats match), tenancy, transfer audit

## M4 — Equipment and timeline `[ ]`
- [ ] Tables, enums, asset-tag generation unique per property
- [ ] CRUD, replace, relocate, photos
- [ ] Timeline endpoint (merged, paginated)
- [ ] UI: equipment list per property, unit detail with timeline and warranty status, quick-add flow
      optimized for entering several units at one property
- [ ] Tests: asset-tag uniqueness and regeneration on relocate; history survives replace and transfer

## M5 — Service catalogue and money engine `[ ]`
- [ ] Table, CRUD, deactivate
- [ ] Document total engine: line subtotals, discount apportionment, per-line SST, grand total
- [ ] SST settings on the business profile with the registration-number constraint
- [ ] Tests: the money engine is the most heavily tested unit in the codebase — rounding boundaries,
      mixed taxable/non-taxable lines, discount apportionment, SST off vs on, rate snapshotting

## M6 — Quotations `[ ]`
- [ ] Tables, status enum, number sequence allocation
- [ ] CRUD on drafts, send, revise, cancel, convert to job
- [ ] Server-side total recomputation (client totals ignored)
- [ ] UI: builder with equipment and catalogue pickers, live totals, print-styled view
- [ ] Tests: approved quotations reject mutation; revision creates n+1 and supersedes n; approval
      snapshot stays readable; expiry

## M7 — Public approval links `[ ]`
- [ ] `approval_tokens`, hashing, expiry, single-decision, revocation
- [ ] Public routes with no session middleware, per-token and per-IP rate limits
- [ ] Narrow public DTO; approval evidence capture
- [ ] `whatsappText` generation and copy-to-clipboard in the UI
- [ ] Public page: mobile-first, works for a non-technical customer, clear approve/reject
- [ ] Tests: expired, used, revoked and malformed tokens; the DTO leaks nothing extra; no raw token
      appears in any log

## M8 — Jobs and scheduling `[ ]`
- [ ] Tables, status machine, assignments, job equipment, job lines
- [ ] CRUD, schedule, assign, cancel; `/schedule` day and week board
- [ ] Conversion from quotation carrying lines and equipment
- [ ] UI: schedule board by technician and slot, unassigned queue, job detail
- [ ] Tests: every illegal transition rejected; assignment changes audited; technician list filtering

## M9 — Technician workspace `[ ]`
- [ ] `/jobs/today`, `/jobs/:id/checklist`, en-route, start
- [ ] Mobile UI: today's list, job detail with maps/call/WhatsApp actions, access notes prominent
- [ ] Device-local draft persistence and submission retry
- [ ] Photo capture with on-device resize and presigned upload
- [ ] Tests: Playwright at 360px for the full day flow; draft survives reload; upload retry

## M10 — Service reports and inspections `[ ]`
- [ ] Tables, `reading_key` enum, `requiredReadings(jobType)` in `packages/shared`
- [ ] Draft/submit lifecycle, readings upsert, photos, amendments
- [ ] Condition written back to the unit; next service date captured
- [ ] Job completion gate
- [ ] UI: per-unit report form designed for under three minutes of thumb work
- [ ] Tests: submission rejected with the missing keys listed; completion blocked until every covered
      unit has a submitted report; amendments preserve the original

## M11 — Additional work approval `[ ]`
- [ ] Tables, request flow, token approval, recorded-verbal approval
- [ ] `AWAITING_APPROVAL` job state handling
- [ ] UI: technician request form, public approval page, staff decision recording
- [ ] Tests: unapproved lines cannot reach an invoice (`ADDITIONAL_WORK_NOT_APPROVED`)

## M12 — Invoices `[ ]`
- [ ] Tables, status machine, number sequence, document snapshots (TIN, SST no, billing address)
- [ ] Create from job/quotation/standalone, issue, void, outstanding list with ageing
- [ ] Immutability enforcement after issue
- [ ] UI: invoice builder, print-styled invoice with payment instructions, outstanding screen
- [ ] Tests: issued invoices reject edits; numbers gapless and per-business under concurrency; void
      blocked once a payment exists

## M13 — Payments, credit notes, audit view `[ ]`
- [ ] `payments`, `credit_notes` tables; locked-transaction payment recording; idempotency keys
- [ ] Reversal as an append-only event; credit note issuance; cash rounding field
- [ ] Audit event writes inside every financial transaction; `/audit` endpoint and view
- [ ] UI: record payment dialog showing the live balance, payment history, credit note flow
- [ ] Tests: overpayment rejected with the real balance; **concurrent** payments cannot exceed the
      balance (two parallel requests, one must fail); reversal leaves an immutable trail; idempotent
      retry does not double-record; every financial mutation produces exactly one audit row

## M14 — Hardening, E2E, staging, pilot readiness `[ ]`
- [ ] Playwright E2E: quote → approve → job → assign → report → complete → invoice → payment
- [ ] Playwright mobile E2E for the technician day at 360px
- [ ] Accessibility pass (axe) on every screen; keyboard-only walkthrough
- [ ] Performance check against seeded volume (NFR-1), with query plans for the list screens
- [ ] Security pass: dependency audit, rate-limit review, presigned-URL expiry, cookie flags,
      token handling review, log redaction verified
- [ ] Postgres RLS backstop policies (ARCHITECTURE §4 step 4)
- [ ] Staging environment on Vercel + Render; migration deploy step; secrets configured
- [ ] **Backup and restore rehearsed and written up** — a gate, not a task
- [ ] Onboarding flow for a new business: first customer, first property, first units, first quote
- [ ] `docs/runbook.md`: deploy, rollback, restore, secret rotation, common failures

---

## Decisions needed before M1

These change architecture, security or schema, so they are not mine to assume.

| # | Decision | Why it matters now | My recommendation |
|---|---|---|---|
| **D1** | **Domain for app and API.** Do you own (or will you buy) a domain, so the app can be `app.<domain>` and the API `api.<domain>`? | `*.vercel.app` + `*.onrender.com` are different registrable domains, so session cookies become third-party cookies and Safari blocks them outright. This decides cookie config and whether `apiFetch` proxies | **Buy a `.my` or `.com` domain now** and use subdomains. Otherwise I implement the Next.js proxy fallback, which costs a latency hop on every technician request |
| **D2** | **Technician equipment creation.** Can technicians create and edit equipment records on site? | Affects the RBAC matrix and the technician UI scope | **Yes.** Finding undocumented units is the most common field reality, and blocking it pushes technicians back to WhatsApp |
| **D3** | **Verbal approval for additional work.** Is an admin recording "customer approved by phone" acceptable, or must every approval be a customer-clicked link? | Determines whether `RECORDED_VERBAL` exists. A link-only rule is cleaner for audit but will be bypassed in practice | **Allow it**, attributed to the staff member with their name on the record, and report link-vs-verbal rates to the owner |
| **D4** | **Who may reverse payments and issue credit notes** — Owner only, or Admins too? | These are the two ways money disappears from the record | **Owner only** for v0.1; relax if pilots find it blocking |
| **D5** | **Invoice email delivery in v0.1.** Confirmed out of scope, so invoices are shared as links/print only? | If invoices must arrive as PDF attachments by email, that adds an email provider and server-side PDF rendering — both outside the stated stack | **Keep it out.** Links plus browser print; revisit after pilot feedback |
| **D6** | **Pilot businesses.** How many, and do we have access to a real technician and a real owner for 30 minutes each before M9? | The technician workspace and the required inspection fields are the two highest-risk designs, and both are cheap to validate and expensive to rebuild | **Two or three pilots**, and talk to a technician before M9 is designed |

Also useful but not blocking: your SST registration status and TIN for the first pilot, and whether
any pilot is above the RM3m e-Invoice threshold (if so, MyInvois moves from v0.2 to a real constraint).

## Assumptions to validate with pilot businesses

Each one is cheap to test now and expensive to discover late. Referenced from `PRD.md`.

| # | Assumption | How it fails | How to test |
|---|---|---|---|
| **A1** | Technicians will fill a structured job sheet if it takes under 3 minutes per unit | Fields left blank or paper used in parallel; equipment history is empty and the differentiator evaporates | Paper prototype of the per-unit form with a real technician; time them on three units |
| **A2** | Customers will approve quotations via a WhatsApp link without an account | They reply "ok boss" in chat instead; approval audit trail is fiction | Send a mock approval link to five real customers of a pilot and count clicks |
| **A3** | Businesses will label units with asset tags on the first visit | Units cannot be identified across visits; history attaches to the wrong unit | Ask a pilot owner whether they will carry a label marker; watch one job |
| **A4** | Half-day slots (morning/afternoon/evening) match how jobs are booked | A calendar grid is needed instead, changing the scheduling model and UI | Review two weeks of a pilot's actual bookings |
| **A5** | The required inspection set (supply/return temp, condition, filter, drainage, rating) is what technicians actually measure | Required fields are guessed at or skipped; data is noise | Have a technician fill the form from memory of their last three jobs |
| **A6** | Device-local drafts plus retry are enough; full offline sync is not needed | Lost job sheets in basements destroy trust permanently | Ask where they lose signal; test the draft flow in a real car park |
| **A7** | Most pilots are below the RM500k SST threshold and the RM3m e-Invoice threshold | SST and MyInvois become v0.1 requirements rather than v0.2 | Ask each pilot owner directly — one question each |
| **A8** | Equipment cold-start can be progressive (capture units during normal visits) rather than a full audit up front | Pilots stall at onboarding because inventorying 200 units is a project nobody has time for | Count units at one pilot's three biggest customers and time entering one property |
| **A9** | Owners will pay for recovered unbilled work rather than for features | Pricing and positioning are wrong even if the product works | Ask each owner to estimate monthly unbilled additional work |
| **A10** | English-only UI is acceptable to technicians | Technicians cannot use it reliably; Bahasa Melayu moves into v0.1 | Watch a technician read the job sheet aloud |

## Decisions log

Architectural decisions carry `AD-n` identifiers in `ARCHITECTURE.md`. Changes recorded here.

| Date | Decision | Rationale |
|---|---|---|
| 2026-11-18 | AD-1 Split Next.js/Express deployment | Given as a stack constraint; consequences documented |
| 2026-11-18 | AD-2 Modular monolith | Correctness over scale at this size |
| 2026-11-18 | AD-3 `packages/shared` owns contracts | Form and endpoint cannot disagree |
| 2026-11-18 | AD-4 Opaque cookie sessions in Postgres, not JWT | FR-A4/A7 require immediate revocation |
| 2026-11-18 | AD-5 One business per user in v0.1 | Simpler login; `memberships` is an additive migration |
| 2026-11-18 | AD-6 Tenancy via scoped Prisma client, RLS as later backstop | Makes the safe path the default path |
| 2026-11-18 | AD-7 Presigned direct-to-bucket uploads | Keeps photo bytes off the API instance |
| 2026-11-18 | Money as `Int` sen, not `BigInt` | RM 21.47m per-field ceiling is ample; `BigInt` poisons JSON |
| 2026-11-18 | npm workspaces | npm is the installed manager; supported by Vercel and Render |
| 2026-11-18 | SST opt-in per business, rate in basis points, snapshotted per document | Most pilots are under the RM500k threshold; 8% must never be hardcoded |
| 2026-11-18 | No MyInvois in v0.1, but carry TIN/SST/MSIC/classification fields | MSME exemption threshold is RM3m turnover; integration stays additive |
| 2026-11-18 | Offset pagination | Small datasets, page-number UIs; revisit with measurements |

## Open questions parked for v0.2

- Maintenance plan model: contract-based (customer buys 4 visits/year) or schedule-based (each unit
  has an interval)? Depends on how pilots actually sell servicing.
- Whether WhatsApp Business API (via a BSP, with template approval) is worth the cost and friction
  versus staff pasting links.
- Whether technicians need a multi-device handover of a part-filled job sheet.
- CSV import shape for equipment onboarding, once a real pilot's existing records have been seen.
