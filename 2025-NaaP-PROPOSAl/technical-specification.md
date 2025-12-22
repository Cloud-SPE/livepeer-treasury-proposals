# Technical Specification

## Goals and Non-Goals

## 1.1 Goals (MVP)

1. **Publicly observable SLA reporting** for the Livepeer AI & Transcodign Network: latency, reliability, jitter, FPS, and (if feasible) CUE.
2. **Decentralized telemetry transport** using Streamr, with clear access controls. 
3. **Durable raw event log** stored in a warehouse to support reproducible SLA calculations and debugging (“observability”). 
4. **Reference implementation**: CloudSPE consumer + storage + API + dashboards; enables others to run their own consumer. 
5. **Job Tester** to generate independent synthetic measurements (latency, swap, failure).

## 1.2 Non-Goals (MVP)

* SLA enforcement, automated routing/selection, protocol policy enforcement.
* Replacing existing Kafka pipelines immediately (can coexist during validation). 

---

# 2) Architecture Overview

## 2.1 Logical Components

**TODO:** need to remove CUE

1. **Telemetry Producers**

   * **Gateway Producer**: emits pathway metrics + reliability events.
   * **Orchestrator Producer**: emits orchestration events + aggregates runner stats if runner can’t publish directly. Includes Transcoder, Runner, and Worker: emits GPU/perf metrics (FPS, jitter, bandwidth, CUE inputs).
2. **Streamr Network**: decentralized pub/sub backbone for telemetry. 
3. **Metrics Access Controller (MAC)**: grants/revokes write permissions for active Gateways/Orchestrators. 
4. **CloudSPE Data Consumer**

   * Subscribes to Streamr streams.
   * Validates schema, deduplicates, persists raw events.
   * Produces derived metrics + SLA scores into warehouse.
5. **Warehouse**

    **TODO:** What tech stack? will ClickHouse/Kafka be used? Perhaps it makes senes to allow easy transistion from the current Inc approach (see Non-goals)

   * ClickHouse Cloud
   * Stores **raw immutable event log** + **derived rollups** (minute/hour).
6. **Analytics Endpoint (API)**

   * Implements `/v1/sla/*` read APIs
7. **Dashboard**

    **TODO:** are these grafana dashboards based on JSON?

   * Metabase dashboards
8. **Job Tester**

    **TODO:** still have open questions about "high"/"low" prompt testing of realtime AI video.
    **TODO:** do batch AI jobs get metrics published? or simply rely on gateway?
    
   * Executes synthetic workloads from dataset catalog.
   * Publishes synthetic results to Streamr.

This matches the diagram flow: Apps/Tester→Gateways→Orchs→Runners + metrics into Streamr → Cloud consumer → storage → analytics endpoint → Analytics data consumer. 

---

# 3) Data Contracts

## 3.1 Event Log vs Metrics (Design Principle)

We treat **raw events as the primary source of truth**. Metrics and SLA scores are **derived**. This enables:

* debugging “why did SLA drop for O X at time Y?”
* retroactive metric derivation when definitions change 

## 3.2 Event Envelope (All producers must emit)

**TODO:** Event taxonomy and schema need updating/validation

All events are JSON objects with a common envelope:

```json
{
  "schema_version": "v1",
  "event_type": "gateway.request.end",
  "timestamp_ms": 1733425490197,
  "source_type": "gateway",
  "publisher": {
    "eth_address": "0x...",
    "software": "go-livepeer",
    "version": "0.8.x"
  },
  "ids": {
    "gateway_id": "gw-...",
    "orchestrator_address": "0x...",
    "model_id": "krea-realtime-v1",
    "stream_id": "user-visible-stream-key-or-name",
    "stream_internal_id": "optional",
    "request_id": "optional",
    "trickle_id": "optional",
    "trace_id": "required_if_available"
  },
  "geo": { "region": "us-east-1", "country": "US" },
  "payload": { }
}
```

### Required IDs (hard requirement)

* `ids.gateway_id`
* `ids.orchestrator_address`
* `ids.model_id`
* `ids.stream_id` (or canonical “stream name/key”)

This aligns with the NaaP MVP “must-have metadata” discussions and is necessary for correlation across producers. 

## 3.3 Event Types (MVP Minimum Set)

### 3.3.1 Gateway pathway & reliability

* `gateway.request.start`

  * payload: request parameters, selected orchestrator, attempt number
* `gateway.request.first_frame`

  * payload: `first_frame_ms`, optional frame sequence
* `gateway.request.end`

  * payload: `status=success|failure`, `failure_reason`, durations
* `gateway.swap`

  * payload: `from_orch`, `to_orch`, `reason`, `attempt`

### 3.3.2 Orchestrator scheduling

**TODO:** This sections needs rework. What are the events and their types. Sample payloads were generated from the existing Open Pool and Inc metrics collected today (for reference).

* `orch.assignment`

  * payload: chosen runner/worker, queue delay, scheduling reason
* `orch.health`

  * payload: active runners count, queue depth, capacity estimate

### 3.3.3 Runner/GPU metrics (direct or bubbled up)

