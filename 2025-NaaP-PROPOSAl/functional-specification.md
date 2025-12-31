# Functional Specification: CloudSPE NaaP Metrics & SLA Foundations

## 0. Purpose

Deliver a **publicly observable, auditable SLA reporting system** for the Livepeer Network by implementing:

1. **Decentralized telemetry collection** Gateway + Orchestrator (including GPU/Runner/Worker/Transcoder)
2. **Durable event transport + warehousing**
3. **Deterministic metric computation + SLA scoring**
4. **Public dashboard + public/authorized APIs**
5. **Job Tester** producing independent synthetic measurements

---

## 1. In-Scope Deliverables

### 1.1 Public Deliverables

**TODO:** Should we remove the Explorer IFrame stuff? i think they will just be RESTful APIs. perhaps a simple JSON based Grafana Dashboard on Cloud SPE's grafana????

**RESPONSE:** NO Explorer at all. We will have a grafana dashboard to link to

**TODO:** Need clarifcation on the actual SLA API endpoints, the Naap documentation mentioned the ones listed below.

**RESPONSE:** We need to spec this out. We dont anticipate the need for all the endpoints. What are the critical events? Need to discuss with Qiang

* **Core Metrics Dashboard**

  * Real-time + historical metrics
  * Filterable by orchestrator, workflow/model, geography

* **Job Tester**

  * Executes standard synthetic workloads from shared datasets
  * Measures and reports pathway metrics (latency, swaps, failures)
* **SLA API** at `https://api.livepeer.cloud/v1/sla`

  * `GET /gpu/metrics`
  * `GET /network/demand`
  * `GET /sla/compliance`
  * `GET /datasets`

### 1.2 Engineering Deliverables

**TODO:** Will the Job Tester publish event data to Streamr? if so, What data?

**RESPONSE:** NO. Inbound/Outbound calculations are based on gateway events. Potentially can add this to the future enhancements due to scope.

**TODO:** Will the Streamr Sink publish to Kafka for ingestion to clickhouse? Or perhaps just write to parquet for storage?

**RESPONSE:** Stream Consumer will have an adaptor for Postgres, Parquet, and Kafka. Our preffered adaptor is Kafka.

**TODO:** Will the use of Streamr's Storage node play a role in this architecture?

**RESPONSE:** No. But this is a potential future enhancement. Potential Streamr Grant.

* **Event taxonomy + schema** (versioned)
* **Publishers**:

  * Event publisher application
    * sidecar application - used by go-livepeer and Job Tester
* **Ingestion consumers**:

  * Streamr → Adapter (Postgres,Parquet, Kafka) Warehouse sink

* **Metric computation jobs**
* **Reference implementation deployment**
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

**TODO:** How do we handle SLA heartbeat/SLA event?

**RESPONSE:** If time permits, this scope can be added. "Nice to have"

**TODO:** What do we do about quality metrics? like bandwidth, heartbeat, uptime, error rates - anything that does not directly relate to SLA???

**RESPONSE** bandwidth, uptime, error rates all relate to SLA.  We must provide these and utilized them in the SLA composite score.

**TODO:** do we capture life cycle events for gateway, orch (transcoder/worker/runner)

**RESPONSE:** not for MVP of NaaP. This plays more of a role with Open Pool. Events will be capptured based on current open pool incision points, but ignored by data aggregation consumer.  We can consider capturing all events by broad categories (lifecycle, payment, etc) that align to event types.  Config options cam dictate to alter the frequency of how these are sent (from nothing to all with certain event mandated at a minimum level).

* **Telemetry events MUST be published to Streamr**
* Applies to:

  * Orchestrator (GPU/Runner/Worker) telemetry
  * Gateway pathway telemetry
  * SLA heartbeats/SLA events (Trickle semantics)

### 3.2 Synthetic Load Plane

* Synthetic workloads flow:
  * Job Tester → Streamr

* Synthetic events MUST still be **correlatable** with Gateway/Orchestrator telemetry (shared IDs).

### 3.3 Data Warehouse Plane (Unified)

* Streamr consumer sinks telemetry
* All public dashboard + API read

---

## 4. Canonical Identity & Correlation Requirements

### 4.1 Required Identifiers (Every event)

**TODO:** The "geo-awareness" of the livepeer nodes makes the event identification important (like region).

**RESPONSE:** Given we have not yet finalized how the region identifier will be used, we can start with a simple, self-reported region value selected from a constrained, code-defined enum (or validated against a lightweight lookup endpoint). Longer term, we can add automated geo-awareness using either (a) an external IP→region web service, or (b) a local GeoIP database. Candidate low-cost/free inputs include ipinfo’s “Lite” endpoint and MaxMind GeoLite2.

