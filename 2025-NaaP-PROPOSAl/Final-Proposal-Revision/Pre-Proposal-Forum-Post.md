# Network-as-a-Product (NaaP) MVP – SLA Metrics, Analytics, and Public Infrastructure

---

## **Abstract**

This proposal seeks treasury funding for the Livepeer Cloud Special Purpose Entity (SPE) to design, build, and operate the **Network-as-a-Product (NaaP) MVP SLA Metrics and Analytics Infrastructure**.

The goal of this work is to make the Livepeer network **publicly observable, measurable, and product-grade** by delivering standardized performance, reliability, and demand metrics across orchestrators, GPUs, workflows, and regions. This infrastructure enables transparent SLA reporting, improves network trust, and establishes the data foundation required for future SLA-aware routing, scaling, and orchestration.

This proposal builds directly on the Cloud SPE’s prior gateway, metrics, and visibility work and aligns with the broader NaaP vision introduced by Livepeer Inc and the Livepeer Foundation.

---

## **Rationale**

As Livepeer evolves toward a **Network-as-a-Product**, predictable performance, reliability, and transparency become first-class requirements. While the network today supports real workloads, it lacks a **network-wide, standardized, and publicly verifiable SLA signal** that users, developers, and ecosystem participants can rely on.

Key challenges this proposal addresses:

* **Lack of unified SLA metrics:** Performance and reliability data exists in fragments but is not standardized, aggregated, or publicly consumable at the network level.
* **Limited visibility into network health:** Users and partners cannot easily assess orchestrator performance, regional reliability, or supply-demand dynamics.
* **Barrier to adoption for serious workloads:** Without clear SLA signals, Livepeer is harder to evaluate as production infrastructure for AI and real-time workloads.
* **Missing foundation for future automation:** SLA-based routing, adaptive scaling, and product guarantees require a robust metrics and analytics layer.

The Cloud SPE is uniquely positioned to deliver this work as **neutral, public infrastructure**, extending its existing role operating gateways, test infrastructure, dashboards, and analytics in service of the broader network.

This proposal does **not** enforce SLAs or change protocol rules. Instead, it establishes the **measurement, analytics, and visibility layer** required for Livepeer to function as a credible, productized decentralized network.

---

## **Deliverables**

The Cloud SPE will deliver the following outcomes as part of the NaaP MVP:

### **1. Standardized SLA Metrics Foundation**

* Implementation of a unified telemetry schema covering **performance, reliability, network, and demand metrics**.
* Support for GPU-level, orchestrator-level, workflow-level, regional, and temporal dimensions.
* Metrics designed for public observability and future SLA-based orchestration.

### **2. Network Test & Verification Infrastructure**

* Operation of **dedicated load-test gateways** that continuously execute synthetic workloads.
* Public, reproducible testing datasets to independently verify network performance and reliability.
* Transparent contribution of test results to the analytics layer.

### **3. Analytics & Data Processing Layer**

* ETL and analytics pipelines to aggregate, validate, and compute network-wide SLA signals.
* Core analytics models such as:

  * SLA compliance scoring
  * GPU efficiency and utilization indicators
  * Network supply-demand visibility
* Data structured for efficient querying and long-term analysis.

### **4. Public Dashboard & APIs**

* A standalone public dashboard displaying live and historical SLA metrics across the network.
* Public read APIs for aggregate metrics and authenticated endpoints for private/operator-level data.
* Infrastructure designed to integrate with existing Livepeer tooling and future protocol features.

### **5. Ongoing Operations & Maintenance**

* Reliable operation of testing, analytics, storage, and dashboard infrastructure.
* Ongoing support, maintenance, and community bug bounty / ticket payouts related to SLA metrics.

---

## **Key Milestones**

**Milestone 1 – SLA Metrics MVP (Initial Release)**

* Core performance, reliability, and network metrics implemented
* Load-test gateways operational
* Initial public dashboard displaying network-wide metrics

**Milestone 2 – Analytics & Demand Visibility**

* Network demand and capacity metrics added
* SLA compliance scoring and analytics models deployed
* Expanded APIs for ecosystem consumption

**Milestone 3 – Stabilization & Operations**

* Infrastructure hardened for long-term reliability
* Documentation, monitoring, and operational processes finalized
* Public reporting and transparency updates to the community

---

## **Budget**

**Total Requested Budget: $90,000**

This budget covers:

* Engineering and implementation of SLA metrics, analytics, APIs, and dashboards
* Development of test and verification infrastructure
* Operation of cloud infrastructure, data pipelines, storage, and monitoring for one year
* Ongoing support, maintenance, documentation, and community ticket/bounty payouts
* A modest buffer for technical risk, overhead, and LPT price volatility

The budget is intentionally scoped to deliver a **focused NaaP MVP**, prioritizing foundational infrastructure and public goods over long-term protocol changes.

---

### **Closing Note**

This proposal represents a critical step in transitioning Livepeer from a powerful decentralized network into a **measurable, trustworthy, and product-grade platform**. By funding this work, the Livepeer treasury invests in infrastructure that benefits **all network participants** and unlocks future growth, automation, and adoption.