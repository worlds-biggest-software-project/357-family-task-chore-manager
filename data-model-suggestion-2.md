# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Family Task & Chore Manager · Created: 2026-05-25

## Philosophy

Core operational entities — families, chores, completions, allowance transactions — are relational tables with indexed columns for family-scoped queries, date-range filtering, and status tracking. Variable-structure data — member profiles with roles, recurrence rules, assignments, reward catalogs, badge definitions, streak progress, photo proof, approval workflows — lives in JSONB columns with GIN indexes.

Family chore managers have a strongly hierarchical access pattern: everything flows from family → members → chores → completions. The dominant UI operation is "load the family dashboard showing all members, their chores for today, and their points/allowance." Embedding members with their badges, streaks, and balances on the family row, and embedding assignments and recurrence on chore rows, means the dashboard is assembled from two queries (family + chores) rather than joins across 6+ tables.

The trade-off is that cross-family analytics (which wouldn't exist in a consumer app — each family is isolated) don't benefit, and per-assignment queries require JSONB extraction. But for a family app where data is always accessed within a single family context, the schema simplicity pays for itself.

**Best for:** Teams building an MVP where rapid iteration on chore types, minimal schema migrations, and fast family dashboard loading are priorities.

**Trade-offs:**
- Pro: 5 tables — simple schema, fast to deploy
- Pro: Family dashboard is a one-row read for members + N chore rows
- Pro: New chore types, reward types, and badge criteria require no migration
- Pro: Member profiles absorb role-specific variation (parent vs child) naturally
- Con: Per-assignment analytics require JSONB path queries
- Con: Large families with many chores can produce oversized JSONB
- Con: No FK enforcement on assignment references within JSONB
- Con: Rotation algorithm must work within JSONB assignment arrays

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| iCalendar (RFC 5545) | `chores.recurrence_json` stores RRULE-compatible rules |
| CalDAV (RFC 4791) | `families.calendar_sync_json` stores CalDAV config |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Chore config and reward validation |
| OAuth 2.0 / OIDC | Parent auth; Sign in with Apple/Google |
| JWT (RFC 7519) | Session tokens with parent/child role claims |
| WebSocket (RFC 6455) | Real-time sync |
| FCM / APNs | Push notifications via `families.members_json` device tokens |
| COPPA (2025 Rule) | `families.coppa_json` for parental consent tracking |
| GDPR Article 8 | Child data minimisation |
| OWASP MASVS | Mobile security baseline |
| MCP | `families.mcp_json` for AI assistant integration |

---

## Families

```sql
CREATE TABLE families (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    invite_code     TEXT UNIQUE,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    locale          TEXT NOT NULL DEFAULT 'en-US',
    currency        TEXT NOT NULL DEFAULT 'USD',
    members_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "display_name": "Mom", "email": "...",
    --   "avatar_url": "...", "role": "parent",
    --   "birth_date": "1985-03-15", "age_group": "adult",
    --   "auth_method": "apple", "pin_hash": null,
    --   "points_balance": 0, "allowance_balance_cents": 0,
    --   "badges": [{"badge_id": "uuid", "name": "Super Mom", "earned_at": "..."}],
    --   "streaks": [{"type": "daily_completion", "current": 12, "longest": 21,
    --               "last_date": "2026-05-25"}],
    --   "device_tokens": ["fcm://...", "apns://..."],
    --   "notification_prefs": {"chore_due": true, "approval_needed": true},
    --   "is_active": true, "created_at": "..."
    -- }, {
    --   "id": "uuid", "display_name": "Jake", "email": null,
    --   "role": "child", "birth_date": "2016-07-22", "age_group": "elementary",
    --   "auth_method": "pin_only", "pin_hash": "...",
    --   "points_balance": 45, "allowance_balance_cents": 1250,
    --   "badges": [{"badge_id": "uuid", "name": "7-Day Streak", "earned_at": "..."}],
    --   "streaks": [{"type": "daily_completion", "current": 7, "longest": 14}],
    --   "is_active": true
    -- }]
    reward_catalog_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "30 min extra screen time",
    --   "reward_type": "screen_time", "points_cost": 20,
    --   "icon": "screen", "is_active": true
    -- }, {
    --   "id": "uuid", "name": "Choose dinner restaurant",
    --   "reward_type": "privilege", "points_cost": 50,
    --   "icon": "restaurant", "is_active": true
    -- }]
    badge_catalog_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "7-Day Streak", "description": "Complete chores 7 days in a row",
    --   "icon": "flame", "criteria": {"type": "streak", "days": 7}, "is_system": true
    -- }, {
    --   "id": "uuid", "name": "Kitchen Master", "description": "Complete 20 kitchen chores",
    --   "icon": "chef", "criteria": {"type": "category_master", "category": "kitchen", "count": 20}
    -- }]
    coppa_json      JSONB,
    -- {
    --   "consented_at": "...", "consent_method": "email_plus_id",
    --   "parent_id": "uuid", "children_covered": ["uuid"],
    --   "data_minimisation_acknowledged": true,
    --   "security_coordinator": "uuid"
    -- }
    calendar_sync_json JSONB,
    -- {"provider": "google_calendar", "calendar_id": "...",
    --  "oauth_token_ref": "vault://...", "sync_enabled": true}
    mcp_json        JSONB,
    -- {"enabled": true, "tools": [...], "resources": [...]}
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- {"quiet_hours_start": "21:00", "quiet_hours_end": "07:00",
    --  "photo_proof_default": true, "dual_parent_approval": false,
    --  "allowance_schedule": "weekly", "allowance_day": "friday"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_families_invite ON families (invite_code)
    WHERE invite_code IS NOT NULL;
CREATE INDEX idx_families_members ON families USING GIN (members_json);
```

---

## Chores

```sql
CREATE TABLE chores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id       UUID NOT NULL REFERENCES families(id),
    title           TEXT NOT NULL,
    description     TEXT,
    category        TEXT CHECK (category IN (
                        'cleaning','kitchen','laundry','outdoor','pet_care',
                        'bedroom','bathroom','homework','personal_care','other'
                    )),
    difficulty      TEXT NOT NULL CHECK (difficulty IN (
                        'easy','medium','hard'
                    )) DEFAULT 'medium',
    points_value    INTEGER NOT NULL DEFAULT 1,
    allowance_cents BIGINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    recurrence_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "type": "weekly", "rrule": "FREQ=WEEKLY;BYDAY=SA",
    --   "days_of_week": ["saturday"],
    --   "condition_based": false, "urgency_days": null,
    --   "start_date": "2026-01-01", "end_date": null
    -- }
    assignment_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "mode": "rotating",
    --   "members": ["uuid-jake", "uuid-emma"],
    --   "current_assignee": "uuid-jake",
    --   "rotation_index": 0,
    --   "schedule": [
    --     {"due_date": "2026-05-25", "member_id": "uuid-jake", "status": "pending"},
    --     {"due_date": "2026-06-01", "member_id": "uuid-emma", "status": "pending"}
    --   ]
    -- }
    -- For up_for_grabs: {"mode": "up_for_grabs", "eligible_members": ["uuid", "uuid"]}
    subtasks_json   JSONB,
    -- [{"title": "Rinse dishes", "order": 1}, {"title": "Load dishwasher", "order": 2}]
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {"requires_photo": true, "requires_approval": true,
    --  "estimated_minutes": 15, "min_age": 8,
    --  "icon": "dish", "colour_hex": "#4285F4"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chores_family ON chores (family_id);
CREATE INDEX idx_chores_category ON chores (category);
CREATE INDEX idx_chores_assignment ON chores USING GIN (assignment_json);
```

---

## Completions

```sql
CREATE TABLE completions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chore_id        UUID NOT NULL REFERENCES chores(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    member_id       UUID NOT NULL,
    due_date        DATE NOT NULL,
    completed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    proof_json      JSONB,
    -- {
    --   "photo_url": "s3://...", "photo_verified": true,
    --   "ai_verification_score": 0.92,
    --   "subtasks_completed": [
    --     {"title": "Rinse dishes", "completed": true},
    --     {"title": "Load dishwasher", "completed": true}
    --   ],
    --   "notes": "Had to rewash some pots"
    -- }
    approval_json   JSONB,
    -- {
    --   "status": "approved", "approved_by": "uuid",
    --   "approved_at": "...", "rejected_reason": null
    -- }
    reward_json     JSONB NOT NULL DEFAULT '{}',
    -- {"points_earned": 5, "allowance_earned_cents": 100}
    time_spent_minutes INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_completions_chore ON completions (chore_id);
CREATE INDEX idx_completions_family ON completions (family_id);
CREATE INDEX idx_completions_member ON completions (member_id);
CREATE INDEX idx_completions_date ON completions (due_date);
CREATE INDEX idx_completions_pending ON completions (family_id)
    WHERE approval_json->>'status' = 'pending' OR approval_json IS NULL;
```

---

## Allowance Ledger

```sql
CREATE TABLE allowance_ledger (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id       UUID NOT NULL REFERENCES families(id),
    member_id       UUID NOT NULL,
    transaction_type TEXT NOT NULL CHECK (transaction_type IN (
                        'chore_earned','allowance_base','bonus',
                        'penalty','withdrawal','parent_adjustment',
                        'reward_redemption','savings_deposit','savings_withdrawal'
                    )),
    amount_cents    BIGINT NOT NULL,
    balance_after_cents BIGINT NOT NULL,
    reference_json  JSONB,
    -- {"completion_id": "uuid", "chore_title": "Wash dishes"}
    -- {"reward_id": "uuid", "reward_name": "Extra screen time", "points_spent": 20}
    -- {"description": "Weekly allowance", "approved_by": "uuid"}
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_ledger_family ON allowance_ledger (family_id);
CREATE INDEX idx_ledger_member ON allowance_ledger (member_id);
CREATE INDEX idx_ledger_date ON allowance_ledger (created_at);
CREATE INDEX idx_ledger_type ON allowance_ledger (transaction_type);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id       UUID NOT NULL REFERENCES families(id),
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
                        'parent','child','system','ai'
                    )),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes_json    JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_family ON audit_log (family_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Family dashboard — all members with points and streaks

```sql
SELECT m->>'display_name' AS name,
       m->>'role' AS role,
       (m->>'points_balance')::INTEGER AS points,
       (m->>'allowance_balance_cents')::BIGINT / 100.0 AS allowance,
       m->'streaks' AS streaks
FROM families f,
     jsonb_array_elements(f.members_json) AS m
WHERE f.id = 'family-uuid'
ORDER BY (m->>'points_balance')::INTEGER DESC;
```

### Workload fairness this week

```sql
SELECT c.member_id,
       COUNT(*) AS completions,
       SUM((c.reward_json->>'points_earned')::INTEGER) AS points
FROM completions c
WHERE c.family_id = 'family-uuid'
  AND c.due_date >= CURRENT_DATE - 7
  AND c.approval_json->>'status' = 'approved'
GROUP BY c.member_id
ORDER BY completions DESC;
```

### Chores pending approval

```sql
SELECT c.id, ch.title, c.member_id,
       c.completed_at, c.proof_json->>'photo_url' AS photo
FROM completions c
JOIN chores ch ON ch.id = c.chore_id
WHERE c.family_id = 'family-uuid'
  AND (c.approval_json IS NULL OR c.approval_json->>'status' = 'pending')
ORDER BY c.completed_at ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Family | 1 | families (embeds members, rewards catalog, badges catalog, COPPA, calendar sync, MCP) |
| Chores | 1 | chores (embeds recurrence, assignments, subtasks, config) |
| Completions | 1 | completions (partitioned; embeds proof, approval, reward) |
| Allowance | 1 | allowance_ledger (partitioned; embeds reference context) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **5** | |

---

## Key Design Decisions

1. **`members_json` on families** — a family typically has 2-6 members; embedding all members with their points, allowance, badges, streaks, and device tokens means the family dashboard is a single-row read.

2. **`assignment_json` on chores** — assignment mode (fixed, rotating, alternating, up_for_grabs), current assignee, and upcoming schedule are chore-level configuration; embedding avoids a separate assignments table and keeps rotation management in one place.

3. **`recurrence_json` on chores** — recurrence rules vary in complexity (daily, weekly with specific days, monthly, condition-based); JSONB with an RRULE-compatible structure accommodates all patterns without fixed columns.

4. **`proof_json` on completions** — photo proof, AI verification score, and subtask completion are variable per completion; JSONB accommodates chores that don't require photos alongside those that do.

5. **`approval_json` on completions** — the parent approval workflow (pending, approved, rejected with reason) is embedded on the completion, keeping the full lifecycle in one row.

6. **`reward_catalog_json` and `badge_catalog_json` on families** — reward and badge definitions are family-scoped configuration with variable structures (points-based vs privilege vs screen time); JSONB accommodates custom reward types without migration.

7. **`allowance_ledger` as a separate partitioned table** — financial transactions are append-only and queried by date range; a dedicated table with `balance_after_cents` creates an auditable ledger even though member balances are also cached on the family row.

8. **`coppa_json` on families** — COPPA 2025 Rule compliance tracking (consent method, timestamp, children covered, security coordinator) is family-level; JSONB accommodates evolving regulatory requirements.

9. **`completions` partitioned by time** — completion records grow indefinitely; partitioning enables efficient recent-history queries while allowing older partitions to be archived.

10. **5 tables** — family apps have a strongly hierarchical data model (family → members → chores → completions); embedding configuration into the top-level entities minimises joins for the dominant dashboard operations.
