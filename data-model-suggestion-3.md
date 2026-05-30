# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Work Journal & Daily Standup · Created: 2026-05-25

## Philosophy

A work journal platform is fundamentally an event ingestion system. Activity events stream in from GitHub (commits, PRs), GitLab (merge requests), Jira (ticket transitions), and the user themselves (standup responses, mood signals). Each event represents something that happened at a point in time. The AI layer processes these events to generate summaries, detect blockers, synthesize achievements, and flag disengagement patterns. An event-sourced model treats all of these inputs — tool activities, user responses, AI outputs — as immutable events in a unified store.

This architecture is particularly natural for work journals because: (1) auto-generated updates ARE events — GitHub webhooks, Jira notifications, and CI pipeline signals are already event-shaped; (2) personal work logs need temporal queries ("what did I do last Tuesday?", "show me the week of May 12") that event replay handles natively; (3) AI blocker detection needs the full activity history to identify patterns (same blocker mentioned 3 days running, decreasing commit frequency, mood decline); (4) achievement synthesis transforms a stream of daily events into narrative — a pipeline that event sourcing models directly.

The daily standup "entry" becomes a read model projected from all events on a given date for a given user — their responses, their mood, their auto-ingested activities, and any AI analysis. The team summary is another projection: aggregate all user entries for a date, run through AI, produce a narrative.

**Best for:** Teams building an activity-aggregation platform where multi-source event ingestion is core, where temporal analytics (blocker persistence, mood trends, productivity patterns) drive product differentiation, and where AI needs rich historical context for pattern detection.

