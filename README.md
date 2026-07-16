# Snowflake FinOps Remediation — Reference Architecture
*Meridian Analytics' bill spikes — diagnosis, root cause, and permanent fix*

A cost-incident reasoning framework for Snowflake: what to do in the first 48-72 hours of a runaway bill, the recurring root causes behind it, and the permanent governance fix so it doesn't recur. Extends the sibling `data_sec` project's `snowflake_compute_finops.md` — that doc designs the cost model up front; this repo is what happens when the design wasn't fully implemented, drifted, or wasn't enough.

## What this is, and isn't

This is a **sibling project to [`data_sec`](https://github.com/Shadoe-42/data_sec)** and [`db_migration`](https://github.com/Shadoe-42/db_migration), reusing the same fictional company (Meridian Analytics) rather than inventing a new one — this scenario extends an existing doc directly, so the same company and cost model apply.

As with the rest of this portfolio: real technology, fictional company. Every Snowflake feature referenced (resource monitors, cost anomaly detection, Adaptive Compute) is real and current as of this writing, verified against Snowflake's own documentation and 2026 release notes; Meridian Analytics and its bill are not.

## Structure

| Path | What's there |
|---|---|
| `cost_governance_loop_hld.svg` / `.png` | Architecture diagram — the cycle from anomaly detection through triage, root cause, and remediation back to prevention, plus the escalating resource monitor thresholds |
| `cost_incident_triage.md` | The first 48-72 hours — where to look (`WAREHOUSE_METERING_HISTORY`, `QUERY_HISTORY`, native cost anomaly detection), finding blast radius, and stop-the-bleeding levers before any permanent fix |
| `root_cause_patterns.md` | The recurring ways a correctly-designed cost model still runs away — auto-suspend drift, oversized warehouses, multi-cluster misconfiguration, notify-only resource monitors, missing per-tenant attribution — plus a note on Adaptive Compute as the emerging alternative |
| `remediation_and_governance.md` | The permanent fix — escalating resource monitors, a repeatable right-sizing methodology, chargeback built on tagging that already exists, and Terraform-enforced guardrails against recurrence |

## Related

Sibling repos in the same portfolio, all reusing or extending `data_sec`'s architecture rather than duplicating it:

- [`data_sec`](https://github.com/Shadoe-42/data_sec) — the landing zone, Snowflake security model, and the original `snowflake_compute_finops.md` cost design this project extends
- [`db_migration`](https://github.com/Shadoe-42/db_migration) — sibling scenario, same fictional company (Meridian Analytics)
- [`cloud_retrofit`](https://github.com/Shadoe-42/cloud_retrofit) — risk-tiered retrofit of a live environment that grew without a plan
- [`dc_decomm`](https://github.com/Shadoe-42/dc_decomm) — full on-prem-to-cloud exit, reusing `data_sec`'s landing zone Terraform directly
- [`ma_consolidation`](https://github.com/Shadoe-42/ma_consolidation) — post-merger data platform consolidation

## License

MIT — see `LICENSE`.
