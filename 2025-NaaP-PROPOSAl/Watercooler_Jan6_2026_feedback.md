# Cloud SPE Metrics & SLA (NaaP) Proposal

*(Main area of community discussion and debate)*

## Problem statement (widely agreed)

* Lack of **network-wide transparency** into orchestrator performance
* Current metrics insufficient for:

  * Developer confidence
  * SLA-style guarantees
  * “Network-as-a-Product” positioning
* No Livepeer-native equivalent to centralized cloud SLAs

## Proposed solution (Cloud SPE)

* Two-milestone project:

  1. Network analytics infrastructure
  2. SLA scoring and reference routing logic
* Components include:

  * Decentralized event publishing from nodes
  * Analytics engine for SLA scoring
  * Public APIs and dashboards
  * Reference implementation for intelligent job routing
* Architecture emphasizes **decentralization**, using **Streamr** for event transport

  * Kafka proposed as fallback if Streamr becomes unavailable

---

## Key community concerns & feedback

### 1. Architecture complexity vs MVP value

**Strong recurring theme**

* Multiple participants questioned whether the architecture is:

  * Overly complex
  * “Production-grade” too early
  * More expensive than necessary for an MVP
* Specific feedback:

  * Could a **simpler, centralized MVP** achieve most of the value?
  * Example alternative:

    * Single container run by orchestrators
    * Tests basic capabilities (GPU, VRAM, bandwidth, health)
    * Publishes to a centralized service with a backup
    * Exposes a public metrics dashboard
* Direct challenge:

  * Why is **$200k** required?
  * Why not prove value with a smaller, cheaper MVP first?

### 2. Decentralization & Streamr dependency

* Questions raised about **Streamr viability**:

  * What if Streamr is deprecated or pivots?
* Cloud SPE response:

  * Streamr can be swapped for Kafka
  * However, Kafka increases centralization and operational burden
  * Decentralization is intended to:

    * Avoid single points of failure
    * Avoid reliance on a complete global service registry

### 3. Overlap with existing Livepeer metrics

* Question:

  * go-livepeer already exposes Prometheus metrics — why not scrape those?
* Cloud SPE response:

  * Existing metrics are:

    * Incomplete for NaaP goals
    * Non-durable
    * Not topology-aware
    * Insufficient for SLA scoring and historical analytics

### 4. Job / model variability

* Concern:

  * How does the system handle rapidly changing job types and models?
* Cloud SPE response:

  * Core metrics are **job-agnostic** (heartbeat, latency, bandwidth, GPU info, failures, capacity)
  * BYOC developers can emit **custom events** for specialized needs

### 5. Service registry & discovery

* Implicit concern:

  * Why not just discover nodes and scrape them?
* Cloud SPE response:

  * Existing registries and `extraNodes` are incomplete
  * Not robust enough for geo-aware, load-balanced routing
  * Event publishing avoids needing to solve global discovery first

### 6. Authority, usage, and “what happens after it’s built?”

* Raised by authoritynil, Navigara, and others:

  * Who will actually use this data?
  * How will it surface in the Explorer?
  * How will developers outside Livepeer learn about it?
* Cloud SPE response:

  * This is **network infrastructure**, not a Cloud SPE-only system
  * Any gateway can consume the APIs
  * Cloud SPE will run a **public reference instance** (similar to AI Tester)
  * Intended use:

    * Intelligent routing
    * SLA- and cost-aware job selection
    * Part of the “Network-as-a-Product” vision

---

## Overall sentiment on Cloud SPE proposal

* **Agreement on the problem**
* **Mixed confidence in the proposed solution**
* Repeated request for:

  * A clearer MVP
  * Reduced scope
  * Stronger justification of cost vs value
  * Explicit articulation of post-build adoption and usage