---
name: reputation-report
description: Generates a reputation health report from the More Good Reviews Project API MCP server over a configurable period (weekly, monthly, quarterly, or yearly), covering review volume, ratings, location and source breakdowns, reply coverage, outreach health, and prioritized action items with period-over-period trends. Use when the user asks for a reputation report, review summary, ratings trend, or reputation health check for a connected More Good Reviews project.
---

# Reputation Report

Produce a clear, action-first reputation report for one More Good Reviews project using only the read tools exposed by the More Good Reviews Project API MCP server. The report works for any project and any period; nothing about a specific business is hardcoded.

**Prerequisite:** Requires a More Good Reviews Project API MCP server connected. This skill calls its tools and has no other data source. If no such server is connected, say so and stop.

## Core principle: insights, not a data dump

The report exists to drive decisions. Numbers are only worth showing when paired with interpretation and a next step.

- Every section ends with a one-line **So what?** that says what changed, why it matters, and what to do.
- The report opens with **Top 3 priorities** before any detail.
- Recommendations are specific and owner-assignable, and each ties to a concrete More Good Reviews follow-up (reply to specific reviews, run a review-request campaign, brief a location). Avoid generic advice.
- Quantify impact and urgency where possible (how many reviews, how many stars, which location).
- Ignore reviews flagged as duplicates (`is_duplicate`); they never count toward any metric, breakdown, or theme.

## Read-only contract

Only call read tools: `listReviews`, `listLocations`, `listSources`, `listMessages`. Never delete, hide, reply to, tag, or otherwise mutate anything in this workflow.

## Step 1: Select the project

Each More Good Reviews Project API MCP connection is already scoped to one project via its API key or OAuth token, so never ask for keys or project IDs. But a user may have several More Good Reviews project MCP servers connected at once.

- If exactly one More Good Reviews project MCP is connected, use it without prompting.
- If more than one is connected, name the connected servers and ask which project to report on, or offer to run the report once per project.
- Use a single chosen project's tools consistently for the entire report. Never mix tools across project connections in one report.

## Step 2: Resolve the period and windows

Accept a `period` input. Default to `weekly` when the user gives nothing.

| `period`    | Current window                                      | Prior window (equal length)        |
|-------------|-----------------------------------------------------|------------------------------------|
| `weekly`    | last 7 days (today minus 6 through today)           | the 7 days before that             |
| `monthly`   | the target calendar month, 1st to last day          | the preceding calendar month       |
| `quarterly` | the target calendar quarter                         | the preceding quarter              |
| `yearly`    | the target calendar year                            | the preceding year                 |
| `custom`    | the user-supplied `date_from`/`date_to`             | the equal-length span just before  |

Rules:

- Express every window as `date_from` and `date_to` in `YYYY-MM-DD`, inclusive.
- Always pair a current window with an equal-length **prior** window for period-over-period (PoP) deltas.
- For calendar periods (`monthly`/`quarterly`/`yearly`), use exact calendar boundaries unless the user asks for trailing windows (trailing 30/90/365 days). State which interpretation you used.
- State the resolved windows in the report headline so the output is unambiguous and reproducible.
- Reviews use `date_key: "created_at"`. Outreach uses `date_key: "sent_at"`.

## Step 3: Collect data

All list endpoints are paginated. Use the response `pagination` object (`current_page`, `last_page`, `total`). For counts, read `pagination.total` rather than fetching every page; only page through review bodies when you need text for theme analysis.

**Exclude duplicates.** Reviews flagged as duplicates must not count toward any metric, distribution, leaderboard, or theme. The `filter` parameter holds only one value (and has no "not-duplicate" option), so it cannot be combined with `without-replies`. Instead, read each returned review's `is_duplicate` field and drop every review where it is `true` before counting. When a count would otherwise come from `pagination.total`, fetch the rows and subtract flagged duplicates rather than trusting the raw total.

Call these tools, parameterized by the windows from Step 2:

- **Reviews (volume + ratings)** — `listReviews` with `date_key: "created_at"`, `date_from`, `date_to` for both the current and prior windows. Use `score` to build the star distribution. Use `pagination.total` for counts.
- **Reply coverage** — `listReviews` with `filter: "without-replies"` and again with `filter: "with-replies"` over the current window.
- **Unreplied negatives** — `listReviews` with `filter: "without-replies"` and `score: 1`, then `score: 2`, `sort_key: "created_at"`, `sort_dir: "desc"`. These feed Needs attention.
- **Locations** — `listLocations` (no params) for the location list, then `listReviews` per `location_slug` over current and prior windows for the leaderboard.
- **Sources** — `listSources` (no params) for source labels, then break review counts down by `source_slug`.
- **Outreach health** — `listMessages` with `channel: "email"`, `date_key: "sent_at"`, over the current window. Compute rates from `status` and the `delivered_at`/`opened_at`/`clicked_at`/`failed_at` timestamps.