We should explicitly account for operator privacy: emitting raw external IPs into a globally consumable event stream (e.g., Streamr) may be unacceptable for some operators. If IP sensitivity is a requirement, the preferred approach is to publish only derived fields (e.g., continent/country/ASN) and avoid including raw IP in the event payload. IP-based enrichment can be performed either locally on the node (no IP disclosure to third parties) or at a trusted ingest layer that observes the source IP at the transport layer and only forwards the derived region.

Operationally, local GeoLite2 usage implies a database distribution and update strategy. MaxMind’s GeoLite terms require deleting older copies within 30 days of a new database release, which effectively means updating on a monthly cadence. External lookup services generally require an API token and introduce a third-party dependency, but reduce client-side footprint and update burden.

**TODO:** The technical details of the event "metadata" need validation and updating.

**RESPONSE:** we need to make sure each event has the relevant data for the given event type. It must be discernible to what the event actually means.

Every event MUST include:

* `stream_id` (or request/session stream ID)
* `orchestrator_addr` (O address)
* `gateway_addr` (originating gateway identifier; eth addr)  

### 4.2 Strongly Recommended IDs

* `runner_id` or `gpu_id` (stable per GPU device) - unclear what value this adds; consider removing
* `job_id` (inference request or batch job)
* `region` (geo region label) - this should come from a config value.  See notes above re: geo-awareness
* `dataset_id` (for synthetic runs)
* `model_id` (model identifier)
* `pipeline_id` (pipeline identifier)
* `trace_id` (end-to-end correlation across hops)

### 4.3 Correlation Rules

**TODO:** These rules need validation and updating.

* A single end-to-end inference session is represented by:

  * A **gateway session event set**
  * A **runner metrics event set**
  * Optional orchestrator mediation events
* Correlation key preference:

  1. `trace_id`
  2. (`job_id`, `orchestrator_address`, `gateway_id`)
  3. (`stream_id`, time-window join)

---

## 5. Event Taxonomy (Functional Requirements)

All events are JSON objects with a shared envelope.

**TODO:** The event taxonomy need validation and updating.

### 5.1 Event Envelope (All Events)

* `schema_version` (e.g., `"v1"`)
* `event_type` (string)
* `timestamp_ms` (UTC epoch millis)
* `source_type` (`gateway` | `job_tester` | `runner` | `worker` | `transcoder` | `orchestrator`)
* Required IDs: `stream_id`, `orchestrator_address`, `model_id`, `gateway_id`
* `region` (manually configured; optional)
* `payload` (event-specific)

### 5.2 Event Types (Minimum Set)

**TODO:** The event types need validation and updating.

#### A.1) Runner / GPU Telemetry

* `runner.heartbeat`
* `runner.metrics.sample`
* `runner.inference.summary`

#### A.2) Transcoder / GPU Telemetry

* `transcoder.heartbeat`
* `transcoder.metrics.sample`
* `transcoder.summary`

#### B) Gateway Pathway Telemetry

* `gateway.request.start`
* `gateway.request.first_frame`
* `gateway.request.end`
* `gateway.request.error`
* `gateway.swap` (if failover occurs)

#### C) Synthetic Test Telemetry

These can be excluded from scope of the MVP since we are not tracking tester actiivty and wnat to keep stream data focused intiially.

* `synthetic.run.start`
* `synthetic.run.end`
* `synthetic.sample`
* `synthetic.error`

#### D) SLA Publishing

* `sla.window.summary` (computed)
* `sla.compliance.score` (computed)

---

## 6. Metric Definitions and Computation (Metrics Catalog)

This section defines **what must be measured, where it comes from, and how it’s computed**.

### 6.1 Economics

#### CUDA Utilization Efficiency (CUE) — **Does Not Exist**

This is documented here for future reference.  It is not included in the scope of the current workstream.

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

**TODO:** Do we measure bandwidth during job processing?

**RESPONSE:** We asssume this is idle time. Sampled 1 or 3 times a day. Job processing bandwidth usage wont be taken into account.

#### Up/Down Bandwidth Mbps — **Does Not Exist**

* **Definition:** measured throughput (not theoretical link speed)
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

* Bandwidth time series per GPU and orchestrator visible in dashboard

---

### 6.3 Performance


**TODO:** The values need to be examined based on where the code event will be inserted. This can determine the proper calculation method.  Additional, any metric relying on first frame received might need to differentiate between a loading image and the actual desired output.

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

