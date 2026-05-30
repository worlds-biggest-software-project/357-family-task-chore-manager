# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Family Task & Chore Manager · Created: 2026-05-25

## Philosophy

Every concept in the family task manager — families, members, chores, assignments, completions, rewards, redemptions, allowance transactions, badges, streaks — is a first-class relational table with foreign keys, indexed columns, and CHECK constraints. This produces a schema where every relationship is explicit and every query can be answered with standard SQL joins.

Family chore management has a core lifecycle: chores are defined with recurrence rules → assigned to members (with rotation support) → completed with optional photo proof → approved by a parent → rewards released → allowance updated. The normalized model treats each stage as a separate entity, enabling analytics that cross entity boundaries: "which chores have the lowest completion rate?", "is workload balanced fairly across children?", "how does streak length correlate with allowance earnings?"

The trade-off is schema complexity — 14 tables — but the payoff is full referential integrity, explicit assignment tracking (critical for rotation and fairness algorithms), and a clean allowance ledger with double-entry-style transaction records.

**Best for:** Teams building a production platform where data integrity, workload fairness analysis, and a robust allowance ledger are priorities over rapid schema iteration.

**Trade-offs:**
- Pro: Full referential integrity across all entities
- Pro: Assignment tracking enables fair rotation algorithms
- Pro: Allowance ledger with explicit transaction records
- Pro: Per-chore, per-member analytics via standard SQL joins
- Con: 14 tables — more complex schema
- Con: Schema migrations required for new reward or chore types
- Con: Dashboard loading requires joins across multiple tables
- Con: Recurrence rule variations need careful constraint modelling

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| iCalendar (RFC 5545) | `chores.rrule` stores iCalendar RRULE for recurrence |
| CalDAV (RFC 4791) | `families.calendar_sync_json` stores CalDAV connection |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Chore config and reward definition validation |
| OAuth 2.0 / OIDC | Parent auth; Sign in with Apple/Google |
| JWT (RFC 7519) | Session tokens with parent/child role claims |
| WebSocket (RFC 6455) | Real-time chore completion sync |
| FCM / APNs | Push notification delivery |
| COPPA (2025 Rule) | `families.coppa_consent_json` for parental consent |
| GDPR Article 8 | Child data handling; data minimisation |
| ISO 27001 | Audit logging; access control |
| OWASP MASVS | Mobile app security baseline |
| MCP | `families.mcp_enabled` for AI assistant integration |
| PCI DSS | Referenced if BaaS card integration is added |

---

## Families & Members

```sql
CREATE TABLE families (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    locale          TEXT NOT NULL DEFAULT 'en-US',
    currency        TEXT NOT NULL DEFAULT 'USD',
    invite_code     TEXT UNIQUE,
    mcp_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    coppa_consent_json JSONB,
    -- {
    --   "consented_at": "...", "consent_method": "email_plus_id",
    --   "parent_id": "uuid", "children_covered": ["uuid"],
    --   "data_minimisation_acknowledged": true,
    --   "security_coordinator": "uuid"
    -- }
    calendar_sync_json JSONB,
    -- {"provider": "google_calendar", "calendar_id": "...",
    --  "oauth_token_ref": "vault://...", "sync_enabled": true}
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- {"quiet_hours_start": "21:00", "quiet_hours_end": "07:00",
    --  "photo_proof_default": true, "dual_parent_approval": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id       UUID NOT NULL REFERENCES families(id),
    display_name    TEXT NOT NULL,
    email           TEXT,
    avatar_url      TEXT,
    role            TEXT NOT NULL CHECK (role IN (
                        'parent','child','guardian'
                    )),
    birth_date      DATE,
    age_group       TEXT CHECK (age_group IN (
                        'toddler','preschool','elementary',
                        'middle_school','high_school','adult'
                    )),
    pin_hash        TEXT,
    auth_method     TEXT NOT NULL CHECK (auth_method IN (
                        'email_password','apple','google','pin_only'
                    )) DEFAULT 'pin_only',
    points_balance  INTEGER NOT NULL DEFAULT 0,
    allowance_balance_cents BIGINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    device_tokens_json JSONB NOT NULL DEFAULT '[]',
    notification_prefs_json JSONB NOT NULL DEFAULT '{}',
    -- {"chore_due": true, "chore_completed": true, "approval_needed": true,
    --  "streak_milestone": true, "quiet_hours": true}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_members_family ON members (family_id);
CREATE INDEX idx_members_email ON members (email)
    WHERE email IS NOT NULL;
CREATE INDEX idx_members_role ON members (role);
```

---

