# Air — System Diagram

One picture of the model in [2_data_model.md](2_data_model.md). Read it **top to bottom as lineage, left to right as stage**:

- **Bands** (top → bottom) are the dbt layers — **staging** (one `stg_` model per source) → **intermediate** (only where a stage needs one) → **mart** → **semantic layer** → **BI**.
- **Within each band**, the columns are the four business stages — Acquisition, Activation & Engagement, Subscriptions, Sales Pipeline — each carried straight down its own column.

`enterprise_daily` is **not** a separate layer: it's just another mart in the marts column that happens to pre-aggregate the four stage marts and reach BI directly. Everything else — every rate and ratio — is computed at query time in the semantic layer, never materialized.

Color, not shape, carries meaning: staging · intermediate (dashed) · mart · enterprise mart · semantic layer · BI · shared aside.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "ui-monospace, SFMono-Regular, Consolas, monospace", "fontSize": "15px", "primaryColor": "#D3DEEF", "primaryBorderColor": "#2B4570", "primaryTextColor": "#16202F", "lineColor": "#8B96A6"}}}%%
flowchart TD
  subgraph STG["STAGING &nbsp;·&nbsp; one stg_ model per source"]
    direction LR
    raw_web["ACQUISITION<br/>stg_website_events"]
    raw_act["ACTIVATION &amp; ENGAGEMENT<br/>stg_logged_in_events · stg_workspaces"]
    raw_bill["SUBSCRIPTIONS<br/>stg_subscription_events"]
    raw_crm["SALES PIPELINE<br/>stg_sales_opportunities"]
  end

  subgraph INT["INTERMEDIATE &nbsp;·&nbsp; where a stage needs one"]
    direction LR
    int_web["sessionized<br/>1 row / event"]
    int_evt["event intermediates<br/>typed · 1 row / event"]
    int_mrr["subscriptions__mrr_movement<br/>6 MRR buckets vs prior period"]
  end

  subgraph MART["MARTS &nbsp;·&nbsp; entities &amp; state at a grain"]
    direction LR
    m_acq["web_visitors<br/>sessions"]
    m_act["workspaces · workspaces_daily<br/>users · users_daily"]
    m_sub["subscriptions<br/>subscriptions_daily"]
    m_sales["sales_opportunities<br/>sales_opportunity_history<br/>sales_opportunities_daily"]
    m_ent["enterprise_daily<br/>pre-aggregated across the four stage marts"]
  end

  sem["SEMANTIC LAYER — computed at query time<br/>NRR · GRR · engagement · activation · win rate · ARPA · ARR"]
  bi["BI TOOL — dashboards &amp; explores"]

  spine["date_spine (shared)<br/>every _daily table joins it"]
  docs["macros + schema YAML<br/>uniqueness · freshness · relationships tests<br/>owner &amp; SLA metadata"]

  %% staging → intermediate → mart (intermediate sits between the two bands)
  raw_web  --> int_web --> m_acq
  raw_act  --> int_evt --> m_act
  raw_act  --> m_act
  raw_bill --> m_sub
  raw_crm  --> m_sales

  %% plan_tier: single source is the billing feed (workspaces_daily fill-down)
  raw_bill -.-> m_act

  %% mrr_movement feeds NRR/GRR
  raw_bill --> int_mrr

  %% marts → semantic → BI (every rate/ratio)
  m_acq   --> sem
  m_act   --> sem
  m_sub   --> sem
  m_sales --> sem
  sem --> bi

  %% enterprise_daily: a mart that aggregates the marts, straight to BI
  m_acq   --> m_ent
  m_act   --> m_ent
  m_sub   --> m_ent
  m_sales --> m_ent

  %% shared date spine into the daily marts
  spine -.-> m_act
  spine -.-> m_sub
  spine -.-> m_sales
  spine -.-> m_ent
  docs -.-> STG
  docs -.-> MART

  classDef raw fill:#EEF0F4,stroke:#8892A6,color:#1A2029
  classDef intnode fill:#DCE6F2,stroke:#3D5A80,color:#16202F,stroke-dasharray: 4 3
  classDef mart fill:#D3DEEF,stroke:#2B4570,color:#16202F
  classDef entmart fill:#C4D4EC,stroke:#2B4570,color:#16202F,stroke-width:2px
  classDef semantic fill:#E9E1F5,stroke:#6B4E9E,color:#2A1F42,stroke-width:2px
  classDef bi fill:#DCEFEA,stroke:#2E8B79,color:#153F36,stroke-width:2px
  classDef note fill:#FBF0E1,stroke:#C1702F,color:#5C3714,stroke-dasharray: 3 3

  class raw_web,raw_act,raw_bill,raw_crm raw
  class int_web,int_evt,int_mrr intnode
  class m_acq,m_act,m_sub,m_sales mart
  class m_ent entmart
  class sem semantic
  class bi bi
  class spine,docs note

  style STG fill:#FAFBFC,stroke:#C3C9D4,color:#4A5568
  style INT fill:#F2F6FC,stroke:#8FA3C2,color:#2E4160
  style MART fill:#F0F5FC,stroke:#7C93BC,color:#223256
```
