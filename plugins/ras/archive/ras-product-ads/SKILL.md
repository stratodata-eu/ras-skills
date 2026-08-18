---
name: ras-product-ads
description: >-
  Compares marketing costs from the product report (PLA-derived, per product) with PLA/PMAX
  campaign costs from the ecom performance report, to reveal how much of Google Performance Max
  budget flows to product listings (Shopping/PLA) versus non-product inventory (Display/video).
  Use when asked to analyse PLA vs Pmax spend split, "where is Pmax budget going", product-ads
  cost share, Shopping vs Display drift, or when a conversion-rate / traffic-quality drop needs to
  be traced to a shift in Google Ads spend mix. Ecommerce sales channel only; per country when there
  is more than one market; monthly aggregation; all months year-to-date. Every number
  traces to a query; every claim is labelled verified / inferred / unknown. Precision is the
  default — the output must survive an agency picking it apart.
---

# RAS product-ads — PLA vs PMAX spend-mix analysis

This skill measures **what share of Google Performance Max (Pmax) budget actually reaches product
listings (Shopping / PLA)** versus everything else Pmax buys (Display, video, and unattributed
inventory). A falling product-listing share is a strong, defensible signal that Google has drifted
the campaign into cheap non-product traffic — which shows up downstream as inflated sessions and a
collapsing conversion rate.

It exists because budget decisions get made on this number. The split between Shopping and Display
**inside** a Pmax campaign is not a manual setting — the advertiser cannot dial it directly — so
proving the drift from data is the only way to have the conversation. The output is frequently
challenged by the media agency, so **every figure must be reproducible and every interpretation
must stay inside what the data can defend.**

> This skill inherits the precision protocol of **`/ras:ras-analysis`** (ground every number in a
> query; label verified / inferred / unknown; understand the data model before interpreting; run a
> verification pass on client-facing output; cite sources). Read that skill's rules first — they
> apply here in full. Below are the additions specific to the PLA-vs-Pmax comparison.

## Scope (fixed for this analysis)

- **Ecommerce channel only.** Filter `sales_channel = 'E-COM'` in the product report; filter
  `platform = 'Google Ads'` (and the ecom channel) in the ecom-performance report. Exclude B2B and
  any other sales channel — they carry no Google Ads cost anyway (see landmines).
- **Per country.** If there is more than one market, report **one row set per country**.
  Never blend markets into a single number.
- **Monthly base aggregation, all months YTD.** Group by calendar month from the start of the
  current year (`DATE_TRUNC(CURRENT_DATE(), YEAR)`) to today. Flag the current month as partial.

## Known data-model facts (verified — treat as landmines)

These are confirmed properties of the RAS data model. They are the exact mistakes an agency will
use to discredit the analysis, so handle each explicitly.

- **`fact_product_performance` has NO native cost column.** Product-level ("PLA") spend must be
  **derived**: `SUM(gads_clicks * gads_average_cpc)`. This is the standard identity (spend = clicks
  × avg CPC) but it is a derivation — state it, and never call it a native cost. It is the
  Shopping/PLA-attributable portion of Google Ads spend (Google only reports item-level data for
  the Shopping part; Search text ads and Display/video are not attributed to products).
- **Product-level Google Ads cost exists only for `sales_channel = 'E-COM'`.** B2B and other
  channels have `NULL` gads metrics. Filtering to E-COM is mandatory, not optional.
- **The two reports use different geo keys.** `fact_product_performance` uses ISO `country_code`
  (e.g. `CZ`, `SK`). `fact_ecom_performance_agg` uses `site` (e.g. `shop.cz`, `shop.sk`) and
  has **no** country_code. Join per country by mapping `site` → ISO. A TLD extraction
  (`UPPER(REGEXP_EXTRACT(site, r'\.([a-z]+)$'))`) works for simple country-TLD shops but is
  **fragile** (`.com`, multi-market domains, subfolders). **Verify the site↔country mapping for the
  specific account before trusting the join** — this is the #1 thing to get wrong on a new account.
- **`campaign_type` values must be checked per account.** Pmax spend is `campaign_type = 'Pmax'`.
  If the account also runs **standalone Shopping** campaigns (a separate `campaign_type`), those also
  produce product-level cost — so the derived PLA cost would include them while a Pmax-only
  denominator would not, breaking the ratio. Run a `SELECT DISTINCT campaign_type` first; if
  standalone Shopping exists, either add it to the denominator or state the scope explicitly.
- **Date basis is clean for this comparison.** Both cost fields (`gads_*` in product report,
  `cost` in ecom report) are on **calendar / activity date**. Cost-to-cost monthly comparison does
  not mix date bases. (Do NOT pull margin/orders from the product report into the same view — those
  are invoice/order date; see `/ras:ras-analysis`.)
- **`is_service = 1` rows** (shipping etc.) carry no real product ad spend; exclude them.
- **EAN ↔ product_name is not 1:1** and the stock table doesn't cover every brand (per
  `/ras:ras-analysis`). Not needed for the spend-share number, but relevant if you drill to product
  level.

