# Livepeer Network-as-a-Product (NaaP)
## System Requirements Document
**Version 1.0 | January 2026**

---

## Executive Summary

This document provides a comprehensive requirements analysis for building the Livepeer Network-as-a-Product (NaaP), a service-oriented platform that transforms the Livepeer protocol into a measurable, scalable, and self-adaptive GPU network for real-time AI inference workloads.

The system consists of three core components:
1. A permissionless protocol enabling GPU orchestrators
2. Publicly monitorable SLA metrics  
3. Workload management utilities

The implementation follows a phased approach across four major milestones, starting with an MVP focused on metrics visibility and culminating in support for complex multi-GPU workload orchestration.

---

## 1. Vision and Value Proposition

### 1.1 Core Deliverables

1. **Real-time AI Workflow Deployment**: Ability for users to deploy real-time AI workflows and request inference
2. **Industry-Leading Latency**: Industry-leading latency for real-time AI and world model workflows
3. **Cost-Effective Scalability**: Cost-effective scalability with pay-as-you-go model and automatic network scaling

### 1.2 Unique Value Differentiators

#### Open Access and Permissionless Nature
Built on public blockchain, enabling anyone to build on the network and reducing platform risk.

#### Open Source and Community Oriented
Users can control, update, and contribute to the project, benefiting from network improvements as token owners.

---

## 2. System Architecture

### 2.1 Three-Component Architecture

#### Component #1: Permissionless Livepeer Protocol
Enables orchestrators to enroll GPUs to the Livepeer Network with clear understanding of:
- Required SLAs for compliance
- Demand characteristics the GPU will satisfy
- Compensation structure

**User Story**: *As an orchestrator, I am able to enroll my GPUs to the Livepeer Network by knowing what SLAs it must comply with, what demands this GPU is to satisfy, and what compensation it pays.*

#### Component #2: Public Monitorable SLA Framework
Provides users with assurance that all inference service requests will be met with pre-agreed, published SLAs for real-time video inference.

**User Story**: *As users of the network, I am ensured by all my inference services requests will be met with a pre-agreed, and published SLAs.*

#### Component #3: Workload Management Utility
Enables users to manage workload lifecycle including deployment, execution, analysis, and sunset of various real-world model workloads.

**User Story**: *As users of the network, I have a set of utilities allowing me to manage my workloads, from deployment, execution, analyzing to workload sunset.*

### 2.2 User Personas

#### Public Orchestrators
Provide essential GPU computing capacity by meeting SLAs.

#### Gateway Providers  
Provide permissionless orchestrator management and workload management utilities.

#### Inference Service Providers
Deploy workloads to the network and manage services through self-hosted gateways and serviceable APIs (e.g., Daydream).

---

## 3. Technical Architecture (High Level)

### 3.1 Data Flow Architecture

```
┌─────────────────────────────────────────────┐
│ Layer 1: GPU Node / Orchestrator Agent     │
│ - AI Runner with GPU Metrics Collector     │
│ - Metrics: FLOPS, Latency, CUE, Reliability│
│ - Direct reporting to "streamr" infra      │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Layer 2: Trickle Protocol Transport Layer  │
│ - Event streaming (heartbeat, SLA events)  │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Layer 3: Gateway Layer                     │
│ - Daydream Gateway / Load Tester Gateway  │
│ - Synthetic workloads → Kafka → ClickHouse│
│ - Pathway metrics (e2e, swap) → streamr   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Layer 4: Analytics Layer                   │
│ - Cloud SPE sinks streamr to data lake    │
│ - Metabase Dashboard                       │
│ - Public APIs per GPU/O/region/workflow   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Layer 5: Explorer UI                       │
│ - Public Dashboard (standalone initially) │
│ - Later embedded in Explorer              │
└─────────────────────────────────────────────┘
```

### 3.2 Telemetry Event Requirements

All telemetry events defining metrics/measurements must include:

- **Stream ID**: Unique identifier defining what the metric measures
- **O address**: Identifying the target GPU
- **Model ID**: Identifying the model the metric is running on
  - Example: `ai3.brzeczyszczykiewicz.work:8935 #streamdiffusion-sdxl-faceid`
- **Gateway ID/IP**: Identifying the source gateway originating the stream

---

## 4. Milestone 1: NaaP MVP Requirements

### 4.1 Goals and Success Criteria

**Primary Goal**: Make NaaP key characteristics measurable and monitorable in real-time

#### Deliverable #1: Core Metrics Dashboard
A dashboard displaying core metrics from the entire AI Network, enabling community members to learn how any specific orchestrator and workflow is performing historically and in real-time.

**Success Criterion**: From the Explorer SLA dashboard, orchestrators can have full visibility on:
- What GPU SLAs criteria to meet for providing competitive services
- What GPUs real-time SLAs is in real-time, per GPU
- What network demands are throughout the time, to inform the demand vs. supply gaps

#### Deliverable #2: Load Test Gateway
An MVP Gateway specialized in executing test loads to monitor network performance metrics, providing community confidence through continuous monitoring with transparent, public-contributed testing datasets.

**Success Criterion**: Community has confidence in overall network performance because a dedicated Gateway continuously monitors the network performance and reliability, with transparent and public contributed high quality testing datasets.

