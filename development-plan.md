# Work Journal & Daily Standup — Phased Development Plan

> Project: 319-work-journal-daily-standup · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (Entity-Centric Normalized Relational) into a concrete, additive, phased build. The normalized relational model is chosen as the primary schema because the product's differentiating features — mood trends, blocker-persistence patterns, participation rates, and per-question analytics — are all first-class analytic queries that benefit from typed, indexed columns rather than JSONB unpacking.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The product is LLM-heavy (summary synthesis, blocker detection, achievement synthesis, sentiment analysis). Python has the richest LLM/agent tooling and first-party SDKs (Slack Bolt, PyGithub, python-gitlab, jira, notion-sdk-py). |
| API framework | FastAPI | Generates OpenAPI 3.1 (a stated standard) automatically, async-native for fan-out to LLM and integration APIs, Pydantic v2 request/response validation enforces JSON Schema Draft 2020-12. |
| ASGI server | Uvicorn (dev), Gunicorn+Uvicorn workers (prod) | Standard FastAPI deployment. |
| Database | PostgreSQL 16 | The schema uses `UUID`, `TIMESTAMPTZ`, `INT[]`, `JSONB`, GIN full-text indexes, and partial indexes — all native PostgreSQL. Full-text search requirement (`tsvector`) rules out SQLite for production. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Async ORM matches FastAPI; Alembic gives versioned, reversible migrations required by the Definition of Done. |
| Task queue | Celery + Redis | Scheduled standup prompts, timezone-aware reminders, activity polling, and long-running LLM calls are async workloads. Celery Beat handles the cron-like schedule firing; Redis doubles as broker and cache. |
| LLM provider | Anthropic Claude (via `anthropic` SDK) with a provider abstraction | Summary/narrative/achievement synthesis. Provider is wrapped behind an interface so OpenAI/local models can be swapped. Prompt caching used for repeated system prompts. |
| Chat: Slack | Slack Bolt for Python | Native Block Kit rendering, OAuth, Events API, modal-based response collection. |
| Chat: Teams | Bot Framework SDK (botbuilder-core) + Adaptive Cards v1.5 | Separate rendering layer per standards.md note on Block Kit / Adaptive Card incompatibility. |
| Email | SMTP via `aiosmtplib` + Jinja2 templates | First-class email delivery channel for non-chat users. |
| Auth (users) | OIDC (Authlib) — Google + Microsoft Entra | OpenID Connect Core 1.0 for SaaS login and enterprise SSO. |
| Auth (integrations) | OAuth 2.0 (Authlib) | RFC 6749 for GitHub/GitLab/Jira/Slack/Teams; no static keys in production. |
| Secrets at rest | `cryptography` Fernet (envelope on `credentials_enc`) | Integration tokens encrypted in `activity_sources.credentials_enc`. |
| Frontend | Next.js 16 (App Router) + React + TypeScript + shadcn/ui + Tailwind | Mobile-responsive web interface (MVP requirement). Charts via Recharts for mood/participation trends. |
| MCP server | Python MCP SDK (`mcp`) | standards.md identifies MCP as a day-one differentiator; exposes journal entries, summaries, standup history to AI assistants. |
| Containerisation | Docker + docker-compose | Self-hosted deployment is in scope; compose orchestrates api, worker, beat, postgres, redis, web. |
| Testing | pytest + pytest-asyncio + httpx.AsyncClient + testcontainers | Unit, mocked-integration, and real-integration (Postgres via testcontainers) tiers. Frontend: Vitest + Playwright. |
| Code quality | ruff (lint+format), mypy (strict), pre-commit | Standard Python toolchain; enforced in Definition of Done. |
| Package manager | uv (Python), pnpm (web) | Fast, lockfile-based. |
| Scheduling format | iCalendar via `icalendar` lib | RFC 5545 export of standup schedules as recurring events. |
| Outbound events | Webhooks documented with AsyncAPI 2.6 | Summary push to external trackers/HR tools. |

### Project Structure

