# Livepeer Foundation Requirements

## Must Have

1. **Unified Metric Definitions:** A standardized set of network performance and demand metrics, grounded in stakeholder requirements and consistently defined across the network.
2. **Network-Wide Aggregation:** Metrics collected and aggregated across orchestrators and gateways into a single, queryable source of truth.
3. **Public Data Access:** Documented access to aggregated network metrics that the community can independently query and validate.
4. **Verifiable Test Load Results:** Standardized test load execution producing reproducible, independently verifiable results for key SLA metrics.

## Should have:
1. **Entity-Level Attribution:** Metrics attributable to individual orchestrators, gateways, regions, and workflows.
2. **Historical Coverage:** Time-series data sufficient to observe trends, regressions, and improvements over time.
3. **Operational Signals:** Clear, interpretable signals derived from metrics (including test load results) that operators can use to diagnose performance and reliability issues.

## Nice to have:
1. **Health Thresholds:** Defined thresholds or indicators that flag degraded or out-of-range network behavior.
2. **Comparative Baselines:** Network-wide averages or benchmarks for contextualizing individual performance.
3. **Schema Extensibility:** Metric definitions designed to evolve without breaking existing consumers.
4. **Example Analyses:** Illustrative analyses or queries demonstrating how aggregated data can be interpreted and used.