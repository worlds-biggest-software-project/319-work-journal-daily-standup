# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Work Journal & Daily Standup · Created: 2026-05-25

## Philosophy

A work journal and async standup platform manages a clear daily cycle: teams define standup schedules with questions, team members submit responses each day, responses are aggregated into team summaries, and over time entries accumulate into a searchable personal work log. Layered on top are auto-generated updates from external tools (GitHub commits, Jira transitions), mood tracking for team health signals, AI-generated summaries and achievement synthesis, blocker detection, and multi-channel delivery (Slack, Teams, email). A normalized relational model gives each concept its own table, enabling database-level enforcement of the schedule→prompt→response→summary pipeline and clean separation between input (responses, activities), output (summaries, digests), and analytics (mood, blockers, trends).

This mirrors how engineering managers think about standups: a team has a schedule, the schedule triggers prompts at the right time in each person's timezone, each person submits answers to specific questions, the platform aggregates answers into a summary posted to a channel, and over time the system detects patterns (recurring blockers, sentiment trends, participation gaps). Each maps to a table.

**Best for:** Teams building a production-grade standup platform where schedule management across timezones is complex, where multi-source activity ingestion needs explicit tracking, and where historical analytics (mood trends, blocker patterns, participation rates) are first-class features.

**Trade-offs:**
- **Pro:** Database-enforced schedule → prompt → response pipeline
- **Pro:** Activity sources as explicit rows enable source-specific health monitoring
- **Pro:** Mood entries as typed rows enable trend charts and burnout detection
- **Pro:** Standup responses linked to questions enable per-question analytics
- **Con:** 18 tables — moderate complexity
- **Con:** Full standup view (all responses for a day) requires joins across responses, questions, activities
- **Con:** Activity data varies by source but is modeled with shared columns
- **Con:** Daily entry volume is low per user but grows linearly with team size × days

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Scrum Guide 2020 | Daily Scrum question format (what did I do / what will I do / blockers) |
| RFC 5545 (iCalendar) | Schedule export as recurring events |
| Slack Block Kit | Standup prompts and summary rendering |
| MS Adaptive Cards v1.5 | Teams-native standup prompts |
| OAuth 2.0 (RFC 6749) | GitHub, GitLab, Jira, Slack, Teams integration auth |
| OIDC Core 1.0 | Enterprise SSO |
| OpenAPI 3.1 | REST API specification |
| GDPR | Mood data as sensitive personal data; right-to-erasure |
| MCP 2025-11-25 | Journal entries exposed to AI assistants |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Workspaces, Teams & Users

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

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('manager', 'member')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
```

---

## Standup Schedules & Questions

```sql
CREATE TABLE standup_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name TEXT NOT NULL DEFAULT 'Daily Standup',
    cadence TEXT NOT NULL DEFAULT 'daily' CHECK (cadence IN (
        'daily', 'weekdays', 'weekly', 'custom'
    )),
    days_of_week INT[] DEFAULT '{1,2,3,4,5}',
    prompt_time TIME NOT NULL DEFAULT '09:00',
    use_member_timezone BOOLEAN NOT NULL DEFAULT TRUE,
    reminder_minutes INT DEFAULT 30,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    delivery_channels TEXT[] NOT NULL DEFAULT '{slack}',
    delivery_config JSONB NOT NULL DEFAULT '{}',
    -- delivery_config: {
    --   "slack": {"channel_id": "C123", "thread_replies": true},
    --   "teams": {"channel_id": "...", "team_id": "..."},
    --   "email": {"subject_template": "Daily Standup - {team} - {date}"}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedules_team ON standup_schedules(team_id);

CREATE TABLE standup_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id UUID NOT NULL REFERENCES standup_schedules(id) ON DELETE CASCADE,
    question_text TEXT NOT NULL,
    question_type TEXT NOT NULL DEFAULT 'text' CHECK (question_type IN (
        'text', 'blockers', 'mood', 'goals', 'custom'
    )),
    placeholder TEXT,
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_questions_schedule ON standup_questions(schedule_id, sort_order);
```

---

## Standup Entries & Responses

```sql
CREATE TABLE standup_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id UUID NOT NULL REFERENCES standup_schedules(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    entry_date DATE NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'submitted', 'skipped'
    )),
    submitted_at TIMESTAMPTZ,
    submitted_via TEXT CHECK (submitted_via IN ('slack', 'teams', 'email', 'web', 'api')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (schedule_id, user_id, entry_date)
);

