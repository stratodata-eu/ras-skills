---
name: ras-scorecard
description: >-
  Builds a client-facing scorecard artifact that evaluates a defined set of goals and metrics on a
  recurring basis (default: weekly) against RAS / BigQuery data, then keeps it refreshed on that
  cadence. Takes a goal/metric definition as input — plain-language in the prompt or an uploaded
  onepager brief (PDF, doc, or text) — maps each metric to the RAS data model, computes current vs.
  target status (on track / at risk / off track), and publishes a dashboard artifact with per-metric
  trend. Use when asked to build a scorecard, KPI dashboard, weekly/monthly business review artifact,
  "track these goals against our data", "set up a recurring scorecard", or to turn a goals onepager
  into a live-updating dashboard. Every metric traces to a query; every claim is labelled verified /
  inferred / unknown; a metric with no confirmed target or mapping is never silently guessed.
---

# RAS scorecard — recurring goal/metric dashboard artifact

This skill turns a set of business goals and metrics into a scorecard artifact that gets refreshed
on a fixed cadence (default weekly) against the client's own RAS / BigQuery data. It exists so
"are we on track" has one place to look, backed by the same query-grounded rigor as every other RAS
deliverable — not a hand-maintained spreadsheet that drifts from the source data.

> This skill inherits the precision protocol of **`/ras:ras-analysis`** (ground every number in a
> query; label verified / inferred / unknown; understand the data model before interpreting; run a
> verification pass on client-facing output; cite sources; default scope is `sales_channel = 'E-COM'`
> and no country filter unless the input says otherwise). Read that skill's rules first — they apply
> here in full. Before building the artifact itself, load **`dataviz`** and **`artifact-design`** for
> chart/layout guidance — this skill defines *what* goes on the scorecard, not how to theme it.

## What this skill is not

It does not invent goals or targets. It maps goals the client already defined onto RAS data, and it
never fills in a missing target, formula, or mapping with a plausible-looking default — that's a gap
to close with the user, because a wrong mapping wired into a recurring artifact repeats the same
error every refresh, unattended.

## Step 1 — take in the goal/metric definition

Accept whatever form the input arrives in:

- **Plain language in the prompt** — a list of goals/metrics, e.g. "track weekly net sales vs. a 5%
  WoW growth target, and order count vs. 500/week."
- **An uploaded onepager** — PDF, Word, or plain text. Use the `pdf` / `docx` skill as appropriate to
  extract it; don't re-key it by eye if a parsing skill is available.
- **A mix** — a onepager plus prompt clarifications/overrides.

For each metric, extract:

1. **Name** — the business-language label (e.g. "Net sales growth", "Repeat purchase rate").
2. **Definition** — how it's computed, in business terms (not yet mapped to columns).
3. **Target** — a value, a growth rate, or a range. If direction isn't explicit, infer it only when
   unambiguous (revenue: higher is better; return rate: lower is better) — otherwise ask.
4. **Cadence** — defaults to weekly if the input doesn't say otherwise. A single scorecard can mix
   cadences per metric (e.g. one metric weekly, another monthly) if the input specifies that.

**If a metric has no target, no clear formula, or an ambiguous direction, do not guess.** Either ask
the user to confirm before wiring it into a recurring artifact, or — if they say so explicitly — mark
it as **track-only** (trend reported, no on-track/at-risk/off-track status computed for it).

## Step 2 — map every metric to the RAS data model

Before writing any query, map each business-language metric to specific RAS tables/columns and state
the mapping back to the user for confirmation. This is the step most likely to go wrong, and because
the scorecard runs unattended afterward, an unconfirmed mapping error repeats every refresh.

Apply `/ras:ras-analysis`'s known data-model landmines while mapping:

- **Mixed date basis.** Pick one date basis per metric and state it (order date for `orders` /
  `order_value` / `ordered_pcs`; invoice date for `invoice_value` / `sold_pcs` / `gross_margin`;
  activity date for `sessions` / `cost` / GA4 events). Never blend two metrics of different date
  basis into one "period" comparison without a caption saying so.
- **`net_sales`, `gross_margin`, `net_contribution` are estimated**, not actuals — carry that caveat
  onto any tile built from them.
