# Functional Specification: CloudSPE NaaP Metrics & SLA Foundations

## 0. Purpose

Deliver a **publicly observable, auditable SLA reporting system** for the Livepeer Network by implementing:

1. **Decentralized telemetry collection** Gateway + Orchestrator (including GPU/Runner/Worker/Transcoder)
2. **Durable event transport + warehousing**
3. **Deterministic metric computation + SLA scoring**
4. **Public dashboard + public/authorized APIs**
5. **Job Tester** producing independent synthetic measurements

> **Note:** This is a living document. Changes and updates are anticipated before the final specification is published and versioned.

---

## 1. In-Scope Deliverables

### 1.1 Public Deliverables

* **Core Metrics Dashboard**

  * Real-time + historical metrics
  * Filterable by orchestrator, pipeline/model, region

* **Job Tester**

  * Executes standard synthetic workloads from shared datasets stored in Github
  * Measures and reports pathway metrics (latency, swaps, failures)

* **SLA API** at `https://api.livepeer.cloud/v1/`

  * `GET /gpu/metrics`
  * `GET /network/demand`
  * `GET /sla/score`

### 1.2 Engineering Deliverables

* **Event taxonomy + schema** (versioned)
* **Publishers**:

  * Event publisher application
    * sidecar application - used by go-livepeer and Job Tester
* **Ingestion consumers**:

  * Streamr → Adapter (Postgres,Parquet, Kafka) Warehouse sink

* **Metric computation jobs**
* **Reference implementation deployment**

  * Stream tester
  * Kafka consumer of Streamr data
  * ETL to warehouse/lake/lakhouse
  * API service exposure
  * Dashboard accessible from Explorer (via link; not emebdded)

* **Docs + demo per milestone**

---

## 2. System Actors and Responsibilities

### 2.1 Actors

* **Gateway (Production)**

  * Emits user-facing latency + reliability events
* **Job Tester (Synthetic)**

  * Generates controlled demand and independent observations
* **Orchestrator (Runner/Worker/Transcoder/GPU)**

  * Emits GPU + performance counters (FPS, jitter, utilization)
* **Streamr Network**

  * Public decentralized event transport for telemetry

* **SLA API Service**

  * Read-only query service over warehouse views

---

## 3. Data Planes (Transport)

### 3.1 Telemetry Plane (Decentralized)

**Telemetry events MUST be published to Streamr** to ensure a decentralized and reliable transport layer for capturing critical metrics. These metrics include bandwidth, uptime, error rates, and SLA-related events such as heartbeats. While these metrics directly contribute to the SLA composite score, additional lifecycle events for gateways and orchestrators (e.g., transcoder/worker/runner) will not be included in the MVP for NaaP. However, capturing these lifecycle events aligns with broader goals, such as Open Pool integration, and can be considered for future iterations. For now, events will be categorized broadly (e.g., lifecycle, payment) and published based on configurable options, allowing flexibility in the frequency and granularity of data sent. This approach ensures that essential telemetry is captured while maintaining scalability and adaptability for future enhancements.

* Applies to:

  * Orchestrator (GPU/Runner/Worker) telemetry
  * Gateway pathway telemetry
  * SLA heartbeats/SLA events (Trickle semantics)

### 3.2 Synthetic Load Plane (post-MVP optional)

**No plans to send events regarding the synthetic load in the first iteration of this solution.**  This placeholder is left for future expansion.

* Synthetic workloads flow:
  * Job Tester → Streamr

* Synthetic events MUST still be **correlatable** with Gateway/Orchestrator telemetry (shared IDs).
  * The reference implementation will not include events published from the job tester.  As such, the solution should allow the dashboard to differentiate between synthetic and real work.

### 3.3 Data Warehouse Plane (Unified)

* Streamr consumer sinks telemetry
* All public dashboard + API read

---

## 4. Canonical Identity & Correlation Requirements

