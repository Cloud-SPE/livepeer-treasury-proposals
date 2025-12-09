Livepeer Inc/Foundation & Cloud SPE NaaP kicoff meeting (11/18/25)
---

## 1. Meeting Overview

**Topic:** Network-as-a-Product (NaaP) + Metrics / SLA & Data Lake Foundations
**Focus:**

* Why/why now for “Network-as-a-Product” and SLA metrics
* How Cloud SPE’s data/metrics proposal and Inc’s NaaP workstream fit together
* What is realistic to deliver in the next 6–8 weeks (Milestone 1)
* Roles, funding, and coordination across Inc, Cloud SPE, AI SPE, and the wider community

**Key Participants (in transcript):**

* **Qiang Han** – facilitating, providing engineering leadership / vision for NaaP
* **Speedy Bird Tech (“Speedy”)** – Cloud SPE, metrics/data lake & infra concerns
* **Mike Zoop (Mike)** – Cloud SPE (you)
* **Rick Staa** – Inc / community coordination, subgraph, Explorer, product alignment
* Others referenced but not present: **Doug**, **Ben**, **foundation**, **Inc**, **AI SPE**, **Daydream**, **Cloud SPE**, etc.

---

## 2. Objectives & Structure of the Meeting

Qiang framed the meeting in two halves:

1. **Open Discussion (first ~30+ minutes)**

   * Why are we doing “Network-as-a-Product” now?
   * What exactly are we trying to solve, and for whom?
   * How do metrics, SLAs, and control-plane ideas fit together?
   * Scope and feasibility of milestones.

2. **Execution & Structure (second half)**

   * How to organize people and work across Inc + community (Cloud SPE, AI SPE, etc.)
   * What can be realistically done in ~6–8 weeks (Milestone 1)?
   * What does success look like?
   * How Cloud SPE’s existing metrics/data proposal fits into this.

---

## 3. Big Picture: Vision, Scope, and Timeframe

### 3.1 Long-Term Vision

* The **“Network-as-a-Product”** roadmap spans **four milestones** and is realistically **~1.5 years** of work, not something achievable in six months.
* The vision responds to **Dob’s broader vision** of Livepeer as:

  * A **reliable, transparent, real-time AI/video/world-model infrastructure** with comparable SLAs to centralized competitors (Runway, Together, etc.)
  * A **network-level product**, not just a collection of siloed toolchains (e.g., Daydream running its own internal metrics and load testing).

### 3.2 Near-Term Focus (Milestone 1)

* Milestone 1 is described as **“metrics-focused MVP”**:

  * Make the **network visible and measurable** from the outside (capacity, performance, reliability).
  * Use this visibility to **build trust** with:

    * Design partners (e.g., Daydream).
    * Orchestrators (O’s) and GPU providers.
    * The community & future integrators.

* Qiang stressed:

  * The **four milestones** are drawn so the roadmap isn’t full of “blank space” between today and the long-term vision.
  * But **only a subset** (Milestone 1) is being targeted for delivery in the **next 6–8 weeks**.

---

## 4. Speedy’s Core Concerns & Questions

Speedy raised several big strategic questions:

### 4.1 Scope and Driver

* The scope of the NaaP document and metrics roadmap feels **“quite massive”** relative to the proposed timeline.
* Questions:

  * **Who is this for, concretely, in the near term?**
  * **What specific gaps or problems** are design partners like **Daydream** facing that require this scope now?
  * Is the timeline driven by:

    * Actual **market / product needs**, or
    * A **top-down internal estimate** (“this seems like 6 weeks”)?

### 4.2 Commodity vs Differentiation

* Speedy strongly emphasized the risk of Livepeer spending too much time and money on **commodity infrastructure**:

  * Defining workloads
  * Loading/assigning compute
  * Running & metering jobs
  * Pricing & control plane logic

* These problems are **not unique** to Livepeer and **“have been solved”** in other places:

  * Example: **Prime Intellect** + integrations with **RunPod** (community compute marketplace, dashboards, pricing, control plane, etc.)

* Concern:

  * If Livepeer invests heavily in a **generic compute mesh** / control plane, it risks:

    * **Diluting focus and capital** away from its core differentiator: **real-time AV and world-model–specific capabilities**.
    * Ending up in 5 years as **one commodity platform among many**, instead of a recognized specialist.

