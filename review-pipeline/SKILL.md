---
name: review-pipeline
description: Finds customers eligible for a review request in a More Good Reviews project and schedules the requests after confirmation. Builds a ranked queue from a customer segment (default never asked), then sends via createAsk on approval. This SENDS outreach; it is not a report or a to-do list. Use when the user asks who to ask for reviews, wants a weekly ask list, or asks to send review requests to eligible customers.
---

# Review Pipeline

Turn a customer segment into scheduled review requests for one More Good Reviews project: build the queue, confirm, send.

**Prerequisite:** Requires a More Good Reviews Project API MCP server connected. This skill calls its tools and has no other data source. If no such server is connected, say so and stop.

## Core principle: show the list, then ask before sending

- Find eligible customers with `listCustomers`; do not over-fetch per-customer history.
- The `not-asked` filter already excludes unsubscribed and archived customers from outreach eligibility — still confirm before sending.
- Never call `createAsk` without explicit user approval of the final list.
- Requests send through the **project's configured strategy and templates** (subject, copy, timing rules). This skill chooses *who* and *which channel*; it does not write custom message text.

## Step 1: Select the project

Each More Good Reviews Project API MCP connection is scoped to one project. A user may have several connected at once.

- If exactly one is connected, use it without prompting.
- If more than one is connected, name them and ask which to use.
- Use one project's tools consistently for the whole run.

## Step 2: Resolve the segment and window

Defaults when the user gives nothing:

| Input | Default |
|-------|---------|
| `filter` | `not-asked` |
| `days` | `30` (signup lookback) |
| `date_key` | `signed_up_at` |
| `sort_key` / `sort_dir` | `signed_up_at` / `desc` (newest first) |
| `max_results` | `50` |

- Express the window as `date_from` / `date_to` in `YYYY-MM-DD`, inclusive.
- State the resolved segment and window in the output headline.

`listCustomers` filters: `archived`, `asked`, `has-notes`, `missed`, `not-asked`, `not-messaged`, `not-reviewed`, `reviewed`, `unsubscribed`. Do not use `has-charges`.

Optional refinements:

- `tag_slug` — limit to one tag.
- Location: `listCustomers` has no location param; if the user needs one location, filter the returned rows by each customer's location, or ask them to narrow by tag.

## Step 3: Fetch the queue

Call `listCustomers` with the `filter`, `date_key`, `date_from`, `date_to`, `sort_key`, `sort_dir`, and `limit: max_results`.

- Paginate only if the user wants more than `max_results`, or `pagination.total` exceeds the cap and they ask to see the rest.
- Drop any row lacking contact for the chosen channel: needs `email` for email, `phone` for SMS.

## Step 4: Present the queue

```markdown
# Review Pipeline — [Project] — [filter] — signed up [date_from] to [date_to]
Up to [max_results], newest first. Requests will use this project's configured request templates.

## Summary
- Eligible: [n]   ·   Skipped (no contact): [n]   ·   Total in segment: [pagination.total]

## Queue
| # | Customer | Email / phone | Signed up | Last asked |
|---|----------|---------------|-----------|------------|
| 1 | Jane Doe | jane@example.com | 2026-06-15 | never |
| 2 | ... | ... | ... | ... |

## Next step
Confirm which rows to schedule. Default channel: email when present, else SMS. Default reminders: 2.
```

When the queue is empty, say so and suggest widening the window or switching segment (e.g. `not-reviewed`).

## Step 5: Schedule asks (write — approval required)

Only after the user confirms specific rows or approves the full list:

1. Call `createAsk` per approved customer using their `email` or `phone`.
2. Set `channels` to `["email"]` or `["sms"]` by available contact and user preference.
3. Set `reminders_count` from user input or default `2`.
4. Report success and failure per customer. Do not retry failures without asking.

Never schedule customers the user removed. Never call `createAsk` before Step 5.

## Configurable inputs

All optional; use defaults when the user gives nothing.

- `filter` — segment. Default `not-asked`.
- `days` — signup lookback. Default `30`.
- `date_from` / `date_to` — explicit window; overrides `days`.
- `date_key` — customer date column. Default `signed_up_at`.
- `sort_key` / `sort_dir` — queue order. Default `signed_up_at` / `desc`.
- `max_results` — queue cap. Default `50`.
- `channel` — `email` or `sms`. Default: email when available, else SMS.
- `reminders` — `reminders_count`. Default `2`.
- `tag_slug` — optional tag filter.
