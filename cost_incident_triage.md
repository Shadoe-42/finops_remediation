# Cost Incident Triage
*Meridian Analytics — the first 48-72 hours of a runaway bill | July 2026*

---

## Purpose

`snowflake_compute_finops.md` (the sibling `data_sec` project's original doc) designs a cost model — sizing, auto-suspend, scaling policy, resource monitors, per-tenant attribution, all specified up front. This doc is what happens when someone calls with a bill that's already run away: the design assumptions either weren't implemented, drifted, or were never enough. Triage isn't redesign. It's finding out what's actually driving spend, fast, before touching anything permanent.

---

## Where to Look First

Two Account Usage views answer almost every triage question, and the discipline is starting with them before opening a support ticket or guessing:

**`WAREHOUSE_METERING_HISTORY`** — credit consumption by warehouse, by hour. The first query in any triage, because it answers the only question that matters in the first ten minutes: *which warehouse*.

```sql
SELECT
    warehouse_name,
    DATE_TRUNC('day', start_time) AS day,
    SUM(credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -14, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 2 DESC, 3 DESC;
```

**`QUERY_HISTORY`** — once the warehouse is identified, this is where the actual cause lives: a single runaway query, a loop, a scheduled task firing far more often than intended, or a genuine and sustained increase in legitimate load.

```sql
SELECT
    query_id,
    user_name,
    warehouse_name,
    query_text,
    total_elapsed_time / 1000 AS seconds,
    credits_used_cloud_services
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = '<warehouse from the previous query>'
  AND start_time >= DATEADD('day', -3, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 50;
```

**Native anomaly detection, if it's already on.** Snowflake's cost anomaly detection reached general availability in December 2025 and now runs daily against the trailing 28 days of billing data, decomposing spend into trend and weekly seasonality and flagging days that deviate from the forecast — and as of the February 2026 access expansion, this is visible to more than just account admins, meaning a properly configured account may have already flagged the spike before the client called. Worth checking `ACCOUNT_USAGE.COST_ANOMALIES` (or the Cost Management view in Snowsight) as a first step, not a last resort — if it's on, someone already did part of the diagnosis. If it isn't on, that absence is itself a root-cause finding for `root_cause_patterns.md`.

---

## Blast Radius, Not Just Root Cause

Identifying the one query or warehouse that spiked is necessary but incomplete — triage also has to answer whether this is isolated or symptomatic. Two follow-up questions, every time:

**Is this one tenant, or systemic?** If Meridian's per-tenant cost attribution (already specified in `snowflake_compute_finops.md`, but frequently the first thing skipped under deadline pressure on a real account) was actually implemented, this is a fast query. If it wasn't, this is the moment that gap gets discovered the expensive way — and it's worth stating plainly in the incident report rather than glossing over, since "we could not immediately attribute this spike to a tenant" is itself the finding that justifies the tagging and attribution work in `remediation_and_governance.md`.

**Is this still happening, or did it already stop?** A query that ran once and finished is a different response than a scheduled task still firing every five minutes. The former needs investigation; the latter needs an immediate lever pulled — see below — before investigation continues, because every hour of triage is another hour of spend on an active leak.

---

## Immediate Levers

Stop-the-bleeding actions, in order of how disruptive they are — reach for the least disruptive one that actually solves the immediate problem, not the most aggressive one by default:

1. **Cancel the specific runaway query**, if one is actively running (`SYSTEM$CANCEL_QUERY` or the Snowsight UI). Zero impact on anything else using the warehouse.
2. **Suspend the specific warehouse**, if the problem is warehouse-wide rather than one query (a misconfigured task looping, for instance). Impacts every workload on that warehouse, so confirm blast radius first.
3. **Apply or tighten a resource monitor** on the affected warehouse with an immediate `SUSPEND_IMMEDIATE` trigger, if this is likely to recur before a permanent fix lands. A temporary hard ceiling, not the permanent governance design — that's `remediation_and_governance.md`.
4. **Account-level warehouse suspension**, reserved for a genuine emergency where the source can't be isolated fast enough and the bleeding has to stop account-wide. This is the most disruptive option on the list and should be treated as a last resort, not a first instinct, precisely because it stops legitimate work along with the problem.

None of these are the fix. They're what buys the time to design the fix properly, which is the entire point of separating this doc from `remediation_and_governance.md` rather than collapsing triage and permanent remediation into one rushed pass.

---

## Sources

- Internal: `root_cause_patterns.md`, `remediation_and_governance.md`, `snowflake_compute_finops.md` (sibling repo `data_sec`)
- [Introduction to cost anomalies](https://docs.snowflake.com/en/user-guide/cost-anomalies)
- [Detect Spending Patterns with Anomaly Insights](https://www.snowflake.com/en/engineering-blog/anomaly-insights-spending-patterns/)
