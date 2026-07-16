# Remediation & Governance
*Meridian Analytics — fixing it so it doesn't come back | July 2026*

---

## Purpose

`cost_incident_triage.md` stops the bleeding. `root_cause_patterns.md` explains why it happened. This doc is the permanent fix — and the distinction matters, because a resource monitor bumped up temporarily during triage that never gets formalized into governance is just next quarter's incident with a delay on it.

---

## Resource Monitors, Escalating

A single-threshold monitor is a blunt instrument. The correct default is an escalating sequence, giving humans a chance to intervene before the hard stop fires:

```sql
CREATE OR REPLACE RESOURCE MONITOR MERIDIAN_PROD_MONITOR
WITH
    CREDIT_QUOTA = 5000
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND
        ON 110 PERCENT DO SUSPEND_IMMEDIATE;

ALTER RESOURCE MONITOR MERIDIAN_PROD_MONITOR SET
    NOTIFY_USERS = ('PLATFORM_ENGINEERING_TEAM');
```

`NOTIFY`-only thresholds early, giving the team time to act before anything is forcibly stopped; `SUSPEND` at quota, allowing in-flight statements to finish rather than killing them mid-execution; `SUSPEND_IMMEDIATE` past quota as the genuine emergency stop. Every production and Regulated-tier warehouse gets a monitor with this shape as a baseline — the specific credit quota is sized per warehouse, but the escalation structure isn't optional per `root_cause_patterns.md`'s finding that `NOTIFY`-only monitors are a known failure mode, not a hypothetical one.

---

## Right-Sizing Methodology

A repeatable exercise, not a one-time guess corrected only after an incident:

1. **Baseline current size against actual query latency**, not against a comfort margin nobody measured.
2. **Test one size down** against real production query patterns (during a low-risk window, with rollback ready) and measure whether latency actually degrades meaningfully, or whether the larger size was headroom nobody needed.
3. **Separate a sizing problem from a concurrency problem** before changing anything — per `root_cause_patterns.md`, queuing under concurrent load gets fixed by multi-cluster scaling, not by warehouse size, and applying the wrong fix to the wrong symptom is how oversized warehouses accumulate in the first place.
4. **Re-run this exercise on a schedule**, not only reactively after an incident — workload shape changes gradually enough that "right-sized six months ago" and "right-sized today" are different claims, and the whole point of a schedule here is catching that drift before it becomes the next incident's root cause.

Watching, not yet adopting for Meridian: Adaptive Compute's per-query allocation would eventually replace steps 1-3 with a platform-managed ceiling rather than a periodic manual exercise, per `root_cause_patterns.md`. Public Preview and regional availability limits mean that's a future-phase note, not a current recommendation.

---

## Chargeback, Built on Tagging That Already Exists

`tagging_standard.md` already specifies `cost_center` and `workload` as mandatory tagging dimensions — the governance fix here isn't inventing new tags, it's actually querying by the ones that already exist, consistently, which is exactly the gap `root_cause_patterns.md` found missing during the incident:

```sql
SELECT
    w.warehouse_name,
    t.tag_value AS cost_center,
    SUM(wm.credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wm
JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES t
    ON t.object_name = wm.warehouse_name
    AND t.tag_name = 'cost_center'
JOIN SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES w
    ON w.warehouse_name = wm.warehouse_name
WHERE wm.start_time >= DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 3 DESC;
```

This is the query that should have existed before the incident, not the one written for the first time during triage. Standing up a scheduled version of it — a weekly chargeback report landed somewhere the platform team actually looks — is the governance deliverable, not a one-time SQL script run once and forgotten.

---

## Guardrails Against Recurrence

The honest failure mode this whole doc exists to prevent: a fix applied by hand during an incident, that quietly reverts the next time someone provisions a warehouse the normal way. Two mechanisms, both already available given what `data_sec`'s Terraform already establishes:

- **Resource monitors defined in Terraform**, attached to every warehouse module by default rather than as an opt-in a busy engineer might skip under deadline pressure — the same "structurally hard to skip" logic `tagging_standard.md` already applies to tagging, applied here to cost controls.
- **`terraform plan` diff review as the place drift gets caught**, not a quarterly audit. A resource monitor manually loosened outside Terraform shows up as configuration drift on the next apply, which is a far earlier catch than discovering it during the next cost incident.

Native anomaly detection (GA'd December 2025, with Cortex-powered plain-language explanations added March 2026) is the complementary layer, not a replacement for either mechanism — it catches the spike that gets past the guardrails, and the March 2026 update means the first triage step in `cost_incident_triage.md` may already have an AI-generated explanation waiting by the time a human looks at it. Worth enabling as a standing account setting, not a tool reached for only after the fact.

---

## Sources

- Internal: `cost_incident_triage.md`, `root_cause_patterns.md`, `snowflake_compute_finops.md`, `tagging_standard.md` (sibling repo `data_sec`)
- [CREATE RESOURCE MONITOR](https://docs.snowflake.com/en/sql-reference/sql/create-resource-monitor)
- [Snowflake's March 2026 anomaly detection update](https://www.anavsan.com/blog/snowflake-native-anomaly-detection-vs-apex-what-happens-after-a-cost-spike-is-found)