We will use a configurable region identifier for now due to the complexities and constraints associated with automated geo-awareness. Licensing restrictions for GeoIP databases, such as MaxMind GeoLite2, require regular updates and deletion of outdated copies within 30 days, which imposes operational overhead. Additionally, using external IP-to-region lookup services like ipinfo’s Lite endpoint or MaxMind GeoLite2 introduces third-party dependencies and potential privacy concerns. Publishing raw external IPs in globally consumable event streams may not be acceptable for some operators due to IP sensitivity. To address this, we will start with a self-reported region value selected from a constrained, code-defined enum or validated against a lightweight lookup endpoint. This approach avoids IP leakage while allowing flexibility for future enhancements, such as automated geo-awareness using derived fields (e.g., continent, country, or ASN) instead of raw IPs. These derived fields can be enriched locally on the node or at a trusted ingest layer without disclosing raw IPs to third parties.

### 4.2 Strongly Recommended IDs

* `runner_id` or `gpu_id` (stable per GPU device) - unclear what value this adds; consider removing
* `job_id` (inference request or batch job)
* `region` (geo region label) - this should come from a config value.  See notes above re: geo-awareness
* `model_id` (model identifier)
* `pipeline_id` (pipeline identifier)
* `trace_id` (end-to-end correlation across hops)

### 4.3 Correlation Rules

* A single end-to-end inference session is represented by:

  * A **gateway session event set**
  * A **runner metrics event set**
  * Optional orchestrator mediation events

* Correlation key preference:

  1. `trace_id`
  2. `request_id`
  3. (`stream_id`, time-window join)

---

## 5. Event Taxonomy (Functional Requirements)

All events are JSON objects with a shared envelope.

### 5.1 Event Envelope (All Events)

* `event_id` (UUID)
* `schema_version` (e.g., `"v1"`)
* `event_type` (string)
* `timestamp_ms` (UTC epoch millis)
* `publisher_id` (eth address, job tester id, unique runner id)
* `publisher_type` (`gateway` | `runner` | `worker` | `transcoder` | `orchestrator` | `job_tester`)
* `nodes_uuid` (unique identifier/fingerprint generated at startup and stored for reuse)

* `region` (manually configured; optional)
* `tags` (optional; configure holder for custom fields)
* `payload` (event-specific)

Given the reality that many nodes will share the same node address, we use node type and uuid to differentiate them.  Node type is based on the mode the node is operating under during the event generation.  This would enable a node to be configured to run as different node types simultaneously and produce differentiated events.  Node UUID will be auto-generated and stored locally for reuse.  This will remain durable unless the node storage is removed, at which point a new uuid will be generated.

### 5.2 Event Types (Minimum Set)

This section outlines the event types designed to capture the lifecycle of a node during its routine operations. The data generated by these events will enable the derivation of metrics through aggregation and analysis.

These event types are preliminary and may evolve as technical discovery advances. Additional event types could be introduced to more effectively meet the metric requirements.

#### A.1) Runner Telemetry

* `runner.heartbeat`
* `runner.request.start`
* `runner.request.end`
* `runner.request.error`

#### A.2) Transcoder Telemetry

* `transcoder.heartbeat`
* `transcoder.request.start`
* `transcoder.request.end`
* `transcoder.request.error`

#### A.3) Orchestrator Telemetry

* `orchestrator.heartbeat`
* `orchestrator.session.start`
* `orchestrator.session.end`
* `orchestrator.request.start`
* `orchestrator.request.end`
* `orchestrator.request.error`

#### B) Gateway Telemetry

* `gateway.heartbeat`
* `gateway.session.start`
* `gateway.session.end`
* `gateway.request.start`
* `gateway.request.end`
* `gateway.request.first_frame`
* `gateway.request.error`
* `gateway.swap` (if failover occurs)

---

## 6. Metric Definitions and Computation (Metrics Catalog)

This section defines **what must be measured, where it comes from, and how it’s computed**.

### 6.1 Economics

#### CUDA Utilization Efficiency (CUE) — **Does Not Exist**

This is documented here for future reference.  It is not included in the scope of the current solution.

* **Definition:** `CUE = Achieved_FLOPS / Peak_FLOPS`
* **Source:** Runner/GPU (primary)
* **Unit:** ratio or %
* **Target:** `>= 80%`

**Functional requirements**

