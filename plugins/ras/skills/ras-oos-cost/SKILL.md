---
name: ras-oos-cost
description: >-
  Estimates revenue lost because a product was out of stock (OOS), per calendar month, using a
  baseline of average daily order value computed only from days the product was actually in stock
  over a trailing ~12-month window, applied to the days it was OOS in the target month. Use when
  asked about stockout cost, out-of-stock revenue loss, lost sales due to no stock, OOS impact,
  "how much did being out of stock cost us", or similar availability/lost-revenue questions. Default
  target period is the last fully closed calendar month. Every number traces to a query; every claim
  is labelled verified / inferred / unknown — the headline loss figure itself is always a
  counterfactual estimate (inferred), never a verified fact.
---

# RAS OOS cost — out-of-stock revenue loss

This skill estimates how much revenue a stockout cost, per month, per product. It exists because
restocking and safety-stock decisions get made on this number — it turns "we were out of stock" into
a defensible euro figure the business can weigh against holding cost.

> This skill inherits the precision protocol of **`/ras:ras-analysis`** (ground every number in a
> query; label verified / inferred / unknown; understand the data model before interpreting; run a
> verification pass on client-facing output; cite sources; default scope is `sales_channel = 'E-COM'`
> and no country filter unless the prompt says otherwise). Read that skill's rules first — they apply
> here in full. Below are the additions specific to OOS-cost estimation.

This logic was agreed directly with the client (confirmed on a call: "we'll adjust it this way") —
treat the rules below as fixed, not a starting point to redesign.

## Scope (default, unless the prompt says otherwise)

- **Target period = the last fully closed calendar month** relative to today. Never estimate loss for
  the current (still-open) month — it isn't closed yet. If the user names a different month or range,
  compute one loss figure **per month**, never one blended number across months.
- **Baseline window = trailing ~12 months ending at the start of the target month** (exclusive of the
  target month), or the product's full sales history if it has been on the market for less than a
  year. The baseline window must never overlap the month being measured — that would make the loss
  circular (measuring loss against a baseline partly made of OOS days from the same window).
- Apply `/ras:ras-analysis`'s default scope: `sales_channel = 'E-COM'`, no country filter, unless the
  prompt names other channels or a specific country.

## Known data-model facts (verified — treat as landmines)

- **Stock history lives in `fact_stock_daily`, one row per EAN × day.** `stock_units = 0` is the
  primary, verified signal that a product was sold out that day.
- **A missing stock row is not evidence of anything** — per `/ras:ras-analysis`, the stock table does
  not cover every brand. Where no `fact_stock_daily` row exists for an EAN/day, that day is **unknown**
  for OOS purposes, not "in stock" and not "out of stock."
- **Fallback OOS proxy (inferred, not verified).** Only for EAN/days with no stock row at all: a
  sustained drop of `ordered_pcs` to 0 in `fact_product_performance_daily` while `items_viewed` stays
  > 0 is a plausibility signal for a stockout, not proof (could also be a pricing, feed, or ranking
  issue). Always label days flagged this way **inferred**, and report them separately from the
  verified `stock_units = 0` days — never merge the two into one "OOS days" count without saying which
  is which.
- **Baseline must exclude every OOS day, including the first day stock hits zero.** That first day can
  still show partial sales (stock depleted mid-day), which would inflate — not deflate — an "in-stock"
  average if wrongly included. The rule is unconditional: `stock_units > 0` is what qualifies a day for
  the baseline, full stop.
