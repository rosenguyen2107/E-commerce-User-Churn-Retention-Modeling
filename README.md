# E-commerce Customer Retention Analysis

**From behavioural signals to a value-based retention strategy** — an end-to-end analytics
case study on 232,079 shoppers across five months of the REES46 cosmetics-shop dataset:
EDA → leakage-free churn model → segmentation → value-based targeting → financial impact →
A/B validation → implementation roadmap.

> **Slide deck:** https://github.com/rosenguyen2107/E-commerce-User-Churn-Retention-Modeling/blob/main/customer_retention_analysis_deck.pdf
> 
> **Report:** https://github.com/rosenguyen2107/E-commerce-User-Churn-Retention-Modeling/blob/main/customer_retention_analysis_report.pdf
> 
> **Notebook:** https://github.com/rosenguyen2107/E-commerce-User-Churn-Retention-Modeling/blob/main/Customer_Retention_Analysis.ipynb

## Project Overview

Imagine you run an online cosmetics store. Every day, hundreds of thousands of people drop in to
browse lipsticks, powders, creams. Your biggest headache isn't "how do we get new people in" -
it's that **most people visit once and vanish, never to return.** Concretely: out of every 100
people who visited in October, **77 never came back at all** over the next two months.

For a store like this, with cheap products and impulse purchases, money doesn't come from one big
order. It comes from customers coming back and buying again and again. So **keeping customers
matters more than attracting new ones.**

Which raises the question: *"Can we predict who's about to leave, in time to win them back? And
who should we spend our retention budget on?"*

This project answers exactly three things:

**1. Who's about to leave.** Based on how someone behaved in their first month: how much they
browsed, how many times they visited, whether they explored widely or fixated on one item, the
algorithm learns which kinds of people tend to come back and which tend to disappear. It predicts
this reasonably well.

**2. A counter-intuitive discovery.** The segment that leaves the most (91% never return) is
actually **barely worth saving**, because they spend almost nothing. Meanwhile, the regulars who
buy steadily leave less often, but because they spend a lot, **losing them is what actually
costs money.** Measured in dollars, one single segment accounts for **~$400k of the $572k of
"revenue at risk"** - more than the other five segments combined.

**3. Where the retention money should go.** Instead of sending offers to the group that churns
most (expensive, and it rescues almost no revenue), aim at the group that is **both at risk of
leaving and spending real money.** Done right, it costs about **5 cents to save $1** of revenue.
Done wrong, that same $1 costs **$99**.

## Business question

> **Which users active in October will not return over the next 60 days, and where should
> limited retention budget be spent?**

On a browse-heavy, low-ticket store, value depends on repeat visits, so **retention, not
acquisition, is the binding constraint.**

## Headline results

| | |
|---|---|
| **Churn base rate** | 77% of October shoppers do not return within 60 days |
| **Model** | ROC-AUC **0.82** (gradient boosting), beating a recency-only baseline (0.74); stable at ±0.001 across 5-fold CV |
| **Revenue at risk** | **$572k** over 60 days — **98% concentrated in two of six segments** |
| **Targeting** | A value-based rule defends **$1 of at-risk revenue for ~$0.05**, vs ~$99 for a naive churn-first campaign |


## Data overview

**Source** — REES46 *eCommerce Events History in a Cosmetics Shop*
([Kaggle](https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-cosmetics-shop)),
a real multi-month behavioural log from an online cosmetics retailer.

**Coverage:** October 2019 → February 2020 (5 monthly CSVs, ~20M events). This analysis uses
October as the feature window and November–December as the label window.

**Grain** - one row per user interaction ("event"):

| Column | Meaning |
|---|---|
| `event_time` | timestamp of the interaction (UTC) |
| `event_type` | `view` · `cart` · `remove_from_cart` · `purchase` |
| `product_id`, `category_id`, `category_code`, `brand` | what was interacted with |
| `price` | item price at the time of the event |
| `user_id` | the customer — stable across months, which is what makes churn measurable |
| `user_session` | one browsing visit |

**October at a glance (after cleaning)**

| Metric | Value |
|---|---|
| Events | **3.7M** (from 4.1M raw) |
| Users | **232,079** (from 399,664 raw, after dropping single-event users) |
| Event mix | view 1.69M · cart 1.20M · remove-from-cart 0.58M · purchase 0.24M |
| Buyers | **25,755** (~11% of users) |
| Median item price | **$4.11** — an impulse-priced category |
| Median buyer spend | **$32.38** |
| Median user | **1 session, 1 active day** — most shoppers form no habit at all |
| Churn base rate | **77.1%** |

**Cleaning decisions & data-quality notes** — each is a judgement call worth knowing:

- **213,811 duplicate events removed** (identical timestamp + user + product + event type).
- **Non-positive prices dropped** (~0.15% of rows, including some negatives).
- **Single-event users dropped** — too little behavioural signal to learn from; this is what
  takes the user count from 399,664 to 232,079.
- **`category_code` is ~98% missing** in this feed, so `category_id` (always present, 490
  distinct values) is used for category breadth instead.
- **`brand` is ~40% missing.** We back-fill from each product's modal brand — but honestly this
  recovers almost nothing here (127 of ~1.56M missing), because those products have no brand
  recorded anywhere. Kept for correctness, not impact.
- **The label window overlaps Black Friday.** November GMV spikes, so some "returns" are
  promo-chasing rather than organic habit — which makes the measured churn rate a **lower
  bound**. Stated openly rather than buried.

**Why this dataset works for churn:** It spans multiple months with stable `user_id`s, so a
genuine 60-day non-return outcome can be observed rather than assumed. A single-month dataset
cannot support a 30-day churn label at all (see leakage, below) — that is precisely the trap this
project was rebuilt to avoid.

## Pipeline

```
raw events → cleaning → feature engineering (Oct)
                              ↓
                     churn label (Nov–Dec)   ← disjoint window = no leakage
                              ↓
              logistic + gradient boosting → cross-validation → drivers
                              ↓
        segmentation (RFM + personas) → revenue at risk → calibration
                              ↓
              value-based targeting (EV > 0) → A/B design → roadmap
```

## Key drivers of churn

Within-October **recency** (risk), **category breadth** and **active days** (protective), they are
consistent across both models and above their permutation-importance error bars.

## How to run

1. Download the cosmetics-shop monthly CSVs (`2019-Oct` … `2020-Feb`) and put them into a `data/` folder
   (Kaggle: `mkechinov/ecommerce-events-history-in-cosmetics-shop`).
2. Install all necessary libraries.
3. Open `customer_retention_analysis.ipynb`, set `DATA_DIR`, and run top-to-bottom.

*Tip:* convert the CSVs to Parquet once for much faster re-reads, and sample ~20% of `user_id`
while iterating.

## Limitations

"Churn" = non-return (not subscription cancellation) · no demographics or marketing-channel data,
so recommendations are product/UX levers only · the Nov-Dec label window overlaps Black Friday,
so the churn rate is a lower bound · dollar inputs to the targeting rule are illustrative
assumptions to which the targeted fraction is sensitive.

---
*Data: REES46 Marketing Platform · Tooling: Python (pandas, scikit-learn, scipy, seaborn).*