**Trade-offs:**
- **Pro:** GitHub/GitLab/Jira webhooks store directly as events — zero transformation
- **Pro:** Full personal activity timeline is a simple stream replay
- **Pro:** Blocker persistence tracking is a projection over events ("mentioned in 3 consecutive days")
- **Pro:** Mood trend analysis has millisecond-level granularity
- **Pro:** Achievement synthesis draws from the complete event history
- **Con:** "Show today's standup" requires projecting from events (read model mitigates this)
- **Con:** Event volume grows linearly with team size × active days × activity sources
- **Con:** Event schema versioning needed as source integrations evolve
- **Con:** 14 tables including read models

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Scrum Guide 2020 | Standup response events follow Daily Scrum question pattern |
| CloudEvents 1.0 | Event envelope for all ingested and internal events |
| GitHub Webhooks | Push, PR, issue events ingested directly into event store |
| Jira Webhooks | Issue transition events ingested directly |
| Slack Block Kit | Read models rendered as Block Kit messages |
| MS Adaptive Cards v1.5 | Read models rendered as Adaptive Cards |
| OAuth 2.0 (RFC 6749) | Integration auth |
| GDPR | Event immutability with crypto-erasure for personal data |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    version BIGINT NOT NULL,
    data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- metadata: {
    --   "actor_id": "uuid",
    --   "source": "github",
    --   "correlation_id": "uuid",
    --   "ce_source": "work-journal/github",
    --   "ce_specversion": "1.0",
    --   "webhook_id": "gh-delivery-uuid",
    --   "team_id": "uuid"
    -- }
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_actor ON event_store((metadata->>'actor_id'), occurred_at);
CREATE INDEX idx_events_source ON event_store((metadata->>'source'), occurred_at);
CREATE INDEX idx_events_team ON event_store((metadata->>'team_id'), occurred_at);
```

---

## Event Type Registry

```sql
CREATE TABLE event_types (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Standup response events
INSERT INTO event_types (event_type, stream_type, description) VALUES
('standup.prompt_sent',       'user_day', 'Standup prompt delivered to user via Slack/Teams/email'),
('standup.response_submitted','user_day', 'User submitted standup response'),
('standup.response_edited',   'user_day', 'User edited a previously submitted response'),
('standup.skipped',           'user_day', 'User skipped standup for the day'),
('standup.reminder_sent',     'user_day', 'Reminder sent to user who has not responded'),

-- Mood events
('mood.submitted',            'user_day', 'User submitted mood score and optional comment'),
('mood.updated',              'user_day', 'User updated mood score'),

-- Activity events (from external sources)
('activity.commit',           'user_day', 'Git commit by user'),
('activity.pr_opened',        'user_day', 'Pull request opened'),
('activity.pr_merged',        'user_day', 'Pull request merged'),
('activity.pr_reviewed',      'user_day', 'Code review completed on a PR'),
('activity.issue_updated',    'user_day', 'Issue status changed (Jira, Linear, etc.)'),
('activity.issue_closed',     'user_day', 'Issue closed'),
('activity.issue_commented',  'user_day', 'Comment added to an issue'),
('activity.deployment',       'user_day', 'Deployment triggered or completed'),
('activity.branch_created',   'user_day', 'New branch created'),

-- Summary events
('summary.daily_generated',   'team_day', 'AI-generated daily team summary'),
('summary.daily_delivered',   'team_day', 'Daily summary posted to Slack/Teams/email'),
('summary.weekly_generated',  'team_day', 'AI-generated weekly team rollup'),
('summary.weekly_delivered',  'team_day', 'Weekly rollup delivered to managers'),

-- Blocker events
('blocker.detected',          'user_day', 'Blocker detected from standup response or AI analysis'),
('blocker.resolved',          'user_day', 'Previously reported blocker marked as resolved'),
('blocker.escalated',         'user_day', 'Blocker escalated after N days open'),
('blocker.persisting',        'user_day', 'Same blocker detected on consecutive day'),

-- AI analysis events
('ai.sentiment_analyzed',     'user_day', 'Sentiment analysis completed on user entry'),
('ai.topics_extracted',       'user_day', 'Topics/keywords extracted from entry'),
('ai.auto_summary_generated', 'user_day', 'Auto-generated summary from activities and responses'),
('ai.achievement_synthesized','user_day', 'Achievement statements generated from period entries'),
('ai.disengagement_flagged',  'user_day', 'AI flagged potential disengagement pattern'),
('ai.cross_team_pattern',     'team_day', 'AI detected cross-team blocker or topic pattern'),

-- Schedule events
('schedule.created',          'team', 'Standup schedule created for team'),
('schedule.updated',          'team', 'Standup schedule settings changed'),
('schedule.paused',           'team', 'Standup schedule paused'),
('schedule.resumed',          'team', 'Standup schedule resumed'),

-- Team events
('team.created',              'team', 'Team created'),
('team.member_added',         'team', 'Member added to team'),
('team.member_removed',       'team', 'Member removed from team'),

-- Integration events
('integration.connected',     'workspace', 'External integration connected'),
('integration.disconnected',  'workspace', 'External integration disconnected'),
('integration.webhook_received','workspace', 'Webhook received from external source'),
('integration.sync_completed','workspace', 'Activity sync cycle completed');
```

---

## Event Data Examples

```sql
-- standup.response_submitted
-- data: {
--   "responses": [
--     {"question_id": "q1", "question_text": "What did you accomplish yesterday?",
--      "response": "Completed the auth migration (PR #234). Fixed the flaky test in CI."},
--     {"question_id": "q2", "question_text": "What are you working on today?",
--      "response": "Starting the notification service refactor."},
--     {"question_id": "q3", "question_text": "Any blockers or concerns?",
--      "response": "Waiting on DevOps to provision the staging database."}
--   ],
--   "submitted_via": "slack"
-- }

-- mood.submitted
-- data: {
--   "score": 4,
--   "emoji": "😊",
--   "comment": "Good day, making progress"
-- }

-- activity.pr_merged
-- data: {
--   "source": "github",
--   "external_id": "acme/api#234",
--   "title": "Auth migration to OAuth 2.0",
--   "url": "https://github.com/acme/api/pull/234",
--   "repo": "acme/api",
--   "branch": "feature/auth-migration",
--   "additions": 342,
--   "deletions": 128,
--   "files_changed": 15,
--   "merged_by": "alice-dev"
-- }

-- blocker.persisting
-- data: {
--   "description": "Waiting on DevOps for staging database",
--   "first_mentioned": "2026-05-23",
--   "days_open": 3,
--   "mentioned_in_entries": ["uuid1", "uuid2", "uuid3"]
-- }

-- ai.achievement_synthesized
-- data: {
--   "period": {"start": "2026-05-01", "end": "2026-05-31", "type": "monthly"},
--   "statements": [
--     "Led the authentication migration from legacy cookies to OAuth 2.0, merging 3 PRs with zero production incidents",
--     "Designed the notification service architecture, gaining team approval in first review cycle"
--   ],
--   "source_event_count": 87,
--   "categories": {"engineering": 2},
--   "model_version": "gpt-4o-2026-05"
-- }
```

---

## Read Models

```sql
CREATE TABLE rm_entries (
    user_id UUID NOT NULL,
    team_id UUID NOT NULL,
    entry_date DATE NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    responses JSONB NOT NULL DEFAULT '[]',
    mood JSONB,
    activities JSONB NOT NULL DEFAULT '[]',
    activity_count INT NOT NULL DEFAULT 0,
    ai JSONB,
    blockers JSONB NOT NULL DEFAULT '[]',
    submitted_at TIMESTAMPTZ,
    submitted_via TEXT,
    last_event_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id, entry_date)
);