- **GA4 `traffic_source`/`traffic_medium` are user-scoped first-touch, not session-scoped** — don't
  build a channel-attribution metric on them without the same caveat `/ras:ras-analysis` gives.
- **Stock table doesn't cover every brand; EAN ↔ product_name isn't 1:1** — relevant if any metric
  touches availability or a rolled-up product name.
- Apply the default scope (`sales_channel = 'E-COM'`, no country filter) unless the input names other
  channels/countries — state the scope applied per metric if it differs from the default.

## Step 3 — fix the cadence and status thresholds

- **Default evaluation period = the last fully closed calendar week (Monday–Sunday)** relative to
  today. Never evaluate the current, still-open week/period — it's incomplete. If the input names a
  different cadence (daily, monthly, quarterly), use that instead and state it per metric.
- **Trend window = trailing 12 periods** by default (12 weeks for a weekly metric), for the sparkline
  and week-over-week context. Override if the input asks for a longer/shorter history.
- **Status bands**, unless the input defines its own:
  - **On track** — current period meets or beats target (at/below target for a lower-is-better
    metric).
  - **At risk** — within 10% of target, short of it.
  - **Off track** — more than 10% short of target.
  - **Track-only** — no target defined; report trend only, never default this to "on track."
  State whichever bands were actually applied in the artifact's methodology footer.

## Step 4 — query, don't guess

For each metric, write SQL for: (a) the current closed period's value, and (b) the trailing-window
series for the trend. Keep every query. A generic weekly-aggregation shape (adapt table/column/
aggregation per metric, and the date-truncation unit for other cadences):

```sql
SELECT
  DATE_TRUNC(date, WEEK(MONDAY)) AS period_start,
  SUM(<metric_column>)           AS <metric_alias>   -- or AVG/COUNT as the metric requires
FROM `<project>.<dataset>.<fact_table>`
WHERE sales_channel = 'E-COM'                          -- adjust per confirmed scope
  AND date >= DATE_SUB(DATE_TRUNC(CURRENT_DATE(), WEEK(MONDAY)), INTERVAL <trend_periods> WEEK)
  AND date <  DATE_TRUNC(CURRENT_DATE(), WEEK(MONDAY))  -- excludes the current, open week
GROUP BY period_start
ORDER BY period_start;
```

The most recent row in that series is the current closed period; the rest is the trend. For a
growth-rate target (e.g. "5% WoW"), compute the rate from consecutive rows rather than hard-coding it.

## Step 5 — classify and label

For each metric: compute status against its band, and label the value **verified** (came from the
query), the status classification **inferred** if it rests on an estimated field (e.g. `net_sales`),
and anything blocked by a data gap **unknown** (e.g. a metric needing a column that doesn't exist, or
a period with no coverage) — report it as unknown on its tile, never as zero or as "off track."

## Step 6 — build the artifact

