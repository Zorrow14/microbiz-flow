# ARCHITECTURE — MicroBiz Flow

Version: 0.1 · Last updated: 2026-11-18

This document records the shape of the system and *why* it has that shape. Decisions are numbered
(AD-n) so later changes can supersede them explicitly instead of drifting.

---

## 1. Topology

```
                    ┌──────────────────────────────┐
  Staff browser ───▶│  Next.js (apps/web)          │
  Technician phone  │  Vercel · App Router         │
  Public quote link │  UI, routing, form state     │
                    └──────────────┬───────────────┘
                                   │ HTTPS, JSON, credentialed
                                   ▼
                    ┌──────────────────────────────┐
                    │  Express API (apps/api)      │
                    │  Render · modular monolith   │
                    │  auth · tenancy · RBAC       │
                    │  all business logic          │
                    └───┬───────────────┬──────────┘
                        │               │
              ┌─────────▼────────┐  ┌───▼─────────────────┐
              │ PostgreSQL 16    │  │ S3-compatible store │
              │ Render · Prisma  │  │ private, presigned  │
              └──────────────────┘  └─────────────────────┘
```

Two deployed units, one database, one storage bucket. No message broker in v0.1.

### AD-1: Next.js frontend and Express backend as separate deployments

Given as a stack constraint. Honest consequences, recorded so they are not rediscovered later:

- The browser must talk to a different origin, which forces a deliberate auth-transport decision
  (see AD-4) and CORS configuration.
- We give up Next.js server-side data access and Server Actions as the data layer; `apps/web` is a
  client of the API like any other. React Server Components are still used for rendering, fetching
  from the API with the forwarded session.
- Two deploys must be released in a compatible order. API changes are additive-first: add the new
  field, ship the web change, remove the old field in a later release.
- Upside we actually want: the API is reusable by a future native app, a customer portal, and
  integrations, and business logic cannot leak into UI code because the UI has no database client.

### AD-2: Modular monolith, not services

One deployable API with hard internal module boundaries (see `CLAUDE.md`). Rationale: a 2–10
technician SaaS has no scaling problem that services solve, and it does have a correctness problem —
tenancy and money — that is far easier to get right inside one transaction boundary. Module
boundaries are enforced by convention plus an import lint rule, so extraction stays possible.

## 2. Monorepo

npm workspaces (npm is the installed package manager; no pnpm/yarn on the target machine, and
Vercel/Render both support npm workspaces natively).

```
microbiz-flow/
├── apps/
│   ├── api/                 Express, TypeScript, tsx in dev, tsc build
│   └── web/                 Next.js App Router
├── packages/
│   ├── shared/              Zod schemas, types, enums, money/date/phone, RBAC matrix
│   └── db/                  Prisma schema, migrations, client factory, seeds
├── docs/                    ADRs, pilot notes
├── docker-compose.yml       local Postgres (no local psql on dev machines)
└── package.json             workspaces + top-level scripts
```

### AD-3: `packages/shared` is the single source of truth for contracts

Request/response shapes, enums and money helpers are defined once in Zod and consumed by both sides.
The web form and the API endpoint validate against the same schema, so they cannot disagree about
whether `postcode` is five digits. Types are inferred from the schemas (`z.infer`), never hand-written
twice.

Constraint: `packages/shared` stays dependency-light (Zod only) and contains no I/O, no Prisma types,
no Express types. It must be importable from a browser bundle.

## 3. Authentication and sessions

### AD-4: httpOnly cookie sessions, with a same-site domain requirement

**Mechanism.** On login the API sets an `httpOnly`, `Secure` session cookie holding an opaque session
id (not a JWT). The session row lives in Postgres: revocable immediately, which FR-A4 and FR-A7
require and a stateless JWT cannot do. A short-lived JWT would need a revocation list to meet those
requirements, which is a session table with extra steps.

**The cross-origin problem.** `*.vercel.app` and `*.onrender.com` are different registrable domains,
so a session cookie would be a third-party cookie — blocked outright by Safari and increasingly by
Chrome. Many Malaysian technicians are on iPhones. This is not a theoretical concern: it breaks login.