* Runner must periodically emit:

  * `achieved_flops` over window (or a proxy: `tokens/sec`, `frames/sec` × model FLOP estimate)
  * `peak_flops` (from GPU model table)
  * `gpu_model`, `gpu_uuid`
* CUE computed in warehouse per:

  * GPU, orchestrator, model_id, region, time-window (e.g., 1m, 5m, 1h)

**Warehouse computation rule**

* `cue_pct = 100 * sum(achieved_flops) / sum(peak_flops * window_duration_seconds)`

  * If achieved_flops is emitted as instantaneous rate, integrate over window
  * If only throughput proxy exists, compute “proxy CUE” and label it explicitly

**Acceptance criteria**

* CUE is queryable per GPU and orchestrator
* CUE is displayed in dashboard
* CUE contributes to SLA scoring (weighted) only when confidence flag is true

---

### 6.2 Network

#### Up/Down Bandwidth Mbps — **Does Not Exist**

* **Definition:** measured throughput (not theoretical link speed); Sampled 2-4 times a day when idle/starting up.  Should not be done only during off peak hours and a few times to ensure sampling beyond off peak hours. Job processing should not be interfered with in order to execute this test.
* **Source:** Runner/GPU or orchestrator networking layer
* **Unit:** Mbps
* **Target:** none (baseline/observability)

**Functional requirements**

* Runner emits:

  * `tx_bytes`, `rx_bytes` counters OR `tx_mbps`, `rx_mbps` rates
  * sampling interval
* Warehouse derives Mbps if counters provided:

  * `mbps = (delta_bytes * 8) / (delta_time_seconds * 1e6)`

**Acceptance criteria**

* Bandwidth time series per orchestrator/worker/transcoder visible in dashboard

---

### 6.3 Performance

#### Startup Times — **Exists Today**

* **Definition:** time from job assigned → GPU ready (or pipeline warm start)
* **Source:** Gateway
* **Unit:** ms
* **Target:** “≤ threshold” (per model/workflow) - tbd if we keep this or fold into SLA scoring formula; a threshold can be set at the display level rather than for calculating beforehand

**Functional requirements**

* Ensure startup is recorded as:

  * cold start vs warm start classification
  * model_id-specific thresholds stored in config table

---

#### E2E Stream Latency — **Does Not Exist**

* **Definition:** end-to-end latency observed by gateway between:

  * prompt submission / request start and delivered frame time
  * or from ingress to egress (as defined by product)
  * Measurement from Job Tester in synthetic loads is not included

* **Source:** Gateway (primary)
* **Unit:** ms
* **Target:** `≤ target`

**Functional requirements**

* Gateway emits:

  * `request.start` with `t0`
  * `first_frame` with `t_ff`
  * optionally each frame with `frame_ts` for continuous latency
* E2E latency computed:

  * `e2e_latency_ms = t_ff - t0` (MVP)
  * optionally continuous: `observed_frame_time - expected_frame_time`
  

**Acceptance criteria**

* E2E latency per orchestrator, model_id, region exposed in dashboard and API

---

#### Prompt-to-First-Frame Latency — **Does Not Exist**

* **Definition:** `t_first_frame - t_prompt_received`
* **Source:** Gateway
* **Unit:** ms
* **Target:** `≤ XX ms`

**Functional requirements**

* Gateway must define an unambiguous `prompt_received` timestamp
* Must carry `trace_id` into orchestrator/runner telemetry for correlation
* First frame received should differentiate between a loading image and the actual desired output.

**Acceptance criteria**

* P50/P95 per orchestrator/workflow visible, filterable

---

#### Jitter Coefficient — **Partial**

* **Definition:** `σ(fps) / μ(fps)` over a rolling window
* **Source:** Runner/GPU (preferred) or orchestrator
* **Target:** `≤ 0.1`

**Functional requirements**

* Runner emits FPS samples at interval (e.g., 1s)
* Warehouse computes jitter per window (e.g., 30s/60s):

  * `mu = avg(fps)`
  * `sigma = stddev_pop(fps)`
  * `jitter = sigma / nullif(mu,0)`

**Acceptance criteria**