```
work-journal/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── README.md
├── openapi/                         # exported OAS 3.1 + AsyncAPI specs (CI-generated)
│   └── asyncapi.yaml
├── migrations/                      # Alembic versions
│   └── versions/
├── src/
│   └── workjournal/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory, router mounting
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py              # async engine, session factory
│       │   ├── models/              # SQLAlchemy models (one file per domain)
│       │   │   ├── workspace.py
│       │   │   ├── user.py
│       │   │   ├── schedule.py
│       │   │   ├── entry.py
│       │   │   ├── mood.py
│       │   │   ├── activity.py
│       │   │   ├── summary.py
│       │   │   ├── achievement.py
│       │   │   └── blocker.py
│       │   └── repositories/        # data-access layer (one per aggregate)
│       ├── schemas/                 # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py              # auth, db session, current-user deps
│       │   ├── routers/
│       │   │   ├── auth.py
│       │   │   ├── workspaces.py
│       │   │   ├── teams.py
│       │   │   ├── schedules.py
│       │   │   ├── entries.py
│       │   │   ├── journal.py
│       │   │   ├── summaries.py
│       │   │   ├── activities.py
│       │   │   ├── blockers.py
│       │   │   ├── achievements.py
│       │   │   └── webhooks.py
│       │   └── errors.py            # BOLA-safe exception handlers
│       ├── services/                # business logic (framework-agnostic)
│       │   ├── scheduling.py
│       │   ├── entry_service.py
│       │   ├── summary_service.py
│       │   ├── blocker_service.py
│       │   ├── achievement_service.py
│       │   └── mood_service.py
│       ├── llm/
│       │   ├── client.py            # provider interface + Anthropic impl
│       │   ├── prompts.py           # versioned prompt templates
│       │   └── parsers.py           # structured-output parsing/validation
│       ├── integrations/
│       │   ├── base.py              # ActivitySource protocol
│       │   ├── github.py
│       │   ├── gitlab.py
│       │   ├── jira.py
│       │   └── oauth.py             # OAuth 2.0 flow helpers + token crypto
│       ├── delivery/
│       │   ├── slack/               # Bolt app, Block Kit renderers, modals
│       │   ├── teams/               # Bot Framework, Adaptive Card renderers
│       │   └── email/               # SMTP sender + Jinja2 templates
│       ├── workers/
│       │   ├── celery_app.py
│       │   ├── beat_schedule.py
│       │   └── tasks/               # enqueue prompts, poll activity, run AI
│       ├── mcp/
│       │   └── server.py            # MCP server exposing journal/summaries
│       └── ics/
│           └── export.py            # RFC 5545 schedule export
├── web/                             # Next.js frontend
│   ├── package.json
│   └── src/app/...
└── tests/
    ├── conftest.py                  # fixtures: db, client, factory builders
    ├── fixtures/                    # committed sample payloads (gh, jira, slack)
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure is grouped by concern (api / services / integrations / delivery / workers), not by phase, so every later phase adds files within these directories without restructuring.

---

## Phase 1: Foundation & Tenancy Model

### Purpose
Establish the project skeleton, configuration, database connectivity, the multi-tenant core (workspaces, users, teams, memberships), and OIDC authentication. After this phase a developer can create a workspace, authenticate, and manage team membership through a documented OpenAPI surface. Nothing user-facing about standups exists yet, but every later phase depends on the tenancy + auth primitives built here.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: Bootstrap the repo with FastAPI app factory, Pydantic settings, ruff/mypy/pre-commit, Docker, and docker-compose (api, postgres, redis).

**Design**:
```python
# config.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="WJ_")
    database_url: PostgresDsn
    redis_url: RedisUrl = "redis://localhost:6379/0"
    secret_key: SecretStr                      # session + state signing
    fernet_key: SecretStr                       # integration-token encryption
    anthropic_api_key: SecretStr | None = None
    oidc_google_client_id: str | None = None
    oidc_google_client_secret: SecretStr | None = None
    oidc_microsoft_tenant: str | None = None
    smtp_host: str | None = None
    environment: Literal["dev", "test", "prod"] = "dev"
    log_level: str = "INFO"

@lru_cache
def get_settings() -> Settings: ...
```
```python
# main.py
def create_app() -> FastAPI:
    app = FastAPI(title="Work Journal API", version="0.1.0",
                  openapi_version="3.1.0")
    register_error_handlers(app)
    mount_routers(app)          # /healthz returns {"status":"ok"}
    return app