**Required resolution (one of these must be chosen before Milestone 1 — see Decision D1):**

1. **Preferred — one registrable domain.** `app.microbizflow.my` → Vercel, `api.microbizflow.my` →
   Render. Requests become same-site, cookies work with `SameSite=Lax`, and nothing exotic is needed.
   Requires owning a domain and configuring DNS on both platforms.
2. **Fallback — proxy through Next.js.** The web app exposes `/api/*` and rewrites to the Render API,
   making every request same-origin. Works on free `*.vercel.app` domains; costs one network hop of
   latency per request and makes the proxy a dependency of the technician flow.

Both are implemented behind one `apiFetch` helper, so the choice is a configuration change and not a
rewrite. **Option 1 is assumed for planning.**

**CSRF.** Cookie auth needs CSRF protection: double-submit token, `SameSite=Lax`, and an `Origin`
allowlist check on every mutating request. State-changing requests are never `GET`.

**Passwords.** Argon2id (via `@node-rs/argon2`, no native build toolchain needed), with parameters
recorded in config so they can be raised later. Never bcrypt-with-defaults.

**Rate limiting.** Per-email and per-IP limits on login, invite acceptance, password reset and public
approval endpoints. In-memory limiter in v0.1 with a single API instance; moves to Redis in v0.2 when
Redis arrives for BullMQ. Documented limitation: scaling the API to two instances before then
weakens the limiter.

### AD-5: A user belongs to exactly one business in v0.1

`users.business_id` is non-null and the email is globally unique, so login needs no tenant selection
step. Limitation: one person cannot work for two businesses on one email. Migration path is a
`memberships` join table; everything downstream already resolves the active business from the
session, so this is additive. Recorded as a known limitation, not an oversight.

## 4. Multi-tenancy

### AD-6: Shared schema, `business_id` on every owned row, enforced by a scoped Prisma client

Rejected alternatives:

- *Schema-per-tenant*: migration pain scales with customer count; Prisma support is poor.
- *Database-per-tenant*: absurd at this size and on Render pricing.
- *Postgres RLS alone*: the right long-term defence, but with Prisma it requires a `SET LOCAL` on a
  transaction-bound connection for every request, which is easy to get subtly wrong and awkward with
  connection pooling. Planned as a hardening step (see below), not the primary mechanism.

**Primary mechanism.** A `tenantClient(businessId)` factory built on Prisma Client Extensions
(`$extends` with a `query` hook) injects `where: { businessId }` into every `find*`, `update*`,
`delete*` and `count` on tenant-owned models, and sets `businessId` on every `create`. Repositories
receive this client; they never import the base client.

```ts
// packages/db/src/tenant.ts  (shape, not final code)
export function tenantClient(businessId: string) {
  return basePrisma.$extends({
    query: {
      $allModels: {
        async $allOperations({ model, operation, args, query }) {
          if (!TENANT_OWNED.has(model)) return query(args);
          return query(applyTenantScope(operation, args, businessId));
        },
      },
    },
  });
}
```

**Why an extension rather than discipline:** a forgotten `where` clause is a cross-tenant data leak
with no symptom in development, where there is one business. The extension makes the safe path the
default path, and `TENANT_OWNED` is an explicit list so adding a model forces a decision.

**Defence in depth, in order of implementation:**

1. The scoped client (Milestone 1).
2. Integration tests that seed two businesses and assert every list and get endpoint returns 404 for
   the other tenant's ids (Milestone 1, and extended with every new module).
3. A lint rule banning imports of the base Prisma client outside `packages/db`.
4. Postgres RLS policies as a backstop (post-MVP hardening, tracked in the plan).

**Technician isolation** (FR-A6) is a second scope on top of tenancy, applied in the jobs/reports
services: the accessible job set is `jobs` joined to `job_assignments` for that user. Because it is a
row-level predicate rather than a column filter, it lives in explicit repository queries with tests,
not in the generic extension.

## 5. Request lifecycle

