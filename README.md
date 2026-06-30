# mgr-skills

[Claude Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) for [More Good Reviews](https://moregoodreviews.com). Each skill pairs with the More Good Reviews MCP server to turn review and reputation data into action.

## Skills

| Skill | Use when | What it does |
|-------|----------|--------------|
| [reputation-report](reputation-report/SKILL.md) | Reputation report, review summary, ratings trend, health check | Read-only report over a configurable period (weekly, monthly, quarterly, yearly): review volume, ratings, location and source breakdowns, reply coverage, outreach health, and prioritized next steps |
| [reputation-actions](reputation-actions/SKILL.md) | What to do, how to improve, what to fix, priorities | Ranked to-do list (not a report, sends nothing): judges signals against the project's own history and recommends the next 7 highest-impact actions |
| [review-pipeline](review-pipeline/SKILL.md) | Who to ask for reviews, weekly ask list, send review requests | Builds an eligible-customer queue from a segment (default never asked) and schedules requests via the project's templates after confirmation |

## Requirements

These skills call tools exposed by the More Good Reviews Project API MCP server. Connect that server before using a skill; the skills have no other data source.

## Installing a skill

- **Claude Code** — copy a skill directory into `~/.claude/skills/` (personal) or `.claude/skills/` (project).
- **claude.ai** — zip a skill directory and upload it under Settings > Capabilities.
- **Claude API** — upload via the Skills API (`/v1/skills`).

Each skill is a self-contained directory with a `SKILL.md`; copy or zip the directory, not just the file.