## The core metric

For each country × month:

- **PLA cost** = `SUM(gads_clicks * gads_average_cpc)` over `fact_product_performance` (E-COM,
  non-service).
- **Pmax cost** = `SUM(cost WHERE campaign_type='Pmax')` over `fact_ecom_performance_agg` (Google
  Ads, ecom).
- **PLA share of Pmax** = PLA cost ÷ Pmax cost.
- **Non-product share** = 1 − PLA share = the portion of Pmax going to Display/video + unattributed.

**Interpretation guardrails (defensibility):**
- Non-product share is **not provably 100 % Display** — it is Display + video + unattributed Pmax
  spend. The *direction* (less to product listings, more to non-product) is verifiable; the exact
  channel split is **not** in RAS (it lives in the Google Ads UI). Label it `INFERRED` and stop
  there.
- The Shopping/Display reallocation is executed **automatically by the Pmax algorithm** (`INFERRED`
  — the advertiser can't set it). It is normally **triggered** by an account-side change (budget
  increase, brand-exclusion / audience-signal changes, bid-strategy loosening such as max-clicks,
  feed issues, new image/video assets). Which specific trigger fired is `UNKNOWN` from RAS — it is
  in the Google Ads **change history**. Phrase for the client: *"Google runs the Shopping/Display
  split itself, but it was switched on by account changes — the exact cause needs the change
  history."*

## Sanity checks that make the output agency-proof

Run these every time and report the result:

1. **Subset consistency.** PLA cost must be **≤ Pmax cost** every month (typically ~30–45 %). If it
   ever exceeds Pmax cost, the numerator is capturing non-Pmax product spend (standalone Shopping)
   → fix the denominator scope before reporting.