```
request
  │ requestId          crypto.randomUUID, echoed in responses and logs
  │ helmet             security headers
  │ cors               strict origin allowlist from config
  │ body parse         size-limited JSON
  │ rate limit         on auth and public routes
  │ session            cookie → session row → user → business; attaches req.ctx
  │ csrf               double-submit + Origin check on mutations
  ├─ route
  │    zod parse       body / query / params → typed input
  │    authorize       requirePermission('invoice:issue') against the RBAC matrix
  │    service call    business rules, one transaction per unit of work
  │    audit write     inside the same transaction for financial changes
  │ serialize          explicit DTO mapping — never return a Prisma row directly
  │ error handler      AppError → wire format; unknown → 500 + logged with requestId
response
```

`req.ctx` is `{ requestId, user, businessId, role, db }` where `db` is the tenant-scoped client. A
route that needs the base client is a design error.

**Explicit DTO mapping** is a rule, not a preference: returning Prisma rows leaks columns
(`passwordHash`, `tokenHash`, internal flags) the moment someone adds one.

## 6. Public (unauthenticated) surfaces

Only two in v0.1: quotation approval and additional-work approval. Both share one mechanism.

- Token: 32 bytes from `crypto.randomBytes`, base64url-encoded, in the URL path.
- Stored as SHA-256 only. The raw value exists in the response that creates it and nowhere else.
- One token, one purpose, one subject, one expiry (quotation validity date, or 14 days, whichever is
  sooner), revocable, single-decision (approve or reject marks it used).
- Lookup is by hash, constant-time compared, rate-limited per token and per IP.
- The public view returns a deliberately narrow DTO: business letterhead, document lines, totals,
  terms. No customer list, no other documents, no internal ids beyond what is needed.
- Approval records typed name, timestamp, IP and user agent as evidence.
- These routes bypass the session middleware entirely and mount under `/api/v1/public/*`, so there is
  no chance of an authenticated code path being reached with a null user.

## 7. Files and photos

### AD-7: Direct-to-bucket presigned uploads, private bucket, keys never guessable

The API never proxies file bytes — a Render instance streaming technician photos is wasted memory and
a latency trap on 3G.

Flow: client requests an upload intent (`POST /files/intents`) with content type and byte size → API
validates type and size against a per-subject policy, creates a `file` row in `PENDING`, returns a
presigned `PUT` URL valid for minutes → client uploads directly → client confirms
(`POST /files/:id/confirm`) → API marks `READY` and attaches it to its subject.

- Keys are `{businessId}/{subjectType}/{subjectId}/{uuid}.{ext}` — tenant-prefixed for auditability,
  random for unguessability.
- Reads are short-lived presigned `GET` URLs issued only after an authorization check, or via the
  public token for documents shown on an approval page.
- Images are resized and JPEG-compressed **on the device** before upload (target ~1600px, under
  ~400KB). Malaysian mobile data matters, and a 12MP phone photo of a filter is no more diagnostic
  than a 1600px one.
- Orphaned `PENDING` files are swept by a cleanup routine. In v0.1 that is an on-demand admin
  endpoint; in v0.2 it becomes a BullMQ repeatable job.

## 8. Money, time and numbering

- **Money**: integer sen in `Int` columns, field suffix `_sen`. `Int` not `BigInt` — the RM 21.47m
  per-field ceiling is far above any single aircon document, and `BigInt` poisons JSON serialization
  across the whole stack for no benefit. The ceiling is asserted in Zod and documented.
- **Arithmetic**: `packages/shared/money.ts` owns `roundSen` (half-up), line total computation,
  discount apportionment and SST computation. Nothing else does money arithmetic. The rule is: round
  at the line, then sum; never sum then round.
- **SST**: stored per business as `sstRegistered` plus `sstRateBasisPoints` (800 = 8%), applied only
  to lines whose catalogue item is taxable. Documents snapshot the rate used, so a later rate change
  does not rewrite history.
- **Time**: `timestamptz` everywhere, UTC in the database, `Asia/Kuala_Lumpur` for display. Calendar
  concepts (`issueDate`, `scheduledDate`, `nextServiceDueOn`) are `date` columns, because "19
  November in Malaysia" is not an instant and storing it as one creates off-by-one-day bugs at
  UTC+8.
