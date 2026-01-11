# Cloud SPE Pre-Proposal: Network-as-a-Product (NaaP) MVP – SLA Metrics, Analytics, and Public Infrastructure

---

## **Abstract**

This pre-proposal seeks treasury funding for the Livepeer Cloud Special Purpose Entity (SPE) to design, build, and operate a **focused Network-as-a-Product (NaaP) MVP** for SLA metrics, analytics, and public visibility.

The objective of this work is to make the Livepeer network **measurable, comparable, and trustworthy at a network level** by delivering a small but complete set of standardized performance, reliability, and demand metrics. These metrics will be publicly observable and designed to support gateway providers, orchestrators, and ecosystem builders evaluating Livepeer as production infrastructure.

This MVP intentionally prioritizes **time-to-value, architectural simplicity, and reuse of existing Livepeer infrastructure**, while establishing a durable foundation for future SLA-aware routing, scaling, and productization efforts led by Livepeer Inc, the Livepeer Foundation, and the community.

---

## **Rationale**

As Livepeer advances toward the **Network-as-a-Product** vision, predictable service characteristics and transparent performance signals become essential. While the network supports real workloads today, participants lack a **shared, network-wide view of performance, reliability, and demand** that can be used to assess suitability for production use.

Community discussion around earlier drafts of this initiative strongly aligned on the *problem*, while raising important concerns around **scope, cost, architectural risk, and MVP clarity**. This pre-proposal reflects that feedback by narrowing focus to a practical MVP that:

* Demonstrates clear value with minimal complexity
* Leverages existing data sources and pipelines wherever possible
* Avoids protocol changes, enforcement mechanisms, or premature decentralization
* Produces immediately usable outputs for real network participants

Key challenges addressed by this proposal include:

* **Fragmented metrics:** Existing performance and reliability data is dispersed across systems and difficult for non-core teams to consume.
* **Limited network-level visibility:** Gateway providers and orchestrators cannot easily compare performance across regions, workloads, or peers.
* **Adoption friction:** Without transparent, shared metrics, external developers and partners struggle to evaluate Livepeer for serious workloads.
* **Missing foundation for NaaP evolution:** Future SLA-aware routing, scaling, and automation require a trusted measurement layer first.

The Cloud SPE is well positioned to deliver this work as **neutral, public infrastructure**, building on its prior experience operating gateways, test tooling, dashboards, and analytics for the Livepeer network.

Importantly, this proposal **does not attempt to enforce SLAs, modify protocol incentives, or introduce new routing logic**. Its purpose is to establish shared measurement and learning infrastructure as a prerequisite for those future decisions.

---

## **Deliverables**

The NaaP MVP will deliver a constrained, end-to-end metrics system focused on observability and learning.

### **1. Core SLA Metrics (MVP Scope)**

* A standardized set of **network, performance, and reliability metrics** sufficient to evaluate orchestrator and GPU behavior across regions and workflows.
* Metrics sourced primarily from **existing gateway- and orchestrator-emitted telemetry**, with targeted additions only where clear gaps exist.
* Clear separation between:

  * **Operational SLA indicators** (MVP focus)
  * **Extended engineering or optimization metrics** (informational, non-blocking)

### **2. Network Test & Verification Signals**

* Operation of one or more **reference load-test gateways** to generate consistent, reproducible performance signals.
* Public test datasets designed to reflect real workloads while remaining transparent and community-verifiable.
* Test results contributed into the same analytics layer as organic network traffic to enable comparison.

### **3. Analytics & Aggregation Layer**

* Lightweight ETL and aggregation pipelines to transform raw metrics into network-level views.
* Computation of a small number of **derived indicators**, such as:

  * SLA compliance summaries
  * Relative performance distributions
  * High-level supply and demand signals
* Data structured for efficient querying without requiring dashboards to load raw event streams.

### **4. Public Dashboard & APIs**

* A standalone public dashboard presenting live and historical network metrics.
* Public, read-only APIs for aggregate data access.
* Clear paths for gateways and ecosystem teams to consume the data directly or mirror it into their own analytics systems.

### **5. Operations & Stewardship**

* Ongoing operation of testing, analytics, and dashboard infrastructure.
* Maintenance, monitoring, and community support for the MVP.
* Regular public updates on system status, coverage, and known limitations.

---

## **Key Milestones**

**Milestone 1 – MVP Metrics & Visibility**

* Define and implement the minimal SLA metrics set
* Aggregate existing telemetry into a unified analytics layer
* Launch an initial public dashboard with core network views

**Milestone 2 – Test Signals & Derived Analytics**

* Deploy reference load-test gateways
* Add basic SLA summary and comparison metrics
* Expand APIs for ecosystem consumption

**Milestone 3 – Stabilization & Review**

* Harden infrastructure for reliability and cost efficiency
* Document metrics, assumptions, and known gaps
* Review outcomes with the community to determine next steps

---

## **Budget**

**Total Requested Budget: $90,000**

This budget supports:

* Engineering work to aggregate, validate, and expose SLA-relevant metrics
* Development of minimal analytics and public-facing dashboards
* Operation of testing, analytics, and storage infrastructure for approximately one year
* Ongoing maintenance, documentation, and community support
* A modest buffer for technical risk, overhead, and LPT price variability

The budget is intentionally sized for a **thin but complete MVP**, designed to validate assumptions, inform future investment, and avoid long-term commitments before value is demonstrated.

---

## **Closing Note**

This pre-proposal reflects extensive community and Livepeer Inc feedback and represents a **deliberate step toward a simpler, clearer, and more actionable NaaP MVP**.

By focusing on shared measurement rather than enforcement or protocol change, this work aims to give the Livepeer ecosystem a common understanding of network behavior today — and a solid foundation for deciding what to build next.