- **Date basis:** `order_value` in `fact_product_performance_daily` is order-date (per
  `/ras:ras-analysis`'s mixed-date-basis landmine). Confirm `fact_stock_daily.date` is on the same
  calendar grain before joining — don't assume.
- **EAN ↔ product_name is not 1:1** (per `/ras:ras-analysis`). This analysis runs at EAN level; if you
  roll up to product name for reporting, state that multiple EANs may be summed and that stock status
  can differ EAN by EAN within the same product name.
- **`gads_cost` may not be a native column.** If `fact_product_performance_daily` has no direct cost
  field, derive it the same way as `/ras:ras-analysis`'s known model: `gads_clicks × gads_average_cpc`.
  Verify against the connector schema before assuming either way.

## The core metric

For each EAN, for the target month:

1. **OOS days (verified)** = `COUNT(date)` where `stock_units = 0` in the target month.
2. **OOS days (inferred, fallback only)** = days with no stock row where the `ordered_pcs`/`items_viewed`
   proxy fired. Report separately from (1).
3. **Baseline daily order value** = `AVG(order_value)` over the trailing baseline window, restricted to
   days where `stock_units > 0`.
4. **Estimated OOS loss for the month** = (OOS days, verified + inferred, stated separately or summed
   with a caveat) × baseline daily order value.
5. **Total loss for the month** = `SUM` of estimated loss across every EAN that had ≥1 OOS day.

**Bonus signal — wasted ad spend on OOS products.** For EAN/days where `stock_units = 0` and
`gads_clicks > 0` (or derived `gads_cost > 0`), report that spend separately, per EAN per month. This
is money spent driving traffic to a product nobody could buy — a distinct, additional finding, not
part of the revenue-loss number itself.

**Interpretation guardrail:** the loss figure is a **counterfactual estimate**, always labelled
**inferred** — even though its inputs (verified OOS days, verified historical order values) are
themselves verified. It assumes the trailing-12-month average is a reasonable proxy for what the
product would have sold on the specific OOS days; a strongly seasonal product (e.g. OOS during its
peak month) can make this assumption weak. Say so explicitly when the target month coincides with a
known seasonal peak or trough.

## Sanity checks that make the output defensible

Run these every time and report the result:

1. **Baseline sample size.** If an EAN has too few in-stock days in the baseline window (e.g. new
   product, or chronic stockouts) to compute a stable average, say so and label the baseline
   **low-confidence** rather than presenting it at full precision. If an EAN has **zero** in-stock days
   ever, the baseline is **unknown** — do not substitute a guess or default to zero loss.
2. **OOS days ≤ days in month.** A basic bound; if violated, the join or date filter is wrong.
3. **Total estimated loss vs. total actual revenue.** The loss figure should be a minority of the
   product's (or portfolio's) actual revenue for the same baseline period — if it isn't, something is
   off (wrong baseline window, wrong join granularity) and the number should not go out as-is.
4. **Verified vs. inferred OOS days.** State how much of the total loss comes from verified
   `stock_units = 0` days vs. the inferred proxy fallback. If the inferred share is large, flag that the
   overall number rests more on inference than the client should assume.

## SQL templates

Use your RAS BigQuery connector. Replace `<project>.<dataset>` and `<target_month_start>` (first day of
the target month, e.g. `DATE '2026-07-01'`).

**Step 1 — baseline daily order value per EAN (in-stock days only, trailing 12 months before target month):**

```sql
WITH instock_days AS (
  SELECT p.ean,
         p.date,
         p.order_value
  FROM `<project>.<dataset>.fact_product_performance_daily` p
  JOIN `<project>.<dataset>.fact_stock_daily` s
    ON s.ean = p.ean AND s.date = p.date
  WHERE p.sales_channel = 'E-COM'
    AND p.is_service = 0
    AND s.stock_units > 0
    AND p.date >= DATE_SUB(<target_month_start>, INTERVAL 12 MONTH)
    AND p.date < <target_month_start>
)
SELECT ean,
       AVG(order_value)         AS baseline_daily_order_value,
       COUNT(*)                 AS instock_days_in_baseline
FROM instock_days
GROUP BY ean;
```

**Step 2 — verified OOS days per EAN in the target month:**

```sql
SELECT ean,
       COUNT(*) AS oos_days_verified
FROM `<project>.<dataset>.fact_stock_daily`
WHERE stock_units = 0
  AND date >= <target_month_start>
  AND date <  DATE_ADD(<target_month_start>, INTERVAL 1 MONTH)
GROUP BY ean;
```

**Step 3 — inferred OOS-day fallback, EAN/days with no stock row at all (report separately, never merge silently):**

```sql
-- INFERRED signal only — no fact_stock_daily row exists for these ean/date pairs.
-- Verify persistence (e.g. N+ consecutive days) before treating as a stockout signal.
SELECT p.ean, p.date
FROM `<project>.<dataset>.fact_product_performance_daily` p
LEFT JOIN `<project>.<dataset>.fact_stock_daily` s
  ON s.ean = p.ean AND s.date = p.date
WHERE s.ean IS NULL
  AND p.ordered_pcs = 0
  AND p.items_viewed > 0
  AND p.date >= <target_month_start>
  AND p.date <  DATE_ADD(<target_month_start>, INTERVAL 1 MONTH);
```

