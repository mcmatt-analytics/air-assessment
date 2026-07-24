# Air — Supporting Data Model

Downstream tables built on the raw event, subscription, and CRM sources to serve the KPIs in metrics_list.md, organized by the same four stages.

Raw event streams are first transformed into intermediate tables — deduped, typed, one row per event — which the marts below build on. Most intermediate models aren't enumerated here; the ones carrying real logic (e.g. the MRR-movement rollup) are called out where they matter.

---

## Raw sources

Everything below is built from a handful of raw feeds landed by ingestion (CDC / Fivetran-style), untouched before staging: a web clickstream (GA4/Segment) at one row per hit — pageviews, visits, signup clicks, auth events; an in-product event stream at one row per action — asset added, organized, commented, variant created, shared; CDC copies of the app database's workspace, user, and workspace-membership (seat) records, one row per entity; a subscription lifecycle feed from the billing system (e.g. Stripe), one row per billing change, carrying plan tier and MRR changes; and CRM pipeline data (e.g. Salesforce) — one row per deal, plus one row per stage change. Staging cleans and dedupes each of these one-to-one; the marts never read raw directly.

**Plan tier has exactly one source: the subscription lifecycle feed.** The workspace record doesn't carry its own copy of it. Plan changes are billing events, so the subscription lifecycle is the system of record; every mart below that shows a workspace's plan tier — current or daily — reads it from that lineage, never from the workspace record, so the two can't disagree.

---

## Semantic layer

The marts below materialize **entities and their state at a grain** (current-state and daily). They do **not** materialize metrics. Rates and ratios — engagement rate, activation rate, NRR, win rate — are defined once in a semantic layer (dbt Semantic Layer / MetricFlow, Cube, LookML) that sits on top of these marts and compiles to SQL at query time.

So there is deliberately no `nrr` table. There is `subscriptions_daily` (and the movement rollup beneath it), and the semantic layer computes NRR from those components on demand. This keeps each metric defined in exactly one place, lets any metric be sliced by any dimension the marts expose without building new tables, and keeps the warehouse to entities rather than a combinatorial sprawl of pre-aggregated metric tables. The one deliberate exception is the enterprise daily rollup (below) — a single wide snapshot materialized for dashboard load speed. A worked example is in the appendix.

---

## dbt conventions

- **Layering.** staging (one model per raw source: rename, recast, clean — never joined) → intermediate (reusable joins and rollups) → marts (the tables below). Rebuilds flow one direction only.
- **Naming.** Staging as source__entity (e.g. `billing__subscription_events`); intermediate as entity__verb. Marts named for entity and grain — plural entity for current state, entity_daily for date-grain state, entity_events for event streams (entity_history where the stream is a specific state-transition log, like opportunity stages, rather than raw activity).
- **Deduplication happens once, in staging.** Event streams carry replays and late arrivals; each staging model dedupes to one row per natural key so nothing downstream re-solves it.
- **Cost awareness.** Partition/cluster on the date column. Full rebuilds are the default; the daily models can build incrementally where a nightly rebuild would be too costly.
- **Testing.** Uniqueness and not-null on keys, relationships on foreign keys, and source freshness, plus the stage-specific checks noted below.

---

## Data quality

The per-mart checks listed under each stage are the concrete tests; this section is the framework they run under.

**Layered tests.** Generic tests (`unique`, `not_null`, `relationships`, `accepted_values`) live in schema YAML at every layer. Cheap key and freshness tests run at staging so bad data fails fast, before it fans out into joins.

**Freshness and volume.** Source freshness on every raw feed, plus row-count anomaly detection — a mart landing far below its trailing-average row count is flagged even when every individual row is valid.

**Late-arriving data.** Full rebuilds are preferred where reasonable — they absorb replays and late events for free. Incremental models are reserved for cases where a nightly full rebuild is too costly.

**On failure.** A failing test halts the DAG at that node; downstream marts serve their last-good build rather than a partial one. Failures notify the owning team — each mart carries an owner and a freshness SLA in its schema metadata.

---

## Shared

### date_spine

A one-column calendar every daily table joins to, so a metric exists for every day including days with zero activity.

```sql
-- macros/date_spine.sql  (dbt_utils.date_spine wrapper)
{{ dbt_utils.date_spine(
     datepart="day",
     start_date="cast('2021-01-01' as date)",
     end_date="date_add(current_date, interval 1 day)"
) }}
```