CREATE INDEX idx_entries_schedule_date ON standup_entries(schedule_id, entry_date DESC);
CREATE INDEX idx_entries_user ON standup_entries(user_id, entry_date DESC);

CREATE TABLE standup_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID NOT NULL REFERENCES standup_entries(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES standup_questions(id),
    response_text TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_responses_entry ON standup_responses(entry_id);
CREATE INDEX idx_responses_fulltext ON standup_responses
    USING GIN (to_tsvector('english', response_text));
```

---

## Mood Tracking

```sql
CREATE TABLE mood_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID NOT NULL REFERENCES standup_entries(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    score INT NOT NULL CHECK (score BETWEEN 1 AND 5),
    emoji TEXT,
    comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entry_id)
);

CREATE INDEX idx_mood_user ON mood_entries(user_id, created_at DESC);
```

---

## Activity Sources & Auto-Generated Updates

```sql
CREATE TABLE activity_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider TEXT NOT NULL CHECK (provider IN (
        'github', 'gitlab', 'jira', 'linear', 'bitbucket'
    )),
    status TEXT NOT NULL DEFAULT 'connected',
    credentials_enc TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, provider)
);

CREATE TABLE user_activity_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    source_id UUID NOT NULL REFERENCES activity_sources(id) ON DELETE CASCADE,
    external_username TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, external_username)
);

CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    source_id UUID NOT NULL REFERENCES activity_sources(id),
    activity_type TEXT NOT NULL CHECK (activity_type IN (
        'commit', 'pull_request_opened', 'pull_request_merged',
        'pull_request_reviewed', 'issue_updated', 'issue_closed',
        'issue_commented', 'deployment', 'branch_created'
    )),
    title TEXT NOT NULL,
    description TEXT,
    external_id TEXT NOT NULL,
    external_url TEXT,
    repo_name TEXT,
    occurred_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, external_id)
);

CREATE INDEX idx_activities_user ON activities(user_id, occurred_at DESC);
CREATE INDEX idx_activities_date ON activities(user_id, (occurred_at::DATE));
```

---

## Summaries & Digests

```sql
CREATE TABLE daily_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id UUID NOT NULL REFERENCES standup_schedules(id) ON DELETE CASCADE,
    summary_date DATE NOT NULL,
    summary_text TEXT NOT NULL,
    participation_count INT NOT NULL DEFAULT 0,
    total_members INT NOT NULL DEFAULT 0,
    blockers_detected INT NOT NULL DEFAULT 0,
    delivered_to TEXT[] NOT NULL DEFAULT '{}',
    ai_narrative TEXT,
    model_version TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (schedule_id, summary_date)
);

CREATE INDEX idx_summaries_schedule ON daily_summaries(schedule_id, summary_date DESC);

CREATE TABLE weekly_rollups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    week_start DATE NOT NULL,
    week_end DATE NOT NULL,
    executive_summary TEXT NOT NULL,
    highlights TEXT[],
    blockers_resolved INT NOT NULL DEFAULT 0,
    blockers_ongoing INT NOT NULL DEFAULT 0,
    participation_rate REAL,
    mood_avg REAL,
    mood_trend TEXT CHECK (mood_trend IN ('up', 'down', 'stable')),
    model_version TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, week_start)
);

CREATE INDEX idx_rollups_team ON weekly_rollups(team_id, week_start DESC);
```

---

## Achievement Synthesis

```sql
CREATE TABLE achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    period_type TEXT NOT NULL CHECK (period_type IN (
        'weekly', 'monthly', 'quarterly', 'yearly', 'custom'
    )),
    achievement_statements TEXT[] NOT NULL,
    source_entry_ids UUID[] NOT NULL DEFAULT '{}',
    source_activity_ids UUID[] NOT NULL DEFAULT '{}',
    is_reviewed BOOLEAN NOT NULL DEFAULT FALSE,
    model_version TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_achievements_user ON achievements(user_id, period_start DESC);
```

---

## Blockers

```sql
CREATE TABLE blockers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id),
    user_id UUID NOT NULL REFERENCES users(id),
    entry_id UUID REFERENCES standup_entries(id),
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'resolved', 'escalated'
    )),
    detected_by TEXT NOT NULL DEFAULT 'user' CHECK (detected_by IN ('user', 'ai')),
    days_open INT NOT NULL DEFAULT 0,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_blockers_team ON blockers(team_id, status);
CREATE INDEX idx_blockers_open ON blockers(team_id, days_open DESC)
    WHERE status = 'open';