* Jitter is reported per GPU and per orchestrator aggregated

---

#### Output FPS — **Exists Today**

* **Definition:** measured output frames per second
* **Source:** Runner/GPU
* **Target:** `>= target fps (model specific)`

**Functional requirements**

* Maintain table of per-model target FPS:

  * `model_id → target_fps`
* Compute compliance as:

  * `fps_compliant = fps >= target_fps`

---

### 6.4 Reliability

#### Failure Rate — **Partial**

* **Definition:** `% failed requests / total requests` over time window
* **Source:** Gateway
* **Target:** `< 1%`

**Functional requirements**

* Gateway emits `request.end` with `status=success|failure`
* Failure includes:

  * timeout
  * model error
  * transport disconnect
* Warehouse computes per orchestrator/workflow/region/window:

  * `failure_rate = failures / total`

**Acceptance criteria**

* Failure rate queryable and included in SLA score

---

#### Swap Rate — **Partial**

* **Definition:** `% requests that required failover/swap / total requests`
* **Source:** Gateway
* **Target:** `< 5%`

**Functional requirements**

* Gateway emits explicit `gateway.swap` event when rerouting occurs
* Must include:

  * `from_orchestrator_address`
  * `to_orchestrator_address`
  * `reason` enum (timeout, error, overload, etc.)

**Acceptance criteria**

* Swap rate computed and displayed; swap reasons summarized
* Swap reason standardize in go-livepeer codebase and cannot be generic `OrchCap` message

---

## 7. Storage Model

### 7.1 Raw Event Tables

1. `events_raw`

* `event_id`
* `timestamp`
* `event_type`
* `publisher_id`
* `publisher_type`
* ids: `stream_id`, `trace_id`, `node_addr`, `pipeline_id`, `model_id`, `region`
* `payload` (JSON)

2. `events_rejected`

  * `event_id`
  * `reason`
  * `raw_payload`
  * `ingest_ts`

### 7.2 Derived Metric Tables / Materialized Views

Purpose: provide stable, pre-aggregated inputs for dashboards, APIs, and SLA scoring. These definitions are iterative and will be refined as the metric catalog and producer payloads harden.

1) Type-Specific Staging (flattened from events_raw)
   - gateway_events: start/first_frame/end/error/swap with enums and normalized regions; feeds latency, failure, swap metrics.
   - runner_samples: fps, tx/rx bytes, gpu stats; feeds jitter, fps, bandwidth.
   - runner_summaries: windowed gpu/runner stats if emitted by producers.
   - synthetic_runs: start/end/error for synthetic-job filtering.
   - sla_published: optional store of published sla.window.summary / sla.compliance.score events.

2) Derived Metrics (rollups)

    *Time Window Reasoning:* 5m balances freshness with stability (reduces noise and sensitivity to out-of-order events), and 1h provides trend and capacity insights without the cost of per-second queries. If finer granularity is ever required (e.g., for debugging spikes), ad hoc queries can still be run directly on gateway_events or by generating a temporary 1m view, but 5m/1h cover the primary observability and SLA use cases.

   - latency_window_5m / latency_window_1h: P2FF and E2E latency (p50/p95) from gateway_events; success/failure counts.
   - swap_window_5m / swap_window_1h: swap count, swap_rate, reason breakdown from gateway.swap.
   - jitter_window_5m / jitter_window_1h: jitter_coeff, avg/p95 fps from runner_samples.
   - bandwidth_window_1d: tx/rx Mbps derived from byte deltas in runner_samples.
   - gpu_metrics_1m: per gpu_id/model/orchestrator/region fps, utilization, bandwidth (if needed for GPU-level drill-down).
   - orchestrator_metrics_5m: orchestrator-level aggregates (latency, failure, swap, fps, jitter) for leaderboard/API.
   - network_demand_1h: stream counts, inference minutes (for demand visibility).
   - sla_scores_5m / sla_scores_1h: SLA scores and subscores produced by scoring jobs.