### 4.2 User Stories

| User Role | As a... | I want to... | So that I can... |
|-----------|---------|--------------|------------------|
| **Orchestrator** | Enroll and monitor my GPU capacity on the Livepeer Network | Know real-time SLA compliance and competitive positioning | Optimize service competitiveness through SLA visibility |
| **Gateway Provider** | Operate a public gateway utility and load test tool | Validate GPU reliability and feed metrics into network analytics | Select the best resources for workload requests |
| **Inference Provider** | Deploy AI workflows and view service SLA data | Ensure inferences meet industry-leading latency and cost targets | Have full confidence in underlying infrastructure and its SLAs |
| **Community Observer** | Access public dashboard and APIs | Monitor network health and transparency of decentralized GPU performance | Have comprehensive view of NaaP status and performance |
| **Core Engineer** | Validate metrics pipeline integrity | Know QoS of network before committing engineering efforts | Reduce overall infrastructure risk |

---

## 5. Metrics and SLA Requirements

### 5.1 GPU Performance and Reliability Metrics

#### Core Performance KPIs

1. **Target Output FPS**
   - Measure: Frames per second per model
   - Target: ≥ target fps (model-specific)
   - Aggregation: 5s intervals
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, workflow, region, time

2. **Minimum GPU Requirements**
   - GPU type, VRAM size, per GPU
   - Source: GPU/Orchestrator

3. **Prompt-to-First-Frame Latency**
   - Measure: milliseconds
   - Target: ≤ XX ms
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, workflow, region, time

4. **E2E Stream Latency**
   - Measure: milliseconds (from ingest to egress)
   - Target: ≤ target
   - Source: Gateway
   - Dimensions: O wallet, GPU ID, workflow, region, time

5. **Jitter Coefficient**
   - Measure: σ(fps)/μ(fps)
   - Target: ≤ 0.1
   - Calculation: Standard deviation FPS / mean FPS per workflow
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, workflow, region, time

6. **Startup (Cold) Time**
   - Measure: seconds
   - Target: ≤ threshold
   - Time to first inference
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, workflow, region, time

7. **Network Bandwidth**
   - Measure: Mbps (upload/download speeds)
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, VRAM, CUDA version, workflow, region, time

#### Reliability KPIs

1. **Inference Failure Rate**
   - Measure: Percentage
   - Target: < 1%
   - Calculation: Percentage of failed inference requests
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, workflow, region, time

2. **Swap Rate**
   - Measure: Percentage
   - Target: < 5%
   - Calculation: Percentage of times gateway must swap orchestrators
   - Source: Gateway
   - Dimensions: O wallet, GPU ID, workflow, region, time

#### Cost Economics KPIs

1. **CUDA Utilization Efficiency (CUE)**
   - Measure: Percentage
   - Formula: `(Achieved FLOPS / Peak FLOPS) × 100%`
   - Target: ≥ 80%
   - Source: GPU/Orchestrator
   - Dimensions: O wallet, GPU ID, workflow, region, time
   
   **Insights Provided**:
   - Whether GPU is compute-bound or I/O/memory/network-bound
   - How well kernels are optimized for GPU architecture
   - Whether getting expected value per dollar of GPU usage

### 5.2 Network Demand Metrics

| Metric | Measure | Dimensions | Source | Purpose |
|--------|---------|------------|--------|---------|
| **Total Streams** | Count | Gateway, Orchestrator, Workflow, Region, Time | Gateway | Demand profile |
| **Total Inference Minutes** | Minutes | Gateway, Orchestrator, Workflow, Region, Time | Gateway | Utilization |
| **Inference Minutes by GPU Type** | Minutes | Gateway, GPU Type, Workflow, Region, Time | Gateway | Supply matching |
| **Capacity Rate** | % (total inference mins / all GPU inference cap mins) | Gateway, Orchestrator, GPU Type, Workflow, Region, Time | Gateway | Supply/Demand Gap monitoring |
| **Missing Capacity** | Count | Gateway, Orchestrator, Workflow, Region, Time | Gateway | Supply/Demand Missing Gap |
| **Staking LPT** | LPT | Orchestrator, Workflow, Region, Time | Gateway | Supply |
| **Fee Payment** | ETH | Orchestrator, Workflow, Region, Time | Gateway | Demand |

**Note**: Network demand metrics are computed hourly and can be aggregated at daily, weekly intervals.

### 5.3 Implementation Status Matrix

| Metric Name | Category | Measure | Implementation Complexity | Status |
|-------------|----------|---------|---------------------------|--------|
| Output FPS | Performance | Frames per second | Low | ✅ Exists Today |
| Startup Time | Performance | seconds | Low | ✅ Exists Today |
| Failure Rate | Reliability | % | Medium | ⚠️ Partial |
| Jitter Coefficient | Performance | σ(fps)/μ(fps) | Medium | ⚠️ Partial |
| Swap Rate | Reliability | % | Medium | ⚠️ Partial |
| Up/Down Bandwidth | Network | Mbps | Medium | ❌ Does Not Exist |
| E2E Stream Latency | Performance | milliseconds | High | ❌ Does Not Exist |
| CUDA Utilization Efficiency (CUE) | Economics | Achieved FLOPS / Peak FLOPS | High | ❌ Does Not Exist |
| Prompt-to-First-Frame Latency | Performance | milliseconds | High | ❌ Does Not Exist |

