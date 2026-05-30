# Family Task & Chore Manager — Phased Development Plan

> Project: 357-family-task-chore-manager · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. The backend adopts **data-model-suggestion-1 (Entity-Centric Normalized Relational, PostgreSQL, 12 tables)** because the allowance ledger, rotation/fairness algorithms, and per-member analytics all depend on explicit assignment and transaction records.

The product is delivered as: a **TypeScript REST + WebSocket API** (the system of record), a **React Native (Expo) mobile app** for iOS/Android (parent and child experiences), and a **Next.js web app** for desktop parent management. AI features run server-side against an LLM provider. Push uses FCM (which bridges to APNs for iOS). An **MCP server** exposes family chore state to AI assistants — a documented competitive gap (no incumbent ships one).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | One language across API, web, mobile, and MCP server. Strong typing for the financial ledger and shared DTO types between server and clients. The domain is integration/CRUD/realtime-heavy, not numeric-compute-heavy, so TS beats Python or Go here. |
| API framework | Fastify 5 | High throughput, first-class JSON Schema validation (aligns with the project's JSON Schema 2020-12 requirement), built-in serialization, and `@fastify/swagger` auto-generates the OpenAPI 3.1 spec the standards doc mandates. |
| API style | REST (OpenAPI 3.1) + WebSocket | REST for CRUD; WebSocket (RFC 6455) for real-time chore/completion sync across family devices, as required by research §4. Pagination uses RFC 8288 `Link` headers. |
| Database | PostgreSQL 16 | Data-model-suggestion-1 requires referential integrity, partial indexes, JSONB columns, CHECK constraints, and range-partitioned audit logs — all native to Postgres. `gen_random_uuid()` via `pgcrypto`. |
| ORM / query layer | Drizzle ORM | Type-safe SQL-first ORM. Schema defined in TS mirrors the DDL exactly; migrations are generated and versioned. Avoids the abstraction tax of heavier ORMs while keeping the ledger queries explicit. |
| Migrations | drizzle-kit | Deterministic, reviewable SQL migrations checked into the repo. |
| Task queue / scheduler | BullMQ on Redis | Async workloads: scheduled chore-instance generation (cron-like), push-notification fan-out, AI photo verification, and quiet-hours-deferred deliveries. |
| Cache / pub-sub | Redis 7 | Backs BullMQ and the WebSocket pub/sub adapter for horizontal scaling. |
| Auth | OAuth 2.0 / OIDC (Sign in with Apple + Google) + email/password for parents; PIN/biometric for children | Standards doc requires OIDC and NIST SP 800-63B-aligned AAL separation: parents = AAL2 (password + optional MFA), children = AAL1 (PIN). |
| Session tokens | JWT (RFC 7519), short-lived access + rotating refresh | Token claims carry `family_id` and `role` (parent/child/guardian) to enforce role-based access at the API edge. |
| AI provider | Anthropic Claude (server-side), abstracted behind an `LLMProvider` interface | NL chore parsing, age-appropriate suggestions, and photo-proof vision verification. Provider abstraction allows swapping models without touching call sites. |
| Push notifications | Firebase Cloud Messaging (FCM v1 HTTP API) | Single API for Android + iOS (FCM bridges to APNs). 4096-byte payload limit respected. |
| Mobile app | React Native via Expo (SDK 53+) | Single codebase for iOS + Android, OTA updates, native push + camera (photo proof) + home-screen widgets via config plugins. |
| Web app | Next.js 16 (App Router) | Desktop parent management UI; server components for the dashboard, W3C Push API for browser notifications. |
| Local store (mobile offline) | WatermelonDB (SQLite-backed) | Offline-first chore lists with background reconciliation against the server (a documented competitor gap). |
| Recurrence | `rrule` (JS) over iCalendar RRULE strings (RFC 5545) | `chores.rrule` stores standard RRULE; the same string drives calendar export (VTODO) and instance generation. |
| Calendar export | `ical-generator` producing VTODO components | iCalendar export/import per RFC 5545 + CalConnect CC/11001:2024 task extensions. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | First-party MCP server exposing family chore tools to Claude/AI assistants — the standards doc flags this as a distribution advantage no competitor has. |
| Object storage | S3-compatible (MinIO in dev, S3 in prod) | Photo-proof image storage with signed, expiring URLs. |
| Validation | TypeBox (JSON Schema 2020-12) | Single source of truth: TypeBox schemas validate requests AND emit the OpenAPI component schemas. |
| Testing | Vitest (unit/integration), Supertest (HTTP), Testcontainers (real Postgres/Redis), Detox (mobile E2E), Playwright (web E2E) | Layered strategy; Testcontainers gives real-DB integration tests without external setup. |
| Code quality | ESLint + Prettier + `tsc --noEmit` + Biome (fast CI lint) | Lint, format, and full type-check gates per phase. |
| Package manager / monorepo | pnpm workspaces + Turborepo | Shared `packages/shared` (DTOs, validation schemas) consumed by api, web, mobile, mcp. |
| Containerisation | Docker + docker-compose | Self-hostable stack: api, worker, postgres, redis, minio. |
| CI | GitHub Actions | Lint → type-check → unit → integration (Testcontainers) → build per package. |

### Project Structure

```
family-task-chore-manager/
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # postgres, redis, minio, api, worker
├── .github/workflows/ci.yml
├── packages/
│   └── shared/                       # consumed by all apps
│       ├── src/
│       │   ├── schemas/              # TypeBox schemas (chore, member, completion, ...)
│       │   ├── types/                # inferred TS types / DTOs
│       │   ├── enums.ts              # roles, statuses, categories, txn types
│       │   └── index.ts
│       └── package.json
├── apps/
│   ├── api/
│   │   ├── Dockerfile
│   │   ├── drizzle.config.ts
│   │   ├── src/
│   │   │   ├── index.ts              # Fastify bootstrap
│   │   │   ├── config.ts             # env parsing + defaults
│   │   │   ├── db/
│   │   │   │   ├── schema.ts         # Drizzle schema (12 tables)
│   │   │   │   ├── client.ts
│   │   │   │   └── migrations/
│   │   │   ├── plugins/              # auth, swagger, websocket, error-handler
│   │   │   ├── auth/                 # OIDC, JWT, RBAC guard
│   │   │   ├── routes/               # families, members, chores, assignments,
│   │   │   │                         #   completions, rewards, allowance, analytics
│   │   │   ├── services/             # business logic (domain layer)
│   │   │   │   ├── scheduling/       # rrule expansion, assignment generation, rotation
│   │   │   │   ├── ledger/           # allowance transaction posting
│   │   │   │   ├── gamification/     # streaks, badges
│   │   │   │   ├── ai/               # LLMProvider, nl-parse, suggestions, vision
│   │   │   │   └── notifications/    # FCM, quiet-hours
│   │   │   ├── realtime/             # WebSocket hub + Redis pub/sub
│   │   │   └── worker/               # BullMQ processors (scheduler, push, ai)
│   │   └── test/
│   │       ├── unit/
│   │       ├── integration/          # Testcontainers
│   │       └── fixtures/
│   ├── mcp/
│   │   └── src/index.ts              # MCP server (tools over the API)
│   ├── web/                          # Next.js 16 parent dashboard
│   └── mobile/                       # Expo RN parent + child app
└── README.md
```

---

## Phase 1: Foundation — Monorepo, Database Schema, Config

### Purpose
Establish the monorepo, the PostgreSQL schema (all 12 tables from data-model-suggestion-1), shared validation schemas, and a bootable Fastify server with health checks. After this phase the data layer exists, migrations run, and every later phase adds routes/services without restructuring.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Create the pnpm + Turborepo workspace with `packages/shared` and `apps/api`, plus lint/format/type-check pipelines.

**Design**:
- `pnpm-workspace.yaml` globs `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test` with `dependsOn: ["^build"]`.
- Root `tsconfig.base.json` with `strict: true`, `noUncheckedIndexedAccess: true`, path alias `@shared/*`.
- `docker-compose.yml` services:
  ```yaml
  postgres: { image: postgres:16, ports: ["5432:5432"], env: POSTGRES_PASSWORD }
  redis:    { image: redis:7, ports: ["6379:6379"] }
  minio:    { image: minio/minio, ports: ["9000:9000","9001:9001"] }
  ```
- `apps/api/src/config.ts` parses env via TypeBox; required vars + defaults:
  ```
  DATABASE_URL                 (required)
  REDIS_URL                    (default redis://localhost:6379)
  JWT_ACCESS_SECRET            (required)
  JWT_REFRESH_SECRET           (required)
  ACCESS_TOKEN_TTL_SEC         (default 900)
  REFRESH_TOKEN_TTL_SEC        (default 1209600)
  S3_ENDPOINT / S3_BUCKET / S3_KEY / S3_SECRET
  FCM_PROJECT_ID / FCM_CREDENTIALS_JSON
  ANTHROPIC_API_KEY
  PORT                         (default 3000)
  ```
  Invalid/missing required env → process exits with a message naming the offending var.

**Testing**:
- Unit: `loadConfig({})` with no `DATABASE_URL` → throws listing `DATABASE_URL`.
- Unit: `loadConfig({DATABASE_URL, JWT_ACCESS_SECRET, JWT_REFRESH_SECRET})` → defaults applied (`PORT===3000`, `ACCESS_TOKEN_TTL_SEC===900`).
- CI: `pnpm -w lint && pnpm -w typecheck` pass on the empty scaffold.

#### 1.2 — Database schema (12 tables) + migrations

**What**: Implement the Drizzle schema mirroring data-model-suggestion-1 and generate the initial migration.

**Design**:
- Enable `pgcrypto` (`gen_random_uuid()`) in migration 0000.
- Tables, columns, CHECK constraints, and indexes exactly per data-model-suggestion-1: `families`, `members`, `chores`, `chore_assignments`, `completions`, `rewards`, `reward_redemptions`, `allowance_transactions`, `badges`, `member_badges`, `streaks`, `audit_log` (range-partitioned by `created_at`, with a default monthly partition).
- Enums implemented as Postgres `CHECK` constraints (not native enums) to allow additive values without type migrations; mirrored in `packages/shared/src/enums.ts`.
- Money is `BIGINT` cents throughout; never floats.
- Partial indexes preserved, e.g.:
  ```sql
  CREATE INDEX idx_assignments_pending ON chore_assignments (member_id, due_date)
    WHERE status = 'pending';
  CREATE INDEX idx_completions_pending ON completions (family_id)
    WHERE approved_by IS NULL AND rejected_reason IS NULL;
  ```
- `created_at`/`updated_at` triggers (or app-layer `updated_at` on write).

**Testing**:
- Integration (Testcontainers real Postgres): run all migrations up → assert 12 tables + expected indexes present (`pg_indexes`).
- Integration: insert `members.role = 'sibling'` → fails CHECK constraint.
- Integration: insert `allowance_transactions` with negative `amount_cents` for `withdrawal` and assert balance math is left to the service (no DB-level prohibition) — documents the contract.
- Integration: migrations down → up is idempotent (clean re-up).

#### 1.3 — Shared schemas & DTO types

**What**: TypeBox schemas in `packages/shared` for every entity, exporting both runtime validators and inferred TS types.

**Design**:
- One schema module per entity: `ChoreSchema`, `ChoreCreateSchema` (omits server-set fields), `MemberSchema`, `CompletionSchema`, etc.
- Enums centralised: `Role`, `AgeGroup`, `ChoreCategory`, `Difficulty`, `RecurrenceType`, `AssignmentMode`, `AssignmentStatus`, `RewardType`, `RedemptionStatus`, `TransactionType`, `StreakType`.
- `export type Chore = Static<typeof ChoreSchema>` pattern so server + clients share types.
- A `Paginated<T>` envelope schema for list endpoints.

**Testing**:
- Unit: valid chore payload → `Value.Check(ChoreCreateSchema, payload)` true.
- Unit: `points_value: -1` → check fails (schema enforces `minimum: 0`).
- Unit: enum drift guard — test asserts `Role` values match the DB CHECK list (hard-coded expected array).

#### 1.4 — Fastify bootstrap, health, error handling, OpenAPI

**What**: Bootable server with `/health`, global error handler, and `@fastify/swagger` serving OpenAPI 3.1 at `/docs`.

**Design**:
- Plugins registered in order: error-handler → swagger → (auth/websocket added later).
- `GET /health` → `{ status: 'ok', db: 'up'|'down', redis: 'up'|'down' }` (pings both).
- Error handler maps domain errors to RFC 9457 problem+json:
  ```ts
  class AppError extends Error { constructor(public status:number, public code:string, msg:string){...} }
  // 400 VALIDATION, 401 UNAUTHENTICATED, 403 FORBIDDEN, 404 NOT_FOUND, 409 CONFLICT
  ```
- TypeBox schemas attached to routes feed `@fastify/swagger` component schemas automatically.

**Testing**:
- Integration (Supertest): `GET /health` with DB up → 200, `db:'up'`.
- Integration: unhandled `AppError(404,...)` → response is problem+json with `status:404`, `code`.
- Integration: `GET /docs/json` → valid OpenAPI 3.1 document (validate with `@apidevtools/swagger-parser`).

---

## Phase 2: Identity, Families & Role-Based Access

### Purpose
Implement family creation, member management, the auth system (parent OIDC/password + child PIN), JWT issuance with role claims, and the RBAC guard that every later route depends on. COPPA/GDPR-K parental-consent capture is foundational here, per the README and standards doc.

### Tasks

#### 2.1 — Family & member CRUD

**What**: Endpoints to create a family, add/edit/deactivate members, and read the family roster.

**Design**:
- `POST /families` → creates `families` row + the founding parent `members` row; returns family + access/refresh tokens. Generates unique `invite_code`.
- `GET /families/:id` (parent only) → family + members.
- `POST /families/:id/members` (parent only) → add child/guardian; `birth_date` → server derives `age_group`:
  ```
  toddler 0-2, preschool 3-5, elementary 6-10, middle_school 11-13, high_school 14-17, adult 18+
  ```
- `PATCH /members/:id`, `POST /members/:id/deactivate` (sets `is_active=false`).
- All writes append an `audit_log` row (`actor_type`, `action`, `entity_type`, `entity_id`, `changes_json`).

**Testing**:
- Integration: `POST /families` → 201, family persisted, parent member created, tokens returned.
- Integration: `POST /families/:id/members` with `birth_date` 2018-01-01 (as of 2026) → `age_group:'elementary'`.
- Integration: child token calls `POST /families/:id/members` → 403.
- Integration: every successful write creates an `audit_log` entry.

#### 2.2 — Authentication (OIDC, password, child PIN) + JWT

**What**: Parent sign-in via Sign in with Apple/Google (OIDC) or email+password; child sign-in via family-scoped PIN; JWT issuance and refresh rotation.

**Design**:
- OIDC: verify ID token against Apple/Google JWKS (`openid-client`); match/create parent member by verified email.
- Password: Argon2id hashing; login → access + refresh JWT.
- Child PIN: `pin_hash` (Argon2id); `POST /auth/child` `{family_id, member_id, pin}` → AAL1 access token (shorter TTL, restricted scope claim `aal:1`).
- JWT claims: `{ sub: member_id, fam: family_id, role, aal }`.
- Refresh rotation: refresh tokens stored hashed in Redis with jti; reuse of a rotated token revokes the chain (NIST 800-63B reuse detection).

**Testing**:
- Unit: Argon2 verify round-trip; wrong PIN → false.
- Integration (mocked JWKS): valid Google ID token → parent token issued.
- Integration: child PIN login → token has `role:'child'`, `aal:1`.
- Integration: reuse a rotated refresh token → 401 and active sessions for that chain revoked.

#### 2.3 — RBAC guard & family-scoping middleware

**What**: A Fastify preHandler enforcing role + family scope on every protected route.

**Design**:
- `requireAuth()` verifies JWT; attaches `req.actor = {memberId, familyId, role, aal}`.
- `requireRole('parent')` / `requireParentOrSelf(paramMemberId)` decorators.
- Family-scoping: any `:familyId`/resource lookup must match `req.actor.familyId` or → 403. Children may only read/act on their own assignments and rewards.
- Default-deny: routes without an explicit auth decorator are rejected by a catch-all in non-test env.

**Testing**:
- Integration: parent of family A requests family B resource → 403.
- Integration: child requests another child's allowance ledger → 403.
- Integration: missing/expired token → 401.
- Unit: `requireParentOrSelf` allows child reading own member; denies reading sibling.

#### 2.4 — COPPA / GDPR-K parental consent capture

**What**: Capture and store verifiable parental consent before any child profile becomes active.

**Design**:
- Adding a child sets `members.is_active=false` until `families.coppa_consent_json` covers that child.
- `POST /families/:id/consent` (parent) records:
  ```json
  { "consented_at":"<ts>", "consent_method":"email_plus_id|knowledge_based|digital_signature|video",
    "parent_id":"uuid", "children_covered":["uuid"], "data_minimisation_acknowledged":true,
    "security_coordinator":"uuid" }
  ```
- On valid consent covering a child → set that child `is_active=true`.
- Data minimisation: child members reject `email` unless explicitly allowed; only `birth_date`→`age_group` retained where possible.

**Testing**:
- Integration: child added without consent → `is_active:false`, cannot authenticate.
- Integration: `POST consent` covering the child → child activated.
- Integration: consent missing `consent_method` → 400 naming the field.

---

## Phase 3: Chores & Scheduling Engine (Core Value)

### Purpose
The heart of the product: defining chores with recurrence (RFC 5545 RRULE) and assignment modes, then generating `chore_assignments` instances — including rotation, alternating, and "up for grabs". After this phase a family can define real chore schedules and see today's/this-week's assignments.

### Tasks

#### 3.1 — Chore CRUD

**What**: Create/read/update/deactivate chores with categories, difficulty, points/allowance value, subtasks, recurrence, and assignment mode.

**Design**:
- `POST /families/:id/chores` (parent) validated against `ChoreCreateSchema`.
- `recurrence_type` ∈ once|daily|weekly|biweekly|monthly|custom; if not `once`, `rrule` required and must parse via `rrule` lib.
- `assignment_mode` ∈ fixed|rotating|alternating|up_for_grabs; `fixed`/`rotating`/`alternating` require a non-empty assignee member list (stored in a `chore_assignees` join — add as part of this task: a small table `chore_assignees(chore_id, member_id, rotation_order)`).
- `up_for_grabs` requires no assignees.
- `requires_photo`, `requires_approval`, `condition_based`+`urgency_days` supported.

**Testing**:
- Integration: weekly chore with `RRULE:FREQ=WEEKLY;BYDAY=SA` → persisted, validates.
- Integration: recurrence_type=weekly with no rrule → 400.
- Integration: rotating mode with empty assignee list → 400.
- Unit: invalid RRULE string → validation error before DB write.

#### 3.2 — Recurrence expansion & assignment generation

**What**: A service that expands a chore's RRULE over a date window and creates `chore_assignments` rows, applying the assignment mode.

**Design**:
```ts
function generateAssignments(chore: Chore, assignees: ChoreAssignee[], window: {from: Date; to: Date}): NewAssignment[]
```
- Expand RRULE via `rrule` to occurrence dates within `[from, to]`.
- Per occurrence, choose assignee(s) by mode:
  - `fixed`: every listed assignee (or the single one).
  - `rotating`: round-robin by `rotation_order`, advancing an index persisted per chore (`chores`-adjacent counter) so successive due dates cycle members.
  - `alternating`: two-member A/B by occurrence parity.
  - `up_for_grabs`: create one unassigned-claimable assignment (member_id nullable for this mode — relax FK with a sentinel "open" handling; store claimant on completion).
- Idempotency: `UNIQUE(chore_id, member_id, due_date)` (add index) so re-running generation never duplicates.
- `once` chores generate exactly one assignment on `due_date`.

**Testing**:
- Unit: `FREQ=WEEKLY;BYDAY=MO` over a 4-week window → 4 dates.
- Unit: rotating with 3 members over 6 occurrences → each member assigned exactly twice in order.
- Unit: alternating with members [A,B] over 4 occurrences → A,B,A,B.
- Integration: run generation twice for the same window → no duplicate rows (idempotent).

#### 3.3 — Scheduled instance generation (worker)

**What**: A BullMQ repeatable job that rolls the assignment window forward daily for all active chores.

**Design**:
- Daily cron job (per family timezone) materialises assignments for a rolling horizon (default 14 days ahead, configurable `ASSIGNMENT_HORIZON_DAYS`).
- Marks past-due `pending` assignments as `missed` once `due_date < today` (family tz) and the chore was not condition-based.
- Emits a WebSocket `assignments.updated` event (Phase 6) and queues due-reminder notifications (Phase 7).

**Testing**:
- Integration (real Redis + Postgres via Testcontainers): run the job → assignments exist for the horizon.
- Integration: an un-acted pending assignment with `due_date` = yesterday → transitions to `missed`.
- Integration: condition-based chore never auto-`missed`.

#### 3.4 — Assignment listing & query endpoints

**What**: Endpoints to fetch assignments by member/family/date with status filters and RFC 8288 pagination.

**Design**:
- `GET /families/:id/assignments?member=&from=&to=&status=` → paginated; children scoped to self.
- `GET /members/:id/today` → today's assignments for a child's home screen.
- Response uses `Paginated<Assignment>`; `Link` header with `rel="next"`/`"prev"`.

**Testing**:
- Integration: child fetches `/today` → only own assignments, only today (family tz).
- Integration: parent filters `status=missed` → only missed returned.
- Integration: pagination beyond page 1 → `Link: rel="next"` honored.

---

## Phase 4: Completion, Approval & Allowance Ledger

### Purpose
Close the core loop: a child submits a completion (optionally with photo proof), a parent approves/rejects, and approval posts points and an allowance ledger transaction. This phase makes the app deliver value — earning and tracking.

### Tasks

#### 4.1 — Completion submission

**What**: Child marks an assignment complete, optionally attaching photo proof and subtask checkmarks.

**Design**:
- `POST /assignments/:id/complete` (child = assignee, or any for up_for_grabs → becomes claimant):
  - Sets assignment `status` → `submitted` (if `requires_approval`) else `approved`.
  - Creates a `completions` row (`member_id`, `subtasks_completed_json`, `notes`, optional `photo_url`, `time_spent_minutes`).
  - For `up_for_grabs`, sets the claiming member and prevents a second claim (row lock / status check → 409).
- Photo upload: `POST /uploads/photo-proof` returns a signed S3 PUT URL; client uploads directly; `photo_url` references the object key.
- If `requires_approval=false`, immediately invoke the ledger posting (4.3).

**Testing**:
- Integration: assignee completes a no-approval chore → assignment `approved`, completion row created, ledger posted.
- Integration: two children race an up_for_grabs chore → first 200, second 409.
- Integration: non-assignee child completes a fixed chore → 403.

#### 4.2 — Parent approval / rejection

**What**: Parent approves or rejects a submitted completion; dual-parent approval optionally required.

**Design**:
- `POST /completions/:id/approve` / `POST /completions/:id/reject {reason}` (parent).
- If `families.settings_json.dual_parent_approval=true`, first approval records a pending second-approver state; the ledger posts only after the second distinct parent approves.
- Approve → assignment `status:'approved'`, completion `approved_by/approved_at`, trigger ledger posting (4.3) and gamification (Phase 5).
- Reject → assignment `status:'rejected'`, `rejected_reason`; no ledger effect; notifies child.

**Testing**:
- Integration: parent approves → assignment approved, allowance/points posted once.
- Integration (dual approval on): one parent approves → no ledger yet; second parent approves → ledger posts exactly once.
- Integration: same parent approves twice under dual mode → still no posting (must be distinct).
- Integration: reject → no `allowance_transactions` row created.

#### 4.3 — Allowance & points ledger posting (transactional)

**What**: A ledger service that, in a single DB transaction, increments balances and writes an `allowance_transactions` row with `balance_after_cents`.

**Design**:
```ts
async function postEarning(tx, { memberId, familyId, completionId, points, cents }): Promise<void>
```
- Within a `SERIALIZABLE` (or `SELECT ... FOR UPDATE` on the member row) transaction:
  1. Lock member row; read current `allowance_balance_cents`, `points_balance`.
  2. Update balances.
  3. Insert `allowance_transactions` (`transaction_type:'chore_earned'`, `amount_cents:cents`, `balance_after_cents`, `reference_type:'completion'`, `reference_id:completionId`).
  4. Update `completions.points_earned`, `allowance_earned_cents`.
- Idempotency: unique `(reference_type, reference_id)` on chore_earned to prevent double-posting on retries.
- Parent manual adjustments: `POST /members/:id/allowance/adjust {amount_cents, description}` → `transaction_type:'parent_adjustment'`.

**Testing**:
- Integration: approve completion worth 200 cents → balance +200, one transaction, `balance_after` correct.
- Integration: retry the same approval (idempotency key) → no second transaction.
- Integration: concurrent approvals on the same member → balances consistent (no lost update); run with two parallel transactions.
- Unit: `balance_after_cents` always equals prior balance + amount.

#### 4.4 — Allowance ledger & balance endpoints

**What**: Read endpoints for a member's balance and paginated transaction history.

**Design**:
- `GET /members/:id/balance` → `{ points_balance, allowance_balance_cents, currency }`.
- `GET /members/:id/allowance/transactions` → paginated, newest first (mirrors the data-model example query).
- Child sees own; parent sees any child in family.

**Testing**:
- Integration: child reads own balance → matches ledger sum.
- Integration: transactions endpoint ordered by `created_at DESC`, paginated.
- Integration: child reads sibling's transactions → 403.

---

## Phase 5: Rewards, Gamification (Streaks, Badges) & Analytics

### Purpose
Sustain engagement: a family reward catalog with redemption + parent approval, streak tracking, badge awarding, and the parent-facing workload-fairness dashboard (a documented market gap).

### Tasks

#### 5.1 — Reward catalog & redemption

**What**: Parents define rewards; children redeem with points; parents approve and fulfil.

**Design**:
- `POST /families/:id/rewards` (parent) → `rewards` row (`reward_type`, `points_cost`/`allowance_cents`).
- `POST /rewards/:id/redeem` (child) → checks `points_balance >= points_cost`; creates `reward_redemptions` (`status:'pending'`); does NOT yet deduct points (held).
- `POST /redemptions/:id/approve` (parent) → deducts points within a ledger transaction (points side), sets `fulfilled`; for `allowance_bonus` type posts an `allowance_transactions` bonus.
- `reject` → no deduction.
- Insufficient points → 409.

**Testing**:
- Integration: redeem with enough points → pending; approve → points deducted once, status fulfilled.
- Integration: redeem with insufficient points → 409, no row.
- Integration: reject redemption → points unchanged.

#### 5.2 — Streak engine

**What**: Maintain per-member streaks on approved completions.

**Design**:
- On completion approval, update `streaks`:
  - `daily_completion`: if `last_completed_date` == yesterday → `current_count+1`; == today → no change; else reset to 1. Update `longest_count = max(longest, current)`.
  - `category_streak`: same logic scoped by `category`.
- `weekly_all_done`: a weekly job checks whether all of a member's week assignments are approved → increment.

**Testing**:
- Unit: completions on consecutive days → `current_count` increments; gap day → resets to 1.
- Unit: two completions same day → count unchanged.
- Unit: `longest_count` never decreases.

#### 5.3 — Badge awarding

**What**: Award badges when criteria are met; expose earned badges.

**Design**:
- System badges seeded (`is_system=true`) with `criteria_json` e.g. `{"type":"streak","days":7}`, `{"type":"completion_count","count":50}`, `{"type":"category_master","category":"kitchen","count":20}`.
- After approval/streak update, evaluate criteria for the member; on satisfy, insert `member_badges` (UNIQUE prevents re-award) and emit notification + WebSocket event.
- `GET /members/:id/badges`.

**Testing**:
- Integration: reach a 7-day streak → 7-day badge awarded once; further days do not re-award (UNIQUE).
- Integration: 50th approved completion → completion_count badge awarded.
- Unit: criteria evaluator returns the correct badge set for given member stats.

#### 5.4 — Workload-fairness & analytics dashboard

**What**: Parent analytics: per-member contribution, completion rates, and fairness over a window.

**Design**:
- `GET /families/:id/analytics/workload?days=7` → per-member `{ completions, points, allowance_cents }` (the data-model fairness query).
- `GET /families/:id/analytics/chore-completion?days=30` → per-chore completion rate (lowest-rate query).
- `GET /families/:id/analytics/fairness` → a normalized fairness score (e.g. Gini or max/min ratio of completion counts among children) with a "balanced/imbalanced" flag and the most over/under-loaded member.

**Testing**:
- Integration: seed completions skewed to one child → fairness flagged imbalanced, names the overloaded member.
- Integration: workload endpoint sums match seeded data.
- Integration: chore-completion rate computed correctly for a chore with 3/5 approved.

---

## Phase 6: Real-Time Sync & Offline Support

### Purpose
Deliver the cross-device real-time experience and offline-first mobile behaviour that competitors largely lack. Family members see chore/completion/balance changes live; mobile works offline and reconciles on reconnect.

### Tasks

#### 6.1 — WebSocket hub with Redis pub/sub

**What**: Authenticated WebSocket channel broadcasting family-scoped events.

**Design**:
- `GET /ws` upgrade; auth via access token in the connection (subprotocol or first message); subscribe the socket to room `family:{familyId}`.
- Server publishes events to Redis; all API/worker instances fan out to local sockets in that room.
- Event envelope:
  ```ts
  type RTEvent =
    | {type:'assignment.updated'; assignmentId:string; status:string}
    | {type:'completion.submitted'; completionId:string; memberId:string}
    | {type:'completion.approved'; completionId:string}
    | {type:'balance.updated'; memberId:string; points:number; allowanceCents:number}
    | {type:'badge.awarded'; memberId:string; badgeId:string};
  ```
- Children receive only events relevant to them or family-wide non-sensitive events (no sibling balance details).

**Testing**:
- Integration: two clients in family A; parent approves → both receive `completion.approved`; client in family B receives nothing.
- Integration: unauthenticated WS connect → closed with 4401.
- Integration: child socket does not receive sibling `balance.updated`.

#### 6.2 — Sync protocol (delta pull) & offline reconciliation

**What**: A pull-based delta sync endpoint plus mobile write queue for offline actions.

**Design**:
- `GET /families/:id/sync?since=<cursor>` → entities changed since cursor (`updated_at`-based), with a new cursor. Used on reconnect to catch up missed WS events.
- Mobile (WatermelonDB): local writes (complete chore, check subtask) queued with a client-generated `idempotency_key`; on reconnect, replay against the API; server dedupes by key.
- Conflict policy: server is authoritative; an offline completion of an assignment already approved/missed → server returns the canonical state and the client reconciles.

**Testing**:
- Integration: change made while client offline → appears in `?since` delta on reconnect.
- Integration: replay the same queued completion twice (same idempotency_key) → single effect.
- Integration: offline completion of an already-`missed` assignment → server responds with canonical `missed`, no duplicate ledger.

---

## Phase 7: Notifications (FCM, Quiet Hours)

### Purpose
Deliver reliable, well-timed push notifications — due reminders, completion alerts, approval requests, and milestone celebrations — respecting per-family quiet hours.

### Tasks

#### 7.1 — Device registration & FCM integration

**What**: Register device tokens and send via FCM v1.

**Design**:
- `POST /members/:id/devices {token, platform}` → appends to `members.device_tokens_json`.
- `NotificationService.send(memberId, payload)`: looks up tokens, sends via FCM v1 HTTP (`https://fcm.googleapis.com/v1/projects/{id}/messages:send`), auth via service-account JWT. Payload kept under 4096 bytes.
- Stale tokens (FCM `UNREGISTERED`) are pruned from `device_tokens_json`.

**Testing**:
- Integration (mocked FCM): send → POST to FCM with correct token and payload shape.
- Integration: FCM returns `UNREGISTERED` → token removed from member.
- Unit: payload exceeding 4096 bytes → truncated/rejected with error.

#### 7.2 — Notification triggers & quiet hours

**What**: Generate notifications from domain events, honouring quiet hours and per-member prefs.

**Design**:
- Triggers (respecting `members.notification_prefs_json`):
  - `chore_due`: from the scheduler when an assignment becomes due.
  - `completion_submitted` / `approval_needed`: to parents.
  - `completion_approved` / `completion_rejected`: to the child.
  - `streak_milestone` / `badge.awarded`.
- Quiet hours (`families.settings_json.quiet_hours_start/end`, family tz): non-urgent notifications scheduled during quiet hours are deferred (BullMQ delayed job) to `quiet_hours_end`.
- Fan-out runs in the worker, not the request path.

**Testing**:
- Integration: assignment due at 21:30 with quiet hours 21:00–07:00 → delivery deferred to 07:00 (delayed job scheduled).
- Integration: member with `chore_due:false` pref → no notification enqueued.
- Integration: approval request goes to parents only.

---

## Phase 8: AI-Native Features

### Purpose
The differentiators: natural-language chore creation, age-appropriate chore suggestions, and AI photo-proof verification — none well-served by incumbents. All run server-side behind an `LLMProvider` abstraction.

### Tasks

#### 8.1 — LLMProvider abstraction

**What**: A provider interface wrapping Claude for text and vision, with structured-output enforcement.

**Design**:
```ts
interface LLMProvider {
  complete(opts:{system:string; user:string; schema?:TSchema}): Promise<unknown>;   // JSON-mode when schema given
  vision(opts:{system:string; user:string; imageUrl:string; schema?:TSchema}): Promise<unknown>;
}
```
- Anthropic implementation; outputs validated against the supplied TypeBox schema; one repair retry on schema-invalid output, then error.
- All AI actions write `audit_log` with `actor_type:'ai'`.

**Testing**:
- Unit (mocked client): schema-valid response → parsed object.
- Unit: invalid-then-valid responses → repair retry succeeds.
- Unit: invalid twice → throws `AppError(502,'AI_OUTPUT_INVALID')`.

#### 8.2 — Natural-language chore creation

**What**: Parse a free-text chore description into a structured chore + RRULE + assignment plan.

**Design**:
- `POST /families/:id/chores/parse {text, context?}` (parent). Context includes family members (names, ages) so the model can resolve "split between the kids".
- System prompt instructs output strictly matching `ChoreParseSchema`:
  ```ts
  { title, category, recurrence_type, rrule, assignment_mode, assignee_member_ids[],
    points_value, requires_photo, subtasks?[] }
  ```
- Example: "clean the bathroom every Saturday, split between the kids" → `{title:"Clean the bathroom", category:"bathroom", recurrence_type:"weekly", rrule:"FREQ=WEEKLY;BYDAY=SA", assignment_mode:"rotating", assignee_member_ids:[<child ids>]}`.
- Returns a draft for parent confirmation; a separate `POST /chores` actually persists (no auto-create from AI without confirmation).

**Testing**:
- Unit (mocked LLM): canned NL → expected structured draft; RRULE validates via `rrule`.
- Integration: parse → returned draft references only real family member ids.
- Unit: model returns unknown `category` → repair/normalize to `'other'`.

#### 8.3 — Age-appropriate chore suggestions

**What**: Suggest chores tailored to each child's age group, season, and household history.

**Design**:
- `GET /families/:id/suggestions?member=` → 5–8 suggested chores.
- Prompt inputs: child `age_group`, existing chores (avoid duplicates), current month (seasonal), recent completion categories (variety).
- Output validated to a list of `{title, category, difficulty, suggested_points, min_age, rationale}`; filtered so `min_age <= child age`.

**Testing**:
- Unit (mocked LLM): elementary child → no chore with `min_age>10` returned (post-filter enforced).
- Integration: suggestions exclude chores the family already has (by title match).
- Unit: empty/garbage LLM output → returns empty list, no crash.

#### 8.4 — AI photo-proof verification

**What**: Vision model scores submitted photo proof; high-confidence passes can auto-approve if the family opts in.

**Design**:
- On completion with `photo_url`, the worker calls `LLMProvider.vision` with the chore title/description and a signed image URL → `{ verified:boolean, score:0..1, reason }`.
- Writes `completions.photo_verified`, `photo_ai_score`.
- If `families.settings_json.ai_auto_approve=true` and `score >= AI_APPROVE_THRESHOLD` (default 0.85) → auto-approve (posts ledger as a system/AI actor); otherwise surfaces the score to the parent to speed manual approval.
- Never auto-rejects; low score only routes to a parent.

**Testing**:
- Integration (mocked vision): score 0.9 + auto-approve on → completion approved by AI actor, ledger posted, audit `actor_type:'ai'`.
- Integration: score 0.4 + auto-approve on → stays `submitted`, awaits parent.
- Integration: auto-approve off → never auto-approves regardless of score.

---

## Phase 9: Web App, MCP Server & Calendar Export

### Purpose
Surface the platform beyond the mobile app: a desktop parent dashboard, a first-party MCP server (no competitor has one), and iCalendar/CalDAV export for interoperability.

### Tasks

#### 9.1 — Next.js parent web dashboard

**What**: Desktop web UI for parents: roster, chore management, approvals queue, analytics, allowance ledgers.

**Design**:
- Next.js 16 App Router; server components fetch via the REST API using a parent session (OIDC/password); W3C Push API for browser notifications.
- Key screens: Dashboard (today + pending approvals), Chores (CRUD + NL create), Approvals queue (with AI photo score), Members & balances, Analytics (workload/fairness charts).
- Reuses `packages/shared` types and the WebSocket client for live updates.

**Testing**:
- E2E (Playwright): parent logs in → dashboard shows pending approvals; approve → row clears live.
- E2E: create a chore via the NL box → draft shown → confirm → appears in list.
- E2E: child credentials rejected from the parent web app.

#### 9.2 — MCP server

**What**: An MCP server exposing family chore tools to AI assistants (Claude, etc.).

**Design**:
- `@modelcontextprotocol/sdk` server; auth via a family-scoped API token (parent-issued).
- Tools: `list_chores`, `list_today_assignments(member?)`, `complete_assignment(assignment_id)`, `create_chore(natural_language)`, `get_balance(member)`, `pending_approvals()`, `approve_completion(id)`.
- Each tool calls the REST API with the bound token; respects RBAC (a child-scoped token cannot approve).
- Enables "Mark Jake's bathroom chore as done" / "What chores does Emma have this week?".

**Testing**:
- Integration: `list_today_assignments` tool → returns the same data as the REST endpoint.
- Integration: `approve_completion` with a child-scoped token → 403 surfaced as a tool error.
- Integration: `create_chore` NL tool → delegates to the parse endpoint, returns a draft.

#### 9.3 — iCalendar export & CalDAV-friendly feed

**What**: Export family chore assignments as iCalendar VTODO items.

**Design**:
- `GET /families/:id/calendar.ics?token=<feed_token>` → `ical-generator` VTODO components per upcoming assignment: `SUMMARY` = chore title, `DUE` = due_date, `STATUS` mapped (`NEEDS-ACTION`/`IN-PROCESS`/`COMPLETED`) per CalConnect CC/11001:2024, `RRULE` carried from the chore for recurring series.
- Read-only signed feed token (not a JWT) so calendar clients can poll without login.

**Testing**:
- Unit: assignment → valid VTODO (parse with an ICS parser; `DUE` and `SUMMARY` correct).
- Unit: approved assignment → `STATUS:COMPLETED`.
- Integration: feed with bad token → 403.

---

## Phase 10: Mobile Apps (Parent & Child Experiences)

### Purpose
Ship the primary platform: native iOS/Android apps with age-progressive child UI, photo proof capture, offline support, push, and a home-screen widget — pulling together every prior phase into the end-user product.

### Tasks

#### 10.1 — App shell, auth, navigation, role-based UI

**What**: Expo app with parent and child flows and age-progressive child rendering.

**Design**:
- Auth: parent via OIDC/password (Expo AuthSession + Sign in with Apple/Google) and child via family PIN screen.
- Navigation branches on `role`: parent (dashboard, chores, approvals, members, analytics) vs child (my chores, rewards, progress).
- Age-progressive child UI keyed off `age_group`: large icons + stickers (preschool/elementary), points/streaks (middle_school), budgeting/balance (high_school).
- WatermelonDB local store; WebSocket + delta-sync wiring from Phase 6.

**Testing**:
- Detox E2E: child PIN login → child home; parent login → parent dashboard.
- Detox E2E: elementary child sees sticker UI; high_school child sees balance UI (snapshot/assertion on key elements).
- Unit: navigation guard hides parent tabs for child role.

#### 10.2 — Chore completion & photo proof capture

**What**: Child completes chores; captures/uploads photo proof; works offline.

**Design**:
- Today list → tap chore → check subtasks → "Done"; if `requires_photo`, open camera (Expo Camera), capture, request signed S3 URL, upload, then submit completion referencing the key.
- Offline: completion queued locally with idempotency key; uploads/submits flush on reconnect (Phase 6 protocol).
- Live balance/streak/badge updates via WebSocket.

**Testing**:
- Detox E2E: complete a photo-required chore → camera → submit → appears as `submitted`; parent device receives push.
- Detox E2E (airplane mode): complete offline → queued → reconnect → server reflects completion once.
- Unit: completion missing required photo → blocked client-side with a clear message.

#### 10.3 — Parent approval flow & notifications

**What**: Parent reviews submissions (with AI photo score), approves/rejects, receives push.

**Design**:
- Approvals screen lists `submitted` completions with photo + `photo_ai_score`; approve/reject actions hit Phase 4 endpoints; dual-parent state surfaced when enabled.
- Push handling (Expo Notifications + FCM): tapping an `approval_needed` push deep-links to the approvals screen.

**Testing**:
- Detox E2E: child submits → parent push arrives → tap → approvals screen → approve → child balance updates live.
- Detox E2E: reject with reason → child sees rejection.
- Integration: dual-approval family → UI shows "awaiting second parent" after first approval.

#### 10.4 — Home-screen widget

**What**: Ambient widget showing today's chores with quick-complete (matching Chorsee's differentiator).

