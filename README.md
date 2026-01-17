# E-commerce User Churn & Retention Modeling
*A Product Analytics & Predictive Modeling Case Study*

## Project Overview
This project analyzes user behavior and churn dynamics in a large-scale e-commerce platform using event-level data.
The objective is to answer three core product questions:
1. Where do users drop off along the behavioral funnel from first view to purchase and repeat usage?
2. Which early-stage behavioral signals are the strongest predictors of 30-day churn?
3. Which product levers can most effectively improve long-term retention?
   
The project combines:
- Funnel and cohort analysis (descriptive)
- Churn prediction (logistic regression)
- Causal interpretation (Partial Dependence)
- Behavioral segmentation (value & engagement)
- Prescriptive business recommendations
- This structure mirrors how Product Analytics and Data Science teams support growth and retention decisions in real-world companies.

## Data Source
Dataset:

E-commerce Events History (October 2019)

Source: Kaggle – Multi-category online retail platform

Granularity: Event-level (view, cart, purchase)

Key columns:

| Column        | Description              |
| ------------- | ------------------------ |
| event_time    | Timestamp of user action |
| event_type    | view / cart / purchase   |
| user_id       | Unique user identifier   |
| user_session  | Session identifier       |
| product_id    | Product identifier       |
| category_code | Product category         |
| price         | Transaction price        |

This analysis focuses on a one-month window to study early activation, short-term retention, and churn dynamics. While longer horizons enable seasonality analysis, this time frame is sufficient to identify behavioral drivers of attrition and high-leverage product interventions.

## Methodology

**1. Data Processing**
   
Steps:

Parse timestamps and sort events chronologically per user.

Aggregate to user-level behavioral table.

Construct lifecycle features:

- `first_seen, last_seen`
- `n_views, n_carts, n_purchases`
- `n_sessions`
- `category_diversity`
- `avg_events_per_day`
- `days_to_first_cart`
- `recency_days`

These features capture:
- Activation speed (time to first intent)
- Engagement depth (sessions, intensity)
- Exploration breadth (category diversity)
- Habit strength (return frequency)
- Disengagement signals (recency)
  
**2. Funnel & Cohort Analysis**

Activation Funnel:

View → Cart → Purchase

Purpose:

- Identify where intent formation breaks down.
- Diagnose whether churn is driven by lack of traffic or lack of conversion.
  
Cohort Retention Heatmap:
Users grouped by first active week, tracked weekly.

Purpose:

- Separate structural churn from random noise.
- Observe early-lifecycle decay patterns.

**3. Churn Definition**

Churn label (`churn_30d`):

A user is considered churned if no activity is observed in the 30 days following their last interaction.
This aligns with industry practice in subscription, marketplace, and app analytics for short-cycle products.

**4. Predictive Modeling**

Model: Logistic Regression (standardized features)
- Interpretable coefficients (direction & magnitude)
- Direct mapping to business drivers
- Suitable for causal-style reasoning (unlike black-box models)

Feature set:
- `n_views`
- `n_carts`
- `n_purchases`
- `days_to_first_cart`
- `n_sessions`
- `category_diversity`
- `avg_events_per_day`
- `recency_days`

Missing values were imputed using business logic:
- No cart → long activation delay
- Single category → low exploration
- No repeat sessions → low habit strength

**5. Partial Dependence (Causal Interpretation)**

Partial Dependence Plot (PDP) was used to estimate:

“If time-to-first-cart decreases from 7 days to 2 days, how much does churn probability change, holding all other features constant?”

This converts correlation into actionable product insight.

**6. Segmentation**

Users were segmented into High-Value and Low-Value groups using:
- Total spend (monetary)
- Purchase frequency (frequency)

Churn drivers were compared across segments to design differentiated retention strategies.


## Key Insights

**1. Activation Funnel: View → Cart → Purchase**
   
<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/fa215184-b137-4761-a249-59cefd163225" />

A large proportion of users remain stuck at the browsing stage. The sharp drop from “view” to “cart” confirms that intent formation, rather than traffic acquisition, is the primary bottleneck. This aligns with the churn model, where delayed time-to-first-cart strongly increases churn risk.

**2. Cohort Retention Heatmap**

<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/f9e0723a-282c-4850-ac36-31588fb8bb2e" />
<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/efb112ac-9b07-4879-9aa9-950a63614a89" />


**Strong early drop-off across all cohorts**

All cohorts experience a sharp decline after the first week. For example, the earliest cohort (2019-09-30) drops from 100% to ~42% in Week 1 and further to ~20% by Week 4. This indicates that the majority of churn happens very early in the user lifecycle, suggesting that onboarding and first-week experience are the most critical phases for retention.

