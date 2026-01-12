# Livepeer NaaP Data Gap Analysis
**Analysis Date: January 2026**

Based on analysis of streaming event JSON files against Livepeer Foundation Requirements and NaaP MVP specifications.

---

## Executive Summary

This analysis identifies critical gaps between **what data currently exists** in the streaming events and **what data is required** to meet the Livepeer Foundation Requirements and NaaP MVP goals.

**Key Finding**: While ~60% of required metrics can be derived from existing data, significant gaps exist in:
1. GPU-level attribution (no GPU_ID in most events)
2. Gateway identification (empty gateway fields)
3. Advanced performance metrics (CUDA utilization, cold start times)
4. Geographic/region data
5. Comprehensive error categorization

---

## 1. Foundation Requirements Compliance Analysis

### 1.1 MUST HAVE Requirements

#### ✅ PARTIALLY MET: Unified Metric Definitions
**Status**: 60% Complete

**What EXISTS**:
- Output FPS (from `ai_stream_status.inference_status.fps`)
- Input FPS (from `ai_stream_status.input_status.fps`)
- Network jitter (from `stream_ingest_metrics.stats.track_stats[].jitter`)
- Packet loss (from `stream_ingest_metrics.stats.track_stats[].packet_loss_pct`)
- Connection quality (from `stream_ingest_metrics.stats.conn_quality`)
- Basic error events (from `ai_stream_events` with `type: error`)
- Restart counts (from `ai_stream_status.inference_status.restart_count`)
- Stream state (from `ai_stream_status.state`)
- Payment/pricing data (from `create_new_payment`)

**What's MISSING**:
- ❌ **Prompt-to-First-Frame Latency**: Can be calculated from trace events but requires correlation logic not yet implemented
- ❌ **E2E Stream Latency**: Requires gateway-to-gateway timing not captured
- ❌ **Jitter Coefficient**: Requires standard deviation calculation over FPS time series (not just single jitter value)
- ❌ **CUDA Utilization Efficiency (CUE)**: No FLOPS data captured
- ❌ **Cold Start Time**: No explicit cold start timing captured
- ❌ **Swap Rate**: No orchestrator swap events captured
- ❌ **Bandwidth utilization metrics**: Bytes transferred exist but not normalized to capacity
- ❌ **GPU utilization percentage**: No GPU load metrics

---

#### ⚠️ CRITICAL GAP: Network-Wide Aggregation
**Status**: Major Gaps in Attribution Data

**Current State**:
```
✅ Orchestrator Address: Available in most events
✅ Orchestrator URL: Available in most events  
❌ GPU ID: ONLY in network_capabilities event, NOT in operational events
❌ Gateway ID: Field exists but is EMPTY STRING in all events
❌ Region: Not captured anywhere
❌ Workflow consistent naming: Inconsistent (pipeline vs model_id vs manifestID)
```

**Critical Issues**:

1. **GPU-Level Attribution MISSING**
   - `ai_stream_status` has orchestrator info but NO `gpu_id` field
   - Cannot attribute performance to specific GPUs
   - Cannot track GPU-level metrics over time
   - GPU info only appears in `network_capabilities` event (happens once)
   
   **Impact**: Cannot meet requirement "Metrics attributable to individual orchestrators"

2. **Gateway Identification BROKEN**
   - All events have `"gateway": ""` (empty string)
   - Cannot identify which gateway originated requests
   - Cannot analyze gateway-specific performance
   - Cannot track gateway demand patterns
   
   **Impact**: Cannot distinguish between gateway providers' usage

3. **No Region Data**
   - No geographic region captured in any event
   - Cannot perform region-based analysis
   - Cannot meet requirement for regional attribution
   
   **Impact**: Cannot provide region-specific SLAs