```
- `.env.example` lists every `WJ_*` var. `docker-compose.yml` defines `postgres:16`, `redis:7`, and `api` with healthchecks.
- `pyproject.toml` pins ruff, mypy (strict), pytest, sqlalchemy[asyncio], asyncpg, alembic, pydantic-settings.

**Testing**:
- `Unit: get_settings() with full env → Settings populated, cached (identity equal on second call).`
- `Unit: missing WJ_DATABASE_URL → ValidationError naming database_url.`
- `Integration: GET /healthz → 200 {"status":"ok"}.`
- `E2E: docker compose up → api container healthcheck passes within 30s.`

#### 1.2 — Database layer & base migration

**What**: Async SQLAlchemy engine/session, declarative `Base`, and the first Alembic migration creating `workspaces`, `users`, `teams`, `team_members` exactly as specified in data-model-suggestion-1.

**Design**:
- `db/base.py`: `async_engine`, `async_session_factory`, `get_session()` dependency yielding a session with commit/rollback.
- Models mirror the DDL (UUID PKs `gen_random_uuid()`, `TIMESTAMPTZ`, role CHECK constraints, `users.email UNIQUE`, `idx_users_workspace`, `teams (workspace_id, slug) UNIQUE`).
- `users.role IN ('admin','manager','member')`; `team_members.role IN ('manager','member')`.
- Migration `0001_tenancy` is reversible (downgrade drops in FK order).

**Testing** (real Postgres via testcontainers):
- `Integration: alembic upgrade head then downgrade base → no error, schema empty after.`
- `Integration: insert user with duplicate email → IntegrityError.`
- `Integration: insert user role='owner' → CheckViolation.`
- `Integration: delete workspace → cascades to users, teams, team_members.`

#### 1.3 — OIDC authentication & session

**What**: Login via Google and Microsoft Entra (OIDC Core 1.0), issuing an httpOnly session JWT; `get_current_user` dependency.

**Design**:
- Endpoints: `GET /auth/login/{provider}` → redirect with PKCE+state; `GET /auth/callback/{provider}` → verify, upsert user into the workspace inferred from invite/domain, set session cookie, `302 /`.
- `GET /auth/me` → `UserOut`. `POST /auth/logout` → clears cookie.
- Session JWT signed with `secret_key`, 12h expiry, claims `{sub: user_id, ws: workspace_id, role}`.
- `deps.get_current_user(session)` decodes cookie → `CurrentUser`; raises 401 on missing/expired.

**Testing**:
- `Unit: valid JWT → CurrentUser with sub/ws/role.`
- `Unit: expired JWT → 401.`
- `Integration (mocked OIDC discovery+token): callback with valid code → user upserted, Set-Cookie present.`
- `Integration: callback with mismatched state → 400, no user created.`

#### 1.4 — Workspace, team, and membership API

**What**: CRUD for workspaces/teams and member management, with role-based authorization.

**Design**:
- `POST /workspaces` (creates ws + makes caller admin); `GET /workspaces/me`.
- `POST /workspaces/{id}/teams`, `GET /teams/{id}`, `PATCH /teams/{id}`.
- `POST /teams/{id}/members {user_id, role}`, `DELETE /teams/{id}/members/{user_id}`.
- Authorization rule: only `admin`/team `manager` may mutate teams/members. All queries scoped to `current_user.workspace_id` (OWASP BOLA mitigation — never trust path IDs alone).
- Pydantic schemas: `WorkspaceOut`, `TeamCreate/Out`, `TeamMemberCreate`.

**Testing**:
- `Integration: member POSTs team member → 403.`
- `Integration: manager adds member → 201, row in team_members.`
- `Integration: user from workspace A GETs team in workspace B → 404 (not 403, to avoid leaking existence).`
- `Unit: TeamCreate with blank name → ValidationError.`

---

## Phase 2: Schedules, Questions & Standup Entry Pipeline

### Purpose
Build the heart of the product's data flow: teams define standup schedules with questions, and members submit responses creating entries and per-question responses. This is the schedule→prompt→response pipeline enforced at the database level. After this phase, standups can be created and answered via the web/API (chat delivery comes later), and the personal journal has data to index.

### Tasks

#### 2.1 — Schedules & questions

**What**: Migration + API for `standup_schedules` and `standup_questions`.

**Design**:
- Migration `0002_schedules` creates both tables per the DDL (`cadence IN ('daily','weekdays','weekly','custom')`, `days_of_week INT[]`, `prompt_time TIME`, `use_member_timezone`, `delivery_channels TEXT[]`, `delivery_config JSONB`; questions with `question_type IN ('text','blockers','mood','goals','custom')`, `sort_order`).
- On schedule creation, seed three default questions following the Scrum Guide 2020 pattern (type `text`, `text`, `blockers`).
- API: `POST /teams/{id}/schedules`, `GET/PATCH /schedules/{id}`, `PUT /schedules/{id}/questions` (replace set, preserving historical responses by not deleting old question rows still referenced).
- `ScheduleCreate` validates `prompt_time` HH:MM and `days_of_week ⊆ {0..6}`.

**Testing**:
- `Unit: ScheduleCreate cadence='hourly' → ValidationError.`
- `Integration: create schedule → 3 default questions seeded in sort_order 0,1,2.`
- `Integration: PUT questions replacing set → new rows added, old rows retained if referenced by responses.`

#### 2.2 — Standup entries & responses

**What**: Migration + service/API to create an entry per (schedule, user, date) and store responses per question.

**Design**:
- Migration `0003_entries` creates `standup_entries` (`status IN ('pending','submitted','skipped')`, `submitted_via IN ('slack','teams','email','web','api')`, UNIQUE `(schedule_id,user_id,entry_date)`) and `standup_responses` (FK to entry+question, GIN full-text index on `response_text`).
- `entry_service.submit(schedule_id, user_id, date, answers, via)`:
  1. upsert entry (create `pending` if absent),
  2. validate every required question answered,
  3. write `standup_responses`, set `status='submitted'`, `submitted_at=now()`, `submitted_via=via`,
  4. return assembled `EntryOut`.
- API: `POST /schedules/{id}/entries` (submit), `GET /schedules/{id}/entries?date=`, `GET /entries/{id}`. Submitting twice the same day updates responses idempotently.
- Authorization: a user may only submit their own entry; managers may read team entries.

**Testing**:
- `Integration: submit with all required answers → status submitted, responses persisted.`
- `Integration: submit missing required question → 422, entry remains pending.`
- `Integration: submit same (schedule,user,date) twice → single entry row, responses updated.`
- `Integration: full-text query response_text via to_tsvector → matching row returned.`

#### 2.3 — Personal work journal & full-text search

**What**: Read API aggregating a user's own entries/responses over time with full-text search and pagination.

**Design**:
- `GET /journal?from=&to=&q=&cursor=` → paginated `JournalEntryOut[]` (date, schedule name, answers, mood if present, linked activities later).
- Full-text: when `q` present, filter via `to_tsvector('english', response_text) @@ plainto_tsquery(q)`.
- Pagination uses RFC 8288 `Link` headers (`rel="next"`) with opaque keyset cursor on `(entry_date, id)`.
- BOLA: journal endpoint always filters `user_id = current_user.id` — no path-supplied user id.

**Testing**:
- `Integration: journal for user with 40 entries, page size 20 → 20 items + Link rel=next.`
- `Integration: journal?q=deployment → only entries whose responses match.`
- `Integration: user A requests journal → never returns user B rows even with crafted cursor.`

---

## Phase 3: AI Daily Summaries

### Purpose
Deliver the first AI-native differentiator: synthesising individual standup responses into one coherent team narrative, persisted as `daily_summaries`. This is the category's "standard expectation" feature and the foundation for weekly rollups and blocker detection. After this phase the system can generate and store a leadership-readable narrative for any team-day.

### Tasks

#### 3.1 — LLM provider abstraction & prompt library

**What**: A provider-agnostic LLM client with versioned prompts and structured-output parsing.

**Design**:
```python
# llm/client.py
class LLMClient(Protocol):
    async def complete(self, *, system: str, user: str,
                       max_tokens: int, response_schema: type[BaseModel] | None
                       ) -> LLMResult: ...

