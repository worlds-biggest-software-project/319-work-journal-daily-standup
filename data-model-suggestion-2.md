# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Work Journal & Daily Standup · Created: 2026-05-25

## Philosophy

A standup and work journal platform has a fixed daily rhythm — prompt, respond, summarize — but the content within each step varies widely. Teams customize questions (classic three-question format, goal-linked, mood-only, free-form), activity sources produce different shaped data (GitHub commits vs. Jira ticket transitions vs. Linear issues), and AI outputs evolve rapidly (summaries, blocker detection, sentiment scores, achievement statements). A hybrid approach keeps the schedule→entry pipeline relational while pushing variable content into JSONB columns.

The key insight is that a standup entry is a self-contained document: a user's responses, their mood, their auto-populated activities, and any AI-generated insights all belong together and are always loaded as a unit. There is no use case for "give me response #3 from yesterday's standup independent of the other responses." Similarly, a team summary is a complete document — the narrative, participation stats, and blocker alerts are always rendered together. By inlining these as JSONB, the "show today's standup" query returns everything in one row per user.

**Best for:** Teams building a standup tool where rapid iteration on question formats and AI features matters, where the development team values simplicity over strict normalization, and where the primary access pattern is "load one user's entry for one day."

**Trade-offs:**
- **Pro:** Full standup entry loads in a single row — no joins
- **Pro:** New question types require zero schema changes
- **Pro:** Activity data shape varies by source without schema complexity
- **Pro:** AI outputs evolve freely within JSONB
- **Pro:** 8 tables total — very low complexity
- **Con:** Per-question analytics require JSONB unpacking
- **Con:** Full-text search across responses needs GIN indexes on JSONB
- **Con:** Mood trend queries require extracting scores from JSONB entries
- **Con:** Activity deduplication is harder without a dedicated table

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Scrum Guide 2020 | Default question template follows Daily Scrum pattern |
| RFC 5545 (iCalendar) | Schedule export as recurring events |
| Slack Block Kit | JSONB entries map directly to Block Kit message payloads |
| MS Adaptive Cards v1.5 | JSONB entries map to Adaptive Card content |
| OAuth 2.0 (RFC 6749) | Integration auth |
| OIDC Core 1.0 | Enterprise SSO |
| GDPR | Mood and sentiment data as sensitive personal data |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Workspaces & Users

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings: {
    --   "billing_plan": "pro",
    --   "integrations": {
    --     "slack": {"bot_token_enc": "...", "team_id": "T123"},
    --     "teams": {"bot_id": "...", "tenant_id": "..."},
    --     "github": {"app_id": "...", "installation_id": "...", "credentials_enc": "..."},
    --     "gitlab": {"access_token_enc": "...", "base_url": "https://gitlab.com"},
    --     "jira": {"base_url": "...", "credentials_enc": "...", "default_project": "ENG"},
    --     "linear": {"api_key_enc": "..."}
    --   },
    --   "ai": {"provider": "openai", "model": "gpt-4o-2026-05"},
    --   "notifications": {
    --     "email_from": "standups@acme.com",
    --     "digest_time": "17:00"
    --   }
    -- }
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
    profile JSONB NOT NULL DEFAULT '{}',
    -- profile: {
    --   "external_accounts": {
    --     "github": "alice-dev",
    --     "gitlab": "alice",
    --     "jira": "alice@acme.com",
    --     "slack": "U12345",
    --     "teams": "alice@acme.onmicrosoft.com"
    --   },
    --   "preferences": {
    --     "preferred_channel": "slack",
    --     "receive_email_digest": true,
    --     "achievement_cadence": "monthly"
    --   }
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_workspace ON users(workspace_id);
```

---

## Teams & Schedules

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    members JSONB NOT NULL DEFAULT '[]',
    -- members: [
    --   {"user_id": "uuid", "role": "manager", "joined_at": "2026-01-15T10:00:00Z"},
    --   {"user_id": "uuid", "role": "member", "joined_at": "2026-02-01T09:00:00Z"}
    -- ]
    schedule JSONB NOT NULL DEFAULT '{}',
    -- schedule: {
    --   "name": "Daily Standup",
    --   "cadence": "weekdays",
    --   "days_of_week": [1, 2, 3, 4, 5],
    --   "prompt_time": "09:00",
    --   "use_member_timezone": true,
    --   "reminder_minutes": 30,
    --   "is_active": true,
    --   "questions": [
    --     {"id": "q1", "text": "What did you accomplish yesterday?", "type": "text", "required": true, "sort_order": 0},
    --     {"id": "q2", "text": "What are you working on today?", "type": "text", "required": true, "sort_order": 1},
    --     {"id": "q3", "text": "Any blockers or concerns?", "type": "blockers", "required": false, "sort_order": 2},
    --     {"id": "q4", "text": "How are you feeling today?", "type": "mood", "required": false, "sort_order": 3}
    --   ],
    --   "delivery": {
    --     "slack": {"channel_id": "C123", "thread_replies": true, "post_summary": true},
    --     "teams": {"channel_id": "...", "team_id": "..."},
    --     "email": {"enabled": true, "subject": "Daily Standup - {team} - {date}"}
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE INDEX idx_teams_workspace ON teams(workspace_id);
CREATE INDEX idx_teams_members ON teams USING GIN (members jsonb_path_ops);
```

---

## Standup Entries

```sql
CREATE TABLE standup_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    entry_date DATE NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'submitted', 'skipped'
    )),
    responses JSONB NOT NULL DEFAULT '[]',
    -- responses: [
    --   {"question_id": "q1", "question_text": "What did you accomplish yesterday?",
    --    "response": "Completed the auth migration (PR #234). Fixed the flaky test in CI."},
    --   {"question_id": "q2", "question_text": "What are you working on today?",
    --    "response": "Starting the notification service refactor. Code review for Sarah's PR."},
    --   {"question_id": "q3", "question_text": "Any blockers or concerns?",
    --    "response": "Waiting on DevOps to provision the staging database."}
    -- ]
    mood JSONB,
    -- mood: {
    --   "score": 4,
    --   "emoji": "😊",
    --   "comment": "Good day, making progress"
    -- }
    activities JSONB NOT NULL DEFAULT '[]',
    -- activities: [
    --   {"source": "github", "type": "pull_request_merged", "title": "Auth migration",
    --    "url": "https://github.com/acme/api/pull/234", "repo": "acme/api",
    --    "occurred_at": "2026-05-24T16:30:00Z"},
    --   {"source": "github", "type": "commit", "title": "Fix flaky auth test",
    --    "url": "https://github.com/acme/api/commit/abc123", "repo": "acme/api",
    --    "occurred_at": "2026-05-24T17:00:00Z"},
    --   {"source": "jira", "type": "issue_closed", "title": "ENG-1234: Auth migration",
    --    "url": "https://acme.atlassian.net/browse/ENG-1234",
    --    "occurred_at": "2026-05-24T16:35:00Z"}
    -- ]
    ai JSONB,
    -- ai: {
    --   "auto_summary": "Completed auth migration and fixed CI. Starting notification refactor today.",
    --   "sentiment": "positive",
    --   "sentiment_score": 0.72,
    --   "blockers_detected": [
    --     {"description": "Waiting on DevOps for staging database", "severity": "medium",
    --      "days_mentioned": 1, "related_entries": []}
    --   ],
    --   "topics": ["auth", "migration", "notifications", "code-review"],
    --   "model_version": "gpt-4o-2026-05"
    -- }
    submitted_at TIMESTAMPTZ,
    submitted_via TEXT CHECK (submitted_via IN ('slack', 'teams', 'email', 'web', 'api')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, user_id, entry_date)
);

CREATE INDEX idx_entries_team_date ON standup_entries(team_id, entry_date DESC);
CREATE INDEX idx_entries_user ON standup_entries(user_id, entry_date DESC);
CREATE INDEX idx_entries_responses ON standup_entries USING GIN (responses jsonb_path_ops);
CREATE INDEX idx_entries_activities ON standup_entries USING GIN (activities jsonb_path_ops);
CREATE INDEX idx_entries_fulltext ON standup_entries
    USING GIN (to_tsvector('english',
        COALESCE(responses::TEXT, '') || ' ' || COALESCE(activities::TEXT, '')));
```

---

## Summaries & Rollups

```sql
CREATE TABLE team_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    summary_date DATE NOT NULL,
    summary JSONB NOT NULL,
    -- summary: {
    --   "narrative": "The team made strong progress on the auth migration, with Alice closing the main PR and Bob completing the related database changes...",
    --   "participation": {"submitted": 5, "total": 7, "rate": 0.71},
    --   "mood": {"avg_score": 3.8, "respondents": 4, "trend": "stable"},
    --   "blockers": [
    --     {"description": "Staging database provisioning", "reported_by": ["Alice"],
    --      "days_open": 2, "status": "open"}
    --   ],
    --   "highlights": [
    --     "Auth migration PR merged (Alice)",
    --     "New notification service design doc started (Bob)"
    --   ],
    --   "topics": {"auth": 3, "notifications": 2, "testing": 1},
    --   "delivered_to": ["slack:C123", "email:digest"],
    --   "model_version": "gpt-4o-2026-05"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, summary_date)
);

CREATE INDEX idx_summaries_team ON team_summaries(team_id, summary_date DESC);

CREATE TABLE weekly_rollups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    week_start DATE NOT NULL,
    rollup JSONB NOT NULL,
    -- rollup: {
    --   "executive_summary": "The engineering team completed the auth migration ahead of schedule...",
    --   "highlights": [
    --     "Auth migration completed — 3 PRs merged, zero production incidents",
    --     "Notification service design doc approved, implementation starting next week"
    --   ],
    --   "metrics": {
    --     "participation_rate": 0.85,
    --     "mood_avg": 3.9,
    --     "mood_trend": "up",
    --     "blockers_opened": 3,
    --     "blockers_resolved": 2,
    --     "blockers_ongoing": 1
    --   },
    --   "blocker_summary": "One ongoing blocker: staging DB provisioning (5 days, escalated to DevOps lead)",
    --   "achievement_candidates": [
    --     {"user_id": "uuid", "statement": "Led the auth migration from legacy system to OAuth 2.0, completing 3 PRs with zero production incidents"}
    --   ],
    --   "model_version": "gpt-4o-2026-05"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, week_start)
);

CREATE INDEX idx_rollups_team ON weekly_rollups(team_id, week_start DESC);
```

---

## Achievements

```sql
CREATE TABLE achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    period_type TEXT NOT NULL CHECK (period_type IN (
        'weekly', 'monthly', 'quarterly', 'yearly', 'custom'
    )),
    content JSONB NOT NULL,
    -- content: {
    --   "statements": [
    --     "Led the authentication migration from legacy cookies to OAuth 2.0, merging 3 PRs with zero production incidents",
    --     "Designed and documented the notification service architecture, gaining team approval in first review cycle",
    --     "Mentored two junior engineers through their first production deployments"
    --   ],
    --   "source_entries": 22,
    --   "source_activities": 47,
    --   "categories": {"engineering": 2, "leadership": 1},
    --   "is_reviewed": false,
    --   "model_version": "gpt-4o-2026-05"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_achievements_user ON achievements(user_id, period_start DESC);
```

---

## Example Queries

### Full standup view for a team on a date

```sql
SELECT u.name, u.avatar_url,
       se.responses, se.mood, se.activities, se.ai,
       se.submitted_at, se.submitted_via
FROM standup_entries se
JOIN users u ON u.id = se.user_id
WHERE se.team_id = 'team-uuid'
  AND se.entry_date = '2026-05-25'
  AND se.status = 'submitted'
ORDER BY se.submitted_at;
```

### Mood trend over 30 days

```sql
SELECT entry_date,
       AVG((mood->>'score')::INT) AS avg_mood,
       COUNT(*) FILTER (WHERE mood IS NOT NULL) AS respondents
FROM standup_entries
WHERE team_id = 'team-uuid'
  AND entry_date >= CURRENT_DATE - 30
  AND status = 'submitted'
GROUP BY entry_date
ORDER BY entry_date;
```

### Search across all entries for a keyword

```sql
SELECT u.name, se.entry_date, se.responses, se.activities
FROM standup_entries se
JOIN users u ON u.id = se.user_id
WHERE se.team_id = 'team-uuid'
  AND to_tsvector('english', se.responses::TEXT) @@ plainto_tsquery('english', 'auth migration')
ORDER BY se.entry_date DESC
LIMIT 20;
```

### Recurring blockers (mentioned multiple days)

```sql
SELECT blocker->>'description' AS blocker,
       COUNT(DISTINCT se.entry_date) AS days_mentioned,
       MIN(se.entry_date) AS first_mentioned
FROM standup_entries se,
     jsonb_array_elements(se.ai->'blockers_detected') AS blocker
WHERE se.team_id = 'team-uuid'
  AND se.entry_date >= CURRENT_DATE - 14
GROUP BY blocker->>'description'
HAVING COUNT(DISTINCT se.entry_date) > 1
ORDER BY days_mentioned DESC;
```

### Personal activity log for performance review

```sql
SELECT entry_date,
       jsonb_array_length(activities) AS activity_count,
       mood->>'score' AS mood,
       ai->>'auto_summary' AS summary
FROM standup_entries
WHERE user_id = 'user-uuid'
  AND entry_date BETWEEN '2026-01-01' AND '2026-06-30'
  AND status = 'submitted'
ORDER BY entry_date;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workspace & Users | 2 | workspaces (with integrations JSONB), users (with external accounts JSONB) |
| Teams & Schedules | 1 | teams (with members, schedule, questions as JSONB) |
| Entries | 1 | standup_entries (with responses, mood, activities, AI as JSONB) |
| Summaries | 2 | team_summaries, weekly_rollups (both with content as JSONB) |
| Achievements | 1 | achievements (with statements as JSONB) |
| **Total** | **7** | Versus 16 in normalized model |

---

## Key Design Decisions

1. **Entries as self-contained documents** — `standup_entries` stores responses, mood, activities, and AI analysis in a single row. This matches the primary access pattern: "load Alice's entry for today" is one row, not a join across 4 tables. The Slack/Teams rendering code receives the JSONB directly and maps it to Block Kit / Adaptive Cards.

2. **Schedule and questions on teams** — `teams.schedule` embeds the standup configuration including questions. Since questions rarely change and are always loaded with the schedule configuration, inlining avoids a separate table. Historical question text is preserved in `standup_entries.responses` (each response includes its question text).

3. **Activities inline on entries** — `standup_entries.activities` stores auto-populated Git/Jira activities as a JSONB array. This avoids a separate activities table and means the entry is self-contained — the auto-generated update renders from the same row as the manual responses.

4. **Mood inline on entries** — `standup_entries.mood` stores the mood score, emoji, and optional comment. Since mood is always displayed with the entry, inlining is natural. Mood trend queries use `(mood->>'score')::INT` with a functional index if needed.

5. **External accounts on users** — `users.profile.external_accounts` maps the user to their GitHub, GitLab, Jira, Slack, and Teams usernames. This replaces a separate linking table while keeping the mapping co-located with the user profile.

6. **Summaries as JSONB documents** — `team_summaries.summary` stores the AI narrative, participation stats, mood average, blockers, and highlights as a single JSONB document. This matches how summaries are rendered (posted to Slack as one message) and evolves freely as the AI summary format improves.

7. **Achievements with category tagging** — `achievements.content` stores achievement statements with source counts and categories. The JSONB format allows the AI to add new metadata (impact scores, relevance tags) without migrations.

8. **Integrations on workspace** — All external service connections (Slack, Teams, GitHub, GitLab, Jira) live in `workspaces.settings.integrations`. This keeps the integration config centralized and avoids a separate table for each provider.