**Step 4 — combine into the headline loss table:**

```sql
SELECT o.ean,
       o.oos_days_verified,
       b.baseline_daily_order_value,
       b.instock_days_in_baseline,
       ROUND(o.oos_days_verified * b.baseline_daily_order_value, 0) AS estimated_oos_loss
FROM oos_days_verified o
JOIN baseline b USING (ean)
ORDER BY estimated_oos_loss DESC;
```

**Bonus — wasted ad spend on OOS EANs, same month:**

```sql
SELECT s.ean,
       FORMAT_DATE('%Y-%m', s.date) AS month,
       SUM(p.gads_clicks)                          AS clicks_while_oos,
       SUM(p.gads_clicks * p.gads_average_cpc)      AS spend_while_oos  -- derive if gads_cost isn't native; verify first
FROM `<project>.<dataset>.fact_stock_daily` s
JOIN `<project>.<dataset>.fact_product_performance_daily` p
  ON p.ean = s.ean AND p.date = s.date
WHERE s.stock_units = 0
  AND s.date >= <target_month_start>
  AND s.date <  DATE_ADD(<target_month_start>, INTERVAL 1 MONTH)
GROUP BY s.ean, month
HAVING clicks_while_oos > 0
ORDER BY spend_while_oos DESC;
```

## Workflow

1. **Load the model.** Read `/ras:ras-analysis`'s landmines plus the ones above. Confirm
   `fact_stock_daily`'s date grain and join key (EAN) against `fact_product_performance_daily` before
   writing anything else.
2. **Fix the target month.** Default to the last fully closed calendar month; confirm with the user if
   ambiguous. Fix the baseline window to the trailing ~12 months strictly before that month (or full
   history if the product is newer).
3. **Run Step 1** for the baseline, **Step 2** for verified OOS days, **Step 3** only if stock coverage
   gaps exist for the EANs in question. Keep all SQL.
4. **Run the sanity checks.** Especially baseline sample size and the loss-vs-revenue bound. Downgrade
   or flag any EAN that fails them rather than reporting a shaky number at full confidence.
5. **Compute the bonus wasted-spend signal** for the same OOS EANs/month.
6. **Draft with confidence labels.** The final loss figure is always **inferred** (counterfactual).
   Verified: OOS day counts from `stock_units = 0`, historical order values. Inferred: the fallback
   proxy OOS days, and the loss estimate itself. Unknown: any EAN with no usable baseline.
7. **Verification pass (client-facing).** Re-derive one or two EANs by hand from source; check the
   baseline window didn't leak into the target month, and that verified/inferred OOS days weren't
   silently merged.
8. **Deliver.** Per month: total estimated loss, top EANs by loss, verified-vs-inferred split, the
   wasted-ad-spend bonus finding, and a short verified/inferred/unknown read. End with methodology +
   Sources.
9. **Feed gaps back.** If stock-table coverage gaps force frequent use of the inferred fallback, raise
   it as a data-backlog item rather than leaning on the proxy every time.

## Confidence tags — quick reference

| Tag | Means | How to phrase |
|-----|-------|---------------|
| Verified | Ran the query / read the source | State it plainly, keep the query |
| Inferred | Deduced from verified facts | "Inferred: … — rests on …" |
| Unknown | Not in available data | "Not available in RAS; to answer we'd need …" |

## Standard deliverable footer (adapt per account)

> **Methodology & defensibility:** Loss computed per calendar month. Baseline = average daily
> order value per EAN over the trailing ~12 months before the target month (or shorter, if the
> product is newer), restricted to days with `stock_units > 0` (OOS days, including the first
> zero-stock day, are excluded from the baseline to avoid deflating or inflating it). Estimated loss =
> verified OOS days in the target month × baseline. The loss figure is an **inferred counterfactual
> estimate**, not a verified fact. Separately reported: OOS days detected only via the inferred
> `ordered_pcs`/`items_viewed` proxy (no stock record available), and Google Ads spend continuing on
> OOS EANs in the same month. Filters: `sales_channel='E-COM'`, `is_service=0`, EAN-level.
>
> **Sources:** `<project>.<dataset>` — fact_stock_daily, fact_product_performance_daily.
