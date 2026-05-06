# Work Journal & Daily Standup

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native async standup and personal work journal that auto-generates updates from real tool activity and turns daily entries into team narratives and career-grade achievement logs.

Work Journal & Daily Standup is an open-source platform for async daily standups, progress logging, and weekly summaries. It serves distributed engineering teams, individual contributors who want a searchable record of their work, and managers who need visibility without synchronous meetings.

---

## Why Work Journal & Daily Standup?

- Existing standup bots are platform-locked: Standup Bot is Slack-only, Geekbot covers Slack and Teams but lacks git-activity awareness, and most tools ignore non-chat users entirely.
- Pricing for paid tiers runs $2.50–$6/user/month or $25–$100/month flat for small teams, yet AI features beyond basic summarisation remain shallow across the category.
- GitMore proves zero-input updates work for developers, but its developer-only scope leaves managers, designers, and PMs invisible.
- Sunsama is the only major tool treating the standup entry as a personal work journal for career growth, and it offers no team broadcast features.
- The category lacks large venture-backed players (Loom was acquired by Atlassian for $975 M in 2023; Geekbot is bootstrapped), leaving room for a well-positioned open-source entrant.

---

## Key Features

### Async Standups & Team Summaries

- Customisable daily standup prompts (yesterday / today / blockers and team-defined variants)
- Team response aggregation posted to Slack or Microsoft Teams
- Email digests for non-chat users
- AI-generated daily summaries that synthesise individual updates into a team narrative
- Customisable schedules with per-person timezone support

### Personal Work Journal

- Searchable personal log of daily accomplishments
- Historical data storage with full-text search
- Personal achievement synthesis that turns standup entries into resume-ready statements for performance review and career growth

### Auto-Generated Updates from Tool Activity

- Git activity auto-population from GitHub and GitLab (commits, pull requests, issue updates)
- Goal and task integration linking daily updates to project tasks
- Jira issue integration for ticket-driven progress

### Manager Visibility & Team Health

- Manager narrative generation that rolls up team updates into a weekly executive summary
- Automated blocker detection that flags stalled tasks before a manager has to ask
- Mood and sentiment tracking for team-health signals
- Trend visualisation showing blocker patterns and participation over time

### Platform & Access

- Mobile-responsive web interface
- Dual-platform chat support (Slack and Microsoft Teams simultaneously)
- Access control with public, private, and team-specific sharing

---

## AI-Native Advantage

AI is used to generate daily updates automatically from GitHub commits, Jira ticket transitions, and similar tool activity, removing the daily form-filling burden that drives churn in incumbent tools. It detects stalled tasks and surfaces blockers proactively, synthesises five engineers' updates into one coherent leadership-ready paragraph, and transforms standup entries into structured achievement logs for performance reviews. Language-pattern analysis across updates also flags early signs of disengagement or burnout risk for managers.

---

## Tech Stack & Deployment

The platform is expected to deliver via Slack Block Kit and Microsoft Teams Adaptive Cards for native chat integration, with email as a first-class delivery channel. Auto-generated updates rely on the GitHub and GitLab APIs, with Jira for issue correlation. Webhooks (HTTP POST) push summaries to external trackers and HR tools. The standup format is grounded in the Scrum Daily Scrum questions, and iCalendar (.ics) integration is relevant for schedule syncing. A private self-hosted deployment option is in scope alongside the cloud offering.

---

## Market Context

The async standup and team communication niche sits within a collaboration software sector that exceeded $30 billion globally in 2025, with dedicated standup tools individually operating in the $1–20 M ARR range and the segment still pre-consolidation. Paid plans for teams of 10–50 typically run $25–$100/month flat or $2–$6/user/month, while individual productivity journaling tools cluster at $10–$20/month. Primary buyers are engineering managers at distributed companies, remote-first teams across timezones, individual contributors building a performance-review evidence trail, and team leads who want automated weekly summaries.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