`listReviews` filters available: `date_from`, `date_to`, `date_key` (`created_at`/`replied_at`/`updated_at`), `filter`, `score`, `location_slug`, `source_slug`, `tag_slug`, `sort_key`, `sort_dir`, `limit`. `listMessages` filters: `channel`, `date_key`, `date_from`, `date_to`, `status`, `template_slug`, `sort_key`, `sort_dir`, `limit`.

## Step 4: Compute metrics

- Total reviews (current vs prior, with PoP delta and percent change).
- Average rating (current vs prior, delta in stars).
- Star distribution (count and percent per score 1-5).
- Percent 5-star.
- Reply coverage percent = with-replies / total for the current window.
- Per-location: review count, average rating, percent 5-star, and PoP rating trend.
- Per-source: review count and average rating.
- Outreach: emails sent, delivery rate, open rate, click rate, failure rate.

### Alert flags (defaults, overridable)

- Average rating dropped more than **0.2 stars** PoP (overall or for any location).
- Reply coverage below the **90%** target.
- Review volume below the period target (see Configurable inputs).
- Any spike in unreplied 1-2 star reviews vs the prior period.

## Step 5: Diagnose negative trends (where and why)

Never report a decline without isolating its driver. When a metric drops PoP, decompose it across the dimensions already gathered:

1. **Where** — Compare the drop across `location_slug` (which location regressed most) and `source_slug` (which platform). Determine whether it is volume-driven (fewer reviews) or sentiment-driven (lower scores), using the per-location/per-source breakdowns and PoP deltas.
2. **Why** — For the locations or sources driving the decline, pull the underlying negative reviews (`listReviews` with `score: 1`/`2` over the current window, scoped by `location_slug`/`source_slug`) and read the text. Extract the top 2-3 recurring themes or complaints as the likely cause.
3. **Fix** — Convert each finding into a targeted recommendation. When the cause is too few recent reviews, recommend a review-request campaign. When sentiment-driven, name the theme and point to the specific reviews to address.

Example finding: "Rating at the downtown location fell 0.5 stars PoP, driven by 6 reviews citing wait times. Reply to those 6 and brief that location on staffing."

## Report template

Output in this fixed order. Keep it tight; lead with action.

```markdown
# Reputation Report — [Project] — [period] ([date_from] to [date_to])
Compared to prior [period] ([prior_date_from] to [prior_date_to]).

## Top 3 priorities
1. [Most important insight + the concrete action to take]
2. [...]
3. [...]

## Reviews
- New reviews: [n] ([+/-x%] vs prior)
- Average rating: [x.x] ([+/-0.x] vs prior)
- 5-star share: [x%]; distribution: 5:[n] 4:[n] 3:[n] 2:[n] 1:[n]
- Reply coverage: [x%] ([n] unreplied)
- So what?: [interpretation + next step]

## Location leaderboard
[Per location: reviews, avg rating, % 5-star, PoP trend. Flag any drop > 0.2 stars.]
- So what?: [where to focus]

## Source breakdown
[Per source: reviews, avg rating.]
- So what?: [which platform needs attention]

## Outreach health
- Emails sent: [n]; delivered [x%], opened [x%], clicked [x%], failed [x%]
- So what?: [is outreach driving enough reviews?]

## Needs attention
[Ranked unreplied 1-2 star reviews: location, source, score, date, snippet.]

## Recommended actions
1. [Specific, owner-assignable action tied to a More Good Reviews follow-up]
2. [...]
```

When a single location or source has no data for a window, say so rather than implying zero performance.

## Configurable inputs

All optional, with sensible defaults. Use defaults when the user gives nothing; accept overrides at run time.

- `period` — `weekly` (default), `monthly`, `quarterly`, `yearly`, or `custom` with explicit `date_from`/`date_to`.
- `review_target` — expected new reviews for the period. Scale any per-week target to the chosen period (e.g. a weekly target of 10 becomes ~43 for a 30-day month). No default volume target unless the user provides one; skip the volume flag when absent.
- `rating_alert_drop` — PoP average-rating drop that triggers a flag. Default `0.2` stars.
- `reply_coverage_target` — minimum acceptable reply coverage. Default `90%`.

The report auto-adapts to whatever locations and sources the project returns, so it works for single-location and multi-location projects alike. Omit the leaderboard breakdown only when the project has a single location, and say so.