---

## Stage 1 — Acquisition

### web_visitors

One row per visitor: first-seen timestamp, acquisition channel, total events/visits, `is_signup_click`, `is_authenticated`, `is_engaged` (GA4 rule), and `is_returning` flags, and the visitor-to-user id crosswalk (`user_id`, populated once the visitor authenticates). Serves unique visitors, signup/auth conversion, returning-visitor and engagement rates.

Because this is a lifetime rollup — one row per visitor, not per visitor-day — it can't answer "how many visitors were active on this specific date." Daily acquisition trends (unique visitors per day, new signups per day) are read from `sessions` instead, using each session's date.

### sessions

Session grain — session id, visitor id, session date, duration, pageviews, `is_engaged` flag — for session-level engagement, and for the day-by-day acquisition numbers `web_visitors`' lifetime grain can't provide.

**Data quality:** dedupe raw hits in staging; visitor id unique on the mart; channel validated against an accepted set; authenticated visitors reconcile to users.

## Stage 2 — Activation & Engagement

A current-state table and a daily table per entity.

### workspaces

One row per workspace, current state: creation timestamp, activation timestamp, `onboarding_completed_at`, `is_active` and `is_engaged` flags, `assets_to_date` (lifetime asset count), `seat_count`, `plan_tier`.

### workspaces_daily

One row per workspace × day, off the historical event stream and date spine: `is_active_today` and `is_engaged_today` flags, `assets_to_date` (cumulative — the same fill-down as `plan_tier`, so a day's value is a running count, not that day's increment), `active_seats`, and a creation-week cohort key. Powers WAW/MAW, engagement and activation rates over time, retention cohorts, and stickiness.

The daily tables record end-of-day state — the value as of one microsecond before midnight. Multiple changes can happen within a day; only the last one that day survives into the row.