class AnthropicClient:  # uses prompt caching on the system block
    model = "claude-…"   # configurable via WJ_LLM_MODEL
```
- `LLMResult` carries `text`, `parsed` (validated Pydantic), `model_version`, `usage`.
- `prompts.py` exposes `DAILY_SUMMARY_V1` with `version` string stored on outputs (`model_version`). System prompt instructs: produce a concise team narrative, group by themes, list distinct blockers, never invent facts.
- `parsers.py` validates structured JSON output (`{narrative, blockers:[...], highlights:[...]}`) and retries once with a repair prompt on schema failure.

**Testing**:
- `Unit (mocked transport): complete() returns parsed model matching schema.`
- `Unit: malformed JSON then valid on retry → parsed result, two transport calls.`
- `Unit: prompt template renders with team name/date/responses, no unfilled placeholders.`

#### 3.2 — Daily summary generation & storage

**What**: Service + migration generating and persisting a team daily summary.

**Design**:
- Migration `0004_summaries` creates `daily_summaries` (UNIQUE `(schedule_id,summary_date)`, `participation_count`, `total_members`, `blockers_detected`, `delivered_to TEXT[]`, `ai_narrative`, `model_version`).
- `summary_service.generate(schedule_id, date)`:
  1. load submitted entries+responses (the data-model "full standup view" query),
  2. build prompt input, call `LLMClient`,
  3. upsert `daily_summaries` with narrative, counts, `model_version`,
  4. return `DailySummaryOut`.
- API: `POST /schedules/{id}/summaries {date}` (idempotent regenerate), `GET /schedules/{id}/summaries?date=`.

**Testing**:
- `Integration (mocked LLM): 4 submitted entries → summary row with participation_count=4, narrative stored.`
- `Integration: zero submissions → summary with participation_count=0, generic "no updates" narrative, no LLM call.`
- `Integration: regenerate same day → single row updated, not duplicated.`

---

## Phase 4: Scheduling Engine & Reminders

### Purpose
Make standups happen automatically. Celery Beat fires schedule checks; the worker materialises `pending` entries, sends reminders in each member's timezone, and triggers summary generation at end-of-window. After this phase the platform operates the daily cycle without manual API calls — the operational backbone for all delivery channels.

### Tasks

#### 4.1 — Celery + Beat infrastructure

**What**: Wire Celery app, Redis broker, and a Beat tick that evaluates active schedules every 5 minutes.

**Design**:
- `workers/celery_app.py` configures broker/result backend from `redis_url`; `beat_schedule.py` registers `tick_schedules` every 5 min.
- `tasks/scheduling.py:tick_schedules()` finds active schedules whose `prompt_time` (resolved per member timezone when `use_member_timezone`) falls in the current window and have no entry for today → enqueue `open_standup(schedule_id, user_id, date)`.
- docker-compose gains `worker` and `beat` services.

**Testing**:
- `Unit: tick at 09:02 UTC, schedule 09:00 daily, member tz UTC → user selected.`
- `Unit: member tz America/New_York, prompt 09:00 local → fires at 13:00/14:00 UTC per DST.`
- `Unit: schedule cadence weekdays, date=Saturday → no users selected.`
- `Integration (eager mode): tick → open_standup enqueued once per member, idempotent on re-tick.`

#### 4.2 — Entry materialisation & reminders

**What**: Create `pending` entries on open, and send reminder + closing tasks.

**Design**:
- `open_standup`: create `pending` entry (idempotent on UNIQUE), enqueue `remind(entry_id)` at `prompt_time + reminder_minutes`, enqueue `close_standup(schedule_id, date)` at end-of-day local.
- `remind`: if entry still `pending`, dispatch reminder via configured channels (channel adapters from Phases 5/7; in Phase 4 a no-op logger adapter is acceptable behind the `Notifier` interface).
- `close_standup`: mark unsubmitted entries `skipped`, then enqueue `generate_summary(schedule_id, date)` (Phase 3 service).

**Testing**:
- `Integration (eager): open_standup → pending entry; second call → no duplicate.`
- `Integration: remind after submission → no notification sent.`
- `Integration: close_standup → pending entries become skipped, generate_summary enqueued.`

---

## Phase 5: Slack Delivery & Response Collection

### Purpose
Ship the primary chat integration (MVP requires at least one). Slack Bolt collects responses via modals (Block Kit) and posts AI summaries to channels, removing form-filling friction. After this phase a Slack team can run a complete async standup without touching the web app.

### Tasks

#### 5.1 — Slack OAuth & app install

**What**: Slack OAuth 2.0 install flow storing bot tokens encrypted.

**Design**:
- `GET /integrations/slack/install` → Slack OAuth (scopes: `chat:write`, `commands`, `users:read`, `im:write`).
- `GET /integrations/slack/callback` → exchange code, encrypt bot token with Fernet, store on a `slack_installations` row keyed by workspace (added via migration `0005_slack`).
- Map Slack user IDs ↔ platform users by email.

**Testing**:
- `Integration (mocked Slack OAuth): callback → token stored encrypted (ciphertext ≠ plaintext), workspace linked.`
- `Unit: Fernet round-trip encrypt/decrypt token.`

#### 5.2 — Prompt rendering, modal collection, summary posting

**What**: Block Kit prompt DM, modal submission → `entry_service.submit(via='slack')`, and channel summary posts.

**Design**:
- `delivery/slack/renderers.py`: `render_prompt(schedule, questions)` → Block Kit with a "Submit standup" button opening a modal (one input block per question; `mood` type → 1–5 static_select).
- Bolt handlers: `view_submission` → parse to `answers`, call `entry_service.submit`, ack with confirmation.
- `render_summary(daily_summary)` → Block Kit (uses Section + new Card blocks); posted to `delivery_config.slack.channel_id`, threading replies if configured; append channel to `daily_summaries.delivered_to`.
- Implements the `Notifier` interface so Phase 4 `remind`/`open_standup` dispatch through it.
- Bolt runs in Socket Mode for dev, Events API (request URL with signature verification) in prod.

**Testing**:
- `Unit: render_prompt with 3 questions → ≤50 blocks, one input per question.`
- `Integration (mocked Slack signature): valid view_submission → entry submitted via='slack'.`
- `Integration: invalid Slack signature → 401, no entry written.`
- `Integration (mocked chat.postMessage): generate+deliver summary → delivered_to contains channel_id.`

---

## Phase 6: Activity Integrations & Auto-Generated Updates

### Purpose
Deliver the headline AI-native advantage from the README: zero-input updates synthesised from real tool activity. Connect GitHub, GitLab, and Jira via OAuth, ingest activities, and pre-fill standup drafts from a user's day of activity. After this phase developers get auto-drafted updates and managers see objective activity alongside self-reports.

### Tasks

#### 6.1 — Activity source connection & token storage

**What**: OAuth 2.0 connection for GitHub/GitLab/Jira and per-user external-username linking.

**Design**:
- Migration `0006_activity` creates `activity_sources` (`provider IN ('github','gitlab','jira','linear','bitbucket')`, `credentials_enc`, `config JSONB`, UNIQUE `(workspace_id,provider)`), `user_activity_links` (UNIQUE `(source_id,external_username)`), `activities` (typed `activity_type`, UNIQUE `(source_id,external_id)`, indexes on `(user_id,occurred_at)` and `(user_id,(occurred_at::DATE))`).
- API: `POST /integrations/{provider}/connect` (OAuth start), callback stores encrypted creds; `POST /sources/{id}/links {user_id, external_username}`.

**Testing**:
- `Integration (mocked OAuth): connect github → activity_sources row, creds encrypted.`
- `Integration: link same external_username twice on a source → IntegrityError.`

#### 6.2 — Ingestion adapters & polling

**What**: `ActivitySource` adapters that fetch and normalise recent activity into `activities`.

**Design**:
```python
# integrations/base.py
class ActivitySource(Protocol):
    provider: str
    async def fetch(self, link: UserActivityLink, since: datetime
                    ) -> list[NormalizedActivity]: ...

