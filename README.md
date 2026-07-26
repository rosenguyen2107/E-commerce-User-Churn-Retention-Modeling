# E-commerce Customer Retention Analysis

**From behavioural signals to a value-based retention strategy** — an end-to-end analytics
case study on 232,079 shoppers across five months of the REES46 cosmetics-shop dataset:
EDA → leakage-free churn model → segmentation → value-based targeting → financial impact →
A/B validation → implementation roadmap.

> **Slide deck:** https://github.com/rosenguyen2107/E-commerce-User-Churn-Retention-Modeling/blob/main/customer_retention_analysis_deck.pdf
> **Report:** https://github.com/rosenguyen2107/E-commerce-User-Churn-Retention-Modeling/blob/main/customer_retention_analysis_report.pdf
> **Notebook:** https://github.com/rosenguyen2107/E-commerce-User-Churn-Retention-Modeling/blob/main/Customer_Retention_Analysis.ipynb
---

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

**1. Who's about to leave.** Based on how someone behaved in their first month — how much they
browsed, how many times they visited, whether they explored widely or fixated on one item — the
computer learns which kinds of people tend to come back and which tend to disappear. It predicts
this reasonably well.

**2. A counter-intuitive discovery.** The segment that leaves the *most* (91% never return) is
actually… **barely worth saving**, because they spend almost nothing. Meanwhile, the regulars who
buy steadily leave less often — but because they spend a lot, **losing them is what actually
costs money.** Measured in dollars, one single segment accounts for **~$400k of the $572k of
"revenue at risk"** — more than the other five segments combined.

**3. Where the retention money should go.** Instead of sending offers to the group that churns
most (expensive, and it rescues almost no revenue), aim at the group that is **both at risk of
leaving and spending real money.** Done right, it costs about **5 cents to save $1** of revenue.
Done wrong, that same $1 costs **$99**.

**In one line:** *this project teaches a computer to predict who's about to leave the store, then
works out who's worth spending money to keep.*

Now the details.

---

## Business question

> **Which users active in October will *not return* over the next 60 days, and where should
> limited retention budget be spent?**

On a browse-heavy, low-ticket store, value depends on repeat visits — so **retention, not
acquisition, is the binding constraint.**

## Headline results

| | |
|---|---|
| **Churn base rate** | 77% of October shoppers do not return within 60 days |
| **Model** | ROC-AUC **0.82** (gradient boosting), beating a recency-only baseline (0.74); stable at ±0.001 across 5-fold CV |
| **Revenue at risk** | **$572k** over 60 days — **98% concentrated in two of six segments** |
| **Targeting** | A value-based rule defends **$1 of at-risk revenue for ~$0.05**, vs ~$99 for a naive churn-first campaign |

---

## Data overview

**Source** — REES46 *eCommerce Events History in a Cosmetics Shop*
([Kaggle](https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-cosmetics-shop)),
a real multi-month behavioural log from an online cosmetics retailer.

**Coverage** — October 2019 → February 2020 (5 monthly CSVs, ~20M events). This analysis uses
**October as the feature window** and **November–December as the label window**.

**Grain** — one row per user interaction ("event"):

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

**Why this dataset works for churn** — it spans multiple months with stable `user_id`s, so a
genuine 60-day non-return outcome can be *observed* rather than assumed. A single-month dataset
cannot support a 30-day churn label at all (see leakage, below) — that is precisely the trap this
project was rebuilt to avoid.

---

## Glossary: the terms used here

### E-commerce

- **Event** — one recorded action by a shopper. The four types form the **conversion funnel** (a
  funnel because fewer people reach each stage): `view` (browsed a product) → `cart` (added it,
  so intent exists) → `remove_from_cart` (changed their mind) → `purchase` (actually bought).
  Here **58% of viewers add to cart, but only 19% of carts become orders** — the cart→purchase
  step is the biggest leak, and the one most worth fixing, because those people already wanted
  the item.
- **Churn** — leaving. For subscriptions (Netflix, Spotify) it means cancelling. This store has
  no subscription, so **churn = non-return within 60 days**. Worth stating explicitly: a softer
  notion than "cancelled".
- **Retention** — the opposite of churn. **Cohort retention** = take everyone who first arrived
  in the same week, then track what share come back in later weeks. Here it falls off a cliff
  after week one.
- **Session** — one visit (open the site, do something, leave). The median user has exactly
  **one** — most "looked once and left", no habit formed.
- **Segment / persona** — a group of customers who behave alike ("steady repeat buyers",
  "one-time visitors", "browsers who never buy").
- **CLV (Customer Lifetime Value)** — total value a customer brings over time. This project uses
  October spend as a **proxy** for it.