Building the daily grain uses a gaps-and-islands fill-down: state is only written when it changes, so to know the state on a quiet day you carry the last known value forward — cross-join entities to the date spine, then take the last non-null value over the day window. Below, `plan_tier` fills down from the subscription lifecycle feed specifically (see the Raw sources note above on why there's exactly one source for it, not two). The same pattern drives `subscriptions_daily`.

```sql
-- marts/workspaces_daily.sql  (plan_tier fill-down; is_active_today / is_engaged_today follow the same pattern)
select
    d.date_day,
    w.workspace_id,
    coalesce(
      e.plan_tier,
      last_value(e.plan_tier ignore nulls) over (
        partition by w.workspace_id
        order by d.date_day
        rows between unbounded preceding and current row
      )
    ) as plan_tier
from date_spine d
cross join {{ ref('workspaces') }} w
left join {{ ref('billing__subscription_events') }} e
       on e.workspace_id = w.workspace_id
      and e.event_date   = d.date_day
where d.date_day between w.created_date and current_date
```

### users and users_daily

The same current + daily pattern at user grain, for WAU/MAU, seat engagement, and multiplayer analysis.

### workspace_members

Membership/seat table mapping users to workspaces — named to match the raw feed it's built on (the workspace-membership CDC records above) rather than the more ambiguous "workspace_users," which reads like it could mean either direction of the relationship. This is the join seat-engagement and multiplayer-rate metrics need, which events alone don't give cleanly.

**Data quality:** one row per grain (workspace, and workspace × day); `is_engaged_today` implies `is_active_today` on every daily row; daily active counts reconcile to the raw event stream on a sample of days.

## Stage 3 — Subscriptions

Same current + daily pattern, at subscription grain.

### subscriptions

Current state per subscription: workspace id, status (trial/active/cancelled), `plan_tier`, current `mrr`, start and cancel dates. Only `mrr` is stored — ARR is `mrr × 12`, computed at query time rather than kept as a second column, for the same reason rates aren't materialized elsewhere in this model (see Semantic layer, above): a derived number stored next to the number it's derived from can only ever drift, never stay more correct. Answers active subs, total ARR, and ARPA as of now.

### subscriptions_daily

One row per subscription × day via the date spine and fill-down. With `plan_tier` and `mrr` carried forward across quiet days, it reports any subscription metric at any historical point: ARR trend, active subs over time, and the expansion/contraction/churn movement behind NRR, GRR, and the churn rates.

### subscriptions__mrr_movement  (intermediate)

The model NRR and GRR actually depend on, and the one worth making visible. For each subscription each period, it classifies how recurring revenue changed versus the prior period into one of six buckets — new, expansion, contraction, churn, reactivation, or flat (no change from the prior period) — by comparing that period's `mrr` to the prior period's. NRR/GRR are then just sums of these buckets over a cohort ÷ start ARR. Deriving the buckets once here (rather than re-deriving them inside every metric) is what keeps NRR, GRR, and revenue churn from drifting apart.

```sql
-- intermediate/subscriptions__mrr_movement.sql
with monthly as (
    select
        subscription_id,
        date_trunc(date_day, month) as month,
        max(mrr) as mrr                       -- end-of-month MRR from subscriptions_daily
    from {{ ref('subscriptions_daily') }}
    group by 1, 2
),
change as (
    select
        subscription_id,
        month,
        mrr,
        lag(mrr) over (partition by subscription_id order by month) as prev_mrr
    from monthly
)
select
    subscription_id,
    month,
    prev_mrr,
    mrr,
    mrr - coalesce(prev_mrr, 0) as mrr_delta,
    case
        when prev_mrr is null and mrr > 0        then 'new'
        when prev_mrr = 0     and mrr > 0        then 'reactivation'
        when mrr > prev_mrr                       then 'expansion'
        when mrr > 0 and mrr < prev_mrr           then 'contraction'
        when mrr = 0 and prev_mrr > 0             then 'churn'
        else 'flat'
    end as movement_type
from change
```

**Data quality:** valid status state machine (no post-cancel activity); `mrr` non-negative; ARR movement reconciles (start + expansion − contraction − churn = end); relationships back to workspaces.

## Stage 4 — Sales pipeline

### sales_opportunities

Current state of deals at opportunity grain: opportunity id, workspace id, `stage`, amount, created and closed dates, source. Won/lost/open status lives in `stage` alone, as a fixed set of terminal and non-terminal values — there's no separate `is_won`/`is_lost`/`is_closed` shadowing it, so there's nothing that can fall out of sync with the stage a deal is actually in. Serves win rate, open pipeline, new/closed counts, sales-cycle time, and — via workspace id — sales-led conversion and opportunity-to-subscription linkage.

### sales_opportunity_history

One row per stage transition — an event stream of stage changes — for pipeline history and time-in-stage velocity.

### sales_opportunities_daily

One row per opportunity × day, filled down from `sales_opportunity_history` the same way `workspaces_daily` and `subscriptions_daily` fill down from their own event sources: each day carries forward the stage as of the last transition on or before it. This is what "open pipeline over time" and the enterprise rollup below actually read from — `sales_opportunities` alone only tells you where a deal stands today, not where it stood on any past date.

**Data quality:** every won opportunity (`stage = closed_won`) reconciles to a subscribed event (this is the opportunity-to-subscription metric); closed date on or after created date; stage transitions are monotonic — a deal never revisits an earlier stage.

---

## Enterprise daily rollup

### enterprise_daily

One row per day: a pre-joined, pre-aggregated rollup across all four stages — the single wide table an executive dashboard reads so it never fans out across marts at query time. Totals, rates, and averages (unique visitors, active/engaged workspaces, ARR, ARPA, open pipeline) sit side by side on one date grain.

This is the deliberate exception to the semantic-layer rule up top. Because the exec view is fixed and read constantly, materializing it once a day beats recomputing through the semantic layer on every load, and it also freezes a point-in-time snapshot for the historical record. It sources only from the stage marts (never raw): `sessions` and `web_visitors` for daily visitor and signup counts, `workspaces_daily`, `subscriptions_daily`, and `sales_opportunities_daily`.

```sql
-- marts/enterprise_daily.sql   (partitioned on date_day)
with acquisition as (
    select
        session_date as date_day,
        count(distinct visitor_id)                          as unique_visitors
    from {{ ref('sessions') }}
    group by date_day
),
new_signups as (
    select
        first_seen_date as date_day,
        count(distinct visitor_id)                          as new_signups
    from {{ ref('web_visitors') }}
    where is_authenticated
    group by date_day
),
engagement as (
    select
        date_day,
        countif(is_active_today)  as active_workspaces,
        countif(is_engaged_today) as engaged_workspaces,
        sum(assets_added_today)   as assets_added
    from (
        select
            date_day,
            workspace_id,
            is_active_today,
            is_engaged_today,
            assets_to_date - coalesce(
              lag(assets_to_date) over (partition by workspace_id order by date_day), 0
            ) as assets_added_today                          -- daily delta off the cumulative column
        from {{ ref('workspaces_daily') }}
    )
    group by date_day
),
subs as (
    select
        date_day,
        countif(status = 'active')                          as active_subscriptions,
        sum(mrr) * 12                                       as total_arr
    from {{ ref('subscriptions_daily') }}
    group by date_day
),
pipeline as (
    select
        date_day,
        countif(stage not in ('closed_won', 'closed_lost'))            as open_opps,
        sum(if(stage not in ('closed_won', 'closed_lost'), amount, 0)) as open_pipeline_amount
    from {{ ref('sales_opportunities_daily') }}
    group by date_day
)
select
    d.date_day,
    -- acquisition
    a.unique_visitors,
    n.new_signups,
    safe_divide(n.new_signups, a.unique_visitors)          as signup_conversion_rate,
    -- activation & engagement
    e.active_workspaces,
    e.engaged_workspaces,
    safe_divide(e.engaged_workspaces, e.active_workspaces) as workspace_engagement_rate,
    e.assets_added,
    safe_divide(e.assets_added, e.active_workspaces)       as avg_assets_per_active_workspace,
    -- subscriptions
    s.active_subscriptions,
    s.total_arr,
    safe_divide(s.total_arr, s.active_subscriptions)       as arpa,
    -- sales
    p.open_opps,
    p.open_pipeline_amount
from {{ ref('date_spine') }} d
left join acquisition a using (date_day)
left join new_signups n using (date_day)
left join engagement  e using (date_day)
left join subs        s using (date_day)
left join pipeline    p using (date_day)
```

Rolling metrics (WAW/MAW, WAU/MAU, stickiness, trailing win rate) are added as window functions over `date_day` on top of these point-in-time columns.

**Data quality:** one row per day (unique and not-null on `date_day`, no gaps against the date spine); each column reconciles to its source mart on a sample of days.

---

## Appendix — a semantic layer over these marts

The NRR definition from Stage 3, expressed as a semantic-layer metric instead of a table. The measures are declared once against `subscriptions__mrr_movement`; the engine compiles them to SQL for whatever grain and slice is asked for. Syntax below is illustrative (dbt Semantic Layer / MetricFlow flavour) rather than drop-in config.

```yaml
# semantic_models/subscriptions.yml
semantic_model:
  name: subscription_movement
  model: ref('subscriptions__mrr_movement')
  dimensions:
    - name: month
      type: time
    - name: plan_tier
      type: categorical
  measures:
    - name: expansion
      agg: sum
      expr: case when movement_type = 'expansion'   then mrr_delta  else 0 end
    - name: contraction
      agg: sum
      expr: case when movement_type = 'contraction' then -mrr_delta else 0 end
    - name: churn
      agg: sum
      expr: case when movement_type = 'churn'       then -mrr_delta else 0 end
    - name: start_arr
      agg: sum
      expr: prev_mrr * 12

metrics:
  - name: nrr
    label: Net Revenue Retention
    type: ratio
    numerator:   start_arr + 12 * (expansion - contraction - churn)
    denominator: start_arr
```

Asking the layer for `nrr` grouped by `month, plan_tier` compiles to roughly:

```sql
select
    month,
    plan_tier,
    (start_arr + expansion - contraction - churn) / nullif(start_arr, 0) as nrr
from (
    select
        month,
        plan_tier,
        sum(prev_mrr) * 12                                                   as start_arr,
        sum(case when movement_type = 'expansion'   then mrr_delta  else 0 end) * 12 as expansion,
        sum(case when movement_type = 'contraction' then -mrr_delta else 0 end) * 12 as contraction,
        sum(case when movement_type = 'churn'       then -mrr_delta else 0 end) * 12 as churn
    from {{ ref('subscriptions__mrr_movement') }}
    group by 1, 2
) m
```

The same measures answer any slice — `nrr by month`, `nrr by plan_tier`, GRR (drop expansion from the numerator) — without a new table. Win-rate and engagement-rate metrics follow the identical pattern over `sales_opportunities` and `workspaces_daily`.
