# Air — Product Success KPIs (Deliverable 01)

A tiered list of KPIs and supporting metrics for evaluating product success, grouped into four stages of the customer journey: **Acquisition**, **Activation & Engagement**, **Subscriptions**, and **Sales**. Within each stage, **Primary** metrics are the dashboard headlines and **Supporting** metrics are the diagnostics that explain them.

**North Star — Engaged Workspaces:** the count of workspaces actively storing and interacting with their assets. This is the truest measure of value delivered — a team gets nothing from Air until it has assets in the workspace and is acting on them — and it leads the metrics that follow: engaged workspaces retain, and retained workspaces convert to **ARR**, the lagging business outcome the North Star ultimately drives.

---

## 1. Acquisition

### Primary

| Metric | Formula | Definition |
|---|---|---|
| Unique Visitors | Distinct visitor IDs in period | Distinct visitors in period; the marketing-driven volume input. |
| Signup Conversion Rate | Unique Visitors who auth ÷ Unique Visitors | Share of visitors who complete signup/auth. |

### Supporting

| Metric | Formula | Definition |
|---|---|---|
| Total Visits | Count of visit events | Raw visit volume, repeats included. |
| Signup Click Rate | Distinct visitors who click signup ÷ Unique Visitors | Signup intent. |
| Auth Completion Rate | Successful Auths ÷ Signup Clicks | Intent that converts to an account. |
| Returning Visitor Rate | Distinct returning visitors ÷ Unique Visitors | Re-engagement vs. new. |
| Engagement Rate | Engaged sessions ÷ Total sessions | GA4 definition — a session is engaged if it lasts >10s, has a conversion event, or has ≥ 2 pageviews. |

## 2. Activation & Engagement

### Primary

| Metric | Formula | Definition |
|---|---|---|
| Total Active Workspaces | Distinct workspaces with ≥ 1 event in period | Volume base for the stage; the denominator most rates run against. |
| Total Engaged Workspaces | Distinct workspaces with ≥ 1 value interaction (organize, comment, variant, share) in period | Active workspaces actually getting value, not just logged in. |
| Workspace Engagement Rate | Total Engaged Workspaces ÷ Total Active Workspaces | Quality of the active base. |
| Workspace Activation Rate | Activated workspaces ÷ New workspaces | % of new workspaces reaching the activation threshold. Threshold is derived empirically — the early behavior with the strongest predictive lift on retention/conversion — not a fixed assumption. |
| Workspace Retention Rate (W1/W2/W4) | Active workspaces in week _n_ ÷ creation-cohort workspaces | Cohort return rate by week. |

### Supporting

| Metric | Formula | Definition |
|---|---|---|
| Weekly Active Workspaces (WAW) | Distinct workspaces with ≥ 1 event in the week | Weekly active workspace base. |
| Monthly Active Workspaces (MAW) | Distinct workspaces with ≥ 1 event in the month | Monthly active workspace base. |
| Weekly Active Users (WAU) | Distinct users with ≥ 1 event in the week | Weekly active user base. |
| Monthly Active Users (MAU) | Distinct users with ≥ 1 event in the month | Monthly active user base. |
| Workspace Stickiness | Weekly Active Workspaces ÷ Monthly Active Workspaces | Engagement intensity — how habitual usage is. |
| Median Time to Activate | Median time from creation to activation | Speed to value. |
| Onboarding Completion Rate | Workspaces completing onboarding ÷ New workspaces | Onboarding funnel. |
| Assets per Workspace | Total assets added ÷ Active Workspaces | Depth of the core value loop. |

## 3. Subscriptions

### Primary

| Metric | Formula | Definition |
|---|---|---|
| Total Active Subscriptions | Distinct workspaces in a paid state | Workspaces currently paying. |
| Total ARR | Sum of current ARR across active subscriptions | Annualized recurring revenue. |
| Net Revenue Retention (NRR) | (Start ARR + expansion − contraction − churn) ÷ Start ARR | Revenue retained plus expansion, on a subscriber cohort. |
| Subscription Churn Rate | Subscriptions cancelled in period ÷ Active subscriptions at period start | Rate of paying workspaces lost (logo churn). |

### Supporting

| Metric | Formula | Definition |
|---|---|---|
| New Subscriptions | Distinct workspaces entering a paid state in period | Gross new logos. |
| Cancellations | Distinct workspaces moving to cancelled in period | Lost logos. |
| Subscription Retention Rate | Subs retained ÷ Subs at period start | Logo-level retention over the period. |
| Gross Revenue Retention Rate | (Start ARR − contraction − churn) ÷ Start ARR | Revenue kept before upsell. |
| Revenue Churn Rate | Churned ARR ÷ Start ARR | ARR lost to downgrades/cancellations. |
| Free-to-Paid Conversion Rate | Workspaces that ever subscribe ÷ Workspaces created | Funnel to revenue. |
| Median Time to Paid | Median time from workspace creation to first subscription | Speed to revenue. |
| ARPA | Total ARR ÷ Total Active Subscriptions | Average revenue per account. |
| Self-Serve Subscription Share | Subs initiated by user ÷ New Subscriptions | Self-serve vs. sales-led mix. |

*Revenue-movement terms (used in NRR, GRR, and the churn metrics above):*
- ***Start ARR*** *— total ARR of the subscriber cohort at the start of the period.*
- ***Expansion*** *— ARR gained from existing subscribers upgrading tiers (e.g. basic → advanced).*
- ***Contraction*** *— ARR lost from existing subscribers downgrading tiers.*
- ***Churn / Churned ARR*** *— ARR lost from subscribers cancelling entirely.*
- ***Retained*** *— subscribers (or ARR) active at both the start and end of the period.*

## 4. Sales pipeline

### Primary

| Metric | Formula | Definition |
|---|---|---|
| Win Rate | Closed Won ÷ (Closed Won + Closed Lost) | Share of closed opportunities won. |
| Sales-Led Conversion Rate | Workspaces reaching paid after opening an opportunity ÷ Workspaces that opened an opportunity | Of workspaces that engaged Sales (opened an opportunity), the share that converted to a paid subscription. |

### Supporting

| Metric | Formula | Definition |
|---|---|---|
| Open Pipeline | Count of opportunities in progress | Live deals at snapshot. |
| New Opportunities Created | Count of opportunities created in period | Pipeline inflow. |
| Opportunities Closed | Count of opportunities closed in period | Won + lost outflow. |
| Median Sales Cycle Time | Median time from opportunity created to closed | Deal velocity (won/lost tracked separately). |
| Opportunity-to-Subscription Rate | Closed Won that fire a subscribed event ÷ Closed Won | Data-linkage integrity. |