**Design**:
- Expo config plugin enabling iOS WidgetKit and Android App Widget; widget reads cached today-assignments from the shared local store; tapping a chore deep-links into the app to complete (or completes simple no-photo chores directly).
- Widget data refreshed on app foreground and via background sync.

**Testing**:
- Manual/instrumented: widget renders today's count; tap → app opens to that chore.
- Unit: widget data provider returns only today's incomplete assignments for the signed-in child.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (schema, config, server)        ─── required by everything
    │
Phase 2: Identity, Families & RBAC                  ─── requires 1
    │
Phase 3: Chores & Scheduling Engine (core)          ─── requires 2
    │
Phase 4: Completion, Approval & Allowance Ledger    ─── requires 3
    │
    ├── Phase 5: Rewards, Gamification & Analytics   ─── requires 4
    ├── Phase 6: Real-Time Sync & Offline            ─── requires 4 (parallel with 5, 7)
    └── Phase 7: Notifications (FCM, Quiet Hours)    ─── requires 4 (parallel with 5, 6)
         │
Phase 8: AI-Native Features                          ─── requires 3,4 (parallel with 5–7)
    │
Phase 9: Web App, MCP Server & Calendar Export       ─── requires 4–8 (3 sub-tasks parallelisable)
    │
Phase 10: Mobile Apps (Parent & Child)               ─── requires 4,6,7,8 (the integrating phase)
```

**Parallelism opportunities**
- After Phase 4: Phases 5, 6, and 7 are independent and can be built concurrently.
- Phase 8 (AI) depends only on Phases 3–4 and can proceed alongside 5–7.
- Within Phase 9, the web app, MCP server, and calendar export are independent of one another.
- Phase 10 is the convergence point and should follow once the API surface (1–8) is stable.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm -w test`), including Testcontainers integration tests for any DB/Redis-touching code.
3. Linting and formatting pass (`pnpm -w lint`).
4. Type checking passes (`pnpm -w typecheck`, `tsc --noEmit`).
5. Docker build succeeds for affected apps (`docker compose build`); `docker compose up` boots the stack with passing `/health`.
6. The phase's user-facing capability works end-to-end (demonstrated by an integration or E2E test).
7. New configuration options documented in `apps/api/src/config.ts` and the repo README.
8. New/changed API endpoints appear in the auto-generated OpenAPI 3.1 spec at `/docs/json` and validate against the spec.
9. Database migrations created, reviewed, and reversible (`drizzle-kit` up/down verified).
10. Every state-changing action writes an `audit_log` entry (COPPA/ISO 27001 alignment).
11. Child-data paths verified against COPPA/GDPR-K rules: no child profile active without recorded parental consent; data minimisation respected.
```