@dataclass
class NormalizedActivity:
    activity_type: str        # maps to activities CHECK set
    title: str
    description: str | None
    external_id: str
    external_url: str | None
    repo_name: str | None
    occurred_at: datetime
```
- `github.py` (PyGithub/httpx): commits, PRs opened/merged/reviewed, issues → map to `activity_type`. `gitlab.py`: commits, MRs, issues. `jira.py`: issue transitions/comments via ADF text extraction.
- Celery `poll_activity(source_id)` (Beat: every 15 min) fans out per link, upserts on `(source_id,external_id)` (dedup), updates `last_synced_at`.

**Testing** (fixture-based using committed sample API payloads):
- `Unit: github commit payload → NormalizedActivity type='commit', correct url/occurred_at.`
- `Unit: github merged PR → 'pull_request_merged'.`
- `Integration: poll twice with same payloads → no duplicate activities (dedup).`
- `Unit: jira ADF description → plain text extracted.`

#### 6.3 — Auto-drafted standup updates

**What**: Generate a suggested draft answer from a user's activities for a date, surfaced in the modal/web entry.

**Design**:
- `GET /schedules/{id}/draft?date=` → for `current_user`, load that date's activities (data-model "auto-generated update" query), call LLM with `AUTODRAFT_V1` prompt → `{suggested_answers: {question_id: text}}`.
- Prompt instructs grouping commits/PRs/issues into prose, grounded only in supplied activities (no fabrication). User edits before submitting.

**Testing**:
- `Integration (mocked LLM): user with 5 activities → draft references those titles only.`
- `Integration: user with zero activities → empty draft, no LLM call.`

---

## Phase 7: Microsoft Teams & Email Delivery

### Purpose
Complete the MVP delivery surface: dual-platform chat (Slack + Teams simultaneously) and first-class email digests for non-chat users. After this phase any team member can receive prompts and summaries on their channel of choice. Can be developed in parallel with Phase 6 (both depend only on Phases 4–5 interfaces).

### Tasks

#### 7.1 — Email delivery channel

**What**: SMTP-based prompt reminders and summary digests via Jinja2 templates.

**Design**:
- `delivery/email/sender.py` implements `Notifier`: `send_prompt`, `send_summary` rendering responsive HTML+text templates.
- Reply-to-submit not supported in MVP; emails contain a magic-link to the web entry form (signed, single-use, 24h).
- Subject from `delivery_config.email.subject_template`.

**Testing**:
- `Unit: render summary template with narrative+stats → HTML contains narrative, no unrendered Jinja.`
- `Integration (mocked SMTP): send_summary → message sent to each member, delivered_to includes 'email'.`
- `Integration: magic-link token valid once → second use 410 Gone.`

#### 7.2 — Microsoft Teams delivery & collection

**What**: Bot Framework app rendering Adaptive Cards v1.5 for prompts and summaries, collecting responses.

**Design**:
- `delivery/teams/` Bot Framework handler; `renderers.py` builds Adaptive Card v1.5 (Input.Text per question, Input.ChoiceSet for mood) and an Action.Submit.
- OAuth via Azure AD (Entra); install flow stores bot credentials (migration `0007_teams`).
- Submit handler maps card data → `entry_service.submit(via='teams')`; summary card posted to configured Teams channel; append `'teams'` to `delivered_to`.
- Separate rendering layer from Slack per standards.md incompatibility note.

**Testing**:
- `Unit: render prompt card → valid Adaptive Card v1.5 JSON (schema-validated), one input per question.`
- `Integration (mocked Bot connector): card submit → entry submitted via='teams'.`
- `Integration: dual delivery (slack+teams+email configured) → summary delivered to all three, delivered_to has 3 entries.`

---

## Phase 8: Mood Tracking & Team Health

### Purpose
Add the engagement-signal layer: capture per-entry mood and expose trend analytics. This feeds burnout detection (Phase 10) and manager rollups (Phase 9). GDPR-relevant: mood is sensitive personal data and gated behind opt-in consent.

### Tasks

#### 8.1 — Mood capture & consent

**What**: Migration + capture of `mood_entries`, gated by per-user consent.

**Design**:
- Migration `0008_mood` creates `mood_entries` (`score 1..5`, `emoji`, `comment`, UNIQUE `(entry_id)`, index `(user_id, created_at)`).
- Add `users.mood_consent BOOLEAN DEFAULT FALSE` (migration) — GDPR explicit opt-in; mood `question_type` only collected when consent true.
- `mood_service.record(entry_id, score, emoji, comment)` (consent-checked).

**Testing**:
- `Integration: record mood without consent → 403, no row.`
- `Integration: mood question in modal with consent → mood_entries row, score validated 1..5.`
- `Integration: score=6 → 422.`

#### 8.2 — Mood & participation trend analytics

**What**: Endpoints powering trend charts (mood over time, participation rate).

**Design**:
- `GET /teams/{id}/analytics/mood?days=30` → the data-model mood-trend query: `[{date, avg_mood, respondents}]`.
- `GET /teams/{id}/analytics/participation?days=30` → participation-rate query per member.
- Manager/admin only. Frontend renders with Recharts.

**Testing**:
- `Integration: 3 mood entries across 2 dates → correct daily averages.`
- `Integration: member (non-manager) → 403.`
- `Integration: participation rate matches submitted/total computation.`

---

## Phase 9: Blocker Detection, Weekly Rollups & Achievement Synthesis

### Purpose
Deliver the manager-facing and career-growth AI differentiators: proactive blocker detection, weekly executive rollups, and resume-ready achievement logs. Requires entries (Phase 2), summaries (Phase 3), activities (Phase 6), and mood (Phase 8).

### Tasks

#### 9.1 — Blocker tracking & AI detection

**What**: Migration + user-reported and AI-detected blockers with persistence counter.

**Design**:
- Migration `0009_blockers` creates `blockers` (`status IN ('open','resolved','escalated')`, `detected_by IN ('user','ai')`, `days_open`, partial index `WHERE status='open'`).
- On submit, responses to `blockers`-type questions create `blockers` rows (`detected_by='user'`).
- Celery `detect_blockers(team_id, date)`: LLM `BLOCKER_DETECT_V1` over recent entries+stalled activities → create `detected_by='ai'` blockers; nightly task increments `days_open` and sets `escalated` past threshold (default 3 days).
- API: `GET /teams/{id}/blockers?status=open`, `PATCH /blockers/{id} {status}`.

**Testing**:
- `Integration: blockers-question answer → blocker row detected_by='user'.`
- `Integration (mocked LLM): stalled activity pattern → AI blocker created.`
- `Integration: nightly increment → days_open+1; >3 → escalated.`
- `Integration: long-running blocker query returns days_open>3 ordered desc.`

#### 9.2 — Weekly manager rollups

**What**: Migration + AI-synthesised weekly executive summary per team.

**Design**:
- Migration `0010_rollups` creates `weekly_rollups` (`executive_summary`, `highlights[]`, blocker counts, `participation_rate`, `mood_avg`, `mood_trend IN ('up','down','stable')`, UNIQUE `(team_id,week_start)`).
- Celery `generate_weekly_rollup(team_id, week_start)` (Beat: Monday): aggregate daily summaries, participation, mood; LLM `WEEKLY_ROLLUP_V1` → executive narrative + highlights; compute `mood_trend` from week-over-week delta.
- API: `GET /teams/{id}/rollups?week_start=`.

**Testing**:
- `Integration (mocked LLM): a week of summaries → one rollup row, participation_rate computed.`
- `Unit: mood_avg up vs prior week → mood_trend='up'.`
- `Integration: regenerate same week → single row.`

#### 9.3 — Personal achievement synthesis

**What**: Migration + AI synthesis of resume-ready achievements with source traceability.

**Design**:
- Migration `0011_achievements` creates `achievements` (`period_type`, `achievement_statements[]`, `source_entry_ids[]`, `source_activity_ids[]`, `is_reviewed`).
- `achievement_service.synthesize(user_id, period_start, period_end, period_type)`: gather the user's entries+activities, LLM `ACHIEVEMENT_V1` → impact-oriented statements; store source id arrays for verification.
- API: `POST /achievements {period_type, start, end}`, `GET /achievements`, `PATCH /achievements/{id} {is_reviewed, achievement_statements}` (user edits/confirms).
- BOLA: scoped strictly to `current_user.id`.

**Testing**:
- `Integration (mocked LLM): quarter of entries → statements with non-empty source_entry_ids.`
- `Integration: user A cannot GET user B achievements → 404.`
- `Integration: PATCH statements → updated, is_reviewed=true.`

---

## Phase 10: Differentiators — MCP Server, Burnout Detection, Webhooks & iCal

### Purpose
Ship the strategic differentiators that position the project ahead of incumbents: a day-one MCP server, GDPR-gated burnout-risk detection, outbound webhooks (AsyncAPI-documented), and RFC 5545 schedule export. These are additive surfaces over the now-complete core.

### Tasks

#### 10.1 — MCP server

**What**: MCP server exposing journal entries, summaries, and standup history to AI assistants.

**Design**:
- `mcp/server.py` (Python MCP SDK) exposes tools: `get_journal(from,to,query)`, `get_daily_summary(team,date)`, `get_blockers(team,status)`, `get_achievements(period)`.
- Auth via per-user MCP token (issued from settings); every tool scoped to that user's workspace + own private data (journals/achievements never cross-user).

**Testing**:
- `Integration: get_journal tool → returns caller's entries only.`
- `Integration: invalid MCP token → unauthorized, no data.`