CREATE INDEX idx_rm_entries_user ON rm_entries(user_id, entry_date DESC);
CREATE INDEX idx_rm_entries_team_date ON rm_entries(team_id, entry_date DESC);
CREATE INDEX idx_rm_entries_fulltext ON rm_entries
    USING GIN (to_tsvector('english', COALESCE(responses::TEXT, '')));

CREATE TABLE rm_daily_summaries (
    team_id UUID NOT NULL,
    summary_date DATE NOT NULL,
    narrative TEXT,
    participation JSONB NOT NULL DEFAULT '{}',
    mood_stats JSONB,
    blockers JSONB NOT NULL DEFAULT '[]',
    highlights TEXT[] NOT NULL DEFAULT '{}',
    topics JSONB NOT NULL DEFAULT '{}',
    delivered_to TEXT[] NOT NULL DEFAULT '{}',
    last_event_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, summary_date)
);

CREATE INDEX idx_rm_summaries ON rm_daily_summaries(team_id, summary_date DESC);

CREATE TABLE rm_weekly_rollups (
    team_id UUID NOT NULL,
    week_start DATE NOT NULL,
    executive_summary TEXT,
    highlights TEXT[] NOT NULL DEFAULT '{}',
    metrics JSONB NOT NULL DEFAULT '{}',
    blocker_summary TEXT,
    achievement_candidates JSONB NOT NULL DEFAULT '[]',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, week_start)
);

CREATE INDEX idx_rm_rollups ON rm_weekly_rollups(team_id, week_start DESC);

CREATE TABLE rm_blocker_tracker (
    blocker_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL,
    user_id UUID NOT NULL,
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'open',
    detected_by TEXT NOT NULL DEFAULT 'user',
    first_mentioned DATE NOT NULL,
    last_mentioned DATE NOT NULL,
    days_open INT NOT NULL DEFAULT 1,
    mention_count INT NOT NULL DEFAULT 1,
    resolved_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_blockers_team ON rm_blocker_tracker(team_id, status);
CREATE INDEX idx_rm_blockers_open ON rm_blocker_tracker(days_open DESC)
    WHERE status = 'open';

CREATE TABLE rm_mood_trends (
    team_id UUID NOT NULL,
    user_id UUID NOT NULL,
    entry_date DATE NOT NULL,
    score INT NOT NULL,
    rolling_avg_7d REAL,
    rolling_avg_30d REAL,
    trend TEXT CHECK (trend IN ('up', 'down', 'stable')),
    PRIMARY KEY (team_id, user_id, entry_date)
);

CREATE INDEX idx_rm_mood_team ON rm_mood_trends(team_id, entry_date DESC);
CREATE INDEX idx_rm_mood_user ON rm_mood_trends(user_id, entry_date DESC);

CREATE TABLE rm_achievements (
    user_id UUID NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    period_type TEXT NOT NULL,
    statements TEXT[] NOT NULL,
    source_event_count INT NOT NULL DEFAULT 0,
    categories JSONB,
    is_reviewed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, period_start, period_type)
);

CREATE INDEX idx_rm_achievements ON rm_achievements(user_id, period_start DESC);
```

---

## Reference Tables

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'manager', 'member')),
    avatar_url TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_workspace ON users(workspace_id);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    version BIGINT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Example Queries

### All events for a user on a date (raw stream replay)

```sql
SELECT event_type, data, occurred_at
FROM event_store
WHERE stream_type = 'user_day'
  AND stream_id = 'user-day-uuid'
ORDER BY version;
```

### Activity timeline from events (cross-source)

```sql
SELECT event_type, data->>'source' AS source,
       data->>'title' AS title, data->>'url' AS url,
       occurred_at
FROM event_store
WHERE stream_type = 'user_day'
  AND (metadata->>'actor_id') = 'user-uuid'
  AND event_type LIKE 'activity.%'
  AND occurred_at BETWEEN '2026-05-25' AND '2026-05-26'