- **Revenue at risk** — churn probability × number of users × average spend. This single number
  is what exposed the counter-intuitive finding above.

### Machine learning

*Condensed here — full line-by-line detail in [`CODE_WALKTHROUGH.md`](CODE_WALKTHROUGH.md).*

- **Binary classification** — for each customer, predict 1 (will leave) or 0 (will return). The
  model learns the pattern from hundreds of thousands of past customers and their real outcomes.
- **Data leakage** — the biggest trap, and the one this project was rebuilt to avoid. Imagine
  predicting "will this student fail the exam?" but accidentally feeding the model *the exam
  score*. It scores 100% and is useless, because at prediction time that score doesn't exist yet.
  The churn version is subtler: if you define churn as "30 days inactive" **and** feed in "days
  since last activity", those two are **the same thing** — the model is just copying the answer
  key. The fix: a **feature window** (describe the customer using October only) and a **label
  window** (check whether they returned in Nov–Dec). Two disjoint periods → the features cannot
  see the future → no leakage.
- **Feature engineering** — turning raw events into numbers describing each customer: view count,
  sessions, active days, category breadth, time to first cart-add.
- **ROC-AUC 0.82** — given one churner and one returner, how often does the model score the
  churner higher? 0.5 = coin flip, 1.0 = perfect. **But** a recency-only rule already reaches
  0.74, so behaviour adds ~0.08 — stated plainly rather than oversold.
- **Cross-validation** — split the data 5 ways, rotate train/test, and confirm the result is
  stable (±0.001) rather than one lucky split.
- **Permutation importance** — shuffle one feature at random and see how much the model degrades.
  Big drop = important. Top three: recency, category breadth, active days.
- **Calibration** — making sure "80% churn risk" really means *~80 out of 100 such people churn*,
  so the probability can be trusted in a money decision (Brier 0.129 — already well-calibrated).
- **Value-based targeting** — contact a customer only when
  **P(churn) × customer value − contact cost > 0**. That is how budget lands on regulars ($81
  average spend) instead of one-time visitors ($2.56).
- **A/B test** — split customers randomly, give one group the offer, compare return rates. Since
  this data is historical with no real experiment, the project provides a **design + simulation**
  and says so, rather than pretending an experiment was run.

---

## Why this project is methodologically sound

- **No target leakage.** Features come from October only; churn is measured from a disjoint
  Nov–Dec window, so recency cannot leak the label.
- **Two models, two jobs.** An interpretable **logistic regression** (signed drivers) benchmarked
  against **HistGradientBoosting** (predictive ceiling, non-linearities).
- **Honest evaluation.** Class imbalance handled; reported with PR-AUC against a base-rate
  baseline, 5-fold cross-validation, and permutation importance with error bars — not accuracy
  alone.
- **Honest experimentation.** The data is observational, so the A/B section is a validated
  *design + power analysis + simulation*, clearly labelled — not a completed experiment.

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

Within-October **recency** (risk), **category breadth** and **active days** (protective) —
consistent across both models and above their permutation-importance error bars.

## Repository structure

```
├── notebooks/           # end-to-end analysis notebook (EDA → model → targeting → A/B)
├── src/                 # standalone scripts (churn pipeline, A/B, validation, targeting, Excel export)
├── deliverables/        # consulting-style deck (.pptx) and report (.docx)
├── figures/             # exported charts
├── CODE_WALKTHROUGH.md  # line-by-line guide to the pipeline and why it's built that way
└── README.md
```

## How to run

1. Download the cosmetics-shop monthly CSVs (`2019-Oct` … `2020-Feb`) into `data/`
   (Kaggle: `mkechinov/ecommerce-events-history-in-cosmetics-shop`).
2. `pip install -r requirements.txt`
3. Open `notebooks/cosmetics_churn_report.ipynb`, set `DATA_DIR`, and run top-to-bottom.

*Tip:* convert the CSVs to Parquet once for much faster re-reads, and sample ~20% of `user_id`
while iterating.

## Limitations

"Churn" = non-return (not subscription cancellation) · no demographics or marketing-channel data,
so recommendations are product/UX levers only · the Nov–Dec label window overlaps Black Friday,
so the churn rate is a lower bound · dollar inputs to the targeting rule are illustrative
assumptions to which the targeted fraction is sensitive.

## Acknowledgment

Data-cleaning, GMV/KPI and RFM techniques follow common practice from public Kaggle EDA notebooks
on this dataset (yiquanxiao); the churn framing, leakage-free label design, modeling, targeting
and experimentation are original to this project.

---
*Data: REES46 Marketing Platform · Tooling: Python (pandas, scikit-learn, scipy, seaborn).*