3) SLA Scoring
   - Inputs: latency_window_*, failure_rate (gateway.request.end), swap_rate (gateway.swap), jitter_coeff (runner_samples), optional efficiency.
   - Apply scoring_config (weights, thresholds, windows, scoring_version).
   - Persist to sla_scores_* and optionally emit sla.compliance.score events.

Note: Table/view names and column shapes are expected to evolve alongside schema_version and scoring_version; breaking changes will be versioned rather than mutated in place.

### 7.3 Model/Config Tables

* `model_targets` (target_fps, latency targets, weights)
  * Used to display compliance metrics in the dashboard, calculate percentile values, and support detailed performance analysis.
* `orchestrator_registry` (active Os, metadata)
  * required since the registry onchain and extra nodes flags are insufficient for discovery.
  

---

## 8. SLA Scoring (Deterministic)

### 8.1 Output

* `compliance_score` in `[0..100]` per orchestrator over a scoring window

### 8.2 Input Metrics (MVP)

* Latency:

  * Prompt-to-first-frame (P50/P95)
  * E2E latency (P50/P95)
* Reliability:

  * failure_rate
  * swap_rate
* Consistency:

  * jitter coefficient

### 8.3 Scoring Rules

* Each metric produces a normalized subscore `[0..1]` with thresholds:

  * e.g., failure_rate < 1% gets full credit; above degrades linearly or with a curve
* Weighted sum:

  * `score = 100 * Σ(w_i * subscore_i)`
* Publish:

  * `sla.compliance.score` event
  * Store in `sla_scores_*` views

**Functional requirements**

* Scoring must be reproducible from warehouse data alone
* Version scoring config with:

  * `scoring_version`
  * weights, thresholds, window definitions

---

## 9. API Specification (Functional)

All endpoints will be publically available with reasonable rate limits in the reference implementation.

Base: `https://api.livepeer.cloud/v1/`

### 9.1 `GET /gpu/metrics` (Public + Private Views)

Provides average metrics by GPU for each Orchestrator.

**Public**

* aggregated metrics per GPU (should not include private GPU UUID or MAC)
* per `gpu_id` / runner-level detail

**Query params**

* `region`
* `orchestrator_address`
* `window` (1m/5m/1h)
* `since`, `until`

**Returns**

* FPS, jitter, bandwidth, latency averaged over time period.
* GPU info (gpu type, vram, cuda version)

---

### 9.2 `GET /network/demand` (Public)

Provides network wide view of total streams and inference minutes

**Query params**

* `region`
* `gateway_address`
* `window` (1m/5m/1h)
* `since`, `until`

**Returns**

* total streams over period
* total stream/inference minutes

---

### 9.3 `GET /sla/score` (Public)

Returns a SLA score using the published and versioned scoring formula, including inputs.

**Query**

* `orchestrator_address`
* optional `window`

**Returns**

* score 0–100
* subscores and inputs used (for transparency)
* scoring_version

---

## 10. Dashboard Requirements

### 10.1 Core Metrics Dashboard (Public)


* Orchestrator leaderboard by SLA score
* Filters:

  * orchestrator
  * model_id / pipeline
  * region
  * time range

* Panels:

  * E2E latency p50/p95
  * Latency per hop (Gateway -> Orch -> Runner/Transcoder)
  * Prompt-to-first-frame p50/p95
  * Output FPS vs target
  * Jitter coefficient
  * Failure rate
  * Swap rate
  * Stream count (by model)
  * GPUs used per model
  * Stream duration

NOTE: There is a vision for a concept of workflows within Livepeer. At this stage, only model id and pipeline are supported and included in this spec.
---

## 11. Job Tester (Functional)

### 11.1 Responsibilities

* Execute test profiles:

  * single-stream baseline.
  * Include "high" and "low" prompt variations as stored in Github repository (aka dataset catalog)

### 11.2 Requirements

* Deterministic workload generator:

  * fixed seeds
  * pinned dataset versions

## 12. ETL Engine: Raw Events → Metrics → SLA Outputs

**Principle:** Producers (Gateway, Orch, Runner, Job Tester) only emit telemetry. All metrics and SLA scores are computed from `events_raw` so results are reproducible, auditable, and re-computable when definitions change.