ORDER BY occurred_at;
```

### Blocker persistence from events

```sql
SELECT data->>'description' AS blocker,
       MIN(occurred_at)::DATE AS first_mentioned,
       MAX(occurred_at)::DATE AS last_mentioned,
       COUNT(DISTINCT occurred_at::DATE) AS days_mentioned
FROM event_store
WHERE event_type IN ('blocker.detected', 'blocker.persisting')
  AND (metadata->>'team_id') = 'team-uuid'
  AND occurred_at >= now() - INTERVAL '14 days'
GROUP BY data->>'description'
HAVING COUNT(DISTINCT occurred_at::DATE) > 1
ORDER BY days_mentioned DESC;
```

### Mood trend from read model

```sql
SELECT entry_date, score, rolling_avg_7d, rolling_avg_30d, trend
FROM rm_mood_trends
WHERE user_id = 'user-uuid'
  AND entry_date >= CURRENT_DATE - 60
ORDER BY entry_date;
```

### Team participation rate from daily summaries

```sql
SELECT summary_date,
       (participation->>'submitted')::INT AS submitted,
       (participation->>'total')::INT AS total,
       (participation->>'rate')::REAL AS rate
FROM rm_daily_summaries
WHERE team_id = 'team-uuid'
  AND summary_date >= CURRENT_DATE - 30
ORDER BY summary_date;
```

### Event volume by source (for monitoring ingestion health)

```sql
SELECT metadata->>'source' AS source,
       event_type,
       COUNT(*) AS event_count,
       MAX(occurred_at) AS last_event
FROM event_store
WHERE occurred_at >= now() - INTERVAL '24 hours'
GROUP BY metadata->>'source', event_type
ORDER BY event_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 4 | event_store (partitioned), event_types, projection_checkpoints, stream_snapshots |
| Read Models | 6 | rm_entries, rm_daily_summaries, rm_weekly_rollups, rm_blocker_tracker, rm_mood_trends, rm_achievements |
| Reference | 3 | workspaces, users, teams |
| **Total** | **13** | Plus 4 quarterly partitions on event_store |

---

## Key Design Decisions

1. **User-day streams** — The primary stream is `user_day` — all events for a single user on a single day. This groups standup responses, mood, activities, and AI analyses into one stream that can be replayed to reconstruct a complete daily entry. The stream ID is a composite of user and date.

2. **External webhooks as events** — GitHub webhooks, Jira notifications, and other external signals store directly in the event store as `activity.*` events. There is no intermediate "activities" table — the event store IS the activity log. This means adding a new source (e.g., Linear) means defining new event types, not new tables.

3. **Blocker lifecycle as events** — Blockers are not a table with status; they are a sequence of events (`blocker.detected`, `blocker.persisting`, `blocker.resolved`, `blocker.escalated`). The `rm_blocker_tracker` read model materializes the current state (open/resolved, days_open) from these events. This provides a complete audit trail of how long blockers persisted and when they were escalated.

4. **Mood trends as read model** — `rm_mood_trends` pre-computes rolling averages (7-day, 30-day) and trend direction. This avoids expensive window function queries on the event store and provides instant rendering for mood dashboards. The projection updates on each `mood.submitted` event.

5. **Team-day stream for summaries** — Daily summaries and weekly rollups are events on a `team_day` stream. The summary generation AI reads from all `user_day` streams for the team, produces a `summary.daily_generated` event, and the delivery system emits a `summary.daily_delivered` event. This separates generation from delivery, enabling retry without regeneration.

6. **Achievement synthesis from event history** — `ai.achievement_synthesized` events are produced by scanning the user's event history for a period. The achievement statements, source event count, and categories are all in the event data. The `rm_achievements` read model stores the latest synthesis for quick access.

7. **Quarterly partitions** — Work journal data accumulates steadily (team_size × working_days × ~5-20 events/day). Quarterly partitions keep the hot partition small while enabling archival of older quarters. A 10-person team generates roughly 50,000-200,000 events per quarter.

8. **Disengagement detection from event patterns** — `ai.disengagement_flagged` events fire when the AI detects patterns suggesting burnout or disengagement: declining mood scores, decreasing activity volume, increasing skip rate, shorter response lengths. The event carries the evidence (which patterns triggered it) for manager review. This is the most GDPR-sensitive feature — the event metadata must flag this as requiring explicit consent.