* `runner.metrics.sample`

  * payload: fps sample, GPU counters, bandwidth counters, utilization
* `runner.inference.summary`

  * payload: per-request throughput/perf summary keyed by `trace_id`/`request_id`

**TODO:** Need a section for Transcoding

**TODO:** Need some information about message bubbling up to Orch from the Runner/Transcoder/Worker

### 3.3.4 SLA / rollups

* `sla.window.summary`
* `sla.compliance.score`

---

# 4) Metric Computation Specification (Catalog-driven)

All metric computations occur in the **CloudSPE consumer + warehouse** so they are reproducible from raw data.

## 4.1 Performance

### Output FPS (Exists today)

* Source: runner sample or runner summary.
* Aggregations: p50/p95, min/max, compliance vs model target.

### Jitter Coefficient (Partial)

**TODO:** The windowed timeframe (if used) needs to be clarified

* Definition: `stddev(fps)/mean(fps)` per window.
* Inputs: `runner.metrics.sample.fps`.
* Window: 30s or 60s rolling; persisted as 1m rollup.

### Prompt-to-First-Frame Latency (Does not exist)

* Definition: `gateway.request.first_frame.timestamp_ms - gateway.request.start.timestamp_ms`
* Must share correlation IDs across retries (`request_id`, `trace_id`).
* Aggregate: p50/p95 per (orch, model, region).

### E2E Stream Latency (Does not exist)

MVP definition (practical and implementable):

* `e2e_ms = gateway.request.end.timestamp_ms - gateway.request.start.timestamp_ms`
  Alternative for streaming video:
* if per-frame events exist: `frame_delivery_time - expected_frame_time` (deferred)

## 4.2 Reliability

### Failure Rate (Partial)

* `failures / total` from `gateway.request.end.status`

### Swap Rate (Partial)

**TODO:** Swap reasons need to be defined and clarified against the feasibility in the go-livepeer code.

* `swap_events / total` from `gateway.swap`
* Must implement `swap.reason` taxonomy:

  * timeout, model_error, runner_disconnect, capacity, gateway_reset, orchestrator_reset, unknown

## 4.3 Network

### Up/Down Bandwidth (Does not exist)

* Preferred: runner emits `rx_bytes`, `tx_bytes` counters.
* Compute: Mbps from deltas over sampling interval.

## 4.4 Economics

### CUDA Utilization Efficiency (CUE) (Does not exist; “High complexity”)

Two-tier approach:

**Tier A (MVP “proxy CUE”, feasible):**

* Inputs:

  * GPU model peak TFLOPS from lookup table
  * Achieved throughput proxy (frames/sec × model FLOP estimate OR tokens/sec × estimate)
* Output: `cue_proxy_pct` with a `confidence="low|medium"`

**Tier B (True CUE, optional if instrumentation exists):**

* Inputs: NVML/DCGM counters or explicit FLOP counters.
* Output: `cue_pct` with `confidence="high"`

This avoids blocking MVP on perfect FLOP accounting while still surfacing efficiency.

**TODO:** What about Gateway tickets for cost calculations

**TODO:** What about Orch ticket redeemtion for cost calculations?

**TODO:** Open pool has data for "compute units" and jobs received/processed
---

# 5) Streamr Design

## 5.1 Stream Topology

**TODO:** What other partitions are needed? This implys partitons based on environment. There may be more needed for 100/sec limit set by Streamr

Recommended: **one “telemetry root” per environment**, partitioned for throughput:

* `livepeer.naap.telemetry.prod`
* `livepeer.naap.telemetry.staging`

Partitioning strategy:

* Streamr suggests partitioning at > ~100 msg/s; we should implement partitions early to avoid rework. 
* Partition key recommendation:

  * `orchestrator_address` as primary partition key (stable, spreads load)
  * optionally secondary bucketing by `region`

## 5.2 Message Ordering and Idempotency

Streamr consumer must handle:

* out-of-order messages
* duplicates
* reconnect bursts

Dedup key (consumer-side):

* `(publisher.eth_address, event_type, timestamp_ms, ids.trace_id, streamr_metadata.sequenceNumber)` where available.

## 5.3 Access Control (Metrics Access Controller)

MAC responsibilities:

1. Identify “active” orchestrators/gateways (onchain registry, allowlist, or hybrid).
2. Grant Streamr publish permissions for active entities.
3. Revoke permissions when entity becomes inactive.

**Design options**

**TODO:** the owner of the Streamr private keys should probably operate the permission service. Is there a better way to ensure no centralization?

* **Option 1 (recommended):** Stream owned by Livepeer Foundation; MAC operated by CloudSPE (or jointly) to manage permissions.
* **Option 2:** CloudSPE owns stream initially, with a formal transfer plan.

---

# 6) Producer Integration Design (go-livepeer and worker/runner)

**TODO:** The current thinking is the runner/worker should be published via the orchestrator. The orch will pull data from the worker/runner.

## 6.1 Where metrics should be published from (worker/runner vs orchestrator)

Your notes contain a key debate: runner publishing directly vs “bubble up to orchestrator” then publish. 

**Design decision: support both, prioritize orchestrator publish path**