```

---

## Example Queries

### Full standup view for a team on a date

```sql
SELECT u.name, u.avatar_url,
       sq.question_text, sr.response_text,
       me.score AS mood_score, me.emoji AS mood_emoji,
       se.submitted_at, se.submitted_via
FROM standup_entries se
JOIN users u ON u.id = se.user_id
JOIN standup_responses sr ON sr.entry_id = se.id
JOIN standup_questions sq ON sq.id = sr.question_id
LEFT JOIN mood_entries me ON me.entry_id = se.id
WHERE se.schedule_id = 'schedule-uuid'
  AND se.entry_date = '2026-05-25'
  AND se.status = 'submitted'
ORDER BY u.name, sq.sort_order;
```

### Auto-generated update for a user on a date

```sql
SELECT a.activity_type, a.title, a.external_url, a.repo_name, a.occurred_at
FROM activities a
WHERE a.user_id = 'user-uuid'
  AND a.occurred_at::DATE = '2026-05-25'
ORDER BY a.occurred_at;
```

### Mood trend for a team over 30 days

```sql
SELECT se.entry_date, AVG(me.score) AS avg_mood,
       COUNT(DISTINCT me.user_id) AS respondents
FROM mood_entries me
JOIN standup_entries se ON se.id = me.entry_id
WHERE se.schedule_id = 'schedule-uuid'
  AND se.entry_date >= CURRENT_DATE - 30
GROUP BY se.entry_date
ORDER BY se.entry_date;
```

### Participation rate by team member

```sql
SELECT u.name,
       COUNT(*) FILTER (WHERE se.status = 'submitted') AS submitted,
       COUNT(*) AS total,
       COUNT(*) FILTER (WHERE se.status = 'submitted') * 100.0 / COUNT(*) AS rate
FROM standup_entries se
JOIN users u ON u.id = se.user_id
WHERE se.schedule_id = 'schedule-uuid'
  AND se.entry_date >= CURRENT_DATE - 30
GROUP BY u.id, u.name
ORDER BY rate DESC;
```

### Long-running blockers

```sql
SELECT b.description, b.days_open, b.detected_by,
       u.name AS reported_by, t.name AS team
FROM blockers b
JOIN users u ON u.id = b.user_id
JOIN teams t ON t.id = b.team_id
WHERE b.status = 'open'
  AND b.days_open > 3
ORDER BY b.days_open DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workspace & Users | 4 | workspaces, users, teams, team_members |
| Schedules | 2 | standup_schedules, standup_questions |
| Entries | 2 | standup_entries, standup_responses |
| Mood | 1 | mood_entries |
| Activity | 3 | activity_sources, user_activity_links, activities |
| Summaries | 2 | daily_summaries, weekly_rollups |
| Achievements | 1 | achievements |
| Blockers | 1 | blockers |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Responses linked to questions** — `standup_responses` references both the entry and the specific question. This enables per-question analytics ("which question gets the most detailed answers?") and supports teams that change questions without losing historical data tied to the old question set.

2. **Mood as separate table** — `mood_entries` stores mood scores independently of standup responses, enabling mood trend analysis without parsing response text. The 1-5 scale with optional emoji maps to the StatusHero-style engagement pattern that managers use for team health monitoring.

3. **Activities as explicit rows** — `activities` stores each Git commit, PR, and Jira transition as its own row. This enables activity-count analytics ("Alice closed 12 issues this week"), deduplication via `(source_id, external_id)` uniqueness, and selective inclusion in auto-generated updates.

4. **User-to-source linking** — `user_activity_links` maps platform users to their external usernames (GitHub handle, Jira account). A single user might have different usernames on GitHub and Jira, so the mapping is per-source.

5. **Daily summaries as materialized output** — `daily_summaries` stores the AI-generated team narrative alongside participation stats and blocker counts. This avoids re-generating summaries on every view and provides a historical record of what was posted to Slack/Teams.

6. **Weekly rollups for managers** — `weekly_rollups` stores executive summaries with mood trends and participation rates. This is the manager-facing view that rolls up daily data into a leadership-ready format.

7. **Achievement synthesis from entries** — `achievements` stores resume-ready achievement statements synthesized from standup responses and activities over a period. The `source_entry_ids` and `source_activity_ids` arrays trace which raw data informed each achievement, enabling the user to verify and edit.

8. **Blockers as tracked entities** — `blockers` tracks blockers with open/resolved status and `days_open` counter. Blockers can be user-reported (from standup responses) or AI-detected (from stalled activity patterns). The `days_open` counter enables escalation rules ("flag blockers open > 3 days").