**Consistent retention pattern across cohorts**

Later cohorts (2019-10-07, 2019-10-14, 2019-10-21) display remarkably similar decay curves: retention falls to around 30% by Week 2 and stabilizes near 15–18% by Week 4. This consistency implies that churn is driven by structural product factors rather than cohort-specific shocks (e.g., seasonality or campaigns).

**Survival of a “core” user base**

After the initial steep decline, retention curves flatten, indicating the existence of a smaller but stable group of highly engaged users who form a long-term core. Once users pass the first 1–2 weeks, their probability of remaining active increases substantially.

**Business Interpretation:**

The product’s main challenge is not long-term loyalty, but early activation and habit formation. Interventions such as improving first-session experience, faster time-to-value, and nudging users to reach key milestones (e.g., first cart or first purchase) within the first few days are likely to yield the highest marginal gains in overall retention.

**3. Churn Driver Model (Logistic Regression Coefficients)**

<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/6be0c88d-0ff0-429d-9a28-544801dedc1a" />

**Browsing without commitment is a strong churn signal**

The number of product views (`n_views`) has the largest positive coefficient, meaning heavy browsing is associated with *higher* churn probability once recency is controlled for. This suggests a pattern of “window shoppers”: users who explore many items but fail to convert are more likely to drop out, possibly due to choice overload, price sensitivity, or weak value proposition.

**Session frequency is the strongest protective factor**

The number of sessions (`n_sessions`) has the largest negative coefficient. Users who return frequently—even if each session is short—are much less likely to churn. This indicates that habit formation and repeated touchpoints matter more than sheer volume of actions within a single session.

**Usage intensity reduces churn, but with diminishing impact**

Higher average events per day (`avg_events_per_day`) also lowers churn, though its effect is smaller than session frequency. This implies that consistent, distributed engagement across days is more valuable than bursty activity concentrated in a short time window.

**Funnel progression variables play a secondary role**

Variables such as `days_to_first_cart`, `n_carts`, `n_purchases`, and `category_diversity` all have relatively small coefficients. This suggests that, conditional on users being active, *behavioral rhythm* (how often they come back) is more predictive of retention than *how deep* they go in the purchase funnel in the early stage.

**Business Interpretation:**

Beyond simple “are they active recently?”, churn is primarily driven by *engagement quality*. Encouraging users to return frequently (e.g., via personalized recommendations, reminders, or daily-use features) may be more effective than only pushing immediate purchases. At the same time, reducing friction between viewing and adding to cart could convert high-view, low-commitment users before they disengage.

**4. Partial Dependence: Time to First Cart → Churn**

<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/6ab2ebe1-8c6c-4288-b77c-eb2527084f0e" />

Holding other behaviors constant, faster progression to first cart is associated with a lower predicted probability of churn. Although the average marginal effect is modest at the population level, it is economically meaningful for the subset of users on the activation margin, validating activation speed as a causal driver rather than a mere correlation.

Quantitatively, reducing the time to first cart from around 7 days to 2 days leads to a measurable (though modest) drop in average churn probability. This suggests that early progression in the funnel (moving from browsing to intent) is a key psychological commitment point. Users who delay their first cart action are more likely to remain in an exploratory, low-attachment state and eventually disengage.

From a product perspective, this highlights the importance of:
- Frictionless onboarding and discovery,
- Strong first-session recommendations,
- Early nudges (e.g., “Add to cart” prompts, personalized deals)

to accelerate the moment of first intent and reduce long-term churn.

**5. Behavioral Segmentation: Engagement vs Exploration**

<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/f928902b-1cf8-41a3-8ec4-45bf585cac4b" />


Users cluster into distinct behavioral segments. Low-session, low-diversity users exhibit the highest churn, corresponding to shallow exploration and weak habit formation. In contrast, users with frequent sessions and broad category exploration show substantially lower churn, supporting the hypothesis that both engagement depth and breadth are key drivers of long-term retention.

**6. Segment-Level Churn Drivers (High vs Low Value Users)**

| Feature             | High Value Users | Low Value Users |
| ------------------- | ---------------- | --------------- |
| Recency (days)      | **0.2444**       | **0.2878**      |
| Avg Events / Day    | 0.0593           | 0.0473          |
| Category Diversity  | -0.0566          | -0.0439         |
| Days to First Cart  | -0.0527          | -0.0441         |
| Number of Views     | -0.0399          | -0.0309         |
| Number of Sessions  | -0.0368          | **-0.0672**     |
| Number of Carts     | -0.0173          | -0.0103         |
| Number of Purchases | -0.0141          | 0.0029          |