2. **PLA cost ≤ total Google Ads cost** every month (a weaker bound; always true if #1 holds).
3. **Cross-check with clicks/cost-per-session.** Confirm the cost-share drop is corroborated by a
   click-share drop and/or a falling Pmax cost-per-session (cheap-inventory signature). One metric
   moving alone is weaker evidence than three moving together.
4. **Partial-month flag.** The current month is incomplete; never present it as a completed data
   point.

## SQL templates

Use your RAS BigQuery connector. Replace `<project>.<dataset>` with your RAS dataset. Keep the
query text with the deliverable.

**Step 0 — verify scope before trusting anything:**

```sql
-- Confirm channel/geo/campaign_type landscape for THIS account
SELECT 'product' src, sales_channel, country_code AS geo, NULL AS campaign_type
FROM `<project>.<dataset>.fact_product_performance`
WHERE date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
GROUP BY 1,2,3,4
UNION ALL
SELECT 'ecom', channel, site, campaign_type
FROM `<project>.<dataset>.fact_ecom_performance_agg`
WHERE platform='Google Ads' AND date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
GROUP BY 1,2,3,4
ORDER BY src, geo;
```

**Step 1 — the headline table (PLA share of Pmax, per country × month, YTD):**

```sql
WITH pla AS (   -- product-level PLA spend (derived = clicks * avg_cpc)
  SELECT country_code AS country,
         FORMAT_DATE('%Y-%m', date) AS month,
         SUM(gads_clicks * gads_average_cpc) AS pla_cost,
         SUM(gads_clicks)                    AS pla_clicks
  FROM `<project>.<dataset>.fact_product_performance`
  WHERE sales_channel = 'E-COM'
    AND is_service = 0
    AND date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
  GROUP BY country, month
),
gads AS (       -- campaign-level spend from the ecom report
  SELECT UPPER(REGEXP_EXTRACT(site, r'\.([a-z]+)$')) AS country,  -- VERIFY mapping per account
         FORMAT_DATE('%Y-%m', date) AS month,
         SUM(IF(campaign_type = 'Pmax', cost, 0)) AS pmax_cost,
         SUM(cost)                                AS gads_total_cost
  FROM `<project>.<dataset>.fact_ecom_performance_agg`
  WHERE platform = 'Google Ads'
    AND date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
  GROUP BY country, month
)
SELECT
  g.country,
  g.month,
  ROUND(p.pla_cost, 0)                                        AS pla_cost,
  ROUND(g.pmax_cost, 0)                                       AS pmax_cost,
  ROUND(g.gads_total_cost, 0)                                 AS gads_total_cost,
  ROUND(p.pla_cost / NULLIF(g.pmax_cost, 0) * 100, 1)         AS pla_share_of_pmax_pct,
  ROUND((g.pmax_cost - p.pla_cost) / NULLIF(g.pmax_cost, 0) * 100, 1) AS nonproduct_share_pct,
  -- defensibility flag: TRUE means numerator scope is wrong (see sanity check #1)
  p.pla_cost > g.pmax_cost                                    AS pla_exceeds_pmax_flag
FROM gads g
LEFT JOIN pla p USING (country, month)
ORDER BY g.country, g.month;
```

**Step 2 — corroborating cross-check (click share + Pmax cost-per-session):**

```sql
WITH pla AS (
  SELECT country_code AS country, FORMAT_DATE('%Y-%m', date) AS month,
         SUM(gads_clicks) AS pla_clicks
  FROM `<project>.<dataset>.fact_product_performance`
  WHERE sales_channel='E-COM' AND is_service=0 AND date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
  GROUP BY country, month
),
pmax AS (
  SELECT UPPER(REGEXP_EXTRACT(site, r'\.([a-z]+)$')) AS country,
         FORMAT_DATE('%Y-%m', date) AS month,
         SUM(clicks) AS pmax_clicks, SUM(sessions) AS pmax_sessions,
         SUM(orders) AS pmax_orders, SUM(cost) AS pmax_cost
  FROM `<project>.<dataset>.fact_ecom_performance_agg`
  WHERE platform='Google Ads' AND campaign_type='Pmax'
    AND date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
  GROUP BY country, month
)
SELECT x.country, x.month,
  ROUND(p.pla_clicks / NULLIF(x.pmax_clicks,0) * 100, 1) AS pla_click_share_pct,
  ROUND(x.pmax_cost / NULLIF(x.pmax_sessions,0), 2)      AS pmax_cost_per_session,
  ROUND(x.pmax_orders / NULLIF(x.pmax_sessions,0) * 100, 2) AS pmax_cr_pct
FROM pmax x LEFT JOIN pla p USING (country, month)
ORDER BY x.country, x.month;
```

**Step 3 (optional drill-down) — which Pmax campaigns drove the drift:**

```sql
SELECT site AS country_site, campaign, FORMAT_DATE('%Y-%m', date) AS month,
       SUM(sessions) sessions, SUM(orders) orders,
       ROUND(SUM(orders)/NULLIF(SUM(sessions),0)*100,2) cr_pct,
       ROUND(SUM(cost),0) cost,
       ROUND(SUM(cost)/NULLIF(SUM(sessions),0),2) cost_per_session
FROM `<project>.<dataset>.fact_ecom_performance_agg`
WHERE platform='Google Ads' AND campaign_type='Pmax'
  AND date >= DATE_TRUNC(CURRENT_DATE(), YEAR)
GROUP BY country_site, campaign, month
ORDER BY country_site, campaign, month;
```

## Workflow

1. **Load the model.** Read `/ras:ras-analysis` landmines + the ones above. Run **Step 0** and
   confirm for this account: which sales channels/countries exist, the `site` values, and the set of
   `campaign_type` values (is there standalone Shopping?).
2. **Verify the geo mapping.** Confirm `site` → `country_code` mapping is correct (don't assume the
   TLD trick works). Adjust the `REGEXP_EXTRACT` or use an explicit `CASE` map.
3. **Run Step 1** for the headline table, **Step 2** for corroboration. Keep the SQL.
4. **Run the sanity checks.** Especially #1 (PLA ≤ Pmax). If `pla_exceeds_pmax_flag` is ever TRUE,
   stop and fix denominator scope.
5. **Draft with confidence labels.** Verified: the monthly cost figures and shares. Inferred: the
   drift direction toward non-product/Display and that the split is Google-automatic. Unknown: the
   exact channel split and the account-side trigger (change history).
6. **Verification pass (client/agency-facing).** Re-derive one or two months from source; try to
   break it (wrong geo mapping? standalone Shopping in the numerator? partial month? a claim about
   Display you can't prove?). On high-stakes output, run this as a separate subagent.
7. **Deliver.** Per country: monthly table (PLA cost, Pmax cost, PLA share, non-product share),
   the trend, and a short verified/inferred/unknown read. End with the methodology + Sources block.
8. **Feed gaps back.** If the site↔country mapping or a missing native cost column keeps biting,
   raise it as a data-backlog item rather than working around it every time.

## Confidence tags — quick reference

| Tag | Means | How to phrase |
|-----|-------|---------------|
| Verified | Ran the query / read the source | State it plainly, keep the query |
| Inferred | Deduced from verified facts | "Inferred: … — rests on …" |
| Unknown | Not in available data | "Not available in RAS; to answer we'd need …" |

## Standard deliverable footer (adapt per account)

> **Methodology & defensibility:** Costs are activity-date; cost-to-cost monthly comparison, no
> mixed date basis. **PLA cost is derived** = Σ(gads_clicks × gads_average_cpc) in
> `fact_product_performance` (no native cost column). Sanity check: PLA cost is X–Y % of Pmax cost
> each month and never exceeds it → internally consistent. Non-product share = Display/video +
> unattributed Pmax spend (direction verifiable; exact channel split is in the Google Ads UI, not
> RAS). Filters: `sales_channel='E-COM'`, `is_service=0`, `platform='Google Ads'`,
> `campaign_type='Pmax'`. Geo per country via `site`→ISO mapping. Current month is partial.
>
> **Sources:** `<project>.<dataset>` — fact_product_performance, fact_ecom_performance_agg.