### 4.3 Timeline & “Why Now?”

Speedy summarized the near-term asks as:

* **Milestone 1:** RFC and initial metrics infrastructure.
* **Milestone 2 (early concept):** Metrics selection–based APIs and gateway logic (workload selection based on metrics like latency/jitter/pricing).

Question:

* Why must **Milestone 1** (metrics MVP) happen **in the next 6–8 weeks**?
* Is there a **market or product driver**, or is it mainly an **internal target**?

---

## 5. Qiang’s Responses & Clarifications

### 5.1 Addressing Scope & Timing

* Qiang acknowledged:

  * The overall roadmap is **large and high-risk**.
  * Full NaaP realization is **~1.5 years** of work given realistic resources.

* The reason to draw all four milestones now:

  * To **bridge Dob’s vision** and today’s reality.
  * To show **continuity**, not to claim everything will be built immediately.

* For the **next 6–8 weeks**, the focus is narrow:

  * **Milestone 1** = **metrics visibility foundation**:

    * “Collect enough data so people understand what the network is.”
    * Play nicely with what **Daydream** already measures.
    * Expand that into **network-wide metrics** and **SLAs**.

### 5.2 Why Metrics & SLAs Matter Now

Drivers:

* **Daydream** today:

  * Already collecting a **set of metrics** for diagnosis, performance, and reliability.
  * Must convince end users the service is **reliable and comparable** to centralized alternatives (Runway, Together, etc.).
  * Currently metrics + decisions are proprietary and not reusable as **network assets**.

* The opportunity:

  * Turn Daydream learnings into **shared network infrastructure**:

    * **Standard metrics** and SLAs.
    * A **community-visible dashboard**.
    * A **transparent selection / fairness model** for orchestrators and GPU providers.
  * Make the infrastructure **credible** and **trustworthy** as a network, not only as a single app’s internal system.

### 5.3 Control Plane Terminology

* Qiang clarified that when he used the term **“control plane”**, he did **not** mean:

  * “We must build a full, bespoke control plane from scratch.”
* Instead, he meant:

  * A **conceptual picture** describing **gateway workflows** and basics like:

    * Knowing how much capacity is available for a workload type with a given SLA.
    * Allowing users to **deploy a workload to that capacity** if it’s available.
* So:

  * **Control plane = minimum necessary workflows & APIs at the gateway level**, not a massive new infrastructure project.
  * The exact implementation (reuse existing platforms vs build pieces) is **TBD** and should avoid re-inventing commoditized parts.

### 5.4 Role of Metrics in Fairness & Transparency

* Qiang wants metrics to:

  * Show orchestrators **how they’re performing** vs peers (FPS, latency, etc.).
  * Show **where demand gaps** exist (e.g., “network needs this workflow on this hardware in this region to meet SLA X”).
  * Make selection **data-driven**, instead of opaque Daydream discretion.
  * Provide a **single public source of truth** the entire community can refer to for:

    * SLA scores
    * Capacity gaps
    * Fairness issues

---

## 6. Funding, Roles, and Relationship to Cloud SPE Proposal

### 6.1 Cloud SPE’s Existing Plan

Speedy explained:

* Cloud SPE already drafted a **metrics/data proposal**:

  * It targeted a **January start**, not immediate.
  * It was conceived as a **multi-month, full-fledged product**, not just an MVP.
  * It focused on **decentralized metrics components** (data pipeline, metrics infra).
  * It did **not originally include** the Explorer selection APIs and other gateway APIs now appearing in Qiang’s doc.

* Foundation/Inc asked Cloud SPE to **pause (“sit tight” for ~2 weeks)**:

  * So they could publish broader NaaP materials.
  * To ensure Cloud SPE’s proposal **aligns** and **looks cohesive** with the new workstream.

### 6.2 Qiang’s Coordination Role

* Qiang was asked by **Doug and Eric** to:

  * Provide **engineering leadership**.
  * **Glue together** multiple efforts (Inc, Cloud SPE, AI SPE, others) into a coherent roadmap.
  * Help **ship something concrete faster** that everyone can benefit from.