#### 10.2 — Burnout / disengagement risk detection

**What**: GDPR-opt-in analysis of language/velocity patterns flagging risk to managers.

**Design**:
- Requires both `mood_consent` and a new `users.ai_analysis_consent`. Celery weekly `assess_risk(team_id)`: features = update length trend, blocker frequency, completion decline, mood slope; LLM `BURNOUT_V1` produces a per-user risk band (`low/medium/high`) with rationale, surfaced only to managers, never storing raw verbatim beyond retention window.
- Configurable retention (`WJ_RISK_RETENTION_DAYS`, default 90) with purge task.

**Testing**:
- `Integration: user without ai_analysis_consent → excluded from assessment.`
- `Integration (mocked LLM): declining-signal fixture → risk band 'high' with rationale.`
- `Integration: purge task removes assessments older than retention window.`

#### 10.3 — Outbound webhooks & iCalendar export

**What**: Signed outbound webhooks for summaries/blockers (AsyncAPI 2.6 documented) and RFC 5545 schedule export.

**Design**:
- `webhooks` config table (migration `0012_webhooks`): target URL, secret, event types. On summary/rollup/blocker events, enqueue `dispatch_webhook` with HMAC-SHA256 signature header; retry with backoff; payload validated against published JSON Schema. `openapi/asyncapi.yaml` documents the channels.
- `ics/export.py`: `GET /schedules/{id}/calendar.ics` → VEVENT recurrence (RRULE) from cadence/days/prompt_time per member timezone.