* **Default**: Runner → Orchestrator (local gRPC/HTTP) → Streamr

  * Solves access control without requiring runner to have separate keys
  * Centralizes schema enforcement per orchestrator
* **Optional**: Runner publishes directly if delegated credentials exist

## 6.2 Delegated publishing / key management

Constraint: “cannot require Orch ETH key to be present; need delegation.” 

**Proposed mechanism**

* Orchestrator signs a delegation statement:

  * “wallet X may publish telemetry on behalf of orchestrator Y until time T”
* MAC validates delegation and grants Streamr permissions to wallet X.
* Producer includes:

  * orchestrator address
  * delegate address
  * delegation proof (signature)

This keeps “publisher auth” separate from “orchestrator identity”.

## 6.3 Producer failure behavior

When Streamr is unavailable:

* queue in memory with bounded size
* drop oldest first (as noted) 
* emit a local log + a special `producer.backpressure` event when reconnected

---

# 7) CloudSPE Consumer and Warehouse Design

## 7.1 Ingestion Pipeline Stages

1. **Subscribe** to Streamr stream partitions
2. **Validate** schema (JSON schema / zod / ajv)
3. **Normalize** (timestamps, ids)
4. **Deduplicate**
5. **Persist raw event log** (immutable table)
6. **Derive rollups** (materialized views or batch jobs)
7. **Compute SLA scores**
8. **Publish SLA events** back to Streamr (optional) and store in warehouse

## 7.2 Storage Model

**TODO:** This assumes tables. Perhaps parquet or streamr storage node????

Tables:

* `telemetry_events_raw` (append-only)
* `telemetry_events_rejected` (validation failures)
* `metrics_gpu_1m`
* `metrics_orch_5m`
* `metrics_network_1h`
* `sla_scores_5m`, `sla_scores_1h`
* `model_targets` (fps/latency targets)
* `sla_scoring_config` (weights/thresholds/versioned)

**Why raw events are mandatory**
This implements the “event log first” requirement for observability and auditability. 

---

# 8) SLA Scoring Design

## 8.1 Deterministic scoring

**TODO:** Is a windowed approach needed? required? desired?

For each orchestrator over a scoring window (e.g., 5m and 1h):

* Compute normalized subscores:

  * Latency score (prompt→first frame, e2e)
  * Reliability score (failure, swap)
  * Consistency score (jitter)
  * Efficiency score (cue if available)
* Weighted sum → `0..100`
* Store `scoring_version` with the result

## 8.2 Reproducibility requirements

* Every SLA score must be reproducible from:

  * raw events + stored scoring config version

---

# 9) SLA API Design (`/v1/sla`)

The API is a read layer over warehouse views.

## 9.1 Endpoints

* `GET /gpu/metrics`
* `GET /network/demand`
* `GET /sla/compliance`
* `GET /datasets`

## 9.2 Auth

* Public read: aggregates and leaderboard

---

# 10) Job Tester Design

## 10.1 Responsibilities

**TODO:** How will this "catalog" be managed? is this simply the data in github (as it is today?)

**TODO:** How to prioritize test jobs over production work??

* Generate synthetic runs using dataset catalog (workflow/model-specific).
* Ensure tests are low priority vs production (as per stakeholder concerns).
* Emit:

  * swap/fail reasons
  * end-to-end timing breakdown
  * orchestrator selection path

**TODO:** Orch selection and swap fail reasons are probably only gateway concerns.

## 10.2 Outputs

* Publish synthetic events to Streamr

---

# 11) Dashboard

## 11.1 Metabase dashboards

**TODO:** This was AI generated. Need input on this!!!

* Leaderboard by SLA score
* Drill-down by:
  * orchestrator, model/workflow, region, time window
* Panels for all catalog metrics

---

# 12) Validation, Testing, and Acceptance Criteria

**TODO:** This was AI generated. Need to discuss the entire testing approach.

## 12.1 Producer acceptance

* correct event envelope + required IDs
* bounded buffering on Streamr outage
* emits swap reasons taxonomy

## 12.2 Consumer acceptance

* schema validation + rejected table
* dedupe works under reconnect
* raw event log is complete and queryable

## 12.3 Metric acceptance (catalog)

* Output FPS: present + model target compliance
* Startup time: present
* Failure + swap: computed and within expected ranges on test runs
* Jitter: computed
* P2FF and E2E: computed from gateway events
* Bandwidth: computed if runner emits counters
* CUE: proxy or true, clearly labeled

## 12.4 SLA acceptance

* SLA score reproducible and versioned
* `/sla/compliance` returns:

  * score, subscores, inputs, scoring_version

---

# 13) Deliverable Artifacts (What gets shipped)

1. **Schema repo**: versioned event definitions + examples.
2. **Producer modules**:

   * go-livepeer instrumentation patches
   * runner → orchestrator reporting module (if needed)
3. **Metrics Access Controller**: permission automation.
4. **Consumer service**: Streamr→ClickHouse ingestion + rollups.
5. **SLA API service**: query layer.
6. **Metabase dashboards**: public embed.
7. **Job Tester** + dataset catalog.