## Chores & Assignments

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
    estimated_minutes INTEGER,
    min_age         INTEGER,
    points_value    INTEGER NOT NULL DEFAULT 1,
    allowance_cents BIGINT NOT NULL DEFAULT 0,
    recurrence_type TEXT NOT NULL CHECK (recurrence_type IN (
                        'once','daily','weekly','biweekly','monthly','custom'
                    )) DEFAULT 'once',
    rrule           TEXT,
    assignment_mode TEXT NOT NULL CHECK (assignment_mode IN (
                        'fixed','rotating','alternating','up_for_grabs'
                    )) DEFAULT 'fixed',
    requires_photo  BOOLEAN NOT NULL DEFAULT FALSE,
    requires_approval BOOLEAN NOT NULL DEFAULT TRUE,
    subtasks_json   JSONB,
    -- [{"title": "Rinse dishes", "order": 1}, {"title": "Load dishwasher", "order": 2}]
    icon            TEXT,
    colour_hex      TEXT,
    condition_based BOOLEAN NOT NULL DEFAULT FALSE,
    urgency_days    INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chores_family ON chores (family_id);
CREATE INDEX idx_chores_category ON chores (category);
CREATE INDEX idx_chores_mode ON chores (assignment_mode);

CREATE TABLE chore_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chore_id        UUID NOT NULL REFERENCES chores(id),
    member_id       UUID NOT NULL REFERENCES members(id),
    due_date        DATE NOT NULL,
    rotation_order  INTEGER,
    status          TEXT NOT NULL CHECK (status IN (
                        'pending','in_progress','submitted','approved',
                        'rejected','missed','skipped'
                    )) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assignments_chore ON chore_assignments (chore_id);
CREATE INDEX idx_assignments_member ON chore_assignments (member_id);
CREATE INDEX idx_assignments_date ON chore_assignments (due_date);
CREATE INDEX idx_assignments_status ON chore_assignments (status);
CREATE INDEX idx_assignments_pending ON chore_assignments (member_id, due_date)
    WHERE status = 'pending';
```

---

## Completions

```sql
CREATE TABLE completions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assignment_id   UUID NOT NULL REFERENCES chore_assignments(id),
    chore_id        UUID NOT NULL REFERENCES chores(id),
    member_id       UUID NOT NULL REFERENCES members(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    completed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    photo_url       TEXT,
    photo_verified  BOOLEAN,
    photo_ai_score  REAL,
    subtasks_completed_json JSONB,
    -- [{"title": "Rinse dishes", "completed": true}, {"title": "Load dishwasher", "completed": true}]
    notes           TEXT,
    approved_by     UUID REFERENCES members(id),
    approved_at     TIMESTAMPTZ,
    rejected_reason TEXT,
    points_earned   INTEGER NOT NULL DEFAULT 0,
    allowance_earned_cents BIGINT NOT NULL DEFAULT 0,
    time_spent_minutes INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_completions_assignment ON completions (assignment_id);
CREATE INDEX idx_completions_member ON completions (member_id);
CREATE INDEX idx_completions_family ON completions (family_id);
CREATE INDEX idx_completions_chore ON completions (chore_id);
CREATE INDEX idx_completions_date ON completions (completed_at);
CREATE INDEX idx_completions_pending ON completions (family_id)
    WHERE approved_by IS NULL AND rejected_reason IS NULL;
```

---

## Rewards & Allowance

```sql
CREATE TABLE rewards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id       UUID NOT NULL REFERENCES families(id),
    name            TEXT NOT NULL,
    description     TEXT,
    reward_type     TEXT NOT NULL CHECK (reward_type IN (
                        'points_redeem','allowance_bonus','screen_time',
                        'privilege','custom','badge'
                    )),
    points_cost     INTEGER,
    allowance_cents BIGINT,
    icon            TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rewards_family ON rewards (family_id);

CREATE TABLE reward_redemptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reward_id       UUID NOT NULL REFERENCES rewards(id),
    member_id       UUID NOT NULL REFERENCES members(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    points_spent    INTEGER NOT NULL DEFAULT 0,
    approved_by     UUID REFERENCES members(id),
    status          TEXT NOT NULL CHECK (status IN (
                        'pending','approved','rejected','fulfilled'
                    )) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    fulfilled_at    TIMESTAMPTZ
);
CREATE INDEX idx_redemptions_member ON reward_redemptions (member_id);
CREATE INDEX idx_redemptions_family ON reward_redemptions (family_id);
CREATE INDEX idx_redemptions_status ON reward_redemptions (status)
    WHERE status = 'pending';

CREATE TABLE allowance_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id       UUID NOT NULL REFERENCES members(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN (
                        'chore_earned','allowance_base','bonus',
                        'penalty','withdrawal','parent_adjustment',
                        'savings_deposit','savings_withdrawal'
                    )),
    amount_cents    BIGINT NOT NULL,
    balance_after_cents BIGINT NOT NULL,
    reference_id    UUID,
    reference_type  TEXT CHECK (reference_type IN (
                        'completion','reward_redemption','manual'
                    )),
    description     TEXT,
    approved_by     UUID REFERENCES members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_allowance_member ON allowance_transactions (member_id);
CREATE INDEX idx_allowance_family ON allowance_transactions (family_id);
CREATE INDEX idx_allowance_date ON allowance_transactions (created_at);
CREATE INDEX idx_allowance_type ON allowance_transactions (transaction_type);
```

---

## Badges, Streaks & Audit

```sql
CREATE TABLE badges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id       UUID REFERENCES families(id),
    name            TEXT NOT NULL,
    description     TEXT,
    icon            TEXT NOT NULL,
    criteria_json   JSONB NOT NULL DEFAULT '{}',
    -- {"type": "streak", "days": 7}
    -- {"type": "completion_count", "count": 50}
    -- {"type": "category_master", "category": "kitchen", "count": 20}
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_badges_family ON badges (family_id);

CREATE TABLE member_badges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id       UUID NOT NULL REFERENCES members(id),
    badge_id        UUID NOT NULL REFERENCES badges(id),
    earned_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (member_id, badge_id)
);
CREATE INDEX idx_member_badges_member ON member_badges (member_id);

