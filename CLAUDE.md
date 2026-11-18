# CLAUDE.md — Working agreement for MicroBiz Flow

Read this before touching anything. It is the operating manual, not a summary of the product.
Product scope lives in `PRD.md`; system design in `ARCHITECTURE.md`; data in `DATABASE.md`;
contracts in `API.md`; what to build next in `DEVELOPMENT_PLAN.md`.

## What this is

Malaysia-first SaaS for small and medium air-conditioning service businesses. The differentiator is
**equipment-centric service management**: every physical AC unit is a first-class record with its own
service history, inspections, warranty and maintenance schedule. The product replaces WhatsApp
threads, spreadsheets and paper job sheets.

Current release target: **v0.1 (MVP)**. See `DEVELOPMENT_PLAN.md` for the scope boundary.

## Stack (do not add to it without written justification)

| Layer | Choice |
|---|---|
| Frontend | Next.js (App Router), TypeScript, Tailwind CSS, shadcn/ui |
| Backend | Node.js, Express.js, TypeScript |
| Database | PostgreSQL 16 |
| ORM | Prisma |
| Validation | Zod (shared between web and api) |
| Background jobs | Redis + BullMQ — **deferred to v0.2**, do not introduce in v0.1 |
| Testing | Vitest (unit), Supertest (HTTP integration), Playwright (E2E) |
| Hosting | Vercel (web), Render (api + Postgres) |
| Files | S3-compatible private bucket, presigned URLs only |

Adding a dependency requires a one-line justification in the change summary and an entry in the
Decisions log in `DEVELOPMENT_PLAN.md`. Prefer the platform (Node 24 has `crypto.randomUUID`,
`fetch`, `AbortSignal.timeout`) over a package.

## Repository layout

```
apps/web          Next.js app (UI only — no business logic, never imports Prisma)
apps/api          Express modular monolith (all business logic, the only DB client)
packages/shared   Zod schemas, DTO types, money/date/phone helpers, enums, RBAC matrix
packages/db       Prisma schema, migrations, generated client, seed scripts
docs/             ADRs and supporting notes
```

npm workspaces. `apps/web` may import `packages/shared` only. `apps/api` may import both.
`packages/db` must never import from `apps/*`.

### Module boundaries inside `apps/api`

```
src/modules/<module>/
  <module>.routes.ts      Express router — HTTP only: parse, authorize, delegate, respond
  <module>.service.ts     business rules, transactions, invariants
  <module>.repository.ts  Prisma access (tenant-scoped client only)
  <module>.schema.ts      request/response Zod (re-exports from packages/shared where shared)
  <module>.test.ts        unit tests
```

A module may call another module's **service**, never its repository, and never another module's
tables directly. Modules for v0.1: `auth`, `businesses`, `users`, `customers`, `properties`,
`equipment`, `catalogue`, `quotations`, `jobs`, `reports`, `invoices`, `payments`, `files`, `audit`.

## Non-negotiable invariants

These are correctness requirements, not style preferences. Every one needs a test.

1. **Tenant scoping.** Every business-owned row carries `business_id`. All reads and writes go
   through the tenant-scoped Prisma client (`tenantClient(businessId)`); the raw client is reachable
   only from `packages/db` internals, migrations and seeds. A query without a tenant filter is a bug
   even if it currently returns correct rows.
2. **Authorization on the server.** The UI hides things for convenience; the API decides. Every
   route declares a required permission. Hiding a button is never the control.
3. **Technician isolation.** A technician reads and writes only jobs they are assigned to, and only
   the customer, property and equipment records reachable from those jobs.
4. **Money is integer sen.** Type `Int`, never floating-point arithmetic on ringgit. Field names end
   in `_sen`. Formatting to `RM 1,234.50` happens at the view edge only. Maximum representable value
   is RM 21,474,836.47 per field — enforced in Zod.
5. **Rounding is explicit.** Round half-up at the line level, then sum lines. Never sum unrounded
   values. One helper, `roundSen`, in `packages/shared`; no ad-hoc `Math.round` on money.
6. **Time is UTC in, Asia/Kuala_Lumpur out.** All timestamps are `timestamptz` stored in UTC.
   Formatting uses the business timezone (`Asia/Kuala_Lumpur`, UTC+8, no DST). Fields that mean "a
   calendar day in Malaysia" (invoice date, service due date, scheduled date) are `date`, not
   timestamp.
7. **Payments cannot exceed the balance.** Enforced inside the same transaction that reads the
   balance, with a row lock on the invoice. Over-collection goes through a credit note, never a
   negative payment.
8. **Financial records are append-only after issue.** An issued invoice is never edited or deleted —
   void it or credit it. Every financial mutation writes an audit event in the same transaction.
9. **Approved quotations keep revision history.** Changing an approved quotation creates a new
   version and supersedes the old one; the approved snapshot stays readable forever.
10. **Job completion requires inspection data.** The required fields depend on job type; the server
    rejects completion when they are missing. Not a UI-only check.
11. **Additional work needs customer approval.** Work beyond the approved quotation cannot be
    invoiced until an approval record exists.
12. **Public tokens are hashed at rest.** 256-bit random, shown once, stored as SHA-256,
    single-purpose, expiring, revocable. Never put a raw token in a log or an error message.
13. **No secrets in the repo.** Config comes from env, validated once at boot by a Zod schema; the
    process exits on invalid config. `.env.example` lists every key with a dummy value.
