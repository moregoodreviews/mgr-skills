---
name: reputation-actions
description: Produces a short, ranked to-do list of what to do next to improve reputation in a More Good Reviews project — unreplied negatives, reply gaps, review-volume shortfalls, outreach problems, and location-specific fixes. This is a to-do list, NOT a metrics report (use reputation-report for the dashboard) and it does NOT send anything (use review-pipeline to schedule asks). Use when the user asks what to do, how to improve, what to fix, what needs attention, or what their priorities are.
---

# Reputation Actions

Tell the user the highest-impact things to do next for one More Good Reviews project. Output is a **ranked to-do list** — not a report, not a data dump.

**Prerequisite:** Requires a More Good Reviews Project API MCP server connected. This skill calls its tools and has no other data source. If no such server is connected, say so and stop.

## Core principle: actions, not analytics

- Every line is something the user can **do** — reply, ask, investigate, fix, brief a team.
- Lead each action with a verb. Include only the context needed to act (count, location, one review reference). No charts, leaderboards, or period-over-period tables.
- Ignore reviews flagged as duplicates (`is_duplicate`); they never count toward a metric or an action.
- Rank by urgency first, then impact. Cap the list at `max_actions` (default 7).
- If nothing needs work, say so plainly and offer one optional maintenance item.

## Read-only contract

Only call read tools: `listReviews`, `listLocations`, `listMessages`, `listCustomers`. Never mutate anything. This skill recommends and drafts; it never sends, replies, or schedules.

## Step 1: Select the project

Each More Good Reviews Project API MCP connection is scoped to one project. A user may have several connected at once.

- If exactly one is connected, use it without prompting.
- If more than one is connected, name them and ask which to use.
- Use one project's tools consistently for the whole run.

## Step 2: Resolve the window

Default to the **last 30 days** when the user gives nothing. Also compute an equal-length **prior** window — used internally to judge what is normal for this project, not printed.

- `days` — default `30`.
- `date_from` / `date_to` — explicit `YYYY-MM-DD`, inclusive; overrides `days`.
- Reviews use `date_key: "created_at"`. Outreach uses `date_key: "sent_at"`.

## Step 3: Gather signals (minimum needed)

List endpoints are paginated; use `pagination.total` for counts and only fetch review text when naming specific reviews or diagnosing a theme.

**Exclude duplicates.** Reviews flagged as duplicates must not count toward any metric or appear in any action. The `filter` parameter holds only one value (and has no "not-duplicate" option), so it cannot be combined with `without-replies`. Instead, read each returned review's `is_duplicate` field and drop every review where it is `true` before counting, computing coverage, or naming reviews. When relying on `pagination.total` for a count, fetch the rows and subtract flagged duplicates rather than trusting the raw total.

- **Unreplied negatives** — `listReviews` with `filter: "without-replies"`, `score: 1`, then `score: 2`, current window, `sort_dir: "desc"`. Pull enough rows to name up to three.
- **Reply gap** — `listReviews` `with-replies` vs `without-replies` over the current window; compute coverage.
- **Volume trend** — `listReviews` totals, current vs prior window.
- **Location regressions** — `listLocations`, then per-location totals and average score for both windows (only when multiple locations exist).
- **Outreach health** — `listMessages` `channel: "email"`, current window; watch for high failure rate or unusually low volume.
- **Ask pipeline** — `listCustomers` `filter: "not-asked"`, `limit: 1`; read `pagination.total` only.

## Step 4: Judge against the project's own normal, then rank

Prefer **learned baselines** over fixed thresholds — what matters is deviation from this project's trajectory:

- **Rating** — flag a location or the project when its current-window average drops noticeably below its prior window (and below its usual level). Use `rating_alert_drop` (default `0.2` stars) only as a fallback when prior data is thin.
- **Volume** — compare current vs prior review count; flag a clear drop. Use `review_target` only if the user supplied one.
- **Reply coverage** — compare to the project's recent norm; if it has historically been near-complete, even a small slip matters. Use `reply_coverage_target` (default `90%`) as a fallback floor.
- **Outreach** — flag failure rate above `failure_rate_alert` (default `5%`) or a sharp drop in sends.

State briefly when you are judging against the project's own history versus a fallback default, so the reasoning is transparent.

Map each surviving signal to one action; skip healthy signals and do not pad.

## Step 5: Output

```markdown
# What to improve — [Project] — [date_from] to [date_to]

## Do these first
1. **[Verb] [specific target]** — [why it matters now: counts, location, or a review reference]
2. ...

## Do next
1. **[Verb] [specific target]** — [one sentence of context]

## Looking good
- [1–3 things that need no action right now]

_Snapshot: [avg rating] · [n] reviews · [reply coverage]% replied · [n] never asked — [date_from] to [date_to]_
```

Rules:

- **Do these first**: urgent items only (unreplied negatives, sharp rating drops, delivery failures). Max 5.
- **Do next**: important, not urgent (volume, coverage gaps, pipeline). Max 2.
- **Looking good**: brief; omit if everything needs work.
- The closing **Snapshot** line is one paste-able row the user can drop into the next run for continuity (this skill keeps no memory between runs).
- When an action is "reply to negative reviews," offer a handoff: ask whether to draft those replies now. Drafting and publishing happen in a separate reply workflow, not here.

## Configurable inputs

All optional; use defaults when the user gives nothing.

- `days` — lookback window. Default `30`.
- `date_from` / `date_to` — explicit window; overrides `days`.
- `max_actions` — cap for Do-these-first plus Do-next combined. Default `7`.
- `reply_coverage_target` — fallback floor. Default `90%`.
- `rating_alert_drop` — fallback star drop. Default `0.2`.
- `failure_rate_alert` — outreach failure flag. Default `5%`.