**Testing**:
- `Integration (mocked endpoint): summary generated → webhook POSTed with valid HMAC signature.`
- `Integration: endpoint 500 → retried per backoff policy.`
- `Unit: schedule weekdays 09:00 → ICS with RRULE FREQ=WEEKLY BYDAY=MO,TU,WE,TH,FR.`

---

## Phase 11: Web Frontend

### Purpose
Provide the mobile-responsive web interface required by the MVP, covering standup entry, personal journal, team summaries, analytics dashboards, blockers, and achievements. Depends on the stable REST API from Phases 1–10 (and can begin against Phase 2–3 endpoints, growing as APIs land).

### Tasks

#### 11.1 — App shell, auth, and standup entry

**What**: Next.js App Router shell with OIDC session, standup entry form (with auto-draft), and journal view.

**Design**:
- Routes: `/login`, `/standup/[scheduleId]` (renders questions, pre-fills from `/draft`, submits to entries API), `/journal` (search + infinite scroll via `Link` cursor).
- shadcn/ui components; mobile-first Tailwind layout; server components for data fetch, client components for forms.

**Testing**:
- `E2E (Playwright, mocked API): submit standup → success toast, entry appears in journal.`
- `E2E: journal search filters list.`
- `Component (Vitest): question renderer shows mood select for mood type.`

