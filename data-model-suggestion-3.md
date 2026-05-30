# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Family Task & Chore Manager · Created: 2026-05-25

## Philosophy

Every state change — a chore created, an assignment made, a completion submitted, a photo uploaded, a parent approval given, points earned, a badge unlocked, an allowance paid — is an immutable event appended to a single `event_store` table. The event store is the sole source of truth. Read-optimised materialised views (read models) are projected from events to serve the family dashboard, child progress screens, and parent analytics.

Family chore management is fundamentally about tracking patterns over time: "is Jake doing his chores consistently?", "has workload been fair this month?", "when did Emma break her streak?", "how has allowance spending changed since we increased the weekly amount?" Event sourcing makes these temporal patterns first-class: every chore completion, every approval, every allowance transaction is a permanent record that can be replayed, aggregated, and analysed.

The event-sourced approach also aligns with the AI-native features: natural language chore creation generates `chore_created` events with AI metadata, photo proof verification generates `photo_verified` events with AI confidence scores, and workload fairness analysis can be computed by replaying assignment and completion events.

The trade-off is infrastructure complexity — but for a family app where engagement and habit-building depend on accurate historical tracking, the immutable event log provides a foundation that mutable-row approaches cannot match.

**Best for:** Teams building a platform where long-term engagement analytics, habit tracking, streak accuracy, and AI-driven fairness analysis are core requirements.

**Trade-offs:**
- Pro: Complete history — every chore, completion, and allowance event is permanent
- Pro: Streak and habit analysis from accurate historical event replay
- Pro: Workload fairness computed from complete assignment/completion history
- Pro: AI insights grounded in specific events ("Jake missed 3 chores last week")
- Pro: Undo support — mistakes can be corrected with compensating events
- Con: Read model projection infrastructure required
- Con: Eventual consistency between events and dashboard
- Con: More complex infrastructure for a conceptually simple domain
- Con: Higher storage for append-only event log

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | All events conform to CloudEvents spec |
| iCalendar (RFC 5545) | Recurrence rules in chore_created event payloads |
| CalDAV (RFC 4791) | Calendar sync config changes tracked as events |
| OpenAPI 3.1 | REST API documented in OpenAPI; read models serve responses |
| OAuth 2.0 / OIDC | Auth events tracked for parent/child access |
| JWT (RFC 7519) | Session tokens referenced in event metadata |
| WebSocket (RFC 6455) | Real-time event projection to connected clients |
| FCM / APNs | Notification delivery events tracked |
| COPPA (2025 Rule) | Consent events with full provenance |
| GDPR Article 8 | Child data erasure via event crypto-shredding |
| OWASP MASVS | Security event logging |
| MCP | MCP config changes tracked as events |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'family','chore','completion','allowance',
                        'reward','engagement','config'
                    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {"family_id": "uuid", "actor_id": "uuid",
    --  "actor_type": "parent|child|system|ai",
    --  "correlation_id": "uuid", "device": "ios|android|web"}
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_id, version);
CREATE INDEX idx_events_type ON event_store (event_type);
CREATE INDEX idx_events_stream_type ON event_store (stream_type, created_at);
CREATE INDEX idx_events_family ON event_store ((metadata->>'family_id'), created_at);

CREATE TABLE stream_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_version    BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types by Stream

### Family Stream
Events for family lifecycle:

```
family_created              — new family; captures name, timezone, currency
member_added                — parent or child joins; captures display_name, role, age_group, auth_method
member_updated              — profile changed; captures changed_fields
member_removed              — member leaves family
coppa_consent_recorded      — parental consent; captures method, children_covered
settings_updated            — family settings changed; captures old/new values
calendar_sync_configured    — CalDAV/Google Calendar linked
invite_code_generated       — new invite code created
```

### Chore Stream
Events for chore lifecycle:

