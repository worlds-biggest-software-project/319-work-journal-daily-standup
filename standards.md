# Standards & API Reference

> Project: Work Journal & Daily Standup · Generated: 2026-05-03

## Industry Standards & Specifications

### Agile Process Standards

- **Scrum Guide (2020 edition)**
  - URL: https://scrumguides.org/scrum-guide.html
  - The canonical reference for the Daily Scrum ceremony. The 2020 edition removed the prescribed three-question format in favour of flexible team-driven daily planning, but the original pattern (what did I do, what will I do, what are my blockers) remains the de-facto template embedded in every async standup tool.

- **ISO/IEC 12207:2017 — Systems and Software Engineering: Software Life Cycle Processes**
  - URL: https://www.iso.org/standard/63712.html
  - Defines software lifecycle processes including project tracking and status reporting, which inform the data models and reporting cadences used in work journal and standup tools.

### Calendaring & Scheduling Standards

- **RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
  - URL: https://www.rfc-editor.org/rfc/rfc5545
  - Defines the `.ics` format for representing events, to-dos, and journal entries. Relevant for scheduling daily standup prompts, integrating with calendar systems (Google Calendar, Outlook), and exporting standup schedules as recurring calendar events.

- **RFC 5546 — iCalendar Transport-Independent Interoperability Protocol (iTIP)**
  - URL: https://www.rfc-editor.org/rfc/rfc5546
  - Defines the interoperability protocol for scheduling operations (requesting, replying, modifying, cancelling meetings) built on top of RFC 5545. Relevant when a work journal tool needs to propose or confirm standup time slots.

### W3C & IETF Web Standards

- **RFC 6749 — The OAuth 2.0 Authorization Framework**
  - URL: https://datatracker.ietf.org/doc/html/rfc6749
  - The standard authorization framework for third-party integrations. Required for connecting to Slack, GitHub, GitLab, Jira, Notion, and other data sources without storing user passwords. All production integrations should use OAuth 2.0 rather than static API keys.

- **OpenID Connect Core 1.0**
  - URL: https://openid.net/specs/openid-connect-core-1_0.html
  - Identity layer built on top of OAuth 2.0. Required for user authentication in multi-tenant SaaS deployments, federated login (Google, Microsoft), and enterprise SSO integration.

- **RFC 7617 — HTTP Basic Authentication**
  - URL: https://datatracker.ietf.org/doc/html/rfc7617
  - Widely used for initial developer API key authentication (as in the Status Hero API). Considered acceptable only over HTTPS for internal tooling; OAuth 2.0 is preferred for user-facing integrations.

- **RFC 8288 — Web Linking**
  - URL: https://datatracker.ietf.org/doc/html/rfc8288
  - Defines the `Link` header for pagination, relevant for REST APIs returning paginated lists of standup reports and journal entries.

### Data Model & API Specifications

- **OpenAPI Specification 3.1 (OAS 3.1)**
  - URL: https://spec.openapis.org/oas/v3.1.0.html
  - The industry standard for describing REST APIs in a machine-readable YAML or JSON document. OAS 3.1 achieves full JSON Schema alignment, eliminating schema discrepancies. A work journal API should publish an OAS 3.1 document to enable automatic client generation, interactive documentation, and contract testing.

- **JSON Schema (Draft 2020-12)**
  - URL: https://json-schema.org/specification
  - Used by OpenAPI 3.1 for validating request and response payloads. Relevant for defining standup entry schemas, journal entry formats, and webhook payload structures.

- **AsyncAPI 2.x**
  - URL: https://www.asyncapi.com/docs/reference/specification/v2.6.0
  - Specification for event-driven and messaging APIs, analogous to OpenAPI but for webhooks and pub/sub streams. Relevant for documenting the outbound webhook payloads that push standup summaries to external systems.

### Messaging Platform Standards

- **Slack Block Kit**
  - URL: https://docs.slack.dev/block-kit/
  - Slack's proprietary JSON-based UI framework for building rich interactive messages within Slack. Used to render standup prompts, collect responses via modal forms, and publish formatted daily summaries to channels. As of April 2026, new blocks include Alert, Card, and Carousel types. Up to 50 blocks are supported per message.

- **Microsoft Adaptive Cards (v1.5)**
  - URL: https://adaptivecards.microsoft.com/
  - Cross-platform JSON UI framework used in Microsoft Teams, Outlook, and Copilot. The Teams platform supports Adaptive Cards up to v1.5 for bot-sent cards. Used to embed standup prompts and collect responses natively within Teams without leaving the platform.

### Observability & Activity Standards