#### 11.2 — Summaries, analytics, blockers & achievements dashboards

**What**: Manager dashboards (daily summary, weekly rollup, mood/participation charts, blocker board) and the personal achievements page.

**Design**:
- Routes: `/teams/[id]/dashboard`, `/teams/[id]/blockers`, `/achievements`.
- Recharts for mood/participation trends; blocker board grouped by status; achievements page with inline edit + "mark reviewed".
- Manager-only routes guard on role from session.

**Testing**:
- `E2E (mocked API): dashboard renders narrative + mood chart with data points.`
- `E2E: member navigating to manager dashboard → redirected/forbidden.`
- `E2E: edit achievement statement → PATCH called, UI updates.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Tenancy        ─── required by everything
    │
Phase 2: Schedules & Entry Pipeline  ─── requires Phase 1
    │
    ├── Phase 3: AI Daily Summaries   ─── requires Phase 2
    │       │
    │   Phase 4: Scheduling Engine    ─── requires Phase 2 (+3 for auto-summary)
    │       │
    │   Phase 5: Slack Delivery       ─── requires Phase 4
    │       ├── Phase 6: Activity Integrations ─── requires Phase 2 (parallel w/ 7)
    │       └── Phase 7: Teams + Email Delivery ─── requires Phase 4/5 ifaces (parallel w/ 6)
    │
    Phase 8: Mood & Team Health       ─── requires Phase 2 (analytics enrich after 3)
    │
    Phase 9: Blockers / Rollups / Achievements ─── requires Phases 2,3,6,8
    │
    Phase 10: MCP / Burnout / Webhooks / iCal  ─── requires Phases 8,9
    │
    Phase 11: Web Frontend            ─── consumes Phases 1-10 (can start at Phase 2)
```

**Parallelism opportunities:**
- Phases 6 and 7 can be developed concurrently once Phases 4–5 interfaces exist.
- Phase 8 (mood) can be built in parallel with Phase 6/7 since it only depends on Phase 2.
- Phase 11 (frontend) can begin against Phase 2–3 endpoints and grow incrementally alongside backend phases.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks in the phase are implemented.
2. All unit and mocked-integration tests pass; real-integration tests (testcontainers Postgres/Redis) pass in CI.
3. `ruff check` and `ruff format --check` pass with no violations.
4. `mypy --strict` passes for `src/workjournal`.
5. `docker compose build` succeeds and affected services start with passing healthchecks.
6. The phase's feature works end-to-end (verified by at least one e2e/integration test exercising the full path).
7. New configuration options are added to `.env.example` and documented in the README.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 document; new outbound events appear in `openapi/asyncapi.yaml`.
9. New tables/columns have a reversible Alembic migration (`upgrade` and `downgrade` both tested).
10. Authorization is verified: every new endpoint enforces workspace scoping and role checks (OWASP BOLA), and any new personal/sensitive data path respects GDPR consent gates.
