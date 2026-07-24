# Air — System Diagram

Lineage from [data_model.md](data_model.md): raw → staging → intermediate → marts → semantic layer → BI.

Shapes: ▭ materialized table · ⬡ computed at query time, not stored · ⬭ application. Dashed = macro/YAML documentation, aside.

## Overview

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "ui-monospace, SFMono-Regular, Consolas, monospace", "fontSize": "20px", "primaryColor": "#D3DEEF", "primaryBorderColor": "#2B4570", "primaryTextColor": "#16202F", "lineColor": "#8B96A6"}}}%%
flowchart TD
  raw["RAW SOURCES<br/>web · product events · CDC · billing · CRM"]
  staging["STAGING<br/>clean · recast · dedupe, 1:1"]
  intermediate["INTERMEDIATE<br/>reusable joins &amp; rollups"]
  marts["MARTS<br/>acquisition · activation · subscriptions · sales · enterprise rollup"]
  semantic{{"SEMANTIC LAYER<br/>NRR · GRR · rates — computed at query time"}}
  bi(["BI TOOL<br/>dashboards &amp; explores"])

  raw --> staging --> intermediate --> marts --> semantic --> bi
  marts -->|"enterprise_daily: pre-aggregated mart, direct"| bi

  classDef raw fill:#EEF0F4,stroke:#8892A6,color:#1A2029
  classDef staging fill:#E4EAF3,stroke:#5B7091,color:#1A2029
  classDef intnode fill:#DCE6F2,stroke:#3D5A80,color:#16202F,stroke-dasharray: 4 3
  classDef mart fill:#D3DEEF,stroke:#2B4570,color:#16202F
  classDef semantic fill:#E9E1F5,stroke:#6B4E9E,color:#2A1F42,stroke-width:2px
  classDef bi fill:#DCEFEA,stroke:#2E8B79,color:#153F36,stroke-width:2px

  class raw raw
  class staging staging
  class intermediate intnode
  class marts mart
  class semantic semantic
  class bi bi
```

## Upstream — raw → staging → intermediate

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "ui-monospace, SFMono-Regular, Consolas, monospace", "fontSize": "16px", "primaryColor": "#D3DEEF", "primaryBorderColor": "#2B4570", "primaryTextColor": "#16202F", "lineColor": "#8B96A6"}}}%%
flowchart TD
  subgraph RAW["RAW SOURCES — landed by ingestion, untouched"]
    direction LR
    raw_web[["web clickstream<br/>GA4 / Segment · 1 row / hit"]]
    raw_evt[["in-product events<br/>1 row / action"]]
    raw_cdc[["app DB CDC<br/>workspace · user · membership"]]
    raw_bill[["subscription lifecycle<br/>billing (Stripe) · 1 row / change"]]
    raw_crm[["CRM pipeline<br/>Salesforce · deal + stage change"]]
  end

  subgraph STG["STAGING — rename, recast, clean · dedupe once"]
    direction LR
    stg_web["stg_web__clickstream"]
    stg_evt["stg_product__events"]
    stg_cdc["stg_app__workspaces / users / members"]
    stg_bill["billing__subscription_events"]
    stg_crm["stg_crm__opportunities"]
  end

  subgraph INT["INTERMEDIATE"]
    int_evt["event intermediates<br/>deduped · typed · 1 row / event"]
  end

  note_docs>"macros + schema YAML<br/>date_spine · uniqueness/freshness tests · owner &amp; SLA metadata"]

  raw_web --> stg_web --> int_evt
  raw_evt --> stg_evt --> int_evt
  raw_cdc --> stg_cdc
  raw_bill --> stg_bill
  raw_crm --> stg_crm

  note_docs -.-> STG

  classDef raw fill:#EEF0F4,stroke:#8892A6,color:#1A2029
  classDef staging fill:#E4EAF3,stroke:#5B7091,color:#1A2029
  classDef intnode fill:#DCE6F2,stroke:#3D5A80,color:#16202F,stroke-dasharray: 4 3
  classDef note fill:#FBF0E1,stroke:#C1702F,color:#5C3714,stroke-dasharray: 3 3

  class raw_web,raw_evt,raw_cdc,raw_bill,raw_crm raw
  class stg_web,stg_evt,stg_cdc,stg_bill,stg_crm staging
  class int_evt intnode
  class note_docs note

  style RAW fill:#FAFBFC,stroke:#C3C9D4,color:#4A5568
  style STG fill:#F5F8FC,stroke:#AAB8CC,color:#3B4A63
  style INT fill:#F2F6FC,stroke:#8FA3C2,color:#2E4160
```

## Downstream — marts → semantic layer → BI

`enterprise_daily` is a mart — it aggregates the marts beside it and reaches BI directly; every other rate/ratio goes through the semantic layer.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "ui-monospace, SFMono-Regular, Consolas, monospace", "fontSize": "16px", "primaryColor": "#D3DEEF", "primaryBorderColor": "#2B4570", "primaryTextColor": "#16202F", "lineColor": "#8B96A6"}}}%%
flowchart TD
  subgraph MARTS["MARTS — entities &amp; state at a grain"]
    direction LR
    s1["Acquisition<br/>web_visitors · sessions"]
    s2["Activation &amp; Engagement<br/>workspaces(_daily) · users(_daily) · workspace_members"]
    s3["Subscriptions<br/>subscriptions(_daily) + mrr_movement (intermediate)"]
    s4["Sales Pipeline<br/>sales_opportunities(_daily/_history)"]
    ent["enterprise_daily<br/>pre-aggregated across the marts above"]
  end

  subgraph SEM["SEMANTIC LAYER — virtual, computed at query time"]
    sem{{"NRR · GRR · engagement rate<br/>activation rate · win rate · ARPA"}}
  end

  subgraph BI["BI TOOL"]
    bi(["dashboards &amp; explores"])
  end

  s1 --> ent
  s2 --> ent
  s3 --> ent
  s4 --> ent

  s1 --> sem
  s2 --> sem
  s3 --> sem
  s4 --> sem

  sem --> bi
  ent -->|"pre-aggregated, direct"| bi

  classDef mart fill:#D3DEEF,stroke:#2B4570,color:#16202F
  classDef semantic fill:#E9E1F5,stroke:#6B4E9E,color:#2A1F42,stroke-width:2px
  classDef bi fill:#DCEFEA,stroke:#2E8B79,color:#153F36,stroke-width:2px

  class s1,s2,s3,s4,ent mart
  class sem semantic
  class bi bi

  style MARTS fill:#F0F5FC,stroke:#7C93BC,color:#223256
  style SEM fill:#F6F2FB,stroke:#9B7FC7,color:#40305F
  style BI fill:#EFF9F6,stroke:#5FAA98,color:#1D4B41
```