```
chore_created               — new chore; captures title, category, difficulty, points, recurrence, assignment_mode
chore_updated               — chore modified; captures changed_fields
chore_assigned              — member assigned to specific due date; captures member_id, due_date, rotation_order
chore_reassigned            — assignment changed; captures old_member, new_member, reason
chore_grabbed               — up_for_grabs chore claimed; captures member_id
chore_deactivated           — chore disabled
chore_ai_suggested          — AI recommended this chore; captures age_group, context, confidence
```

### Completion Stream
Events for chore completion lifecycle:

```
completion_submitted        — child marks chore done; captures chore_id, member_id, due_date
photo_uploaded              — proof photo attached; captures photo_url, file_size
photo_ai_verified           — AI checks photo; captures score, passed, reasoning
subtasks_checked            — subtask completion recorded; captures subtasks array
approval_requested          — notification sent to parent(s)
completion_approved         — parent approves; captures approved_by
completion_rejected         — parent rejects; captures rejected_by, reason
points_awarded              — points added to member; captures points_earned
allowance_awarded           — allowance credited; captures amount_cents
chore_missed                — deadline passed without completion; captures chore_id, member_id, due_date
```

### Allowance Stream
Events for financial lifecycle:

```
base_allowance_paid         — scheduled allowance payment; captures amount_cents, schedule
chore_earning_credited      — chore-linked earning; captures completion_id, amount_cents
bonus_granted               — parent gives bonus; captures amount_cents, reason
penalty_applied             — deduction; captures amount_cents, reason
reward_redeemed             — child redeems reward; captures reward_name, points_spent
withdrawal_made             — cash/card withdrawal; captures amount_cents
savings_deposited           — child saves portion; captures amount_cents, goal_name
balance_adjusted            — parent manual adjustment; captures old_balance, new_balance, reason
```

### Engagement Stream
Events for gamification:

```
streak_continued            — daily streak extended; captures streak_type, new_count
streak_broken               — streak reset; captures streak_type, final_count, reason
streak_milestone_reached    — milestone hit (7, 14, 30 days); captures count
badge_earned                — badge unlocked; captures badge_name, criteria_met
badge_revoked               — badge removed (rule change); captures badge_name
reward_catalog_updated      — reward added/removed/changed; captures reward, action
```

---

## Read Models