* His view:

  * There is **no conflict** between:

    * The NaaP roadmap and
    * Cloud SPE’s metrics/data work.
  * Milestone 1 can be a **foundation** that **Cloud SPE builds on** with its longer-term project.
  * Multiple funding tracks can coexist, coordinated via a **shared vision**.

### 6.3 Rick’s Perspective

* Rick highlighted:

  * The need for **better coordination** as many efforts are currently siloed and duplicative.
  * Two main collaboration areas:

    1. **Data & Metrics (Milestone 1)** – lower risk, strong alignment, Cloud SPE + Inc + AI SPE collaborating.
    2. **SLA / Control Plane / BYOC** – more complex, overlaps with AI SPE work, requires careful coordination.

* Rick’s understanding:

  * Cloud SPE’s **treasury proposal still makes sense**.
  * It should be brought to the community **soon**.
  * Inc wants to avoid Cloud SPE doing extensive work **without funding**.

---

## 7. Technical Pieces Discussed

### 7.1 “Load Testing Gateway” Clarification

* Mike distinguished:

  * **True load testing**: ramping concurrent users until systems break, determining capacity limits, etc.
  * **Stream testing**: running controlled streams to measure performance metrics.

* Qiang clarified:

  * The current term “load testing gateway” is **misleading**; it’s really **stream testing / metrics collection**:

    * Using a small fraction (e.g., 5%) of capacity.
    * Running synthetic streams to measure FPS, latency, etc.
  * There is already a **rudimentary tool** inside Daydream’s gateway:

    * Currently uses a single demo video replicated at various lengths.
  * Future goal:

    * Build a **testing scheduler** that:

      * Only selects *available* capacity.
      * Distributes test streams across GPUs.
      * Limits test length to avoid impacting production.
    * Make it **more objective and publicly visible** (e.g., public dashboard).

### 7.2 Prompt Variety and Real-World Workloads

* Qiang emphasized:

  * For generative workflows (e.g., SD-based, real-world models), **prompt variety** is a key performance driver.
  * Different prompt structures can produce very different FPS and latency.
* Plan:

  * Use a **bootstrap prompt set**, likely derived from Daydream’s **“boot prompts”** for a main real-world model being deployed.
  * Over time, extend to more **diverse prompt categories** to better simulate real-world workloads.

### 7.3 Geography

* Mike asked whether *where* tests originate (US vs EU vs other regions) matters for metrics.
* Qiang’s current stance:

  * For now, geography is a **nuance that can be deferred**.
  * Current traffic patterns are not complex enough to require detailed geographic modeling in the MVP.
  * Most metrics (especially GPU-level metrics) are independent of test origin.

### 7.4 Existing Telemetry Pipeline (Daydream / Gateway)

Qiang outlined the current **metrics stack** Inc uses:

* **AI Runner / Inference pipeline:**

  * `inference.py` performs actual inference (live video, composited streams, real-world models).
  * `process_guidance` hooks into that pipeline to extract metrics.

* **Data transport:**

  * Metrics are pushed back to the gateway via a **QUIC/WebRTC (chico) datachannel** because orchestrators don’t expose port APIs for easy metric extraction.

* **Gateway side:**

  * A `monitor.go` module:

    * Constructs telemetry events.
    * Enriches them with metadata.
    * Sends to:

      1. **Kafka → ClickHouse** (for the internal analytics data lake, multiple analytical models).
      2. **OpenTelemetry / Prometheus** (for dashboards).

* A metrics dashboard already exists internally using these Prometheus metrics.

**Ask to Mike & Speedy:**

* Spend time **reading this code path** (AI runner, process GUIDANCE, inference, monitor.go, telemetry).
* Understand how metrics flow from GPU → gateway → Kafka/ClickHouse/Prometheus.
* Use this understanding to help define the **MVP metric set** & architecture for Milestone 1.

---

## 8. Near-Term Timeline, Deliverables & Success Criteria

### 8.1 Qiang’s Success Criteria for the Next ~6–8 Weeks

Two **concrete deliverables**:

1. **Dashboard**

   * A visible dashboard (can be standalone for now) that surfaces:

     * The **core GPU/network metrics** agreed upon in the RFC.
   * Able to support **“graduation analytics”** (comparing GPUs, O’s, workloads, etc.).