---

## 6. Load Test Gateway Requirements

### 6.1 Purpose and Design

The Load Test Gateway serves as a **community-hosted validation gateway** that continuously performs synthetic workload tests on the Livepeer Network. 

**Goal**: Independently verify network-wide SLAs using transparent, reproducible, and open data sources.

### 6.2 Design Requirements

1. **Community Hosting**: Enable open participation in hosting validation nodes
2. **Shared Datasets**: Run synthetic workloads based on shared, version-controlled datasets
3. **Periodic Scheduling**: Execute jobs on schedule to cover diverse workflows
4. **Telemetry Streaming**: Stream telemetry and result payloads (FPS, latency, reliability) via Trickle Protocol
5. **SLA Verification**: Validate and aggregate results, flag outliers and SLA violations
6. **Public Dashboard Integration**: Feed verified SLA metrics to Explorer Dashboard and Public APIs

### 6.3 Operational Attributes

| Attribute | Description |
|-----------|-------------|
| **Operator** | Community members / contributors (such as Cloud SPE) |
| **Data Source** | Open test datasets curated by Daydream and community |
| **Workload Types** | Synthetic, workload-specific, and random prompt sets |
| **Update Frequency** | Configurable (default: every 10 minutes per region) |
| **Output** | Verified SLA metrics pushed to public dashboard |
| **Visibility** | Fully transparent and queryable via `/gpu/metrics` and `/network/demand` APIs |

### 6.4 Dataset Requirements

Test datasets must evolve to capture the deployment of new workloads and models:

- **Avoid one-size-fits-all datasets**
- **Use prompt diversity and model-specific benchmark prompt sets**
- **Good Prompt Workloads**: Model-specific datasets producing consistent workflow outputs from a given GPU spec
- **Random Prompt Workloads**: Simulate real-world traffic with variants of prompts (bad, edge, and average)

**Example**: Core Stable Diffusion training prompts should be sampled for SD-related testing, not just any random prompt.

### 6.5 Dataset Categories

1. **Good Prompts**: Workload-specific dataset that can have consistent workflow outputs from a given GPU spec
2. **Random Prompts**: Workload tries to simulate real world traffic types, with variants of prompts (bad, edge, and average)

**Important Note**: It is important to have an evolving test dataset that can capture the evolution of:
- Deployed workloads and models
- Should avoid one dataset fit-for-all
- Prompt diversity and model-specific benchmark prompt set need to be used to capture the performance characteristics

---

## 7. Milestone 1 Implementation Requirements

### 7.1 Milestone 1.0: Design Spec RFC

**Deliverable**: RFC for technical document on SLAs and technical architecture

**Details**:
- Taking Daydream as design partner to publish Phase 1 RFCs
- Two sets of metrics:
  1. GPUs metrics on (core performance, reliability, and workload) per GPU in real-time, per workload
  2. Network historical demand distribution statistics

**Timeline**: 2 weeks

**Working Group**: 
- Evan, Qiang, Doug, Rick
- One representative from Cloud SPE
- One representative from network advisory board

**Review By**: Community and Inc engineering, concluded by end of November

**RFC Document**: Network-as-a-Product MVP – SLA Metrics & Analytics Infra

### 7.2 Milestone 1.1: Reference Implementation

**Goal**: Launch Explorer AI SLA monitoring dashboard with detailed real-time orchestrator SLA reporting with dimensions of:
- Per orchestrator
- Per region
- Per workload
- Per period and now

**Recommendation**: Evan and Cloud SPE

**Timeline**: 4 weeks (many works have been done)

**Prerequisite**: Detailed scoping exercise (can be led by Qiang)

#### Implementation Tasks:

1. **Explorer O SLAs Reporting Dashboard**
   - One option is to directly embed Metabase dashboard

2. **Gateway Load Test Scheduler**
   - GPU metric gateway reporting leveraging current Daydream load testing toolkit
   - Dedicated network testing gateway to ensure SLA

3. **Public Git Repository for Load Test Datasets**
   - **Good Prompt**: Workload-specific dataset with consistent outputs from a given GPU spec
   - **Random Prompt**: Workload tries to simulate real-world traffic with prompt variants (bad, edge, average)

4. **SLA Data Pipeline**
   - Analytic model and orchestrator reporting

5. **API for Explorer SLA Dashboard**
   - Public API endpoints for metrics access

### 7.3 Architecture Note

**Initial Approach**: Standalone Cloud SPE dashboard will be step 1, before making it embeddable in Explorer. This reduces interdependencies before the MVP is implemented.

**Data Enrichment**: Daydream Metabase dashboard will be enriched with new set of metrics defined in Section 5.

---

## 8. API Requirements

### 8.1 Base Configuration

- **Base URL**: `https://api.livepeer.network/v1/sla`
- **Authentication**: 
  - Public Read API (no auth) for aggregate views
  - Rate limiting applied
  - JWT auth for orchestrator and gateway private metrics (TBC, reviewed by Cloud SPE)