*(Target: churn_30d; positive = higher churn risk, negative = lower churn risk)*

**High-Value Users**

For high-value users, the strongest churn driver is recency (0.24), indicating that disengagement is the primary signal of attrition. Once these users stop returning, their churn risk rises sharply.

Protective factors (negative correlations) are:

- Category diversity (-0.057): Broader exploration across product categories reduces churn, suggesting that discovery and variety reinforce long-term engagement.
- Time to first cart (-0.053): Faster progression to first intent is associated with better retention, even among already valuable customers.
- Session frequency (-0.037): Habit formation (coming back often) plays a meaningful role in keeping high-value users active.

Interestingly, average events per day shows a slight positive correlation with churn, implying that bursty, short-term activity does not necessarily translate into long-term loyalty for high-value users; sustained return behavior matters more than intensity in a single day.

High-value user churn is primarily a disengagement problem, not an activation problem. Retention strategies should therefore focus on re-engagement loops, habit reinforcement, and long-term relationship building (e.g., loyalty programs, personalized recommendations, and reminders).

**Low-Value Users**

For low-value users, recency is again the dominant churn signal (0.29), but the structure of secondary drivers differs:

- Number of sessions (-0.067) is the strongest protective factor, highlighting that repeated visits are critical for moving low-value users toward stickier behavior.
- Time to first cart (-0.044) and category diversity (-0.044) also reduce churn, indicating that early activation and exploration are central to preventing early drop-off.
- Purchase-related variables have near-zero correlation, meaning that a single transaction alone does not guarantee retention in this segment.

Low-value user churn is primarily an activation and habit formation problem. These users leave because they fail to quickly reach meaningful engagement milestones (first cart, repeated sessions, exploration).

**Strategic Comparison**

Churn dynamics differ structurally by user value segment.

For high-value users, churn is driven mainly by loss of engagement momentum (recency and session frequency), whereas for low-value users, churn is driven by failure to achieve early activation and exploration milestones.

This implies a segmented retention strategy:
- High-Value Users:
  Focus on re-engagement, loyalty, and long-term habit reinforcement.
- Low-Value Users:
  Focus on accelerating time-to-first-intent, onboarding optimization, and guided discovery to convert casual browsers into engaged users.

## Business Recommendations

**Accelerate First Intent**

- Improve onboarding personalization
- Reduce friction to add-to-cart
- Smart recommendations on first session

**Build Habit Loops**

- Price alerts
- Saved items
- Re-engagement notifications
- Session-triggered reminders

**Segment-Specific Strategy**

- High Value: Re-engagement campaigns, loyalty, habit reinforcement
- Low Value: Activation optimization, guided discovery, first-purchase incentives

## Conclusion

This analysis shows that user churn in the e-commerce platform is not primarily driven by lack of purchasing power or product interest, but by how quickly and deeply users reach their first moment of value.

From the funnel and cohort analysis, we observe a sharp drop-off after the initial viewing stage, with long-term retention stabilizing around 15–20% after four weeks. This indicates that most users either fail to form purchase intent early or disengage shortly after their first interaction.

The churn model and partial dependence analysis further reveal that **recency** and **speed to first cart (days_to_first_cart)** are the most structurally important drivers of 30-day churn. Users who take longer to reach their first cart exhibit consistently higher churn probability, even when controlling for total activity volume. This suggests that time-to-intent is more predictive than raw activity counts.

Segmentation analysis shows a similar pattern across both high-value and low-value users:

* Deeper engagement (more sessions, broader category exploration) is associated with lower churn.
* Recency dominates as the strongest churn signal, but behavioral depth acts as a protective factor.
* The mechanisms of churn are structurally similar across segments; the difference lies in magnitude, not direction.

Overall, the evidence supports a causal narrative:
**Churn is primarily a failure of early activation, not a failure of long-term monetization.**
Users who do not quickly discover relevant products, build intent, and form habitual usage patterns are far more likely to disengage permanently.

From a product perspective, the most effective retention levers are therefore:

1. Reducing time to first cart through better onboarding, recommendations, and search relevance.
2. Increasing early session depth and category exploration to accelerate learning and perceived value.
3. Triggering re-engagement before recency crosses the critical threshold identified by the model.

In short, improving retention is less about pushing more promotions, and more about compressing the time it takes for users to experience their first meaningful success in the product.