CREATE TABLE streaks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id       UUID NOT NULL REFERENCES members(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    streak_type     TEXT NOT NULL CHECK (streak_type IN (
                        'daily_completion','weekly_all_done','category_streak'
                    )),
    category        TEXT,
    current_count   INTEGER NOT NULL DEFAULT 0,
    longest_count   INTEGER NOT NULL DEFAULT 0,
    last_completed_date DATE,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_streaks_member ON streaks (member_id);

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

### Workload fairness across family members

```sql
SELECT m.display_name, m.role,
       COUNT(c.id) AS total_completions,
       SUM(c.points_earned) AS total_points,
       SUM(c.allowance_earned_cents) / 100.0 AS total_allowance
FROM members m
LEFT JOIN completions c ON c.member_id = m.id
    AND c.completed_at >= now() - INTERVAL '7 days'
WHERE m.family_id = 'family-uuid'
GROUP BY m.id, m.display_name, m.role
ORDER BY total_completions DESC;
```

### Chores with lowest completion rate

```sql
SELECT ch.title, ch.category,
       COUNT(ca.id) AS total_assigned,
       COUNT(ca.id) FILTER (WHERE ca.status = 'approved') AS completed,
       ROUND(
           COUNT(ca.id) FILTER (WHERE ca.status = 'approved')::NUMERIC
           / NULLIF(COUNT(ca.id), 0), 2
       ) AS completion_rate
FROM chores ch
JOIN chore_assignments ca ON ca.chore_id = ch.id
WHERE ch.family_id = 'family-uuid'
  AND ca.due_date >= CURRENT_DATE - 30
GROUP BY ch.id, ch.title, ch.category
ORDER BY completion_rate ASC
LIMIT 10;
```

### Allowance ledger for a child

```sql
SELECT transaction_type, description,
       amount_cents / 100.0 AS amount,
       balance_after_cents / 100.0 AS balance,
       created_at
FROM allowance_transactions
WHERE member_id = 'member-uuid'
ORDER BY created_at DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Family | 2 | families, members |
| Chores | 2 | chores, chore_assignments |
| Completions | 1 | completions |
| Rewards | 3 | rewards, reward_redemptions, allowance_transactions |
| Engagement | 3 | badges, member_badges, streaks |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **12** | |

---

## Key Design Decisions

1. **`chore_assignments` as a separate table** — assignment tracking per chore per due date enables rotation algorithms (round-robin the next assignee), alternating schedules, and "up for grabs" claims; this data structure is critical for the workload fairness dashboard.

2. **`chores.rrule` for recurrence** — iCalendar RRULE format (RFC 5545) is the standard for expressing complex recurrence patterns ("every Saturday", "first Monday of the month", "every 2 weeks alternating"); storing the RRULE enables calendar export and interoperability.

3. **`chores.assignment_mode`** — distinguishing between fixed, rotating, alternating, and up_for_grabs at the chore level determines how the assignment engine generates `chore_assignments` rows.

4. **`completions` with photo proof fields** — `photo_url`, `photo_verified`, and `photo_ai_score` support the photo proof workflow where children submit evidence and parents (or AI) verify completion.

5. **`allowance_transactions` as a ledger table** — every financial event (chore earned, base allowance, bonus, penalty, withdrawal) is an explicit transaction with `balance_after_cents`, creating an auditable ledger that parents and children can review.

6. **`members.age_group`** — developmental stage classification drives age-appropriate UI rendering, chore suggestions, and financial literacy content without exposing the child's exact birth date in most queries.

7. **`rewards` and `reward_redemptions` as separate tables** — the reward catalog (family-defined) is separate from redemption records, enabling reward management (add, remove, change cost) without affecting historical redemption data.

8. **`streaks` with `current_count` and `longest_count`** — tracking both current and all-time longest streaks enables the gamification layer to show progress and celebrate milestones.

9. **`families.coppa_consent_json`** — COPPA 2025 Rule compliance requires tracking verifiable parental consent method, timestamp, children covered, and security coordinator designation; JSONB accommodates the evolving consent requirements.

10. **`chores.condition_based` and `urgency_days`** — supporting Tody-style condition-based scheduling (urgency indicators rather than fixed dates) alongside traditional calendar scheduling, a differentiating feature.