```sql
CREATE TABLE rm_family_dashboard (
    family_id       UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    members_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "display_name": "Jake", "role": "child",
    --   "age_group": "elementary", "avatar_url": "...",
    --   "points_balance": 45, "allowance_balance_cents": 1250,
    --   "chores_today": 3, "chores_completed_today": 1,
    --   "current_streak": 7
    -- }]
    chores_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "title": "Wash dishes", "category": "kitchen",
    --   "points_value": 5, "assignment_mode": "rotating",
    --   "current_assignee": "uuid-jake", "next_due": "2026-05-25",
    --   "status": "pending", "requires_photo": true
    -- }]
    reward_catalog_json JSONB NOT NULL DEFAULT '[]',
    pending_approvals INTEGER NOT NULL DEFAULT 0,
    stats_json      JSONB NOT NULL DEFAULT '{}',
    -- {"total_completions_this_week": 23, "completion_rate": 0.85,
    --  "total_points_earned": 115, "total_allowance_paid_cents": 2500}
    settings_json   JSONB NOT NULL DEFAULT '{}',
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_member_progress (
    family_id       UUID NOT NULL,
    member_id       UUID NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL,
    period          DATE NOT NULL,
    completions_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "chore_id": "uuid", "title": "Make bed", "due_date": "2026-05-25",
    --   "completed_at": "...", "points_earned": 2, "status": "approved"
    -- }]
    stats_json      JSONB NOT NULL DEFAULT '{}',
    -- {"assigned": 15, "completed": 12, "missed": 2, "rejected": 1,
    --  "completion_rate": 0.80, "points_earned": 35,
    --  "allowance_earned_cents": 800, "avg_completion_minutes": 12}
    streaks_json    JSONB NOT NULL DEFAULT '[]',
    badges_json     JSONB NOT NULL DEFAULT '[]',
    allowance_json  JSONB NOT NULL DEFAULT '{}',
    -- {"balance_cents": 1250, "earned_this_period_cents": 800,
    --  "spent_this_period_cents": 300, "saved_cents": 500}
    PRIMARY KEY (family_id, member_id, period)
);
CREATE INDEX idx_rm_member_family ON rm_member_progress (family_id, period);

CREATE TABLE rm_chore_analytics (
    family_id       UUID NOT NULL,
    period          DATE NOT NULL,
    fairness_json   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "members": [
    --     {"member_id": "uuid", "name": "Jake", "assigned": 8, "completed": 7,
    --      "points_earned": 25, "workload_pct": 0.53},
    --     {"member_id": "uuid", "name": "Emma", "assigned": 7, "completed": 5,
    --      "points_earned": 18, "workload_pct": 0.47}
    --   ],
    --   "fairness_score": 0.94, "imbalance_detected": false
    -- }
    category_stats_json JSONB NOT NULL DEFAULT '{}',
    -- {"kitchen": {"assigned": 10, "completed": 8, "avg_time_minutes": 15},
    --  "bedroom": {"assigned": 14, "completed": 12, "avg_time_minutes": 8}}
    completion_rate REAL,
    missed_chores_json JSONB NOT NULL DEFAULT '[]',
    -- [{"chore_id": "uuid", "title": "Take out trash", "missed_count": 3,
    --   "member_id": "uuid"}]
    ai_suggestions_json JSONB NOT NULL DEFAULT '[]',
    -- [{"suggestion": "Consider reassigning lawn mowing to Emma — Jake has had it for 6 weeks",
    --   "type": "fairness", "confidence": 0.85}]
    PRIMARY KEY (family_id, period)
);
CREATE INDEX idx_rm_chore_analytics_family ON rm_chore_analytics (family_id, period);

CREATE TABLE rm_allowance_ledger (
    family_id       UUID NOT NULL,
    member_id       UUID NOT NULL,
    current_balance_cents BIGINT NOT NULL DEFAULT 0,
    total_earned_cents BIGINT NOT NULL DEFAULT 0,
    total_spent_cents BIGINT NOT NULL DEFAULT 0,
    total_saved_cents BIGINT NOT NULL DEFAULT 0,
    transactions_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "chore_earned", "amount_cents": 100,
    --   "balance_after_cents": 1350, "description": "Wash dishes",
    --   "created_at": "..."
    -- }]
    savings_goals_json JSONB NOT NULL DEFAULT '[]',
    -- [{"name": "New bike", "target_cents": 15000, "current_cents": 5200}]
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (family_id, member_id)
);

CREATE TABLE rm_engagement_dashboard (
    family_id       UUID NOT NULL,
    period          DATE NOT NULL,
    streaks_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "member_id": "uuid", "name": "Jake",
    --   "daily_current": 7, "daily_longest": 21,
    --   "weekly_current": 3, "weekly_longest": 8
    -- }]
    badges_json     JSONB NOT NULL DEFAULT '[]',
    -- [{"member_id": "uuid", "name": "Jake", "badge": "7-Day Streak",
    --   "earned_at": "2026-05-25"}]
    participation_json JSONB NOT NULL DEFAULT '{}',
    -- {"active_members": 3, "total_members": 4,
    --  "most_engaged": "uuid-jake", "least_engaged": "uuid-emma",
    --  "engagement_trend": "improving"}
    PRIMARY KEY (family_id, period)
);
CREATE INDEX idx_rm_engagement_family ON rm_engagement_dashboard (family_id, period);
```

---

## Example Queries

### Replay a child's chore week