### 8.2 API Endpoints

| Endpoint | Method | Description | Parameters | Response |
|----------|--------|-------------|------------|----------|
| `/gpu/metrics` | GET | Retrieve per-GPU real-time metrics | `o_wallet`, `gpu_id`, `region`, `workflow`, `time_range` | JSON with performance/reliability fields |
| `/network/demand` | GET | Aggregate network demand data | `gateway`, `region`, `workflow`, `interval` | Stream/inference mins stats |
| `/sla/compliance` | GET | Return SLA compliance score for orchestrator | `orchestrator_id`, `period` | Score (0-100) |
| `/datasets` | GET | List public load test datasets | `workflow`, `type` (good, random, bad) | Dataset metadata |

### 8.3 Sample API Response

#### `/gpu/metrics` Sample Output:
```json
{
  "o_wallet": "0x8fC1234a5B9dE0C7d43C8aA92b1D1234567890AB",
  "orchestrator_agent_version": "v1.2.7",
  "time_window": "2025-11-12T09:55:00Z/2025-11-12T10:00:00Z",
  "gpu_id": "GPU-a8d13655-197b-2226-d09f-b4b81e07504b",
  "metrics": {
    "output_fps": 19.98,
    "prompt_to_first_frame_latency_ms": 1807,
    "jitter_coefficient": 0.08,
    "failure_rate_pct": 0.2,
    "cuda_utilization_efficiency_pct": 82.5
  }
}
```

---

## 9. Future Milestones Overview

### 9.1 Milestone 2: SLAs Gateway (Scoring, Selection, and Incentive)

**Goal**: Enable NaaP to be self-adaptive and scalable based on core SLAs

#### Key Deliverables:

1. **SLA Score Computing**
   - Well-defined scoring algorithms based on Milestone 1 deliverables
   - Establish fair weighted scoring algorithm
   - What is considered "fair"

2. **Selection API**
   - Real-time selection/swap algorithm per workload
   - Dynamic benchmark publishing (workflow publishers must specify this)

3. **SLA-Based Payment API**
   - Dynamic incentive algorithms enabling diversified workload participation
   - Consider different workload demands and pricing for end users
   - Revenue distribution through the protocol

#### Published API Specification:

- **Query API**: Retrieve any Public O SLA scoring in real-time/historic records
- **SLAs Initialization API**: Get an orchestrator started with initial fair SLA Score
- **Min Qualifying SLA Score Query API**: Query minimum SLA score
- **Min Qualifying SLA Score Update API**: Update minimum SLA score (allow Gateway to update)

#### Reference Implementation Components:

1. **SLAs Scoring**
   - Score update interval
   - Score publishing and repository
   - Score per workload (using different target for different workload)
     - Example: Krea model will have much lower fps vs. StreamDiff

2. **Selection Algorithm**
   - Query for available GPUs using (workload, SLAs, cost)

3. **Selection Statistics Reporting**
   - Based on SLAs
   - Real-time score balanced with historic scores

4. **Payment (SLAs Based Incentive Framework)**
   - Reflected by % selection and swapping
   - Further manifested as total inference minutes
   - Fee per workload

#### Key Assumptions:

- Different GPU has different cost base
- Different workload has different GPU and power consumption
- Different providers offer different levels of SLAs

**Main Design Question**: How to fairly compensate the differences above

**Impact**: SLAs-based incentive will bring "Protocol Changes" on how different stakeholders (Gateway provider, Orchestrators, Delegators) can collaborate together, enabling best economics of demand and QoS-driven scalability with a self-perpetrated dynamic market mechanism.

### 9.2 Milestone 3: Workload Utility Control Plane

**Goal**: Provide well-defined Gateway API spec on workload management enabling community-developed workload utilities

The Control Plane integrates workload management API in ops workflow with an opinionated DevX/UX design.

#### Core Use Cases:

##### User Story #1: Workload Creation

As a workload creator, I need to create a descriptor file (in JSON format) to describe:
- Workload ID
- Workload model card
- Where to download AI models from and access key
- SLAs target values
- Min CUDA
- Min VRAM
- Supported GPUs
- Min # of instances

**Capabilities**:
- Query the network to discover available resources, quantity, and pricing
- Publish workload request to registry for orchestrator visibility until fulfilled

**Network Benefit**: Users' workload resource requests are published as demand requests, producing future demand projection. This projection, together with historical demand statistics, helps orchestrators respond to growth in real-time.

##### User Story #2: Resource Management

Once GPU resources are allocated, as a user I can:
- Deploy the workload to the network
- Start the inference services
- Publish the inference services with a gated endpoint URL
- Stop the inference services
- Release/uninstall the workload from allocated GPUs
- Perform SLA monitoring of published inference services

##### User Story #3: Security Management

As a user, I can manage workload security by:
- Performing all workload management tasks through an authorized API key
- Setting up API key gate for the inference endpoint

##### User Story #4: Orchestrator Response

As an orchestrator, I can respond to the workload request descriptor to provide my GPUs running the workload by:
- Deploying the descriptor
- Updating the demand requests with my capacity

### 9.3 Milestone 4: Complex Workload Handling