Load `dataviz` and `artifact-design` before writing the page; use `artifact-capabilities` only if the
scorecard needs a runtime capability (see note below — usually it doesn't).

Layout, one scorecard artifact:

- **Header** — evaluation period covered, cadence, and a refreshed-at timestamp; an overall summary
  (counts by status: N on track / N at risk / N off track / N track-only).
- **One tile per metric** — current value, target (if any), delta vs. target, status badge, a
  sparkline over the trend window, and a caption naming its date basis and scope if non-default.
- **Footer** — the methodology block (metric definitions, scope, status bands, cadence) and a Sources
  line (tables/queries the artifact rests on), same convention as every other RAS deliverable.

**Important: the artifact is a snapshot, not a live query.** A published artifact runs in a browser
sandbox and cannot reach the client's private RAS/BigQuery MCP connector at runtime. "Refresh" means
Claude re-runs the queries and republishes to the *same artifact URL* on the defined cadence — the
page itself does not pull live data. Don't reach for a `db`/`connectivity` runtime capability to make
it "live"; only add a capability if the client separately asks for viewer interaction (e.g. per-viewer
notes) that genuinely needs one — load `artifact-capabilities` first if so.

## Step 7 — set up the recurring refresh

1. **Persist the definition.** Save the confirmed metric list (name, mapping, target, cadence,
   status bands, scope) and the published artifact's URL to a small config file in the client's
   workspace (e.g. `ras-scorecard/<scorecard-name>.json`). The scheduled run must reproduce the exact
   same mapping every time, not re-derive it from a stale conversation.
2. **Schedule it.** Use the `schedule` skill (or `CronCreate` directly) to create a recurring job on
   the defined cadence that: reads the config, re-runs Step 4's queries, recomputes status (Step 5),
   and republishes to the same artifact URL (`Artifact` with `url` set — this updates the page in
   place rather than creating a new link).
3. **Every automated refresh still runs the sanity checks** below before republishing, and downgrades
   or flags a metric rather than pushing a bad number out unattended.
4. Tell the user the cadence that was set up and where the config lives, so they can edit targets
   later without you re-deriving the whole scorecard from scratch.

## Sanity checks that make the output defensible

Run these every time (first publish and every scheduled refresh) and report the result:

1. **Every metric lands in exactly one status** (on track / at risk / off track / track-only) — no
   metric silently dropped or double-counted in the summary.
2. **Evaluation period is fully closed.** Confirm the query window excludes the current, open period.
3. **Trend window matches the stated one.** If a metric has fewer periods of history than the trend
   window (new metric, new product line), say so rather than showing a truncated sparkline unlabeled.
4. **Status flips get called out.** On a scheduled refresh, note any metric whose status changed since
   the previous run (on track → at risk, etc.) — that's usually the one thing the client actually
   wants to know from a recurring dashboard.
5. **Estimated/unknown fields stay labelled.** A metric built on an estimated field (e.g. `net_sales`)
   keeps that caveat on every refresh, not just the first one.

## Workflow

1. **Take in the definition.** Parse the prompt and/or onepager into a metric list (Step 1). Flag
   anything with a missing target, formula, or direction — confirm with the user or mark track-only.
2. **Map to the data model.** For each metric, confirm table/column/date-basis/scope against
   `/ras:ras-analysis`'s landmines (Step 2). State the mapping back to the user before proceeding.
3. **Fix cadence, trend window, and status bands** (Step 3), using the input's own values where given
   and the stated defaults otherwise.
4. **Query every metric** — current closed period plus trend series (Step 4). Keep all SQL.
5. **Classify and label** each metric's status and confidence (Step 5).
6. **Run the sanity checks.** Downgrade or flag anything that fails rather than shipping it as-is.
7. **Build and publish the artifact** (Step 6), loading `dataviz`/`artifact-design` first.
8. **Set up the recurring refresh** (Step 7) — persist the config, schedule the job, tell the user
   the cadence and where the config lives.
9. **Deliver.** The artifact link, the evaluation period, the status summary, and any metric that
   couldn't be built (missing mapping, no target, data gap) reported as unknown rather than omitted
   silently. End with methodology + Sources.
10. **Feed gaps back.** If a goal can't be mapped to anything in RAS at all, raise it as a data-backlog
    item — don't approximate it with an unrelated column just to fill the tile.

## Confidence tags — quick reference

| Tag | Means | How to phrase |
|-----|-------|---------------|
| Verified | Ran the query / read the source | State it plainly, keep the query |
| Inferred | Deduced from verified facts, or built on an estimated field | "Inferred: … — rests on …" |
| Unknown | Not in available data, or no confirmed mapping/target | "Not available; to answer we'd need …" |

## Standard deliverable footer (adapt per account)

> **Methodology & defensibility:** Each metric is evaluated over the last fully closed period at its
> stated cadence (default weekly, Monday–Sunday) — the current, open period is never included. Trend
> shown over the trailing window (default 12 periods). Status = on track / at risk (within 10 % of
> target) / off track (>10 % short) / track-only (no target defined), unless the brief defined its own
> bands. This artifact is a snapshot as of its refreshed-at timestamp, republished on the stated
> cadence — it does not query RAS live. Filters and date basis are stated per metric where they differ
> from the default (`sales_channel='E-COM'`, no country filter).
>
> **Sources:** `<project>.<dataset>` — tables listed per metric above; scorecard definition stored at
> `ras-scorecard/<scorecard-name>.json`.
