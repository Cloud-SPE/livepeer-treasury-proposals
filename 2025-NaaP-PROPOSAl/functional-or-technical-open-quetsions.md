# Functional open questions

## 1) Exact SLA definitions and targets

* **What are the authoritative thresholds** for:

  * Prompt-to-first-frame (“XX ms” is placeholder)
  * E2E latency target (“Target” unspecified)
  * Startup time threshold (unspecified; “≤ threshold”)
* **Which percentile is used for SLA compliance** (p50, p95, p99)?
* **Windowing**: SLA computed over 1m / 5m / 1h / 24h?
* **Minimum sample sizes** before scoring (avoid scoring on tiny data).

**Why:** without this, SLA scores can’t be “deterministic and comparable”.

---

## 2) What exactly counts as “failure” and “swap”

* Failure taxonomy: timeout vs model error vs transport disconnect vs orchestrator rejection vs gateway bug.
* Swap definition: does retry to same orchestrator count? does “runner swap inside orchestrator” count?
* How to treat partial success (first frame delivered but stream fails later).

**Why:** reliability metrics are core to SLA and must be consistent across gateways.

---

## 3) Canonical “workflow/model_id” registry

* What is the source of truth for `model_id` / workflow naming?
* Are model IDs stable across versions? (e.g. `krea-v1` vs `krea-v1.1`)
* Where do targets live (target FPS, latency targets)? Who owns updates?

**Why:** dashboards, filters, and SLA calculations depend on stable IDs + targets.

---

## 4) Geography definition

* What is “region” for scoring: cloud region, geo-IP country, orchestrator declared region, or both?
* If gateway and orchestrator differ (client geo vs GPU geo), which matters?

**Why:** the dashboard and API require filtering by geography; mismatched definitions cause confusion.

---

## 5) Public vs private data policy

* Which fields must be **public** vs **JWT-protected**?

  * GPU UUID? runner_id? orchestrator internal capacity? queue depth?
* What is considered sensitive for node operators?

**Why:** impacts schema, API behavior, and what can be embedded publicly in Explorer.

---

## 6) Load Test Gateway behavior and fairness

* Test loads should be **low priority** vs production—how is that enforced?
* How often and how much load (burst vs sustained)?
* Should it sample all orchestrators or a subset (top N, random, weighted)?
* Dataset governance: who curates/approves “public load test datasets”?

**Why:** without clarity, load testing can distort network performance or be gamed.

---

## 7) “Network demand metrics” scope

* Are “staking LPT” and “fee payment ETH” required in MVP, and from where?
* Exact definition of “inference minutes” for streaming inference.

**Why:** these require onchain/indexer integrations and clear units.

---

# Technical open questions

## 8) Streamr publishing identity & access control (big one)

* Who **owns** the Streamr stream(s) (Foundation vs CloudSPE)?
* How is write access granted/revoked for:

  * active orchestrators
  * approved gateways
* How is “active orchestrator” determined (onchain set, allowlist, heartbeat)?

**Why:** this gates decentralization, abuse prevention, and operational management.

---

## 9) Runner vs orchestrator publishing architecture

* Do runners publish directly to Streamr, or do they report to orchestrator which publishes?
* If runner publishes:

  * how does it get credentials?
  * how do we prevent impersonation?
* If orchestrator publishes:

  * what protocol runner→orch is used and how is it authenticated?

**Why:** determines key management, code changes, and feasibility timeline.

---

## 10) Delegation model / key management

* The notes imply **not requiring orch ETH key** on the box. So:

  * What key signs Streamr messages?
  * How is that key authorized to represent an orchestrator address?
  * How is delegation rotated/revoked?

**Why:** security + operability + adoption hinge on this.

---

## 11) Stream partitioning and throughput planning

* Expected telemetry volume (msg/sec) per node and network-wide.
* Partition key choice (orch address, region, model, etc.)
* Message size limits and batching rules.

**Why:** Streamr scalability and consumer design depends on this early.

---

## 12) Event ordering, dedupe, and idempotency rules

* Canonical dedupe key: sequence number? trace_id? msgChainId? (Streamr metadata)
* Out-of-order handling: what windows are allowed?
* At-least-once vs exactly-once: what’s acceptable?

**Why:** required for reproducible SLA and correct aggregates.

---

## 13) Correlation identifiers propagation

* Where does `trace_id` come from (gateway)? how is it carried to orchestrator/runner?
* Is there an existing “job_id” in go-livepeer AI flows?
* If retries happen, do we reuse trace_id or generate new attempt IDs?

**Why:** without robust correlation, you cannot compute e2e metrics reliably.

---

## 14) Trickle protocol specifics

* What exactly is “Trickle” here:

  * message format?
  * heartbeat cadence?
  * required fields?
* How does it map onto Streamr events (same schema or separate)?

**Why:** “Trickle” is listed as a requirement but needs concrete definition for implementation.

---

## 15) Warehouse choice & schema decisions

* Is ClickHouse Cloud the committed warehouse, or is Postgres acceptable initially?
* Retention policy for raw events (30/90/365 days)?
* Required query latency for API (real-time vs near-real-time)?

**Why:** impacts cost, infra, and API feasibility.

---

## 16) SLA API hosting + domain ownership

* Who owns `api.livepeer.network` and deploy pipeline?
* Are rate limits required for public endpoints?
* CORS and embedding constraints for Explorer iframe/API usage.

**Why:** operational readiness and product integration requirements.

---

## 17) CUE instrumentation feasibility

* What runner environment exists (NVIDIA drivers, NVML/DCGM availability)?
* Are we allowed to run privileged collectors?
* If not, do we accept “proxy CUE” and how is it labeled?

**Why:** CUE is a stated target (≥80%) but “true” computation may be hard.

---

## 18) Compatibility with existing Kafka pipeline

* Is Kafka→ClickHouse already in use for synthetic loads?
* Should Streamr replace Kafka later, or is Kafka permanent for that path?

**Why:** affects duplication of effort and migration planning.

---

## 19) Security/abuse model for public telemetry

* How do we prevent:

  * fake orchestrators publishing bogus metrics
  * replay attacks
  * spam/flooding
* Do we require signatures per event?
* Is there a canonical “attestation” format?

**Why:** public telemetry without safeguards undermines trust and SLA validity.

---

# Highest-priority questions to resolve first (to avoid rework)

1. Stream ownership + write permissions model (MAC)
2. Runner vs orchestrator publishing + delegation
3. Correlation ID propagation (`trace_id`, retries)
4. Exact SLA definitions (targets, windows, percentiles)
5. Failure/swap taxonomy