```sql
SELECT event_type, payload, ce_time
FROM event_store
WHERE stream_type IN ('completion', 'engagement')
  AND metadata->>'family_id' = 'family-uuid'
  AND payload->>'member_id' = 'member-uuid'
  AND ce_time >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY ce_time ASC;
```

### When and why did a streak break?

```sql
SELECT payload->>'streak_type' AS streak_type,
       (payload->>'final_count')::INTEGER AS was_at,
       payload->>'reason' AS reason,
       ce_time
FROM event_store
WHERE stream_type = 'engagement'
  AND event_type = 'streak_broken'
  AND metadata->>'family_id' = 'family-uuid'
  AND payload->>'member_id' = 'member-uuid'
ORDER BY ce_time DESC
LIMIT 5;
```

### Impact of changing a chore's assignment mode on completion rate

```sql
WITH mode_change AS (
    SELECT ce_time AS change_time,
           payload->>'new_assignment_mode' AS new_mode
    FROM event_store
    WHERE stream_type = 'chore'
      AND stream_id = 'chore-uuid'
      AND event_type = 'chore_updated'
      AND payload ? 'new_assignment_mode'
    ORDER BY ce_time DESC
    LIMIT 1
)
SELECT
    CASE WHEN e.ce_time < mc.change_time THEN 'before' ELSE 'after' END AS period,
    COUNT(*) FILTER (WHERE e.event_type = 'completion_approved') AS approved,
    COUNT(*) FILTER (WHERE e.event_type = 'chore_missed') AS missed
FROM event_store e
CROSS JOIN mode_change mc
WHERE e.stream_type = 'completion'
  AND e.payload->>'chore_id' = 'chore-uuid'
  AND e.ce_time >= mc.change_time - INTERVAL '30 days'
  AND e.ce_time <= mc.change_time + INTERVAL '30 days'
GROUP BY CASE WHEN e.ce_time < mc.change_time THEN 'before' ELSE 'after' END;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_family_dashboard, rm_member_progress, rm_chore_analytics, rm_allowance_ledger, rm_engagement_dashboard |
| **Total** | **8** | 3 infrastructure + 5 read models |

---

## Key Design Decisions

1. **Completion lifecycle as event stream** — every step (completion_submitted → photo_uploaded → photo_ai_verified → approval_requested → completion_approved → points_awarded → allowance_awarded) is an immutable event; this enables full audit trails for disputed chore completions.

2. **`streak_broken` events with reason** — when a streak resets, the event captures why (missed deadline, app bug, parent override), enabling fair streak recovery and accurate habit analytics.

3. **`rm_chore_analytics` with `fairness_json`** — workload fairness scores are computed from assignment and completion events per period, enabling the AI to detect imbalance and suggest redistribution.

4. **`ai_suggestions_json` in `rm_chore_analytics`** — AI-generated fairness and scheduling suggestions are stored per period, enabling the parent dashboard to show actionable recommendations based on historical patterns.

5. **`chore_ai_suggested` events** — when the AI recommends a chore based on the child's age and household context, the suggestion is a first-class event, enabling tracking of which AI suggestions parents accept vs modify.

6. **`photo_ai_verified` events** — AI photo verification is a separate event from human approval, preserving the distinction between automated and manual verification for audit purposes.

7. **`chore_missed` events** — missed deadlines are explicit events (not just the absence of a completion), enabling missed-chore analysis and proactive reminders.

8. **`rm_allowance_ledger` as a dedicated read model** — the allowance ledger (current balance, total earned/spent/saved, recent transactions, savings goals) is projected from allowance stream events, providing an auditable financial view for both parents and children.

9. **`rm_engagement_dashboard` with participation tracking** — engagement metrics (most/least engaged member, engagement trend) are derived from completion and streak events, enabling parents to identify when a child's participation is declining.

10. **Event-sourced undo** — because all state changes are events, mistakes (accidental rejection, wrong point value) can be corrected with compensating events rather than mutating historical records, preserving the complete audit trail.