**Goal**: Enable users to publish and manage complex workloads with higher SLAs (>99.9% reliability) using cluster-based architecture

**Key Changes**:
- Extend paradigm from n:1 (streams per GPU) to 1:m (stream to many GPUs)
- GPU cluster-based workload management for redundancy and in-workload scalability
- Public orchestrators offer GPU pools as clusters, not single GPUs

**Note**: This milestone will be prioritized when enterprise-level use cases emerge. The goal is to enable more robust resource orchestration during runtime.

**Current Workload Topology Assumption** (Milestones 1-3): 
- 1 or many-to-1 mapping between stream and GPUs
- Stream:GPU = n:1

---

## 10. Technology Stack Requirements

### 10.1 Modern Data Lakehouse Architecture

This architecture transitions from a localized data setup to a scalable, distributed Modern Data Lakehouse.

#### A. Ingestion Layer (The Source)

1. **Push Events**
   - Go-Livepeer nodes (Gateways and Broadcasters) push real-time events directly to **Apache Kafka**

2. **Pull Scraper**
   - Asynchronous service scrapes HTTP endpoints from Orchestrator nodes
   - Collects supplemental metadata
   - Publishes to Kafka to unify data flow

3. **Kafka Connect**
   - Utilizes S3 Sink connector
   - Batch streams data from Kafka into **Parquet** files
   - Stored on **MinIO**

#### B. Storage & Table Format (The Lake)

1. **Object Storage**
   - **MinIO** serves as persistent landing zone for raw Parquet data

2. **Table Format**
   - **Apache Iceberg** implemented over Parquet files
   - Provides ACID transactions
   - Schema evolution
   - Hidden partitioning
   - Allows "lake" to be queried like traditional relational database
   - Eliminates "Malformed JSON" errors typical of raw file access

3. **Catalog**
   - **Hive Metastore (HMS)** or **Project Nessie**
   - Maintains metadata for Iceberg tables
   - Tracks snapshots and data file locations for query engine

#### C. Transformation Layer (The "Brains")

1. **Distributed Query Engine**
   - **Trino** serves as compute muscle
   - Performs complex, multi-stage joins across massive datasets in S3

2. **Transformation Logic**
   - **dbt (data build tool)** manages SQL transformation pipeline
   - Orchestrates flow from raw event tables to "Silver" cleaned views
   - Finally to "Gold" KPI tables

3. **Logic Example**
   - dbt matches `gateway_receive_stream_request` events with `gateway_receive_first_processed_segment` using `request_id` to calculate latency

#### D. Serving & Dashboarding Layer (The Hot Layer)

1. **Analytics Database**
   - **ClickHouse** acts as high-performance serving layer
   - Stores finalized aggregates produced by Trino

2. **Visualization**
   - **Metabase** (or custom Explorer UI)
   - Queries ClickHouse
   - Provides Orchestrators and Users with real-time SLA visibility

### 10.2 Technology Decision Tree

| Technology | Selection Criteria | Decision |
|------------|-------------------|----------|
| **Compute** | Needs to scale horizontally and join multiple data sources (S3 + Postgres) | **Trino** |
| **Storage Format** | Needs schema safety and high performance on "naked" files | **Apache Iceberg** |
| **Transformation** | Needs version-controlled SQL, modularity, and automated testing | **dbt** |
| **Serving Layer** | Needs sub-second response times for aggregate queries (Avg/Sum) | **ClickHouse** |

### 10.3 Deployment Component Summary (Docker Stack)

To build this system, the engineering team requires the following containerized services:

1. **MinIO**: Persistent object storage for the data lake
2. **Postgres**: Metadata backend for Hive Metastore
3. **Hive Metastore**: The catalog layer for Iceberg tables
4. **Trino**: Primary query engine for heavy transformations
5. **ClickHouse**: Fast-access database for dashboards and API consumption
6. **dbt Runner**: To compile and execute the transformation DAG

---

## 11. Key Performance Indicators (KPIs) Implementation

The following metrics defined in the NaaP MVP spec will be generated by the **dbt + Trino** pipeline:

### 11.1 Core Performance KPIs

1. **Output FPS**
   - Averaged per model from `ai_stream_status` events

2. **Prompt-to-First-Frame Latency**
   - Time delta between request receipt and first processed frame

3. **E2E Stream Latency**
   - Total time from ingest to egress

4. **Jitter Coefficient**
   - Standard deviation of FPS divided by mean FPS per workflow

5. **Startup (Cold) Time**
   - Latency measured for initial stream initialization

### 11.2 Reliability & Economic KPIs

1. **Inference Failure Rate**
   - Percentage of `error` type events relative to total requests

2. **Swap Rate**
   - Percentage of requests where Gateway had to switch Orchestrators

3. **CUE (CUDA Utilization Efficiency)**
   - Calculated as: `Achieved FLOPS / Peak FLOPS * 100%`

---

## 12. Core Analytics Models

These models provide initial insights for the dashboard:

