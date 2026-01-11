```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Data Sources                                  │
│ - GPU/Orchestrator (push metrics via Trickle)         │
│ - Load Test Gateway (synthetic workloads)             │
│ - Network Scraper (pull supplemental metadata)        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Ingestion (EXISTING)                          │
│ - Apache Kafka (event streaming)                       │
│ - Kafka Connect ClickHouse Sink                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Storage & Transformation                      │
│ - ClickHouse (primary database)                        │
│   • Raw events tables                                  │
│   • Materialized views (real-time aggregations)        │
│   • dbt models (optional, for complex transforms)      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ Layer 4: Serving                                        │
│ - ClickHouse (direct queries)                          │
│ - Public API (FastAPI/Go service)                      │
│ - Metabase Dashboard                                    │
└─────────────────────────────────────────────────────────┘
```