**Required Fixes**:
```json
// CURRENT (broken):
{
  "type": "ai_stream_status",
  "gateway": "",  // ❌ EMPTY
  "data": {
    "orchestrator_info": {
      "address": "0x52cf...",
      "url": "https://..."
      // ❌ NO GPU_ID
    }
  }
}

// REQUIRED:
{
  "type": "ai_stream_status", 
  "gateway": "gateway-daydream-us-east",  // ✅ POPULATED
  "data": {
    "orchestrator_info": {
      "address": "0x52cf...",
      "url": "https://...",
      "gpu_id": "GPU-a8d13655-197b-2226-d09f-b4b81e07504b",  // ✅ ADDED
      "region": "us-east-1"  // ✅ ADDED
    }
  }
}
```

---

#### ✅ MOSTLY MET: Public Data Access
**Status**: 80% Complete

**What EXISTS**:
- JSON event stream is structured and parseable
- Events have consistent schemas
- Timestamps are present
- Request IDs enable correlation

**What's MISSING**:
- ❌ No documented schema definitions
- ❌ No data dictionary
- ❌ No query examples
- ❌ No API documentation (API doesn't exist yet)

---

#### ❌ NOT MET: Verifiable Test Load Results
**Status**: 0% Complete - Test Load Gateway Doesn't Exist Yet

**What's MISSING**:
- ❌ No test load gateway operational
- ❌ No standardized test datasets
- ❌ No test load identification in events
- ❌ No synthetic workload markers
- ❌ No reproducible test scenarios

**Required Implementation**:
1. Deploy load test gateway
2. Create versioned test datasets in GitHub
3. Add `is_test_load: true` flag to events from test gateway
4. Implement test scenarios (good prompts, bad prompts, edge cases)
5. Store test results with versioning

---

### 1.2 SHOULD HAVE Requirements

#### ⚠️ PARTIAL: Entity-Level Attribution
**Status**: 40% Complete

**What EXISTS**:
- ✅ Orchestrator address (wallet)
- ✅ Orchestrator URL
- ✅ Stream ID
- ✅ Request ID
- ✅ Workflow/Pipeline name

**What's MISSING**:
- ❌ GPU ID (in operational events)
- ❌ Gateway ID (empty)
- ❌ Region
- ❌ Hardware specifications in performance events
- ❌ Model version/hash consistency

**Specific Gap Example**:
```json
// Current data can answer:
"What is orchestrator 0x52cf... average FPS?"

// Current data CANNOT answer:
"What is GPU GPU-a8d13655... average FPS?"
"What is gateway daydream-us-east swap rate?"
"What is performance in region us-east-1?"
```

---

#### ✅ YES: Historical Coverage
**Status**: 90% Complete

**What EXISTS**:
- ✅ Timestamps on all events
- ✅ Sequential event ordering
- ✅ Multi-hour coverage in sample data
- ✅ Event history preserved

**What's MISSING**:
- ⚠️ No explicit retention policy documented
- ⚠️ No compression/archival strategy
- ⚠️ No data lifecycle management

---

#### ⚠️ PARTIAL: Operational Signals
**Status**: 50% Complete

**What EXISTS**:
- ✅ Error events with messages
- ✅ State transitions (DEGRADED_INFERENCE, ONLINE)
- ✅ Restart counts
- ✅ Connection quality indicators

**What's MISSING**:
- ❌ Error categorization/codes
- ❌ Warning/degradation signals before failures
- ❌ Performance degradation indicators
- ❌ Capacity/saturation signals
- ❌ Predictive health indicators

---

### 1.3 NICE TO HAVE Requirements

#### ❌ NOT IMPLEMENTED: Health Thresholds
**Status**: 0% Complete

**What's MISSING**:
- ❌ No defined thresholds for any metric
- ❌ No alerting logic
- ❌ No SLA violation detection
- ❌ No automated health scoring

---

#### ❌ NOT IMPLEMENTED: Comparative Baselines
**Status**: 0% Complete

**What's MISSING**:
- ❌ No network-wide averages calculated
- ❌ No percentile calculations
- ❌ No benchmark reference values
- ❌ No peer comparison data

---

#### ⚠️ PARTIAL: Schema Extensibility
**Status**: 60% Complete

**What EXISTS**:
- ✅ JSON format allows adding fields
- ✅ Versioning possible via event type
- ✅ Flexible data structure

**What's MISSING**:
- ❌ No schema version field
- ❌ No backward compatibility strategy
- ❌ No deprecation process

---

#### ❌ NOT PROVIDED: Example Analyses
**Status**: 0% Complete

**What's MISSING**:
- ❌ No example queries
- ❌ No analysis notebooks
- ❌ No visualization examples
- ❌ No interpretation guides

---

## 2. NaaP MVP Metrics Gap Analysis

### 2.1 Core Performance Metrics

| Metric | Status | Data Source | Gap Description |
|--------|--------|-------------|-----------------|
| **Output FPS** | ✅ EXISTS | `ai_stream_status.inference_status.fps` | Complete |
| **Input FPS** | ✅ EXISTS | `ai_stream_status.input_status.fps` | Complete |
| **Prompt-to-First-Frame Latency** | ⚠️ DERIVABLE | Multiple `stream_trace` events | Requires correlation logic: `gateway_receive_first_processed_segment.timestamp - gateway_receive_stream_request.timestamp` |
| **E2E Stream Latency** | ❌ MISSING | N/A | No end-to-end timing captured from client perspective |
| **Jitter Coefficient** | ⚠️ DERIVABLE | `ai_stream_status.inference_status.fps` over time | Requires σ(fps)/μ(fps) calculation over window - data exists but calculation needed |
| **Startup (Cold) Time** | ❌ MISSING | N/A | No cold start indicator or timing |
| **Network Bandwidth** | ⚠️ PARTIAL | `stream_ingest_metrics.stats.peer_conn_stats` | BytesReceived/BytesSent exist but need rate calculation |

**Detailed Gaps**:

#### Prompt-to-First-Frame Latency
**Status**: Data exists but requires complex correlation

**Current Data**:
```json
// Event 1:
{"type": "stream_trace", "data": {
  "type": "gateway_receive_stream_request",
  "request_id": "ce6ee475",
  "timestamp": 1767966622124
}}

// Event 2:
{"type": "stream_trace", "data": {
  "type": "gateway_receive_first_processed_segment", 
  "request_id": "ce6ee475",
  "timestamp": 1767966625424
}}
```

**Calculation**: `1767966625424 - 1767966622124 = 3300ms`

**Gap**: This calculation requires:
1. Matching events by `request_id`
2. Finding the correct pair of trace events
3. Handling missing events
4. No direct latency field

---

#### E2E Stream Latency
**Status**: ❌ MISSING - Cannot be calculated from current data

**Problem**: Current timing only covers Gateway → Orchestrator → Gateway path. Missing:
- Client → Gateway initial latency
- Gateway → Client final delivery latency
- Network path timing outside gateway

**Required**: Add client-side timing beacons or gateway edge measurements

---

#### Jitter Coefficient (σ/μ of FPS)
**Status**: ⚠️ Data exists but needs windowed calculation

**Current Data**: Individual FPS values at 10-second intervals
```json
{"ai_stream_status": {"inference_status": {"fps": 19.987}}} // t=0
{"ai_stream_status": {"inference_status": {"fps": 20.085}}} // t=10
{"ai_stream_status": {"inference_status": {"fps": 19.889}}} // t=20
```

**Gap**: No pre-calculated jitter coefficient. Requires:
1. Collecting FPS samples over window (e.g., 1 minute)
2. Calculating mean: μ = Σfps / n
3. Calculating stddev: σ = sqrt(Σ(fps - μ)² / n)
4. Computing coefficient: σ/μ

**Note**: `stream_ingest_metrics` has a `jitter` field, but it's network jitter (packet arrival variance), not FPS jitter.

---

#### Startup (Cold) Time
**Status**: ❌ COMPLETELY MISSING

**Problem**: No indicator of whether a stream is "cold start" vs "warm"

**Current State**:
- `start_time` field exists showing when stream started
- But no way to distinguish cold boot from warm handoff
- No indication of model loading time
- No GPU initialization timing

**Required**:
```json
{
  "ai_stream_status": {
    "inference_status": {
      "is_cold_start": true,  // ❌ MISSING
      "cold_start_duration_ms": 2500,  // ❌ MISSING
      "model_load_time_ms": 1800,  // ❌ MISSING
      "gpu_init_time_ms": 700  // ❌ MISSING
    }
  }
}
```

---

### 2.2 Reliability Metrics

| Metric | Status | Data Source | Gap Description |
|--------|--------|-------------|-----------------|
| **Inference Failure Rate** | ⚠️ PARTIAL | `ai_stream_events` with `type: error` | Errors captured but no categorization or error codes |
| **Swap Rate** | ❌ MISSING | N/A | No orchestrator swap events captured |

**Detailed Gaps**:

#### Inference Failure Rate
**Status**: ⚠️ Basic errors captured, but needs enhancement

**Current Data**:
```json
{
  "type": "ai_stream_events",
  "data": {
    "type": "error",
    "message": "whip disconnected"
  }
}
```

**Gaps**:
1. ❌ No error code/category
2. ❌ No severity level
3. ❌ No recoverable vs fatal distinction
4. ❌ No error rate pre-calculated
5. ❌ Cannot distinguish transient vs permanent failures

**Required**:
```json
{
  "type": "ai_stream_events",
  "data": {
    "type": "error",
    "error_code": "CONNECTION_LOST",  // ❌ MISSING
    "error_category": "network",  // ❌ MISSING
    "severity": "warning",  // ❌ MISSING
    "is_recoverable": true,  // ❌ MISSING
    "message": "whip disconnected"
  }
}
```

---

#### Swap Rate
**Status**: ❌ COMPLETELY MISSING

**Problem**: No events indicate when gateway switches orchestrators

**Current State**: 
- Can see stream closed events
- Can see new stream start events
- Cannot identify if it's a swap vs natural end

**Required Events**:
```json
{
  "type": "gateway_orchestrator_swap",  // ❌ DOESN'T EXIST
  "data": {
    "stream_id": "...",
    "request_id": "...",
    "old_orchestrator": "0xABC...",
    "new_orchestrator": "0xDEF...",
    "reason": "performance_degradation",
    "timestamp": 1234567890
  }
}
```

---

### 2.3 Economics Metrics

| Metric | Status | Data Source | Gap Description |
|--------|--------|-------------|-----------------|
| **CUDA Utilization Efficiency** | ❌ MISSING | N/A | No FLOPS data, no GPU utilization data |
| **Pricing** | ✅ EXISTS | `create_new_payment` | Complete |
| **Staking** | ❌ MISSING | N/A | No LPT staking data in events |

**Detailed Gaps**:

#### CUDA Utilization Efficiency (CUE)
**Status**: ❌ COMPLETELY MISSING - Most Complex Metric

**Formula**: `CUE = (Achieved_FLOPS / Peak_FLOPS) × 100%`

**Current State**: ZERO GPU utilization metrics

**Required Data**:
```json
{
  "type": "gpu_utilization_metrics",  // ❌ DOESN'T EXIST
  "data": {
    "gpu_id": "GPU-a8d13655...",
    "timestamp": 1234567890,
    "utilization_pct": 82.5,  // Overall GPU utilization
    "memory_utilization_pct": 76.3,
    "achieved_tflops": 67.8,
    "peak_tflops": 82.5,
    "cue_pct": 82.18,
    "power_draw_watts": 320,
    "temperature_celsius": 72,
    "sm_clock_mhz": 2520,
    "memory_clock_mhz": 10001
  }
}
```

**Alternative Approach** (if direct FLOPS unavailable):
- Use GPU utilization % as proxy
- Correlate with performance (FPS) and workload
- Less accurate but provides signal

---

### 2.4 Network Demand Metrics

| Metric | Status | Data Source | Gap Description |
|--------|--------|-------------|-----------------|
| **Total Streams** | ✅ DERIVABLE | Count unique `stream_id` | Need aggregation |
| **Total Inference Minutes** | ⚠️ DERIVABLE | Stream duration from trace events | Requires calculation |
| **Inference Minutes by GPU Type** | ❌ PARTIAL | GPU type in `network_capabilities` only | GPU type not in per-stream events |
| **Capacity Rate** | ❌ MISSING | N/A | No total capacity data available |
| **Missing Capacity** | ⚠️ PARTIAL | Error events | No explicit capacity shortage indicator |
| **Staking LPT** | ❌ MISSING | N/A | Not in event stream |
| **Fee Payment** | ✅ EXISTS | `create_new_payment` | Complete |

---

## 3. Attribution Data Gaps (CRITICAL)

### 3.1 GPU Identification Problem

**CRITICAL ISSUE**: GPU_ID only appears in `network_capabilities` event (once per orchestrator advertisement), but NOT in operational events.

**Current State**:
```
network_capabilities (once) → Has GPU info
    ↓
    ↓  (NO LINK)
    ↓
ai_stream_status (every 10s) → Has performance data, NO GPU_ID
stream_ingest_metrics (every 5s) → Has network data, NO GPU_ID
stream_trace → Has timing data, NO GPU_ID
```

**Impact**:
- ❌ Cannot track individual GPU performance
- ❌ Cannot compare GPUs on same orchestrator
- ❌ Cannot detect GPU-specific issues
- ❌ Cannot provide per-GPU SLAs

**Required Fix**:
```json
// Add gpu_id to ALL operational events:
{
  "type": "ai_stream_status",
  "data": {
    "gpu_id": "GPU-a8d13655-197b-2226-d09f-b4b81e07504b",  // ⚠️ ADD THIS
    "orchestrator_info": {
      "address": "0x52cf...",
      "url": "https://..."
    }
  }
}
```

---

### 3.2 Gateway Identification Problem

**CRITICAL ISSUE**: Gateway field exists but is EMPTY in all events

**Current State**:
```json
{
  "type": "ai_stream_status",
  "gateway": "",  // ❌ ALWAYS EMPTY
  "timestamp": "..."
}
```

**Impact**:
- ❌ Cannot distinguish between gateways
- ❌ Cannot track gateway-specific demand
- ❌ Cannot analyze gateway performance
- ❌ Cannot provide gateway-level attribution

**Required Fix**:
Populate gateway field with unique gateway identifier:
```json
{
  "gateway": "gateway-daydream-us-east-1a"  // ✅ REQUIRED
}
```

---

### 3.3 Region Data Missing

**ISSUE**: No geographic region data anywhere

**Impact**:
- ❌ Cannot provide region-specific SLAs
- ❌ Cannot analyze regional performance differences
- ❌ Cannot optimize regional capacity planning
- ❌ Cannot meet "per region" attribution requirement

**Required**:
Add region to orchestrator_info and gateway identification:
```json
{
  "gateway": "gateway-daydream-us-east-1a",
  "gateway_region": "us-east-1",  // ❌ MISSING
  "data": {
    "orchestrator_info": {
      "address": "0x...",
      "url": "https://...",
      "region": "us-west-2"  // ❌ MISSING
    }
  }
}
```

---

### 3.4 Workload Identifier Inconsistency

**ISSUE**: Multiple fields used for workload identification

**Current State**:
- `pipeline`: "streamdiffusion-sdxl-faceid"
- `manifestID`: "6fa05caf"  
- `manifestID`: "35_streamdiffusion-sdxl-faceid"
- Model hash in different formats

**Problem**: Difficult to consistently group by workload

**Required**: Standardize on single workload identifier field

---

## 4. Data Quality Issues

### 4.1 Missing Fields in Events

**Common Pattern**: Optional fields not populated

Examples:
```json
{
  "pipeline_id": "",  // Usually empty
  "capability": "",   // Usually empty
  "clientIP": "",     // Privacy concern - should this be captured?
  "gateway": "",      // CRITICAL - always empty
  "requestID": ""     // Sometimes empty in payment events
}
```

---

### 4.2 Timestamp Consistency

**Current State**: ✅ Generally good
- Multiple timestamp fields (event timestamp, data timestamp)
- Millisecond precision
- Unix epoch format

**Minor Issue**: Some events have multiple timestamp fields that can be confusing:
```json
{
  "timestamp": "1767966802518",  // Event created time
  "data": {
    "timestamp": 1767966802512  // Data captured time
  }
}
```

---

### 4.3 ID Correlation Challenges

**Current State**: ⚠️ Works but requires effort

**Correlation Keys Available**:
- `request_id`: Links events for single request ✅
- `stream_id`: Links events for single stream ✅
- `sessionID`: Links payment events ✅

**Missing Correlations**:
- No GPU_ID in operational events ❌
- No gateway_id ❌
- No clear workload_id ❌

---

## 5. Prioritized Gap Remediation Plan

### 5.1 CRITICAL (MVP Blockers)

These MUST be fixed for MVP:

1. **Add GPU_ID to all operational events** (HIGHEST PRIORITY)
   - Severity: CRITICAL
   - Impact: Cannot provide per-GPU metrics
   - Effort: Medium (requires go-livepeer changes)
   - Timeline: Before MVP launch

2. **Populate gateway field** (HIGHEST PRIORITY)
   - Severity: CRITICAL
   - Impact: Cannot distinguish gateway sources
   - Effort: Low (configuration change)
   - Timeline: Before MVP launch

3. **Add region data** (HIGH PRIORITY)
   - Severity: HIGH
   - Impact: Cannot provide regional analysis
   - Effort: Medium (requires deployment topology awareness)
   - Timeline: MVP Phase 1

4. **Implement test load gateway** (HIGH PRIORITY)
   - Severity: HIGH
   - Impact: Required by Foundation
   - Effort: High (new component)
   - Timeline: MVP Phase 2

---

### 5.2 HIGH PRIORITY (Core Metrics)

These should be added for complete MVP:

5. **Capture orchestrator swap events** (HIGH)
   - Impact: Cannot calculate swap rate
   - Effort: Medium
   - Timeline: MVP Phase 2

6. **Add cold start indicators** (MEDIUM)
   - Impact: Cannot measure cold start time
   - Effort: Low
   - Timeline: MVP Phase 2

7. **Add error categorization** (MEDIUM)
   - Impact: Limited error analysis capability
   - Effort: Medium
   - Timeline: MVP Phase 2

8. **Calculate and expose jitter coefficient** (MEDIUM)
   - Impact: Derived metric not directly available
   - Effort: Low (calculation layer)
   - Timeline: MVP Phase 1 (in dbt)

---

### 5.3 MEDIUM PRIORITY (Enhanced Metrics)

These enhance but are not essential for initial MVP:

9. **GPU utilization metrics** (MEDIUM)
   - Impact: Cannot calculate CUE
   - Effort: HIGH (requires GPU monitoring)
   - Timeline: Post-MVP

10. **E2E latency from client perspective** (MEDIUM)
    - Impact: Limited end-to-end view
    - Effort: High (requires client instrumentation)
    - Timeline: Post-MVP

11. **Capacity/saturation indicators** (LOW)
    - Impact: Limited predictive capability
    - Effort: Medium
    - Timeline: Post-MVP

---

### 5.4 LOW PRIORITY (Nice to Have)

These can wait for future iterations:

12. **Health thresholds and alerting** (LOW)
    - Impact: Manual monitoring needed
    - Effort: Medium (requires threshold definition)
    - Timeline: Milestone 2

13. **Comparative baselines** (LOW)
    - Impact: No peer comparison
    - Effort: Low (calculation layer)
    - Timeline: Milestone 2

14. **Example analyses** (LOW)
    - Impact: Documentation gap
    - Effort: Low
    - Timeline: Ongoing

---

## 6. Summary of Data Availability

### What We HAVE (60% of MVP needs):

✅ **Performance Basics**:
- Output FPS
- Input FPS  
- Basic network jitter
- Packet loss
- Connection quality

✅ **Identification**:
- Orchestrator address
- Orchestrator URL
- Stream ID
- Request ID
- Timestamps

✅ **Operational Events**:
- Errors (basic)
- Restarts
- State transitions
- Payment/pricing

✅ **Timing Foundation**:
- Stream trace events
- Event sequences
- Correlation keys

---

### What We NEED (40% critical gaps):

❌ **Attribution Data**:
- GPU ID in operational events
- Gateway identification
- Region data
- Consistent workload IDs

❌ **Advanced Performance**:
- CUDA utilization
- GPU load metrics
- Cold start timing
- E2E latency

❌ **Reliability Tracking**:
- Orchestrator swaps
- Error categorization
- Capacity indicators

❌ **Test Infrastructure**:
- Test load gateway
- Test load identification
- Standardized test datasets

---

## 7. Recommendations

### Immediate Actions (Week 1-2):

1. **Fix Critical Attribution** (go-livepeer changes required):
   ```go
   // Add to ai_stream_status events:
   type StreamStatus struct {
       GPUID string `json:"gpu_id"`  // ADD THIS
       Region string `json:"region"`  // ADD THIS
       // ... existing fields
   }
   ```

2. **Populate Gateway Field** (configuration change):
   - Set gateway identifier in environment/config
   - Propagate to all events

3. **Document Current Schema** (documentation):
   - Create schema reference
   - Document all event types
   - Provide field descriptions

### Short-term (Week 3-6):

4. **Implement Derived Metrics** (dbt/analytics layer):
   - Jitter coefficient calculation
   - Prompt-to-first-frame latency
   - Inference minutes calculation
   - Failure rate aggregation

5. **Deploy Test Load Gateway** (new infrastructure):
   - Initial load test infrastructure
   - Basic test dataset (good prompts)
   - Test load identification in events

### Medium-term (Post-MVP):

6. **Add GPU Utilization** (requires GPU monitoring agent):
   - Deploy nvidia-smi or similar monitoring
   - Capture utilization metrics
   - Calculate CUE

7. **Enhance Error Tracking** (go-livepeer enhancements):
   - Add error codes
   - Add error categories
   - Add severity levels

---

## 8. Data Gap Impact Matrix

| Gap | Foundation Impact | MVP Impact | Remediation Effort | Priority |
|-----|------------------|------------|-------------------|----------|
| No GPU_ID in events | HIGH - Cannot attribute to GPUs | CRITICAL - Blocks per-GPU metrics | MEDIUM | P0 |
| Empty gateway field | HIGH - Cannot distinguish gateways | CRITICAL - Blocks gateway attribution | LOW | P0 |
| No region data | MEDIUM - Limits geographical analysis | HIGH - Cannot provide regional SLAs | MEDIUM | P0 |
| No swap events | LOW - Doesn't block initial launch | HIGH - Missing key reliability metric | MEDIUM | P1 |
| No CUE/GPU utilization | LOW - Advanced metric | MEDIUM - Economic metric missing | HIGH | P2 |
| No cold start timing | LOW - Specific use case | MEDIUM - Performance insight missing | LOW | P1 |
| No test load gateway | MEDIUM - Foundation requirement | HIGH - Validation infrastructure | HIGH | P1 |
| Basic error events | LOW - Functional but limited | LOW - Can work with basic errors | MEDIUM | P2 |
| E2E latency missing | MEDIUM - Important but not critical | MEDIUM - Can use approximations | HIGH | P3 |

---

## Conclusion

The current streaming events data provides a **solid foundation** (~60% complete) but has **critical gaps** in attribution data (GPU_ID, gateway, region) that must be fixed for the MVP to meet Foundation requirements.

**Key Takeaway**: Focus remediation on:
1. **Attribution fixes** (GPU_ID, gateway, region) - Essential for "Must Have" requirements
2. **Test load infrastructure** - Required by Foundation
3. **Derived metrics in analytics layer** - Can extract more value from existing data

With these fixes, the NaaP MVP can deliver on its core promise of making the network "measurable and monitorable in real-time."