- **OpenTelemetry Semantic Conventions (v1.41.0)**
  - URL: https://opentelemetry.io/docs/specs/semconv/
  - Defines standardized attribute names for logs, traces, and metrics. Relevant for correlating developer activity events (deployments, test runs, CI pipelines) with daily work journal entries to provide auto-populated context without manual input.

### Security Standards

- **OWASP API Security Top 10 (2023)**
  - URL: https://owasp.org/www-project-api-security/
  - Key risks for work journal APIs include Broken Object Level Authorization (BOLA — users reading other users' private journals), Excessive Data Exposure (returning full standup history in list endpoints), and Broken Authentication. Mitigation patterns for each should be incorporated in API design.

- **GDPR (EU) 2016/679**
  - URL: https://gdpr-info.eu/
  - Daily standup and work journal data constitutes personal data under GDPR. Compliance requires documented data retention policies, right-to-erasure endpoints, and explicit consent for AI analysis of standup language and mood signals. Particularly relevant for burnout/disengagement detection features.

### Model Context Protocol

- **Model Context Protocol (MCP)**
  - URL: https://modelcontextprotocol.io/docs/
  - Anthropic's open protocol for connecting LLM clients (Claude, Cursor, Windsurf) to external data sources and tools. Geekbot has already published an MCP server (`geekbot-com/geekbot-mcp`, MIT licence) that exposes standup reports to AI agents via natural language queries. A work journal tool should publish an MCP server exposing journal entries, summaries, and standup history to AI assistants.

---

## Similar Products — Developer Documentation & APIs

### Geekbot

- **Description:** Async standup and retrospective bot for Slack and Microsoft Teams. The market leader with 200,000+ users. Supports AI topic and sentiment analysis, Sankey/Gantt visualisations, and the only standup tool with a published MCP server.
- **API Documentation:** https://geekbot.com/developers (REST API with OpenAPI/Swagger; requires Geekbot subscription)
- **MCP Server:** https://github.com/geekbot-com/geekbot-mcp (open-source, MIT licence)
- **Webhook Documentation:** https://app.geekbot.com/dashboard/api-webhooks
- **Python SDK (community):** https://github.com/andrewthetechie/geekbot-api-py
- **Standards:** REST/JSON; OpenAPI/Swagger spec; webhook push notifications
- **Authentication:** API Key (Bearer token)

### Status Hero

- **Description:** Daily check-in tool with goal tracking, mood tracking, and minimalist UX. $3/user/month. Targets small-to-medium engineering and product teams.
- **API Documentation:** https://api.statushero.com/ (REST API v1, GitHub: https://github.com/statushero/api-documentation)
- **Developer Guide:** https://help.statushero.com/integrations-and-api
- **Standards:** REST/JSON over HTTPS; versioned endpoints (`/api/v1/`)
- **Authentication:** Custom header authentication — `X-TEAM-ID` and `X-API-KEY` headers; rate limit of 1 request/second

### GitHub REST API

- **Description:** Provides programmatic access to commits, pull requests, issues, and repository activity — the primary data source for auto-generating developer standup updates without manual input.
- **API Documentation:** https://docs.github.com/en/rest
- **Commits Endpoint:** https://docs.github.com/en/rest/commits/commits
- **Pull Requests Endpoint:** https://docs.github.com/en/rest/pulls/pulls
- **Issues Endpoint:** https://docs.github.com/en/rest/issues/issues
- **Standards:** REST/JSON; OpenAPI 3.0; versioned via `X-GitHub-Api-Version` header (current: `2026-03-10`)
- **Authentication:** OAuth 2.0 (GitHub App or OAuth App); fine-grained personal access tokens; GHES supports basic auth for server-to-server

### GitLab REST API

- **Description:** Provides access to commits, merge requests, issues, and repository activity for GitLab-hosted projects. Important for teams not using GitHub.
- **API Documentation:** https://docs.gitlab.com/api/rest/
- **Commits API:** https://docs.gitlab.com/api/commits/
- **Issues API:** https://docs.gitlab.com/api/issues/
- **Merge Requests API:** https://docs.gitlab.com/api/merge_requests/
- **Standards:** REST/JSON; OpenAPI 3.0
- **Authentication:** OAuth 2.0; personal access tokens; project access tokens

### Jira Cloud REST API

- **Description:** Atlassian's issue tracking and project management API. Used to read issue status, transitions, and comments to auto-populate standup updates for non-developer roles working in Jira.
- **API Documentation:** https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- **Jira Software API:** https://developer.atlassian.com/cloud/jira/software/rest/
- **Developer Guide:** https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/
- **Standards:** REST/JSON; OpenAPI 3.0; uses Atlassian Document Format (ADF) for rich text fields
- **Authentication:** OAuth 2.0 (3-LO); API tokens for personal use; webhooks for push events

### Linear GraphQL API

- **Description:** Issue tracking tool popular with modern engineering teams, built entirely on GraphQL. Supports full read/write access to issues, projects, and cycles, making it suitable for auto-generating developer standup updates.
- **API Documentation:** https://developers.linear.app/docs/graphql/working-with-the-graphql-api
- **Developer Portal:** https://linear.app/developers
- **SDKs:** TypeScript/JavaScript SDK at https://github.com/linear/linear
- **Standards:** GraphQL; introspectable schema; webhook support for real-time push notifications
- **Authentication:** OAuth 2.0; personal API keys

### Notion REST API

- **Description:** Flexible workspace tool used by many teams as a knowledge base and project tracker. Reading Notion page and database activity can populate standup context for knowledge workers.
- **API Documentation:** https://developers.notion.com/
- **Authorization Guide:** https://developers.notion.com/docs/authorization
- **Python SDK (community):** https://github.com/ramnes/notion-sdk-py
- **Standards:** REST/JSON; supports filtering, sorting, and pagination of database records
- **Authentication:** OAuth 2.0 (for public integrations); internal integration API tokens (for single-workspace tools)

### Loom REST API

- **Description:** Async video messaging platform (Atlassian). API provides access to recordings, transcriptions, and metadata — useful for integrating video standup clips alongside text-based journal entries.
- **API Documentation:** https://dev.loom.com/docs/record-sdk/getting-started
- **Record SDK:** https://dev.loom.com/docs/record-sdk/details/api
- **Standards:** REST/JSON (`api.loom.com/api/v1`); JavaScript Record SDK for embedding in-browser recording
- **Authentication:** OAuth 2.0; SDK application tokens
- **Note:** Loom does not expose a fully public REST API; capabilities are limited to the Record SDK and specific recording management endpoints.

### Slack API

- **Description:** Messaging platform API used by the majority of existing standup tools. Provides bot messaging, interactive components (Block Kit), event subscriptions, and OAuth integration.
- **API Documentation:** https://docs.slack.dev/ (new), https://api.slack.com/ (legacy, maintained during transition)
- **Block Kit Reference:** https://docs.slack.dev/reference/block-kit/
- **SDKs:** Bolt for JavaScript, Bolt for Python, Bolt for Java — https://api.slack.com/tools/bolt
- **Standards:** REST/JSON; Block Kit JSON UI format; Events API using webhook subscriptions; Socket Mode for development
- **Authentication:** OAuth 2.0 (Slack Apps); bot tokens; user tokens

### Microsoft Teams (Bot Framework / Adaptive Cards)

- **Description:** Microsoft's collaboration platform used by enterprise teams. Integration requires either a Teams App via the Bot Framework or Power Automate connectors. Adaptive Cards are the UI primitive for structured interactive messages.
- **API Documentation:** https://learn.microsoft.com/en-us/microsoftteams/platform/
- **Adaptive Cards Reference:** https://adaptivecards.microsoft.com/
- **Teams SDK:** https://microsoft.github.io/teams-sdk/
- **Standards:** Adaptive Cards JSON schema (v1.5 supported in Teams); Bot Framework protocol; REST/JSON
- **Authentication:** OAuth 2.0 via Azure AD (Microsoft Entra ID); Teams App manifest for distribution

---

## Notes

- **Slack vs. Teams fragmentation:** Slack (Block Kit) and Teams (Adaptive Cards) use incompatible UI component models. Any tool targeting both platforms must maintain separate rendering layers. This is a known pain point and a driver of the market trend towards platform-agnostic web UIs with optional chat delivery.
- **No open standup data standard:** There is no ISO, IETF, or W3C standard defining a common data model for standup entries or work journal records. Each tool defines its own schema, creating integration friction. An open-source tool could define and publish a JSON Schema for standup/journal data that other tools could adopt.
- **MCP as emerging integration layer:** Geekbot's MCP server (MIT licence) suggests that MCP is becoming a relevant integration mechanism for AI-first standup tools. Building an MCP server from day one would differentiate the project and enable zero-friction integration with AI development assistants.
- **GDPR for AI features:** The burnout detection, mood analysis, and career growth features identified in the feature research involve processing personal behavioural data. These features require explicit consent flows and documented data retention limits to comply with GDPR, and should be architect as opt-in.
