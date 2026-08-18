---
name: ras-analysis
description: >-
  Enforces evidence-grounded answers when analysing client data in RAS / BigQuery. Use whenever analysing RAS or BigQuery client data, building a client-facing number, report, PDF or email, answering "why did metric X change", comparing periods, or interpreting fact_ tables and GA4 events. Every number must trace to a query; every claim is labelled verified / inferred / unknown; data gaps are never filled with a guess. Trigger even if the user does not say "be precise" — precision is the default for client-facing data work.
---

# RAS analysis — precision protocol

This is the standard for any analysis over data in RAS / BigQuery. It exists because
commercial decisions (budget shifts, stock buys, campaign changes) get made on these numbers.
A plausible-sounding guess that turns out wrong costs money and costs trust. The protocol is
simple: say what the data says, label how sure we are, and never dress up a guess as a fact.

## Core rules

1. **Ground every number in a query.** No figure reaches the user unless a query produced it in
   this session. Keep the SQL — for client-facing outputs, be ready to show it. If you're tempted
   to state a number from memory or from "how these tables usually work", stop and run the query.

2. **Label every non-trivial claim as verified, inferred, or unknown.**
   - **Verified** — came directly from a query you ran or a source document you read this session.
   - **Inferred** — a logical conclusion from verified facts, but not directly measured. Say so and
     state what it rests on ("if invoice lag is stable, then…").
   - **Unknown** — not answerable from available data. Say "not available" and state what it would
     take to answer it. Do not substitute a plausible guess.

3. **Never present inference as fact — especially about things you did not query.** Column meanings,
   attribution scope, external projects/datasets, whether a product is truly out of stock: if you
   didn't verify it, it's inferred or unknown. "I assume", "probably", "tuším" are signals to stop
   and either verify or downgrade the claim explicitly.

4. **Understand the data model before you interpret it.** A number can be arithmetically correct
   and still wrong if the column doesn't mean what you assumed. Check the data dictionary for date
   basis, attribution scope and coverage. When you put columns with different bases in one table
   (e.g. order-date revenue next to invoice-date margin), say so.

5. **Client-facing output gets a verification pass.** Before a report, PDF, email or dashboard
   number goes out, run a second adversarial check against source: re-derive the headline numbers,
   confirm each claim's label, and actively try to break the conclusion (wrong window? mixed date
   basis? attribution artefact? small-n?). Prefer a subagent for this on high-stakes work.

6. **Cite sources.** End client-facing and data answers with what they're built on — tables/queries
   run, documents or emails read.

## Default scope — channel and country

Unless the prompt says otherwise:

- **Channel defaults to `sales_channel = 'E-COM'`.** Only include other channels (B2B, wholesale,
  etc.) if the user names them explicitly.
- **No country filter by default.** Aggregate across all countries/markets. Only filter to, or
  break out by, a specific country if the user names one explicitly.

State the scope you applied in the answer (e.g. "Scope: E-COM, all countries — say so if you want
B2B or a specific market included") so the user can tell what was excluded by default.

## Confidence tags — quick reference

| Tag | Means | How to phrase |
|-----|-------|---------------|
| Verified | Ran the query / read the source | State it plainly, keep the query |
| Inferred | Deduced from verified facts | "Inferred: … — rests on …" |
| Unknown | Not in available data | "Not available in RAS; to answer we'd need …" |

## Known data-model facts (verified — treat as landmines)

These are confirmed properties of the RAS data model. They are the errors most likely to be made
if the tables are interpreted naively.

- **Mixed date basis inside the daily fact tables.** `orders`, `order_value`, `ordered_pcs` are on
  **order date**. `invoice_value`, `sold_pcs`, `gross_margin` are on **invoice / fulfilment date**.
  `sessions`, `cost`, `impressions` and GA4 events are on **calendar activity date**. Never compute
  "invoiced ÷ ordered" on a short window — the numerator and denominator describe different orders.
  Use a multi-week window, and label mixed-basis tables.
- **`net_sales`, `gross_margin`, `net_contribution` are estimated** (returns are estimated on the
  day). `orders` and `order_value` are actuals. Estimated fields carry a star in the report.
- **`fact_ga_events.traffic_source` / `traffic_medium` are user-scoped first-touch, NOT session.**
  Evidence: ~96 % of sessions carry a paid `cpc` source and only ~4 % `(direct)/(none)`, which is
  impossible for session-scoped attribution; 99.7 % of multi-session users show a single source.
  Consequence: this table cannot measure multi-touch assist. Last-click order attribution lives in
  the ecom report's `platform` field, which disagrees with GA first-touch — that gap is real, not a
  bug. There is **no** `session_source` / `session_medium` in the curated model.
- **The stock table does not cover every brand.** "No stock record" ≠ "out of stock". Confirm brand
  coverage before making any availability or purchasing claim.
- **EAN ↔ product_name mapping is not 1:1.** One product name can map to several EANs (one of which
  may be a component with large stock). Don't infer a set's stock from a name match.

## AI workflow

Follow these steps for any RAS / BigQuery data request.

### Step 1 — Load the model, not just the table
Read the connector's schema description and the "Known data-model facts" section above before
writing SQL. Confirm each column's date basis, attribution scope and coverage first. If the
semantics aren't documented, treat them as unknown and verify empirically before relying on them.

### Step 2 — Query for every number
Write SQL for each figure you'll report. Apply the default scope (`sales_channel = 'E-COM'`, no
country filter) unless the prompt named other channels or a specific country. Filter on the
partition column, name columns explicitly, exclude service items where relevant. Keep the query
text. If a needed field doesn't exist, that's
an **unknown** — record it and note what would be required (e.g. add `session_source` to the model).

### Step 3 — Draft with confidence labels
Write the answer tagging each non-trivial claim verified / inferred / unknown. Keep interpretation
visibly separate from measurement. When mixing columns of different date basis or attribution in one
view, say so in a caption.

### Step 4 — Verification pass (client-facing outputs)
Before it goes out, re-derive the headline numbers from source and try to break the conclusion:
wrong/short window? mixed date basis? attribution artefact? small sample? a claim about a system you
never queried? Downgrade or cut anything that doesn't survive. On high-stakes deliverables, run this
as a separate subagent.

### Step 5 — Deliver with sources
State the conclusion, then the caveats and limits, then a Sources line listing the tables/queries
and any documents or emails the answer rests on.

### Step 6 — Feed gaps back
When you hit an unknown that keeps recurring (a missing column, an unreliable mapping), flag it as a
data-backlog item rather than repeatedly working around it.