**TODO:** How does the job test factor into the E2E Stream latency?

**RESPONSE:** The job tester will not be part of E2E latency score. This could be a future enhancement, but the job tester will NOT publish to Streamr.  We must however be able to determine which metrics produced by the network came from a test job vs a real job.

#### E2E Stream Latency — **Does Not Exist**

* **Definition:** end-to-end latency observed by gateway between:

  * prompt submission / request start and delivered frame time
  * or from ingress to egress (as defined by product)
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

**TODO:** How does the swap rate reason get included? is this part of the go-livepeer code?

**RESPONSE:** go-livepeer does not consistenly deal with swaps and the swap reason is only sent to the log.  The swap reason should be standardized in its message and publication.

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

---

## 7. Storage Model

### 7.1 Raw Event Tables

**TODO:** The event metadata need validation and updating.

1. `events_raw`

* `timestamp`
* `event_type`
* ids: `stream_id`, `trace_id`, `gateway_id`, `orchestrator_address`, `runner_id`, `model_id`, `region`, `dataset_id`
* `payload` (JSON)

### 7.2 Derived Metric Tables / Materialized Views

**TODO:** This assumes data tables. Not sure where derived metrics will be stored.

* `gpu_metrics_1m` (per gpu_id/model/orchestrator/region)
* `orchestrator_metrics_5m`
* `network_demand_1h`
* `sla_scores_5m` and `sla_scores_1h`

### 7.3 Model/Config Tables

**TODO:** This assumes tables. Not sure where config data will be stored.

* `model_targets` (target_fps, latency targets, weights)
* `orchestrator_registry` (active Os, metadata)
* `dataset_catalog`

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
* Efficiency:

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

**TODO:** Updated needs to be reviewed with Qiang.

Base: `https://api.livepeer.cloud/v1/sla`

### 9.1 `GET /gpu/metrics` (Public + Private Views)

**Public**

* aggregated metrics per GPU type/region (no private IDs)
  **Private (JWT)**
* per `gpu_id` / runner-level detail

**Query params**

* `region`
* `orchestrator_address`
* `window` (1m/5m/1h)
* `since`, `until`

**Returns**

* FPS, jitter, bandwidth, latency if available

---

### 9.2 `GET /network/demand` (Public)

Hourly aggregates:

* total streams
* total inference minutes

---

### 9.3 `GET /sla/score` (Public)

**Query**

* `orchestrator_address`
* optional `window`

**Returns**

* score 0–100
* subscores and inputs used (for transparency)
* scoring_version

---

### 9.4 `GET /datasets` (Public)

These will be stored in github.  No need for an API.  Review with Qiang.

---

## 10. Dashboard Requirements

### 10.1 Core Metrics Dashboard (Public)

**TODO:** How will model_id and workflow be included?? is this data we even capture?

**RESPONSE:** Workflow is not yet a concept in go-livepeer. For now, model id and pipeline id (e.g. live-video-to-video and streamdiffusion-sdxl)


* Orchestrator leaderboard by SLA score
* Filters:

  * orchestrator
  * model_id / workflow
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

---

## 11. Job Tester (Functional)

### 11.1 Responsibilities

**TODO:** How do we document the need for prompt variations?

**RESPONSE:** This is more than just the prompt.  It is also the parameters supplied to the pipeline/model.  These need to be captured and stored in the event.  While this is verbose, it will be the only way to track performance across variants.  For the stream tester, we can store the variants in config as two separate tests.

* Execute test profiles:

  * single-stream baseline.
  * Include "high" and "low" prompt variations

* Use dataset catalog entries
* Produce measurements:

  * end-to-end latency
  * swap rate
  * failure rate
  * startup time distribution

### 11.2 Requirements

* Deterministic workload generator:

  * fixed seeds
  * pinned dataset versions
* Emit events to Streamr (with correlation)

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

**TODO:** I dont know how much testing we plan to include. here are some ai generated ideas
**RESPONSE** Updated; The below are sufficient for now.  As we get clarity on the final SLA score formula, we'll want to ensure testing of each component and the formula.  We should automate as much as we can and fill in with manual steps we document.

### 14.1 Metric-Level Tests

* Validate computation correctness for:

  * jitter coefficient
  * failure rate
  * swap rate

### 14.2 End-to-End Tests

* Synthetic run → events published → ingested → API returns expected metrics → dashboard updates
* The run should be done across variants and models.

### 14.3 Public Demo Requirements (Per milestone)

**TODO:** do we still plan to do any public demos????
**RESPONSE** yes, we can include them as part of our progerss updates on the Water Cooler or Forum posts.

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