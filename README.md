# mgr-skills

[Claude Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) for [More Good Reviews](https://moregoodreviews.com). Each skill pairs with the More Good Reviews MCP server to turn review and reputation data into action.

## Skills

- **[reputation-report](reputation-report/SKILL.md)** — Generates an action-first reputation health report over a configurable period (weekly, monthly, quarterly, or yearly): review volume and rating trends, location and source breakdowns, reply coverage, outreach health, root-cause diagnosis of negative trends, and prioritized next steps.

## Requirements

These skills call tools exposed by the More Good Reviews Project API MCP server. Connect that server before using a skill; the skills have no other data source.

## Installing a skill

- **Claude Code** — copy a skill directory into `~/.claude/skills/` (personal) or `.claude/skills/` (project).
- **claude.ai** — zip a skill directory and upload it under Settings > Capabilities.
- **Claude API** — upload via the Skills API (`/v1/skills`).

Each skill is a self-contained directory with a `SKILL.md`; copy or zip the directory, not just the file.