| Model Name | Description | Input Data Sources | Output | Frequency | Example Use |
|------------|-------------|-------------------|--------|-----------|-------------|
| `sla_compliance_score` | Weighted aggregation of latency, reliability, and jitter | `gpu_metrics` | Score (0-100) | Hourly | Ranking orchestrators |
| `cue_efficiency_index` | GPU compute utilization adjusted for cost | `gpu_metrics` + `fee_data` | CUE % | 5 min | Identifying underutilized GPUs |
| `demand_supply_heatmap` | Maps active workloads to available GPUs | `network_demand` + `gpu_inventory` | Region×Workflow matrix | 1 hour | Scaling decisions |

---

## 13. Event Schema Examples

Understanding the actual event data structure is crucial for implementation. Based on the streaming events data provided:

### 13.1 AI Stream Status Event

```json
{
  "type": "ai_stream_status",
  "id": "uuid",
  "timestamp": "unix_timestamp_ms",
  "gateway": "gateway_id",
  "data": {
    "stream_id": "stream-374e1e-mk6xofol",
    "request_id": "ce6ee475",
    "pipeline": "streamdiffusion-sdxl-faceid",
    "pipeline_id": "",
    "state": "ONLINE",
    "start_time": 1767966622510,
    "last_state_update_time": 1767966634193,
    "last_status_timestamp": 1767966802512,
    "timestamp": 1767966802512,
    "orchestrator_info": {
      "address": "0x52cf2968b3dc6016778742d3539449a6597d5954",
      "url": "https://rtav-orch2.xodeapp.xyz:28935"
    },
    "inference_status": {
      "fps": 19.987593721986812,
      "last_output_time": 1767966802505,
      "last_error": null,
      "last_error_time": null,
      "restart_count": 0,
      "last_restart_time": null,
      "last_restart_logs": null,
      "last_params_update_time": 1767966633725,
      "last_params_hash": "8c85e134607acd293c9efae72ce6ebe9",
      "last_params": {
        "width": 512,
        "height": 512,
        "prompt": "spiderman with a business suit"
      }
    },
    "input_status": {
      "fps": 20.087531690596748,
      "last_input_time": 1767966802487
    }
  }
}
```

**Key Metrics Extractable**:
- Output FPS: `data.inference_status.fps`
- Input FPS: `data.input_status.fps`
- Jitter Coefficient: Calculate from FPS variance over time
- Failure indicators: `data.inference_status.last_error`

### 13.2 Stream Ingest Metrics Event

```json
{
  "type": "stream_ingest_metrics",
  "id": "uuid",
  "timestamp": "unix_timestamp_ms",
  "gateway": "gateway_id",
  "data": {
    "stream_id": "stream-374e1e-mk6xofol",
    "request_id": "ce6ee475",
    "pipeline_id": "",
    "timestamp": 1767966802523,
    "stats": {
      "conn_quality": "good",
      "peer_conn_stats": {
        "ID": "iceTransport",
        "BytesReceived": 38372218,
        "BytesSent": 138919
      },
      "track_stats": [
        {
          "type": "video",
          "packets_received": 33061,
          "packets_lost": 0,
          "packet_loss_pct": 0,
          "jitter": 6.874066136065424,
          "latency": 0,
          "rtt": 0,
          "last_input_ts": 180.189,
          "last_output_ts": 0
        },
        {
          "type": "audio",
          "packets_received": 9016,
          "packets_lost": 0,
          "packet_loss_pct": 0,
          "jitter": 20.259131240488912,
          "latency": 0,
          "rtt": 0,
          "last_input_ts": 180.1891,
          "last_output_ts": 0
        }
      ]
    }
  }
}
```

**Key Metrics Extractable**:
- Network bandwidth: Calculate from `BytesReceived` and `BytesSent` over time
- Packet loss rate: `track_stats[].packet_loss_pct`
- Jitter: `track_stats[].jitter`
- Connection quality: `stats.conn_quality`

### 13.3 Stream Trace Events

```json
{
  "type": "stream_trace",
  "id": "uuid",
  "timestamp": "unix_timestamp_ms",
  "gateway": "gateway_id",
  "data": {
    "stream_id": "stream-3c1jio-mk6xu4go",
    "request_id": "12b4ba45",
    "pipeline_id": "",
    "type": "gateway_receive_stream_request",
    "timestamp": 1767966890407,
    "orchestrator_info": {
      "address": "",
      "url": ""
    }
  }
}
```

**Critical Trace Types for Latency Calculation**:
- `gateway_receive_stream_request`: Stream request received
- `runner_send_first_processed_segment`: First segment processed by runner
- `gateway_send_first_ingest_segment`: First segment sent to ingest
- `runner_receive_first_ingest_segment`: First segment received by runner
- `gateway_receive_first_processed_segment`: First processed segment received by gateway
- `gateway_receive_few_processed_segments`: Multiple segments received
- `gateway_ingest_stream_closed`: Stream closed

**Latency Calculations**:
- **Prompt-to-First-Frame**: `gateway_receive_first_processed_segment.timestamp - gateway_receive_stream_request.timestamp`
- **E2E Stream Latency**: Aggregate across full stream lifecycle

### 13.4 AI Stream Events (Errors)

```json
{
  "type": "ai_stream_events",
  "id": "uuid",
  "timestamp": "unix_timestamp_ms",
  "gateway": "gateway_id",
  "data": {
    "stream_id": "stream-374e1e-mk6xofol",
    "request_id": "ce6ee475",
    "pipeline": "streamdiffusion-sdxl-faceid",
    "pipeline_id": "",
    "capability": "",
    "type": "error",
    "message": "whip disconnected"
  }
}
```

