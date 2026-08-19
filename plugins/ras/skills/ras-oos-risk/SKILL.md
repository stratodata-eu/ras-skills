---
name: ras-oos-risk
description: >-
  Flags best-selling products at risk of an upcoming stockout: ranks products by recent sales
  velocity, compares current stock to that velocity, and computes how many days of stock remain at
  the current sales pace. Sales pace is computed from order-date fields over in-stock days only.
  Products with fewer than two weeks of stock left are flagged at risk by default; the user can
  name a different reorder point (the days-of-stock threshold) in the prompt. Use when asked about
  stockout risk, low stock, reorder point, "which products are about to run out", "how many days of
  stock left", or similar forward-looking availability questions. Every number traces to a query;
  every claim is labelled verified / inferred / unknown — the days-remaining figure is always a
  forward-looking projection (inferred), never a verified fact.
---

# RAS OOS risk — upcoming stockout risk on best-sellers

This skill flags which best-selling products are about to run out of stock, so restocking can happen
before the stockout — not after. It is the forward-looking counterpart to `/ras:ras-oos-cost`, which
measures loss after the fact; this one exists to prevent that loss from happening.

> This skill inherits the precision protocol of **`/ras:ras-analysis`** (ground every number in a
> query; label verified / inferred / unknown; understand the data model before interpreting; run a
> verification pass on client-facing output; cite sources; default scope is `sales_channel = 'E-COM'`
> and no country filter unless the prompt says otherwise). Read that skill's rules first — they apply
> here in full. Below are the additions specific to stockout-risk projection.

## Scope (default, unless the prompt says otherwise)