14. **No mock data standing in for a feature.** Development seeds and test fixtures are fine and must
    be obviously labelled as such. A screen that renders invented data instead of calling the API is
    not done.

## Conventions

- **TypeScript**: `strict: true`, `noUncheckedIndexedAccess: true`. No `any` — use `unknown` and
  narrow. No non-null assertions on values that come from I/O.
- **Validation**: every request body, query and param is parsed by Zod at the route edge. Service
  functions take already-validated types. Shared shapes live in `packages/shared` so a form and its
  endpoint cannot disagree.
- **Errors**: throw typed `AppError` subclasses (`ValidationError`, `NotFoundError`,
  `ForbiddenError`, `ConflictError`, `BusinessRuleError`). One Express error handler maps them to the
  wire format in `API.md`. Never leak a Prisma error or a stack trace to a client.
- **Logging**: `pino`, JSON, one line per request with `requestId`, `businessId`, `userId`, `route`,
  `status`, `durationMs`. Never log tokens, password hashes or the bodies of auth routes. The
  `requestId` is returned to the client so a user's screenshot is traceable to a log line.
- **Naming**: `snake_case` in Postgres, `camelCase` in TypeScript (Prisma `@map` / `@@map`). Database
  enums are `SCREAMING_SNAKE`, mirrored by string unions in `packages/shared`.
- **Dates on the wire**: ISO 8601 UTC with `Z` for timestamps; `YYYY-MM-DD` for calendar dates.
- **Phones**: stored E.164-normalized (`+60…`); one normalizer in `packages/shared`.
- **Migrations**: `prisma migrate dev` locally, committed to git; `prisma migrate deploy` in CI/CD.
  Never edit a committed migration. Never run `db push` against a shared database.
- **UI**: server components for data display, client components for interaction. Every list screen
  implements loading, empty, error and success states — an empty state says what to do next, not
  just "No data". Forms use labels tied to inputs, inline field errors, and keyboard-reachable
  controls. Technician screens are mobile-first: touch targets at least 44px, usable one-handed on a
  360px-wide screen by someone wearing gloves in a plant room.

## Commands

Scripts land in Milestone 1. Keep this table accurate as they do.

```
npm run dev            # web + api concurrently
npm run dev:api        # api only
npm run dev:web        # web only
npm run db:up          # docker compose up postgres
npm run db:migrate     # prisma migrate dev
npm run db:seed        # development seed data
npm run db:studio      # prisma studio
npm run typecheck      # tsc --noEmit across workspaces
npm run lint
npm run test           # vitest (unit + integration)
npm run test:e2e       # playwright
npm run verify         # typecheck + lint + test  <- run before reporting a milestone done
```

## How to work on this project

Work in small, verifiable milestones. Never attempt a whole module family in one pass.

For each milestone:

1. Re-read the relevant docs and the code you are about to change.
2. State the approach, dependencies and risks in two or three sentences.
3. Implement only the agreed milestone.
4. Write tests alongside — at minimum one test per invariant the milestone touches.
5. Run `npm run verify`. Fix what your change broke.
6. Update `PRD.md` / `ARCHITECTURE.md` / `DATABASE.md` / `API.md` if behaviour or contracts moved.
7. Update `DEVELOPMENT_PLAN.md`: tick completed items, adjust remaining ones, log decisions.
8. Summarize: files changed, what works now, tests added and their result, known gaps.

Rules of engagement:

- **Never silently skip a failing test.** Report it with the output. A test that is wrong gets fixed
  and the reason recorded; a test that is merely inconvenient stays red until the code is right.
- **Do not overwrite working functionality** without reading its callers and tests first.
- **Ask only when it matters** — architecture, security, data integrity, money, or product
  behaviour. Otherwise pick a sensible default and record the assumption in `DEVELOPMENT_PLAN.md`.
- **No scope creep.** Deferred means deferred; note the idea in the plan and move on.

## Malaysia specifics that are easy to get wrong

- **SST**: service tax is 8% and maintenance/repair is a taxable service, but registration is only
  mandatory above RM500,000 of taxable services in 12 months. Most target businesses are below that.
  SST is therefore **per-business opt-in** with a stored registration number, and taxability is set
  per catalogue item. Never hardcode 8% into a total.
- **LHDN e-Invoice (MyInvois)**: the MSME exemption threshold is RM3,000,000 annual turnover
  (General Guideline v4.8, August 2026), so typical pilots are exempt. No MyInvois submission in
  v0.1. The schema still carries business TIN, SST number, MSIC code and per-line classification so
  corporate customers can self-bill and so integration later is additive rather than a rewrite.
  Re-verify the current LHDN guideline version before scheduling that work.
- **Cash rounding**: cash is rounded to the nearest 5 sen. Record the invoice total and the cash
  rounding adjustment separately; never mutate the invoice total to make cash arithmetic work.
- **Addresses**: Malaysian state is a fixed enum (13 states + 3 federal territories); postcode is
  exactly 5 digits.
- **Language**: UI is English only for v0.1. Keep user-facing strings out of business logic so Bahasa
  Melayu can be added later without a rewrite, but do not build an i18n framework yet.
- **Connectivity**: technicians work in basements, plant rooms and car parks. Treat submission
  failure as normal: drafts persist on the device and submission retries. Never lose a filled job
  sheet.