**Key Metrics Extractable**:
- Inference failure rate: Count of `type: "error"` events
- Error types and messages for debugging

### 13.5 Payment Events

```json
{
  "type": "create_new_payment",
  "id": "uuid",
  "timestamp": "unix_timestamp_ms",
  "gateway": "gateway_id",
  "data": {
    "orchestrator": "https://rtav-orch2.xodeapp.xyz:28935",
    "recipient": "0xd00354656922168815Fcd1e51CBddB9e359e3C7F",
    "sender": "0x5aE4E42dB3671370a0c25AfF451E7482aAEc3D0B",
    "sessionID": "ff083926",
    "manifestID": "6fa05caf",
    "requestID": "ce6ee475",
    "numTickets": "2",
    "faceValue": "2400000000000000 WEI",
    "winProb": "0.0010000000",
    "price": "1646.821 wei/pixel",
    "capability": "",
    "clientIP": ""
  }
}
```

**Key Metrics Extractable**:
- Fee payment per workload
- Pricing per pixel
- Payment frequency

---

## 14. Implementation Phases and Timeline

### Phase 1: Foundation (Weeks 1-2) - Milestone 1.0
- Complete Design Spec RFC
- Establish working group and review process
- Define core metrics and data schemas
- Architecture validation

### Phase 2: MVP Development (Weeks 3-6) - Milestone 1.1
- Set up data lakehouse infrastructure (MinIO, Hive, Trino, ClickHouse)
- Implement ingestion pipelines (Kafka, event scrapers)
- Develop dbt transformation models
- Build initial Metabase dashboards

### Phase 3: Load Test Gateway (Weeks 7-8)
- Deploy load test gateway infrastructure
- Create and version control test datasets
- Implement continuous monitoring
- Integrate with analytics pipeline

### Phase 4: API Development (Weeks 9-10)
- Implement public API endpoints
- Add authentication and rate limiting
- Create API documentation
- Integration testing

### Phase 5: Dashboard Integration (Weeks 11-12)
- Finalize standalone Cloud SPE dashboard
- Prepare Explorer embedding approach
- User acceptance testing
- Production deployment

### Phase 6: Stabilization and Handoff (Weeks 13-14)
- Performance optimization
- Documentation completion
- Community review and feedback
- Operations transition

---

## 15. Success Metrics and Validation

### 15.1 Quantitative Success Criteria

1. **Dashboard Availability**: 99.5% uptime for public dashboard
2. **API Performance**: < 500ms response time for 95th percentile
3. **Data Freshness**: Metrics updated within 5 minutes of event occurrence
4. **Load Test Coverage**: Test all active workflows at least every 10 minutes
5. **Metric Completeness**: 100% of defined core metrics available

### 15.2 Qualitative Success Criteria

1. **Orchestrator Adoption**: Positive feedback from at least 5 orchestrators using dashboard for optimization
2. **Gateway Integration**: At least 2 gateways (including Daydream) actively consuming metrics
3. **Community Engagement**: Active discussion and contributions to test datasets
4. **API Usage**: At least 3 independent tools/dashboards consuming public APIs

### 15.3 Validation Approach

1. **Alpha Testing**: Internal team validation (Week 6)
2. **Beta Testing**: Selected orchestrators and gateway providers (Week 10)
3. **Community Review**: Public RFC review period (Week 12)
4. **Production Launch**: Phased rollout with monitoring (Week 14)

---

## 16. Risk Analysis and Mitigation

### 16.1 Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| **Data Pipeline Failures** | High | Medium | Implement robust error handling, retry mechanisms, and monitoring alerts |
| **Scale/Performance Issues** | High | Medium | Start with horizontal scaling capability, load testing before production |
| **Event Schema Changes** | Medium | High | Version event schemas, implement backward compatibility |
| **Data Quality Issues** | Medium | Medium | Implement data validation, outlier detection, and alerting |

### 16.2 Operational Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| **Load Test Gateway Downtime** | Medium | Low | Deploy multiple instances, community-hosted redundancy |
| **API Rate Limiting Issues** | Low | Medium | Implement adaptive rate limiting, clear documentation |
| **Dashboard Performance Degradation** | Medium | Low | Caching strategy, query optimization, CDN |

### 16.3 Adoption Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| **Low Orchestrator Engagement** | High | Medium | Early involvement, clear value demonstration, documentation |
| **Test Dataset Staleness** | Medium | Medium | Community contribution process, automated validation |
| **API Misuse** | Low | Low | Rate limiting, usage analytics, clear ToS |

---

## 17. Dependencies and Prerequisites

### 17.1 Infrastructure Dependencies

1. **Existing Livepeer Infrastructure**
   - Go-Livepeer nodes emitting events
   - Trickle protocol transport
   - Existing Kafka/ClickHouse setup (Daydream)

2. **Cloud Infrastructure**
   - Cloud SPE hosting capacity
   - Network connectivity
   - Storage allocation

3. **Development Tools**
   - Docker/Kubernetes for containerization
   - CI/CD pipeline
   - Monitoring and alerting systems