2. **Stream-Testing Gateway**

   * A gateway-based **test scheduler** that:

     * Sends synthetic or structured traffic into the network.
     * Collects metrics in a reproducible way.
     * Feeds data into Kafka/ClickHouse/Prometheus and thus into the dashboard.
   * Not full load testing; more **“continuous, controlled probing.”**

Additionally:

* **RFC to the community**:

  * Within ~1 week: publish an RFC outlining:

    * Which metrics are collected.
    * How they’re calculated.
    * The high-level architecture and data flows.
  * Jointly authored by Inc engineers and community members (Cloud SPE, etc.).

### 8.2 Integration With Explorer

* For the MVP:

  * The dashboard **does not need to be embedded in Explorer**.
  * It can be a **separate property** (subdomain or standalone site) with a link from Explorer.
* Longer term:

  * The goal is to **embed these insights in Explorer**, so:

    * Users see network metrics in the same UX where they explore orchestrators and network state.

---

## 9. Risks, Open Questions, and Philosophical Points

### 9.1 Commoditization Risk

* Speedy reiterated:

  * Compute mesh and generic control-plane logic will likely be **commoditized over the next 5 years**.
  * Livepeer must avoid sinking too much capital into this commodity layer.
  * The focus should be on:

    * **RT-AV + world-model “secret sauce”**.
    * Unique streaming primitives, specialized capabilities.
* This is not a call to *avoid* commodity entirely (we still need a base), but to:

  * **Deliberately prioritize** what Livepeer builds vs reuses.

### 9.2 Generalizing Metrics Beyond Daydream

* A likely community question:

  * “My workflow is different from Daydream’s; why do *these* metrics apply to me?”
* Idea raised:

  * Workflow definitions could include:

    * Model locations
    * Container images
    * Pricing
    * SLA requirements
    * **Metric definitions & reporting format**
* Long-term requirement:

  * A **generalized framework** where:

    * Metrics are composable and per-workflow.
    * The architecture can grow to serve **heterogeneous video + AI workloads**, not just Daydream’s current pipelines.

### 9.3 Avoiding “Rush and Bolt-On” Architecture

* Speedy stressed:

  * Even if milestone 1 is built quickly, the architecture should:

    * Be able to **grow and generalize**.
    * Avoid becoming a **fragile one-off** that has to be thrown away later because everything was bolted together “in two weeks.”

* Rick shares this concern:

  * Wants **quick iteration loops** but also **good coordination** so early work is not wasted.

---

## 10. Agreed Next Steps & Action Items

### For Qiang

* Share:

  * **Implementation diagrams** (generated via AI) explaining current metrics architecture.
  * **Pointers to relevant code** (AI runner, process guidance, inference, monitor.go, telemetry pipelines).
* Coordinate:

  * A **working session next week** (Tuesday or Thursday):

    * With Mike & Speedy.
    * Potentially involving engineers like **Evan**, **Victor**, **Josh**, depending on questions.
* Drive:

  * Drafting the **metrics RFC** and getting Inc + community reviewers on it.

### For Mike & Speedy (Cloud SPE)

* **Digest**:

  * The NaaP document and metrics sections more thoroughly.
  * Qiang’s notes + diagram on current metrics stack.
* **Review code**:

  * AI runner (`process_guidance`, `inference.py`).
  * Gateway `monitor.go` and telemetry export.
* **De-brief together**:

  * Align Cloud SPE’s existing proposal with this Milestone 1 framing.
* **Prepare questions**:

  * Especially fundamental concerns (scope, metrics generalization, control-plane vs commodity).
  * Send key questions to Qiang ahead of the next working session to avoid derailing it.
* **Funding alignment**:

  * Work with Rick/Ben/foundation to ensure the **Cloud SPE proposal** is:

    * Sharpened.
    * Submitted at a timing that aligns with NaaP milestones.
    * Adequately funds Cloud SPE’s work on the metrics/data lake pieces.

### For Rick

* Follow up with **Ben**:

  * On **funding** and **timelines** for the Cloud SPE proposal.
  * On coordination between Inc, Cloud SPE, and AI SPE.
* Continue:

  * Subgraph work and Explorer-related changes that could later surface the metrics.
* Help:

  * Set up a **meeting next week** including Ben (if possible) to ensure funding is in place and roles are clear.