- **Sales-velocity window = trailing 30 full days**, ending at the latest fully elapsed day (never
  including the current, still-open day — it's an incomplete count). If the prompt names a different
  window (e.g. "last 14 days", "last quarter"), use that instead, but keep it stated in the output.
- **Best-sellers = top 20 EANs by total units sold (`ordered_pcs`) in the velocity window.** If the
  prompt names a different cut (top 50, top 100, "all products", a specific brand or category), use
  that instead. Ranking is always by units sold unless the prompt asks to rank by revenue instead.
- **Reorder point (risk threshold) defaults to 14 days of stock remaining** ("two weeks"). If the
  prompt names a different reorder point (e.g. "flag anything under 21 days", "use a 10-day
  threshold"), use that number instead and state which threshold was applied.
- Apply `/ras:ras-analysis`'s default scope: `sales_channel = 'E-COM'`, no country filter, unless the
  prompt names other channels or a specific country.

## Known data-model facts (verified — treat as landmines)

- **Sales pace must be computed on order-date fields.** `ordered_pcs` / `order_value` in
  `fact_product_performance_daily` are order-date, per `/ras:ras-analysis`'s mixed-date-basis
  landmine. Do not substitute `sold_pcs` (invoice/fulfilment-date) — it answers a different question
  and will misstate how fast stock is currently depleting.
- **Sales velocity must exclude OOS days, including the first day stock hits zero** — same rule as
  `/ras:ras-oos-cost`'s baseline. A product that was sold out for part of the window will show
  artificially low `ordered_pcs` on those days; including them understates true sales pace and
  overstates days-of-stock-remaining, hiding real risk. Only average over days where
  `stock_units > 0`.
- **Current stock is a snapshot, not an average.** Use the most recent `fact_stock_daily` row per
  EAN. If that row's date lags today by more than a day or two, say so — the risk projection is only
  as fresh as the last stock snapshot.
- **A missing stock row is not evidence of anything.** Per `/ras:ras-analysis`, the stock table does
  not cover every brand. An EAN with no `fact_stock_daily` rows at all has **unknown** current stock —
  do not report it as in stock, out of stock, or "at risk."
- **Zero recent sales is not the same as "not at risk."** If an EAN has no in-stock sales in the
  velocity window, days-of-stock-remaining is undefined (division by zero), not infinite and not
  zero. Report it as a separate **no-recent-sales** case — never fold it into "OK" or "at risk."
- **Already-zero stock is a distinct, more urgent case than "at risk."** If current stock is already
  0, the product isn't projected to run out — it already has. Report it separately from the
  days-remaining ranking (it belongs with `/ras:ras-oos-cost`'s territory, not a projection).
- **EAN ↔ product_name is not 1:1** (per `/ras:ras-analysis`). This analysis runs at EAN level; if you
  roll up to product name for reporting, state that multiple EANs may be summed and that stock and
  risk status can differ EAN by EAN within the same product name.

## The core metric

For each best-selling EAN:

1. **Sales velocity (verified, from query)** = `AVG(ordered_pcs)` over the trailing window,
   restricted to days where `stock_units > 0`.
2. **Current stock (verified, from query)** = `stock_units` from the most recent `fact_stock_daily`
   row for that EAN, with its snapshot date reported alongside it.
3. **Days of stock remaining (inferred projection)** = current stock ÷ sales velocity.
4. **Risk status:**
   - `current stock = 0` → **already out of stock** (report separately, most urgent).
   - `sales velocity = 0` or no in-stock days in the window → **no recent sales** (undefined
     days-remaining — do not compute a number).
   - `days of stock remaining < reorder point` → **at risk**.
   - otherwise → **OK**.

**Interpretation guardrail:** days-of-stock-remaining is a **projection**, always labelled
**inferred**, even though both inputs (current stock, historical sales pace) are verified. It assumes
the trailing-window pace continues unchanged; a product about to enter a demand spike (promotion,
seasonal peak) or lull will make the projection optimistic or pessimistic respectively. Say so
explicitly when the target period overlaps a known seasonal or promotional event.

## Sanity checks that make the output defensible

Run these every time and report the result:

1. **Velocity sample size.** If an EAN has too few in-stock days in the window (e.g. chronic
   stockouts, new listing) to compute a stable pace, label the velocity **low-confidence** rather than
   presenting the days-remaining figure at full precision.
2. **Stock snapshot freshness.** Report the stock snapshot date used. If it's stale (more than ~2
   days behind today), flag that the risk read may already be out of date.
3. **Already-OOS vs at-risk vs no-recent-sales counts don't overlap.** Every best-selling EAN lands in
   exactly one of: already out of stock, at risk, no recent sales, OK. Verify the counts sum to the
   total best-seller count.
4. **Best-seller cut is stated.** State how many EANs were ranked and by what (units sold, over which
   window) so the reader knows what was in scope and what was excluded.

## SQL templates

Use your RAS BigQuery connector. Replace `<project>.<dataset>`, the velocity-window length, the
best-seller cut (`LIMIT`), and the reorder-point threshold as the prompt requires.

**Step 1 — best-sellers by units sold in the trailing window:**

```sql
SELECT ean,
       SUM(ordered_pcs)  AS units_sold_window,
       SUM(order_value)  AS revenue_window
FROM `<project>.<dataset>.fact_product_performance_daily`
WHERE sales_channel = 'E-COM'
  AND is_service = 0
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND date < CURRENT_DATE()
GROUP BY ean
ORDER BY units_sold_window DESC
LIMIT 20;
```

**Step 2 — sales velocity per EAN, in-stock days only, trailing window:**

```sql
WITH instock_days AS (
  SELECT p.ean,
         p.date,
         p.ordered_pcs
  FROM `<project>.<dataset>.fact_product_performance_daily` p
  JOIN `<project>.<dataset>.fact_stock_daily` s
    ON s.ean = p.ean AND s.date = p.date
  WHERE p.sales_channel = 'E-COM'
    AND p.is_service = 0
    AND s.stock_units > 0
    AND p.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND p.date < CURRENT_DATE()
)
SELECT ean,
       AVG(ordered_pcs) AS avg_daily_units_instock,
       COUNT(*)         AS instock_days_in_window
FROM instock_days
GROUP BY ean;
```

**Step 3 — current stock snapshot per EAN (most recent row):**

```sql
SELECT ean, stock_units, date AS stock_date
FROM `<project>.<dataset>.fact_stock_daily` s
WHERE date = (
  SELECT MAX(date)
  FROM `<project>.<dataset>.fact_stock_daily` s2
  WHERE s2.ean = s.ean
);
```

**Step 4 — combine into the headline risk table:**

```sql
SELECT b.ean,
       c.stock_units                                            AS current_stock,
       c.stock_date,
       v.avg_daily_units_instock,
       v.instock_days_in_window,
       ROUND(SAFE_DIVIDE(c.stock_units, v.avg_daily_units_instock), 1) AS days_of_stock_remaining,
       CASE
         WHEN c.stock_units = 0 THEN 'ALREADY_OOS'
         WHEN v.avg_daily_units_instock IS NULL OR v.avg_daily_units_instock = 0 THEN 'NO_RECENT_SALES'
         WHEN SAFE_DIVIDE(c.stock_units, v.avg_daily_units_instock) < 14 THEN 'AT_RISK'  -- reorder point; override per prompt
         ELSE 'OK'
       END AS risk_status
FROM best_sellers b
JOIN current_stock c USING (ean)
LEFT JOIN velocity v USING (ean)
ORDER BY days_of_stock_remaining ASC;
```

## Workflow

1. **Load the model.** Read `/ras:ras-analysis`'s landmines plus the ones above. Confirm
   `fact_stock_daily`'s date grain and join key (EAN) against `fact_product_performance_daily` before
   writing anything else.
2. **Fix the window and thresholds.** Default: trailing 30 days for velocity, top 20 by units sold,
   14-day reorder point. Apply any overrides named in the prompt and state them in the output.
3. **Run Step 1** for the best-seller cut, **Step 2** for velocity, **Step 3** for the current stock
   snapshot. Keep all SQL.
4. **Run the sanity checks.** Especially velocity sample size and stock-snapshot freshness. Downgrade
   or flag any EAN that fails them rather than reporting a shaky projection at full confidence.
5. **Classify every best-seller** into exactly one of: already out of stock, at risk, no recent sales,
   OK. Verify the counts sum correctly.
6. **Draft with confidence labels.** Verified: current stock, historical sales pace, best-seller
   ranking. Inferred: days-of-stock-remaining, at-risk classification. Unknown: any EAN with no stock
   coverage at all.
7. **Verification pass (client-facing).** Re-derive one or two EANs by hand from source; check the
   velocity window excluded OOS days, and that the reorder point actually applied matches what the
   prompt asked for (or the 14-day default).
8. **Deliver.** The best-seller cut and window used, the reorder point applied, the at-risk list
   ranked by days-of-stock-remaining ascending, the already-out-of-stock list, and the no-recent-sales
   list reported separately. End with methodology + Sources.
9. **Feed gaps back.** If stock-snapshot staleness or coverage gaps force frequent low-confidence
   flags, raise it as a data-backlog item rather than leaning on a stale snapshot every time.

## Confidence tags — quick reference

| Tag | Means | How to phrase |
|-----|-------|---------------|
| Verified | Ran the query / read the source | State it plainly, keep the query |
| Inferred | Deduced from verified facts | "Inferred: … — rests on …" |
| Unknown | Not in available data | "Not available in RAS; to answer we'd need …" |

## Standard deliverable footer (adapt per account)

> **Methodology & defensibility:** Best-sellers = top N EANs by units sold (`ordered_pcs`, order-date)
> over the trailing window (default: top 20, 30 days). Sales velocity = average daily units sold per
> EAN over that window, restricted to days with `stock_units > 0`. Current stock = most recent
> `fact_stock_daily` snapshot per EAN. Days of stock remaining = current stock ÷ sales velocity — an
> **inferred, forward-looking projection**, not a verified fact; it assumes recent sales pace
> continues unchanged. Reorder point (default 14 days) is the days-remaining threshold below which a
> product is flagged at risk. Already-out-of-stock and no-recent-sales EANs are reported separately
> from the at-risk ranking. Filters: `sales_channel='E-COM'`, `is_service=0`, EAN-level.
>
> **Sources:** `<project>.<dataset>` — fact_stock_daily, fact_product_performance_daily.
