# **AI Analytics Architecture — Detailed Explanation**

This diagram presents a **high-level overview of the analytics data flow** used in *Livepeer’s AI Subnet* (as of July 2024). It shows how AI jobs move through the network, how metrics are produced, how they are streamed, and how they are ultimately consumed by analytics systems and dashboards.

<img src="https://github.com/Cloud-SPE/livepeer-treasury-proposals/blob/main/2025-NaaP-PROPOSAl/Analytics_Architecture.png?raw=true" height="500" width="500">

---

## **1. Job Submission Layer**

### **Apps**

* These are any applications or clients that submit AI jobs.
* Jobs flow into the network through gateways.

### **AI Tester Tool**

* A separate open-source tool designed specifically for submitting *test* AI jobs.
* Useful for validation, benchmarking, or integration testing.

---

## **2. Ingress Layer — Gateways**

### **Gateways**

* Receive AI jobs from Apps and the AI Tester Tool.
* Responsible for queueing, normalizing, and forwarding jobs into the orchestration layer.
* Gateways publish **Gateway Metrics** into the Streamr network (e.g., job rates, errors, latencies).
* Diagram notes that this includes both **Cloud SPE’s gateway** and other community-run gateways.

---

## **3. Orchestration Layer**

### **Orchestrators**

* Receive jobs from Gateways.
* Decide which AI Worker/Runner should process each job.
* Publish **Orchestrator Metrics** (e.g., assignment counts, health, job success/failure).

---

## **4. Execution Layer**

### **AI Workers / Runners**

* Perform the actual compute tasks (AI inference, processing, etc.).
* Publish **Runner Metrics** into Streamr (e.g., performance KPIs, throughput, system stats).

---

## **5. Analytics Transport Layer**

### **Streamr.Network**

This is the **real-time data pipeline** for all metrics.

* Receives metrics from Gateways, Orchestrators, and Runners.
* Acts as a pub/sub network:

  * **Producers** publish metrics.
  * **Consumers** read real-time streams.
* Feeds data to multiple downstream consumers (analytics systems, dashboards, apps).

### **Metrics Access Controller**

* An application that manages permissions for which Gateways and Orchestrators are allowed to publish to certain Streamr streams.
* Can enable or disable stream write access dynamically.
* Example use: ensuring only active operators can publish metrics.

---

## **6. Analytics Consumption Layer**

### **Cloud SPE Data Consumer**

* A designated consumer that:

  * Reads real-time metrics from Streamr.
  * Aggregates and stores them into a database for long-term analytics, dashboards, and monitoring.
* This is the primary source that powers analytics features.

### **Other Data Consumers**

* Represents any additional applications that want to read the same real-time metrics.
* Could be community dashboards, research tools, monitoring systems, etc.

---

## **7. Analytics Storage & Access Layer**

### **Database (Aggregated Metric Store)**

* Stores aggregated metrics for:

  * historical views,
  * dashboards,
  * performance reports,
  * alerts,
  * long-running analytics.

### **Cloud SPE Analytics Endpoint**

* API endpoint that exposes aggregated analytics data.
* Used by user interfaces (dashboards, explorers, monitoring tools).

---

## **8. User Interface Layer**

### **Dashboard Access Application**

* An open-source example tool that allows users to view analytics dashboards using data served from the Cloud SPE Analytics Endpoint.

### **Livepeer Explorer**

* The main external-facing UI for:

  * Viewing AI performance metrics,
  * Displaying leaderboard rankings,
  * Monitoring orchestrators and gateways.

---

# **9. Full Data Flow Summary (End-to-End)**

Here’s a step-by-step walk-through of how data moves through the architecture:

1. **Apps & AI Tester Tool** submit AI jobs.
2. **Gateways** ingest the jobs and publish gateway-level metrics.
3. **Orchestrators** receive jobs from gateways, schedule work, and publish orchestrator metrics.
4. **AI Workers/Runners** execute the tasks and publish runner metrics to the Orchestrator.
5. All metric streams flow into **Streamr.Network**, the real-time pub/sub backbone.
6. **Metrics Access Controller** governs who may publish to the streams.
7. **Cloud SPE Data Consumer** reads metrics from Streamr and:

   * Aggregates them,
   * Stores them in a database.
8. The **Analytics Endpoint** provides access to stored metrics.
9. Dashboards and tools like **Livepeer Explorer** consume these analytics to display:

   * AI performance leaderboards,
   * Gateway/Orchestrator health,
   * System-wide metrics,
   * Real-time analytics.

---

# **10. Key Insights**

* The architecture is *modular*: job processing and analytics are decoupled.
* Streamr acts as a **shared metrics nervous system** for the entire AI subnet.
* Multiple applications can consume the same metrics without interfering with each other.
* Analytics are generated in **real time**, but also aggregated for longer-term visibility.
* Permissions and stream access are controlled centrally to maintain integrity.