- **Document numbers**: a `number_sequences` table keyed by `(business_id, kind, period)`, incremented
  with `UPDATE … RETURNING` inside the issuing transaction, giving gapless per-business sequences
  like `INV-2026-0042`. Not `count(*) + 1` (races), not a global sequence (leaks volume between
  tenants, and businesses expect their own numbering to start at 1).

## 9. Errors, logging, configuration

- **Error taxonomy**: `ValidationError` 422, `NotFoundError` 404, `ForbiddenError` 403,
  `UnauthorizedError` 401, `ConflictError` 409, `BusinessRuleError` 409 with a machine-readable
  `code`, `RateLimitError` 429. Everything else is a 500 with a generic message and a logged
  `requestId`.
- **Business rule errors carry codes** (`PAYMENT_EXCEEDS_BALANCE`, `JOB_INCOMPLETE_REPORTS`,
  `QUOTATION_ALREADY_APPROVED`) so the UI can render a specific, actionable message instead of
  echoing a server sentence.
- **Logging**: `pino`, JSON to stdout for Render's log drain. One request-completion line plus
  explicit domain events (`quotation.approved`, `payment.recorded`). Redaction list for tokens,
  cookies, password fields and auth bodies.
- **Config**: one `env.ts` per app, parsed by Zod at boot, exits non-zero on failure. No
  `process.env` access anywhere else. `.env.example` is complete and committed; real values never are.

## 10. Testing strategy

| Level | Tool | What it covers | Where |
|---|---|---|---|
| Unit | Vitest | Money arithmetic, rounding, SST, state machines, RBAC matrix, token hashing, phone normalization | beside the code |
| Integration | Vitest + Supertest + **real Postgres** | Every endpoint: validation, authorization, tenancy isolation, transactions, audit writes | `apps/api/test/` |
| E2E | Playwright | Quote → approve → job → report → invoice → payment, and the technician mobile flow at 360px | `apps/web/e2e/` |

Rules:

- Financial and tenancy logic is **never** tested against a mocked Prisma. A mock cannot show you a
  missing `where` clause or a broken transaction boundary.
- Integration tests run against a disposable database (docker-compose locally, service container in
  CI), migrated then truncated between tests. Not mocked, not shared with development data.
- Every invariant in `CLAUDE.md` maps to at least one named test, so the suite reads as a
  specification.
- The cross-tenant test is a helper (`expectTenantIsolation`) applied to every resource route, so new
  modules inherit the check instead of remembering it.

## 11. Deployment and environments

| Environment | Web | API | Database |
|---|---|---|---|
| Local | `next dev` | `tsx watch` | docker-compose Postgres 16 |
| Staging | Vercel preview | Render service | Render Postgres (separate instance) |
| Production | Vercel production | Render service | Render Postgres + daily backup |

- Migrations run as a Render pre-deploy command (`prisma migrate deploy`). Migrations are
  additive-first: add column → deploy API → deploy web → remove old column in a later release. No
  destructive migration ships in the same release as the code that stops using the column.
- CI on every push: install, typecheck, lint, unit + integration tests against a Postgres service
  container, `npm audit` at moderate. E2E on pull requests to `main`.
- Secrets live in Vercel/Render environment settings. Rotation procedure documented in
  `docs/runbook.md` (written in the milestone that first provisions staging).
- **Before the first pilot loads real customer data**: a restore from backup must be rehearsed and
  the result written down. An untested backup is not a backup. Tracked as a plan gate, not a wish.

## 12. Known gaps and planned hardening

Recorded so they are choices rather than surprises:

| Gap | Impact | Planned |
|---|---|---|
| No Postgres RLS | Tenancy depends on application code plus tests | Post-MVP hardening |
| In-memory rate limiter | Weakens if the API runs more than one instance | Redis in v0.2 |
| No background jobs | Anything slow happens in-request; no scheduled work | BullMQ in v0.2 |
| No automated notification delivery | Humans paste links into WhatsApp | v0.2 |
| Single region (Singapore) | Latency fine for Malaysia; no DR region | Accepted for v0.1 |
| Device-local technician drafts only | Multi-device handover of a part-filled job sheet does not work | Validate need with pilots |
| Email-unique users, one business per user | A technician cannot serve two businesses | `memberships` table when needed |
