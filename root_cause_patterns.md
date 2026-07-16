# Root Cause Patterns
*Meridian Analytics — how a well-designed cost model still runs away | July 2026*

---

## Purpose

`snowflake_compute_finops.md` specifies a correct design: right-sized warehouses, auto-suspend, a deliberate multi-cluster scaling policy, resource monitors, per-tenant attribution. A real incident doesn't usually mean that design was wrong — it usually means part of it was never implemented, drifted after implementation, or was correct for the workload it was designed around and stopped being correct once the workload changed. This doc catalogs the recurring ways that happens, each one checked against what the original design already specifies.

---

## Auto-Suspend: Disabled, or Set Too Loose

The single most common finding. A warehouse created with a sane auto-suspend value during initial build, later set higher "temporarily" to avoid cold-start latency during a demo or a deadline crunch, and never set back. Or, less charitably, disabled entirely on a warehouse someone was actively debugging against and forgot to re-enable. `snowflake_compute_finops.md` specifies auto-suspend as a default; it doesn't specify a mechanism preventing someone from overriding it later — that gap is exactly what `remediation_and_governance.md`'s Terraform-enforced guardrails close.

One AI-specific version of this worth naming directly: `cortex_ai_implementation.md` already documents a 1,800-second (30-minute) `AUTO_SUSPEND` floor on the Cortex Search warehouse — a real platform-level constraint, not a configuration choice, driven by how Cortex Search's indexing works. That floor is correct and unavoidable, but it means a Cortex-dedicated warehouse costs more per idle hour than a general-purpose one by design, and that fact needs to be visible in cost attribution rather than treated as an anomaly every time it shows up in `WAREHOUSE_METERING_HISTORY`.

## Oversized Warehouses for the Actual Workload

A warehouse sized for anticipated peak load that never arrived, or sized generously "to be safe" without a right-sizing exercise ever actually running the workload at a smaller size to check. The practical test — documented in `remediation_and_governance.md` — is running the same workload at one size smaller and checking whether query latency actually degrades. Frequently it doesn't, because the original sizing decision was a guess, not a measurement.

## No Multi-Cluster Discipline, in Either Direction

Two failure modes here, and they're opposites: a multi-cluster warehouse with `MAX_CLUSTER_COUNT` set high and no queuing-based justification, scaling out for concurrency the workload doesn't actually have — or the reverse, a single-cluster warehouse under real concurrent load, where users experience queuing and the fix someone reaches for is a larger warehouse size instead of enabling multi-cluster scaling, which solves a concurrency problem far more cost-effectively than a bigger single warehouse does. Both are root-cause findings, and distinguishing them requires actually looking at whether the problem is query duration (a sizing problem) or query queuing (a concurrency problem) — conflating the two is how the wrong fix gets applied.

## Resource Monitors: Absent, or Notify-Only

A resource monitor that only sends a `NOTIFY` action isn't a guardrail, it's a heads-up — useful, but it doesn't stop spend, and an incident where the monitor "worked as designed" by sending a notification nobody acted on for three days isn't a monitor failure, it's a monitor design failure. `remediation_and_governance.md` treats `NOTIFY` as the first rung of an escalating trigger, never the only one, specifically because of this pattern.

## No Per-Tenant Attribution

The gap `cost_incident_triage.md` calls out directly: without the tagging discipline `tagging_standard.md` already specifies applied consistently to warehouses and queries, a spike is visible in aggregate but not attributable to a cause, which means every incident starts from zero instead of building on the last one. This is consistently the finding that turns a one-time incident response into a standing governance conversation, because it's the one root cause that isn't really about any single warehouse — it's about whether the account can even ask the right question when the next spike happens.

## The Emerging Alternative: Adaptive Compute

Worth naming as context rather than a current fix: Snowflake's Adaptive Compute (Public Preview as of mid-2026, and region-limited — not yet available account-wide) replaces manual warehouse sizing and scaling-policy configuration with per-query, workload-aware compute allocation, governed by a performance ceiling (`MAX_QUERY_PERFORMANCE_LEVEL`) and a burst multiplier rather than a fixed t-shirt size. If and when this reaches general availability broadly, a meaningful share of the sizing-related root causes in this doc stop being a human configuration problem at all. It isn't the answer for Meridian today — Public Preview status and regional limits mean the manual right-sizing methodology in `remediation_and_governance.md` is still the practical default — but it's the direction this problem is heading, and worth flagging rather than pretending the current manual approach is a permanent state of the art.

---

## Sources

- Internal: `cost_incident_triage.md`, `remediation_and_governance.md`, `snowflake_compute_finops.md`, `cortex_ai_implementation.md`, `tagging_standard.md` (sibling repo `data_sec`)
- [Adaptive Compute](https://docs.snowflake.com/en/user-guide/warehouses-adaptive)
- [Working with resource monitors](https://docs.snowflake.com/en/user-guide/resource-monitors)