### 17.2 Data Dependencies

1. **Event Telemetry**
   - Consistent event schema from GPU nodes
   - Gateway event emissions
   - Orchestrator metadata availability

2. **Historical Data**
   - Baseline data for trend analysis
   - Existing Daydream metrics for comparison

### 17.3 Team Dependencies

1. **Core Development Team**
   - Backend engineers (data pipeline)
   - Frontend engineers (dashboard)
   - DevOps engineers (infrastructure)

2. **Subject Matter Experts**
   - Evan (go-livepeer expertise)
   - Qiang (architecture design)
   - Doug (product requirements)
   - Rick (community liaison)

3. **External Partners**
   - Cloud SPE (infrastructure and operations)
   - Daydream (design partner, test datasets)
   - Network advisory board (governance)

---

## 18. Glossary

| Term | Definition |
|------|------------|
| **NaaP** | Network-as-a-Product - Service-oriented product framework for Livepeer |
| **SLA** | Service Level Agreement - Published performance and reliability targets |
| **Orchestrator (O)** | Node operator providing GPU capacity to the network |
| **Gateway** | Entry point managing workload routing to orchestrators |
| **CUE** | CUDA Utilization Efficiency - Ratio of achieved to peak FLOPS |
| **E2E Latency** | End-to-end latency from request to response |
| **P2F Latency** | Prompt-to-first-frame latency |
| **Jitter** | Variance in frame rate or latency |
| **Swap Rate** | Frequency of orchestrator changes mid-stream |
| **Load Test Gateway** | Validation gateway running synthetic workloads |
| **Lakehouse** | Architecture combining data lake and data warehouse |
| **dbt** | Data build tool for SQL transformations |
| **Trino** | Distributed SQL query engine |
| **ClickHouse** | Column-oriented database for analytics |
| **Apache Iceberg** | Table format for large analytic datasets |

---

## 19. References

1. **Livepeer Network-as-a-Product Vision** (PDF)
   - Core vision and value proposition
   - Three-component architecture
   - Four-milestone roadmap

2. **Network-as-a-Product MVP – SLA Metrics & Analytics Infra** (PDF)
   - Technical architecture details
   - Metrics specifications
   - API requirements
   - User stories

3. **Cloud SPE Pre-Proposal: Metrics and SLA Foundations** (Forum Post)
   - Budget and timeline
   - Revised scope (reset to MVP)
   - Community feedback integration

4. **High Level Tech Architecture Document** (Markdown)
   - Data lakehouse implementation details
   - Technology decision tree
   - Deployment components

5. **Streaming Events Data** (JSON)
   - Actual event schemas
   - Real-world data examples
   - Event flow patterns

---

## 20. Appendices

### Appendix A: Event Type Reference

Complete list of event types observed in streaming data:

1. `ai_stream_status` - GPU/inference status updates
2. `stream_ingest_metrics` - Network and connection statistics
3. `stream_trace` - Lifecycle and timing events
4. `ai_stream_events` - Pipeline events including errors
5. `create_new_payment` - Payment and pricing information
6. `network_capabilities` - Available orchestrator capabilities
7. `discovery_results` - Orchestrator discovery outcomes

### Appendix B: Dimension Reference

Common dimensions used across metrics:

- `o_wallet`: Orchestrator wallet address
- `gpu_id`: Unique GPU identifier
- `workflow`: Pipeline/model identifier (e.g., streamdiffusion-sdxl-faceid)
- `region`: Geographic region
- `time`: Timestamp or time range
- `gateway`: Gateway identifier
- `stream_id`: Unique stream identifier
- `request_id`: Unique request identifier
- `model_id`: Model identifier with orchestrator URL

### Appendix C: Calculation Formulas

#### Jitter Coefficient
```
jitter_coefficient = σ(fps) / μ(fps)
where:
  σ(fps) = standard deviation of FPS measurements over time window
  μ(fps) = mean FPS over same time window
```

#### CUDA Utilization Efficiency (CUE)
```
CUE = (Achieved_FLOPS / Peak_FLOPS) × 100%
where:
  Achieved_FLOPS = measured floating point operations per second
  Peak_FLOPS = theoretical maximum FLOPS for GPU model
```

#### Capacity Rate
```
capacity_rate = (total_inference_minutes / total_GPU_capacity_minutes) × 100%
where:
  total_inference_minutes = sum of all active inference time
  total_GPU_capacity_minutes = sum of all available GPU time
```

#### Failure Rate
```
failure_rate = (error_events / total_requests) × 100%
where:
  error_events = count of events with type="error"
  total_requests = total number of inference requests
```

#### Swap Rate
```
swap_rate = (orchestrator_swaps / total_streams) × 100%
where:
  orchestrator_swaps = number of times gateway switched orchestrators
  total_streams = total number of streams processed
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | January 2026 | Technical Team | Initial comprehensive requirements document |

**Review Status**: Draft for Community Review

**Next Review Date**: End of milestone 1.0 (Week 2)

**Distribution List**: 
- Livepeer Inc Engineering Team
- Cloud SPE Team
- Network Advisory Board
- Community (Public)

---

*End of Requirements Document*