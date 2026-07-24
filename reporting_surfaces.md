# Air — Reporting Surfaces & Rules of Engagement (Deliverable 04)

Where employees consume and analyze this data, and what each surface is (and isn't) for.

---

## The surface: a BI tool on top of the semantic layer

Employees don't query the marts directly. A BI tool (Looker or Omni) sits on top of the semantic layer defined in `data_model.md` and is the single point of consumption for everyone — analysts included. Each business area gets a dedicated explore, built on the metrics and dimensions that layer exposes, plus a small set of curated one-click dashboards for people who just need the number, not the exploration.

This matters for one reason: the semantic layer is where NRR, engagement rate, win rate, etc. are defined *once*. If an area's explore is built on the semantic layer's metrics rather than hand-rolled SQL or spreadsheet logic, two dashboards can never quietly disagree about what "engaged workspace" means. That consistency is the whole point of the model in `data_model.md` — the BI layer either preserves it or defeats it, depending on whether people build on top of the semantic layer or route around it.

---

## Reporting surfaces by stage

| Area | Explore built on | Primary users | What it's for | Watch-outs |
|---|---|---|---|---|
| Acquisition | `sessions`, `web_visitors` | Marketing, growth | Traffic, signup conversion, channel performance — top-of-funnel volume and quality. | Session-grain vs. visitor-grain (lifetime) numbers don't sum cleanly against each other — the explore should make clear which grain a given field is on. |
| Activation & Engagement | `workspaces`, `workspaces_daily`, `users`, `users_daily`, `workspace_members` | Product, growth, CS | Activation funnel, engagement/retention cohorts, feature adoption — the core product health surface. | Highest-traffic explore, most likely to sprawl into ungoverned custom fields; cohort logic (activation threshold) should stay defined once, not redefined per dashboard. |
| Subscriptions | `subscriptions`, `subscriptions_daily`, `subscriptions__mrr_movement` | RevOps, Finance, CS (renewals) | ARR, NRR/GRR, churn, ARPA — the revenue surface. | NRR/GRR are semantic-layer metrics, not stored columns — self-service users pulling raw MRR movement and recomputing NRR by hand is the failure mode to prevent. |
| Sales Pipeline | `sales_opportunities`, `sales_opportunity_history`, `sales_opportunities_daily` | Sales, Sales Ops | Win rate, open pipeline, sales-cycle time, funnel velocity. | Pipeline snapshots vs. history — "open pipeline today" and "open pipeline as of last quarter" pull from different grains; explore should default to the daily/point-in-time table to avoid silent as-of errors. |

Each area's explore is self-service within its own domain — filters, slices, and one-click visuals — but doesn't cross into another area's tables. A Sales user exploring pipeline shouldn't be able to casually join into subscription MRR and produce a number nobody on RevOps recognizes.

---

## Enterprise rollup: the executive / company-wide surface

`enterprise_daily` gets its own dashboard, separate from the four stage explores. It's for executives and anyone who wants a fast, high-level read on the business — not for diagnostic drill-down. One row per day, pre-aggregated across all four stages, built for load speed and a stable historical snapshot rather than exploration.

Two things to flag for anyone using it: it's a daily snapshot (not real-time — same-day numbers will lag intraday activity elsewhere), and it's intentionally shallow. When a number on it looks off or needs explaining, the follow-up question belongs in the relevant stage explore, not in this dashboard — `enterprise_daily` is where you notice something, not where you diagnose it.

---

## Rules of engagement, generally

Curated dashboards (enterprise rollup, and one per stage) are the answer for "what's the number" — locked fields, governed definitions, built for broad distribution. Explores are the answer for "why" — self-service within a domain, still grounded in semantic-layer metrics so ad hoc analysis doesn't drift from the governed numbers. Anyone building a recurring report that reaches for raw marts instead of the semantic layer's metrics is a signal to pull that logic into the semantic layer itself, not to let it live as one team's private definition.