### Pipeline Stages
1) Ingest & Validate
   - Subscribe to Streamr partitions.
   - Validate against schema_version; reject to `events_rejected` with reason.
   - Normalize timestamps, enforce required IDs, attach ingest_ts.
   - Deduplicate by `event_id` or (`publisher_id`,`sequence`).

2) Land Raw (Immutable)
   - Append to `events_raw` partitioned by event_date.
   - Store envelope + payload intact; no in-flight mutation.

3) Type-Specific Staging
   - Flatten into typed views/tables where necessary to faciliate downstream calculations and improve related performance.  
   - Enforce enums (swap_reason, failure_reason), normalize regions, coerce units.

4) Derived Metrics (Rollups)
   - Latency (P2FF, E2E): join `gateway.request.start` + `first_frame` + `end`; compute p50/p95 per (orchestrator, model_id, pipeline_id, region, window).
   - Failure rate: `failures/total` from `gateway.request.end.status`.
   - Swap rate: count `gateway.swap` over total requests; include reason breakdown.
   - Jitter/FPS: from `runner.metrics.sample` → rolling jitter_coeff, avg/p95 fps.
   - Bandwidth: derive Mbps from `tx/rx_bytes` + `sampling_interval_ms`.
   - Startup: cold/warm from `gateway.request.start.cold_start`.
   - Synthetic flag: use `is_synthetic` and `synthetic_profile_id` to include/exclude in aggregates.

5) SLA Scoring
   - Inputs: latency (p50/p95), failure_rate, swap_rate, jitter_coeff, optional efficiency.
   - Apply `scoring_config` (weights, thresholds, window_defs, scoring_version).
   - Write to `sla_scores_5m` / `sla_scores_1h`; emit `sla.compliance.score` event (optional).

6) Serve
   - APIs/dashboards read from derived views and `sla_scores_*`.
   - Recompute is possible by replaying from `events_raw` with a new `scoring_version`.

### Determinism & Audit
- All derived tables retain `schema_version` and `scoring_version`.
- Idempotent writes via dedupe keys; tolerate out-of-order using `event_ts`.
- No metric computed outside the warehouse; Job Tester never computes metrics—only emits telemetry with `is_synthetic=true`.


---

## 12. Access Control for Publishing (Streamr)

### 12.1 Publisher Eligibility

**TODO:** This requires feasiblity. Ideally the onchain data like gateway reserve/deposit and orchestrator active by address determine the write access.

A “Metrics Access Controller” must ensure only valid entities publish:

* Only **active orchestrators** can publish runner/orchestrator telemetry
* Only approved gateways can publish gateway events

Mechanisms (functional requirement, implementation flexible):

* allowlist by address
* signed payloads
* stream permissions

---

## 13. Reliability & Durability Requirements

* Consumer must tolerate:

  * Streamr reconnects
  * duplicate events
  * out-of-order events
* Warehouse writes must be idempotent where possible:

  * dedupe keys: (`event_type`, `trace_id`, `timestamp_ms`, `publisher_id`, `sequence_number`)
* System must survive restarts without data loss (at-least-once ingestion acceptable)

---

## 14. Testing & Acceptance Criteria

### 14.1 Metric-Level Tests

* Validate computation correctness for:

  * jitter coefficient
  * failure rate
  * swap rate
  * bandwidth

  Many of these can be done with test data rather than live integration tests on live networks to avoid fragility.

### 14.2 End-to-End Tests

* Synthetic run → events published → ingested → API returns expected metrics → dashboard updates
* The run should be done across variants and models.

### 14.3 Public Demo Requirements (Per milestone)

* Live demo showing:

  * active stream of telemetry
  * API queries
  * dashboard filters and time range behavior

---

## 15. Rollout Plan (Functional)

### Phase A (Milestone 1)

* Schema v1 published
* Streamr publishers for gateway + runner (minimal)
* Streamr → Data Aggregation ingestion
* Basic correlation working
* Initial dashboard (may be limited)

### Phase B (Milestone 2)

* SLA scoring implemented + published
* Public SLA API live
* job tester producing synthetic runs
* “MVP complete” dashboard