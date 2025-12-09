# Gateway-Orchestrator Workflow Explainer

## 1. TL;DR Summary

The Livepeer gateway-orchestrator system enables distributed video AI processing where gateways (broadcasters) discover, select, and route live video streams to orchestrators for AI processing. Orchestrators register on-chain or via webhook, advertise their capabilities (models, capacity, pricing), and process video streams sent by gateways. Gateways maintain pools of orchestrator sessions, select orchestrators based on capabilities, latency, and availability, and can swap orchestrators during live streams if needed. Multiple gateways can independently discover and use the same pool of orchestrators, with each gateway maintaining its own session management and selection logic.

---

## 2. Workflow Diagram

### A. Orchestrator Registration & Discovery Flow

```
Orchestrator Startup
   |
   v
[Register on-chain] --> [Set ServiceURI] --> [Activate via HTTP API]
   |                                              |
   v                                              v
[Ethereum Events] --> [OrchestratorWatcher] --> [DB Storage]
   |                                              |
   v                                              v
[Gateway Discovery] <-- [Poll OrchestratorInfo] <-- [GetOrchestrator RPC]
```

### B. Gateway Orchestrator Selection Flow (Live Video-to-Video AI)

```
Gateway Receives Stream Request
   |
   v
[AISessionManager] --> [AISessionSelector.Select()]
   |                        |
   |                        v
   |              [Check Warm/Cold Pools]
   |                        |
   |                        v
   |              [Refresh if Empty/Expired]
   |                        |
   |                        v
   |              [OrchestratorPool.GetOrchestrators()]
   |                        |
   |                        v
   |              [Discovery: GetOrchInfo RPC]
   |                        |
   |                        v
   |              [Filter by Capabilities/Model]
   |                        |
   |                        v
   |              [Sort by Latency]
   |                        |
   |                        v
   +---> [Select Warm Pool First] --> [Select Cold Pool]
                |                              |
                v                              v
         [BroadcastSession]            [BroadcastSession]
                |                              |
                +----------> [AISession] <------+
                            |
                            v
                    [Trickle Publish Stream]
                            |
                            v
                    [Orchestrator Processes]
                            |
                            v
                    [Results Returned]
```

### C. Stream Push & Result Return Flow

```
Gateway Segment Ready
   |
   v
[Trickle Publisher] --> [POST /segment] --> [Orchestrator]
   |                                              |
   |                                              v
   |                                    [Process with AI Worker]
   |                                              |
   |                                              v
   |                                    [Return Results via HTTP]
   |                                              |
   v                                              v
[Receive Results] <-- [Streaming Response] <-----+
   |
   v
[Forward to Client]
```

### D. Multiple Gateways Sharing Orchestrators

```
Orchestrator Pool (On-chain/Webhook/Hardcoded)
   |
   +---> Gateway A --> [Independent Discovery] --> [Own Session Pools]
   |                    [Own Selection Logic]
   |
   +---> Gateway B --> [Independent Discovery] --> [Own Session Pools]
   |                    [Own Selection Logic]
   |
   +---> Gateway C --> [Independent Discovery] --> [Own Session Pools]
                        [Own Selection Logic]
```

---

## 3. Step-by-Step Explanation

### 3.1 Orchestrator Registration & Activation

1. **Orchestrator Startup**: When an orchestrator starts, it needs to register itself so gateways can discover it.

2. **On-Chain Registration**: 
   - Orchestrator calls `activateOrchestratorHandler` HTTP endpoint (`/activateOrchestrator`)
   - Handler checks if orchestrator is already registered
   - If not, it submits an Ethereum transaction to register the orchestrator on-chain
   - Transaction includes: `blockRewardCut`, `feeShare`, `pricePerUnit`, `pixelsPerUnit`, `serviceURI`

3. **Event Monitoring**:
   - `OrchestratorWatcher` monitors Ethereum blockchain for `TranscoderRegistered` events
   - When detected, it stores orchestrator ETH address in the database
   - `ServiceRegistryWatcher` monitors for service URI updates

4. **Service URI Storage**: The orchestrator's service URI (HTTP endpoint) is stored in the database, allowing gateways to connect to it.

5. **Activation**: Once registered, the orchestrator is "active" and can be discovered by gateways.

### 3.2 Gateway Discovery of Orchestrators

1. **Discovery Data Sources**:
   - **On-chain (default)**: Gateway queries database populated by `OrchestratorWatcher` and `ServiceRegistryWatcher`
   - **Webhook**: Gateway queries webhook URL (e.g., Livepeer Studio API) that returns cached orchestrator list
   - **Hardcoded**: Gateway uses `-orchAddr` flag with explicit orchestrator URLs

2. **Orchestrator Pool Initialization**:
   - Gateway creates `OrchestratorPool` from discovery source
   - For on-chain: `DBOrchestratorPoolCache` queries database for active orchestrators
   - Pool maintains list of orchestrator URLs and metadata

3. **Polling Orchestrator Info**:
   - Gateway periodically calls `GetOrchestrator` RPC on each orchestrator URL
   - Orchestrator responds with `OrchestratorInfo` containing:
     - Capabilities (supported models, capacity)
     - Pricing information
     - Ticket parameters (for payment)
     - Auth tokens
     - Additional advertised nodes (`-nodes` flag)

4. **Caching**: Gateway caches orchestrator info and refreshes periodically (default ~10 seconds for capacity reporting).

### 3.3 Gateway Selection for Live Video-to-Video AI

1. **Request Arrival**: Gateway receives live video stream request with pipeline/model ID.

2. **Session Manager Lookup**:
   - `AISessionManager` maintains selectors per `capability+modelID` combination
   - Each selector manages warm and cold pools of `BroadcastSession` objects

3. **Selector Refresh Logic**:
   - If pools are empty → trigger refresh
   - If pool size < threshold → refresh
   - If TTL expired → refresh

4. **Discovery & Filtering**:
   - `AISessionSelector.getSessions()` calls `OrchestratorPool.GetOrchestrators()`
   - Discovery queries all orchestrators in parallel via `GetOrchestrator` RPC
   - Filters orchestrators by:
     - Capability support (must support `LiveVideoToVideo`)
     - Model support (must support requested `modelID`)
     - Blacklist (excluded orchestrators)
     - Compatibility (version, constraints)

5. **Latency Sorting**: Available orchestrators are sorted by measured latency (ascending).

6. **Warm/Cold Pool Assignment**:
   - Orchestrators with warm models → warm pool
   - Orchestrators with cold models → cold pool
   - Warm pool is preferred for faster startup

7. **Session Selection**:
   - First tries warm pool
   - Falls back to cold pool if warm is empty
   - Returns `AISession` wrapping `BroadcastSession`

8. **Retry Logic**:
   - If selection fails, retries up to `maxTries` (default 3)
   - On error, removes failed session and tries another
   - For live video, removes session immediately on error (no retry on same orchestrator)

### 3.4 Stream Push to Orchestrator

1. **Trickle Publish Setup**:
   - Gateway creates `TricklePublisher` connected to orchestrator's trickle endpoint
   - Establishes persistent connection for streaming segments

2. **Segment Processing**:
   - Gateway segments incoming video stream
   - For each segment:
     - Calls `publisher.Next()` to get segment handle
     - Writes segment data via `segment.Write()`
     - Monitors for errors (stream not found, slow orchestrator)

3. **Payment Processing**:
   - `LivePaymentProcessor` periodically sends payment tickets
   - Calculates pixels processed and sends payment based on orchestrator's pricing

4. **Error Handling**:
   - If orchestrator is slow → suspend and swap to different orchestrator
   - If stream not found → terminate connection
   - If segment write fails → retry or drop segment

### 3.5 Result Return from Orchestrator

1. **Processing**: Orchestrator receives segments and processes with AI worker.

2. **Result Streaming**:
   - For streaming responses: Orchestrator sends Server-Sent Events (SSE) format
   - Gateway reads from `resp.Body` and forwards to client
   - Format: `data: {json}\n\n` with `[DONE]` marker

3. **Non-Streaming Responses**:
   - Gateway reads entire response body
   - Forwards to client immediately
   - Updates gateway balance based on processing time

4. **Result Forwarding**:
   - Gateway acts as proxy, forwarding orchestrator response to original client
   - Maintains streaming connection if SSE format detected

### 3.6 Testing Traffic Workflow

1. **Job Request Endpoint**: Gateway exposes `/process` endpoint for testing.

2. **Orchestrator Discovery**:
   - `getJobOrchestrators()` gets all orchestrators from pool
   - Can filter by `Include`/`Exclude` lists in request

3. **Token Request**:
   - Gateway sends `GET /process/token` to each orchestrator in parallel
   - Includes gateway address and signature for authentication
   - Orchestrator returns `JobToken` with ticket params and pricing

4. **Job Submission**:
   - Gateway signs request with private key
   - Sends `POST /process` to orchestrator with:
     - Job request header (base64 encoded)
     - Request body (video data or URL)
     - Capability header

5. **Response Collection**:
   - Gateway collects responses from all orchestrators
   - Can send to multiple orchestrators simultaneously (for testing)
   - Each orchestrator processes independently and returns results

6. **Metrics Collection**:
   - Gateway tracks:
     - Response times per orchestrator
     - Success/failure rates
     - Processing latency
     - Capacity information (from `GetOrchestrator` responses)

---

## 4. Component Responsibilities

### Gateway Components

- **`AISessionManager`** (`server/ai_session.go`): Manages AI session selectors per capability+modelID, coordinates session lifecycle
- **`AISessionSelector`** (`server/ai_session.go`): Selects orchestrators from warm/cold pools, refreshes pools when needed
- **`AISessionPool`** (`server/ai_session.go`): Maintains pool of `BroadcastSession` objects, tracks in-use sessions
- **`OrchestratorPool`** (`discovery/discovery.go`): Interface for orchestrator discovery, provides `GetOrchestrators()` method
- **`DBOrchestratorPoolCache`** (`discovery/db_discovery.go`): On-chain discovery implementation, polls orchestrators and caches info
- **`TricklePublisher`** (`trickle/`): Streams video segments to orchestrator via trickle protocol
- **`LivePaymentProcessor`** (`server/ai_live_video.go`): Sends periodic payment tickets to orchestrator

### Orchestrator Components

- **`orchestrator`** (`core/orchestrator.go`): Main orchestrator logic, handles `GetOrchestrator` RPC, processes payments
- **`RemoteAIWorkerManager`** (`core/ai_orchestrator.go`): Manages pool of AI workers, routes jobs to workers
- **`activateOrchestratorHandler`** (`server/handlers.go`): HTTP endpoint for orchestrator registration/activation

### Discovery Components

- **`OrchestratorWatcher`** (`eth/watchers/orchestratorwatcher.go`): Monitors Ethereum for orchestrator registration events
- **`ServiceRegistryWatcher`** (`eth/watchers/`): Monitors for service URI updates
- **`WebhookPool`** (`discovery/wh_discovery.go`): Webhook-based discovery implementation

---

## 5. Data Flow Breakdown

### Orchestrator Registration Flow

```
Orchestrator HTTP Request
   |
   v
activateOrchestratorHandler
   |
   v
Ethereum Transaction (BondingManager.transcoder())
   |
   v
Blockchain Event (TranscoderRegistered)
   |
   v
OrchestratorWatcher.handleLog()
   |
   v
Database.InsertOrch()
   |
   v
Gateway DB Query (SelectOrchs)
   |
   v
OrchestratorPool.GetInfos()
```

### Discovery & Selection Flow

```
Gateway Request (capability + modelID)
   |
   v
AISessionSelector.Select()
   |
   v
OrchestratorPool.GetOrchestrators()
   |
   v
Parallel GetOrchestrator RPC Calls
   |
   v
OrchestratorInfo Response
   |
   v
Filter by Capabilities/Model
   |
   v
Sort by Latency
   |
   v
Assign to Warm/Cold Pools
   |
   v
Select from Pool
   |
   v
BroadcastSession Created
```

### Stream Processing Flow

```
Video Segment
   |
   v
TricklePublisher.Next()
   |
   v
HTTP POST /segment (orchestrator)
   |
   v
Orchestrator Receives Segment
   |
   v
AI Worker Processing
   |
   v
Results Generated
   |
   v
HTTP Response (SSE or JSON)
   |
   v
Gateway Forwards to Client
```

### Payment Flow

```
Segment Processed
   |
   v
LivePaymentProcessor.process()
   |
   v
Calculate Pixels Processed
   |
   v
Create Payment Ticket
   |
   v
Send via BroadcastSession
   |
   v
Orchestrator Receives Ticket
   |
   v
Orchestrator.node.Recipient.ReceiveTicket()
   |
   v
Credit Added to Balance
```

---

## 6. Edge Cases & Failure Paths

### Discovery Failures

- **No Orchestrators Found**: Gateway logs error, returns `ServiceUnavailableError` to client
- **Timeout**: Discovery uses exponential backoff timeout (starts at `discoveryTimeout`, doubles up to `maxGetOrchestratorCutoffTimeout`)
- **Partial Responses**: Gateway continues with available orchestrators, logs errors for failed ones

### Selection Failures

- **Empty Pools**: Selector triggers refresh, releases suspended orchestrators
- **All Orchestrators Suspended**: Selector considers suspended orchestrators if insufficient non-suspended ones available
- **Capability Mismatch**: Orchestrator filtered out, not added to pools

### Stream Processing Failures

- **Slow Orchestrator**: `SlowOrchChecker` detects if orchestrator falls behind, triggers swap
- **Stream Not Found**: `trickle.StreamNotFoundErr` detected, connection terminated
- **Segment Write Error**: Retries if no data written, drops segment if partial write occurred
- **Orchestrator Swap**: Gateway can swap to different orchestrator mid-stream (limited to prevent thrashing)

### Payment Failures

- **Missing Price Info**: Gateway logs warning, continues without payment
- **Ticket Errors**: Non-fatal errors logged, fatal errors terminate session

### Multiple Gateway Scenarios

- **Independent Discovery**: Each gateway maintains its own orchestrator pool and discovery state
- **Session Isolation**: Each gateway's sessions are independent, no shared state
- **Load Distribution**: Multiple gateways naturally distribute load across orchestrator pool
- **Orchestrator Capacity**: Each orchestrator advertises capacity, gateways respect limits independently

---

## 7. Suggestions for Clarity or Simplification

### Current Architecture Strengths

1. **Separation of Concerns**: Discovery, selection, and processing are well-separated
2. **Retry Logic**: Robust retry mechanisms with exponential backoff
3. **Warm/Cold Pools**: Efficient model startup time optimization
4. **Independent Gateways**: No shared state, scales horizontally

### Potential Improvements

1. **Discovery Refresh Logic**: The refresh conditions in `AISessionSelector.Select()` could be more explicit about when refresh happens vs. when it's skipped
2. **Suspension Mechanism**: The suspension queue logic could benefit from clearer documentation on suspension duration and release conditions
3. **Error Classification**: More granular error types could help distinguish between retryable and non-retryable errors
4. **Metrics Collection**: The testing/metrics workflow could be more explicit about how metrics are aggregated and reported
5. **Orchestrator Capacity**: The capacity reporting mechanism (`refreshOrchCapacity`) runs independently and could be better integrated with selection logic

### Documentation Gaps

1. **Trickle Protocol**: The trickle publish mechanism could use more documentation on protocol details
2. **Payment Flow**: The payment ticket creation and validation flow could be clearer
3. **Multi-Gateway Coordination**: While gateways are independent, there's no explicit coordination mechanism for load balancing (relies on orchestrator capacity limits)

---

## 8. Missing Context Policy

The following areas were inferred from code structure but may need verification:

1. **Trickle Protocol Details**: The `trickle` package implementation details were inferred from usage patterns
2. **Payment Ticket Format**: Exact ticket structure inferred from `pm` package usage
3. **AI Worker Communication**: The communication between orchestrator and AI workers was inferred from `RemoteAIWorker` usage
4. **Webhook Discovery Format**: Exact webhook response format inferred from `WebhookPool` implementation

These inferences are based on consistent patterns in the codebase and should be accurate, but specific protocol details may vary.

---

## Additional Notes

### Multiple Gateways Sharing Orchestrators

**Yes, multiple gateways can share the same pool of orchestrators.** Each gateway:

1. **Independent Discovery**: Queries the same discovery source (on-chain, webhook, or hardcoded list) but maintains its own `OrchestratorPool` instance
2. **Independent Selection**: Each gateway runs its own `AISessionManager` and `AISessionSelector` with independent warm/cold pools
3. **No Coordination**: Gateways don't coordinate with each other; they independently select orchestrators based on their own criteria
4. **Orchestrator Capacity**: Orchestrators advertise their capacity (idle containers per model), and each gateway respects these limits independently
5. **Load Distribution**: Natural load distribution occurs as each gateway selects orchestrators based on latency and availability

This architecture allows horizontal scaling of gateways while sharing the same orchestrator infrastructure.

---

## 9. Orchestrator Selection Algorithm Deep Dive

### 9.1 Selection Algorithm Overview

The gateway uses a **two-tier selection system**:

1. **Discovery Layer**: Finds eligible orchestrators based on capabilities
2. **Selection Layer**: Chooses the best orchestrator from eligible candidates

### 9.2 Selection Algorithm Types

#### A. MinLSSelectorWithRandFreq (Default for Non-Live AI)

**Algorithm Flow**:
```
Available Sessions
   |
   v
[Separate into Known/Unknown]
   |
   +---> Known Sessions (have latency score)
   |        |
   |        v
   |    [Check Best Latency Score]
   |        |
   |        +---> If score > threshold --> Select Best Known
   |        |
   |        +---> If score <= threshold --> Select from Unknown
   |
   +---> Unknown Sessions (no latency score yet)
            |
            v
        [Selection Strategy]
            |
            +---> X% Random Selection
            |
            +---> (100-X)% Stake-Weighted Random
```

**Latency Score Calculation**:
```
LatencyScore = SegmentDuration / RoundTripTime
```
- **RoundTripTime**: Time from segment upload start to transcoded segment download complete
- **Good Score**: > 1.0 (processing faster than real-time)
- **Threshold**: Configurable (default based on `SELECTOR_LATENCY_SCORE_THRESHOLD`)

**Selection Logic**:
1. **Known Sessions**: Select session with **lowest latency score** (best performance)
2. **Unknown Sessions**: 
   - Random selection (X% of time) to explore new orchestrators
   - Stake-weighted random (remaining time) to favor high-stake orchestrators

#### B. LiveSelectionAlgorithm (For Live Video-to-Video AI)

**Algorithm Flow**:
```
Available Sessions
   |
   v
[Filter by Max Price]
   |
   v
[Select First Available]
```

**Characteristics**:
- **Simpler**: No latency score tracking
- **Faster**: Immediate selection from filtered list
- **Price-First**: Filters by maximum acceptable price
- **First-Match**: Returns first orchestrator that meets criteria

### 9.3 Probability Selection Algorithm (Alternative)

For non-live workloads, can use `ProbabilitySelectionAlgorithm`:

**Weighted Selection**:
```
Probability = PriceWeight × PriceProb + StakeWeight × StakeProb + RandWeight × RandProb
```

Where:
- **PriceProb**: Exponential decay based on price (`exp(-price / PriceExpFactor)`)
- **StakeProb**: Proportional to orchestrator stake
- **RandProb**: Uniform random (1/N)

**Filtering**:
1. Filter by **MinPerfScore** (if configured)
2. Filter by **MaxPrice** (if configured)
3. Calculate probabilities
4. Select using weighted random

### 9.4 Selection Process in Code

**Step-by-Step**:

1. **Session Pool Selection** (`AISessionSelector.Select()`):
   ```go
   // Check if refresh needed
   if shouldRefreshSelector() {
       sel.Refresh(ctx)  // Discover new orchestrators
   }
   
   // Try warm pool first
   sess := sel.warmPool.Select(ctx)
   if sess != nil {
       return &AISession{..., Warm: true}
   }
   
   // Fallback to cold pool
   sess = sel.coldPool.Select(ctx)
   if sess != nil {
       return &AISession{..., Warm: false}
   }
   ```

2. **Pool Selection** (`AISessionPool.Select()`):
   ```go
   // Use selector to pick session
   sess := pool.selector.Select(ctx)
   
   // Track in-use
   pool.inUseSess = append(pool.inUseSess, sess)
   
   // Mark segment in-flight
   sess.pushSegInFlight(&stream.HLSSegment{})
   ```

3. **Selector Selection** (`Selector.Select()` or `MinLSSelector.Select()`):
   ```go
   // For MinLSSelector:
   // - Check known sessions with good latency score
   // - If none, select from unknown sessions
   // - Use stake-weighted or random selection
   
   // For LiveSelectionAlgorithm:
   // - Filter by max price
   // - Return first match
   ```

4. **Discovery Refresh** (`AISessionSelector.Refresh()`):
   ```go
   // Get all orchestrators from pool
   sessions, err := sel.getSessions(ctx)  // Calls selectOrchestrator()
   
   // Filter by model constraints
   for _, sess := range sessions {
       if modelConstraint.Warm {
           warmSessions = append(warmSessions, sess)
       } else {
           coldSessions = append(coldSessions, sess)
       }
   }
   
   // Add to pools
   sel.warmPool.Add(warmSessions)
   sel.coldPool.Add(coldSessions)
   ```

### 9.5 Selection Criteria Summary

| Criterion | Live Video AI | Non-Live AI |
|-----------|---------------|-------------|
| **Primary** | Price (max filter) | Latency Score |
| **Secondary** | First available | Stake weight |
| **Tertiary** | N/A | Random exploration |
| **Warm/Cold** | Yes (model warmup) | Yes (model warmup) |
| **Suspension** | No penalty | 3-request penalty |
| **Refresh** | Every 6 min | On-demand + TTL |

---

## 10. Current Metrics & Kafka Integration

### 10.1 Monitor Package Architecture Overview

**Two Independent Systems**:

The `monitor` package provides **two completely independent systems** for metrics collection:

1. **OpenCensus System** (`monitor/census.go`):
   - Records metrics to OpenCensus
   - Exports to Prometheus automatically
   - Functions: `AIFirstSegmentDelay()`, `AILiveVideoAttempt()`, etc.
   - Check: `monitor.Enabled` flag

2. **Kafka System** (`monitor/kafka.go`):
   - Sends events to Kafka
   - Function: `SendQueueEventAsync()`
   - Check: `kafkaProducer != nil`

**Important**: These systems are **NOT automatically connected**. If you want metrics to go to both OpenCensus and Kafka, you must call both functions explicitly in your code.

**Example Pattern**:
```go
// Dual reporting - call both explicitly
if monitor.Enabled {
    monitor.AIFirstSegmentDelay(delayMs, orchInfo)  // OpenCensus
    monitor.SendQueueEventAsync("stream_trace", data)  // Kafka
}
```

### 10.2 Kafka Integration Architecture

**Current Implementation**:

```
Gateway Event
   |
   v
monitor.SendQueueEventAsync(eventType, data)
   |
   v
KafkaProducer.events (buffered channel)
   |
   v
processEvents() goroutine
   |
   v
Batch Collection (100 events or 1 second)
   |
   v
Kafka Writer (with retry logic)
   |
   v
Kafka Topic
```

**Configuration**:
- **Batch Size**: 100 events
- **Batch Interval**: 1 second
- **Channel Size**: 100 events (buffer)
- **Retry Logic**: 3 attempts on failure
- **Topic**: Configurable via `-kafkaGatewayTopic`

### 10.3 Current Metrics Sent to Kafka

**Event Types Currently Implemented**:

1. **`stream_trace`**: Stream lifecycle events
   - `gateway_send_first_ingest_segment`
   - `gateway_ingest_stream_closed`
   - `gateway_no_orchestrators_available`
   - Orchestrator swap events
   - Stream start/end events

2. **`ai_stream_events`**: AI-specific stream events
   - Pipeline status updates
   - Parameter changes
   - Error events

3. **`stream_ingest_metrics`**: Ingest performance metrics
   - Input/output FPS
   - Segment timing
   - Processing latency

4. **`create_new_payment`**: Payment creation events

5. **`discovery_results`**: Orchestrator discovery results
   - Orchestrator addresses
   - Latency measurements
   - Discovery timestamps

**Event Structure**:
```json
{
  "id": "uuid",
  "type": "event_type",
  "timestamp": "unix_ms",
  "gateway": "gateway_eth_address",
  "data": {
    // Event-specific data
  }
}
```

### 10.4 AI Workflow Monitoring & Metrics Collection

#### 10.4.1 Live Video-to-Video Workflow Monitoring

**Complete Monitoring Flow**:

```
Stream Request Received
   |
   v
[Kafka: gateway_receive_stream_request]
   |
   v
[OpenCensus: AILiveVideoAttempt] --> [Kafka: stream_trace]
   |
   v
Orchestrator Selection
   |
   v
[Kafka: gateway_send_first_ingest_segment]
   |
   v
Segment Processing Loop
   |
   +---> [OpenCensus: AIFirstSegmentDelay] --> [Kafka: gateway_receive_first_processed_segment]
   |
   +---> [Kafka: gateway_receive_few_processed_segments] (after 3 segments)
   |
   +---> [Kafka: stream_ingest_metrics] (every 5 seconds)
   |
   +---> [Kafka: orchestrator_swap] (if swap occurs)
   |
   v
Stream End
   |
   v
[Kafka: gateway_ingest_stream_closed]
```

**Step-by-Step Metrics Collection**:

1. **Stream Request Arrival** (`ai_mediaserver.go`):
   ```go
   // Kafka Event: gateway_receive_stream_request
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "gateway_receive_stream_request",
       "stream_id": streamID,
       "pipeline_id": pipelineID,
       "request_id": requestID,
       "timestamp": streamRequestTime
   })
   
   // OpenCensus Metric: ai_live_attempt
   monitor.AILiveVideoAttempt(pipeline) // Counts attempt
   ```

2. **Orchestrator Selection** (`ai_process.go`):
   ```go
   // If no orchestrators available:
   monitor.AIRequestError(errMsg, pipeline, modelID, nil)
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "gateway_no_orchestrators_available",
       "stream_id": streamID,
       ...
   })
   ```

3. **First Segment Sent** (`ai_live_video.go`):
   ```go
   // Kafka Event: gateway_send_first_ingest_segment
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "gateway_send_first_ingest_segment",
       "orchestrator_info": {
           "address": sess.Address(),
           "url": sess.Transcoder()
       },
       ...
   })
   ```

4. **First Processed Segment Received** (`ai_live_video.go`):
   ```go
   // OpenCensus Metric: ai_first_segment_delay_ms
   delayMs := time.Since(startTime).Milliseconds()
   monitor.AIFirstSegmentDelay(delayMs, orchestratorInfo)
   
   // Kafka Event: gateway_receive_first_processed_segment
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "gateway_receive_first_processed_segment",
       ...
   })
   ```

5. **Multiple Segments Received** (`ai_live_video.go`):
   ```go
   // After 3 segments received:
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "gateway_receive_few_processed_segments",
       ...
   })
   ```

6. **Periodic Ingest Metrics** (`ai_mediaserver.go`):
   ```go
   // Every 5 seconds, collect WHIP transport stats
   monitor.AIWhipTransportBytesReceived(bytesReceived)
   monitor.AIWhipTransportBytesSent(bytesSent)
   
   // Kafka Event: stream_ingest_metrics
   monitor.SendQueueEventAsync("stream_ingest_metrics", {
       "timestamp": time.Now().UnixMilli(),
       "stream_id": streamID,
       "stats": {
           "bytes_received": ...,
           "bytes_sent": ...,
           "packets_lost": ...,
           "jitter": ...,
           "rtt": ...
       }
   })
   ```

7. **Orchestrator Swap** (`ai_mediaserver.go`):
   ```go
   // Kafka Event: orchestrator_swap
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "orchestrator_swap",
       "message": err.Error(),
       "orchestrator_info": {...}
   })
   ```

8. **Stream End** (`ai_mediaserver.go`):
   ```go
   // Kafka Event: gateway_ingest_stream_closed
   monitor.SendQueueEventAsync("stream_trace", {
       "type": "gateway_ingest_stream_closed",
       ...
   })
   ```

9. **Live Sessions Tracking** (`ai_mediaserver.go`):
   ```go
   // OpenCensus Metric: ai_current_live_pipelines
   // Updated periodically with count of active sessions per pipeline
   monitor.AICurrentLiveSessions(sessionsByPipeline)
   ```

#### 10.4.2 AI Metrics Collected (OpenCensus/Prometheus)

**AI-Specific Metrics**:

1. **Request Metrics**:
   - `ai_models_requested`: Total AI model requests (tagged by pipeline, model)
   - `ai_request_latency_score`: Latency score per request (ratio of processing time to content duration)
   - `ai_request_price`: Price per unit for AI requests
   - `ai_request_errors`: Error count (tagged by error code, pipeline, model, sender)

2. **Result Metrics**:
   - `ai_result_downloaded_total`: Number of results downloaded
   - `ai_result_download_time_seconds`: Download time from orchestrator
   - `ai_result_uploaded_total`: Number of results uploaded
   - `ai_result_upload_time_seconds`: Upload time to orchestrator
   - `ai_result_upload_failed_total`: Upload failure count

3. **Live Video-to-Video Metrics**:
   - `ai_live_attempt`: Total live video attempts (tagged by model)
   - `ai_first_segment_delay_ms`: Delay from stream start to first processed segment
   - `ai_current_live_pipelines`: Current number of active live pipelines (tagged by pipeline)
   - `ai_orchestrators_available_total`: Number of available orchestrators for AI
   - `live_ai_price_per_pixel`: Price per pixel for live AI processing (tagged by orchestrator)

4. **Capacity Metrics**:
   - `ai_container_in_use`: Containers currently in use
   - `ai_container_idle`: Containers available for processing
   - `ai_gpus_idle`: Idle GPUs (tagged by pipeline, model)

5. **Transport Metrics**:
   - `ai_whip_transport_bytes_received`: Bytes received on WHIP connection
   - `ai_whip_transport_bytes_sent`: Bytes sent on WHIP connection

#### 10.4.3 Kafka Events for AI Workflows

**Event Types Sent to Kafka**:

1. **`stream_trace`** Events:
   - `gateway_receive_stream_request`: Stream request received
   - `gateway_send_first_ingest_segment`: First segment sent to orchestrator
   - `gateway_receive_first_processed_segment`: First processed segment received
   - `gateway_receive_few_processed_segments`: After 3 segments received
   - `gateway_no_orchestrators_available`: No orchestrators found
   - `gateway_ingest_stream_closed`: Stream ended/closed
   - `orchestrator_swap`: Orchestrator swapped mid-stream

2. **`stream_ingest_metrics`** Events:
   - Periodic (every 5 seconds) transport statistics
   - Includes: bytes received/sent, packets lost, jitter, RTT
   - Tagged with: stream_id, pipeline_id, request_id

3. **`ai_stream_events`** Events:
   - Pipeline status updates
   - Parameter changes
   - Error events

**Event Data Structure**:
```json
{
  "id": "uuid",
  "type": "stream_trace",
  "timestamp": "unix_ms",
  "gateway": "gateway_eth_address",
  "data": {
    "type": "gateway_receive_first_processed_segment",
    "stream_id": "stream_uuid",
    "pipeline_id": "pipeline_uuid",
    "request_id": "request_uuid",
    "orchestrator_info": {
      "address": "0x...",
      "url": "https://orch.example.com"
    }
  }
}
```

#### 10.4.4 Metrics Collection Architecture

**Dual Collection System - Independent Routing**:

The monitor package provides **two independent systems** that are NOT automatically connected. Code must explicitly call both if metrics should go to both systems:

```
AI Workflow Event
   |
   +---> [OpenCensus Functions] --> [OpenCensus Stats] --> [Prometheus Export]
   |        |                              |
   |        |                              +---> go.opencensus.io/stats
   |        |                              +---> Tagged with: pipeline, model, orchestrator
   |        |                              +---> Aggregated views (count, distribution, last value)
   |        |                              +---> Prometheus exporter
   |        |
   |        +---> Functions: AIFirstSegmentDelay(), AILiveVideoAttempt(), etc.
   |        +---> Check: monitor.Enabled flag
   |
   +---> [Kafka Functions] --> [Kafka Producer] --> [Kafka Topic]
            |                         |
            |                         +---> Batched (100 events or 1 second)
            |                         +---> Retry logic (3 attempts)
            |
            +---> Function: SendQueueEventAsync()
            +---> Check: kafkaProducer != nil
```

**Key Point**: OpenCensus and Kafka are **separate, independent systems**. They do NOT automatically send to both. Code must explicitly call both functions if dual reporting is needed.

**How It Works**:

1. **OpenCensus Metrics**:
   - Functions like `AIFirstSegmentDelay()`, `AILiveVideoAttempt()` record to OpenCensus
   - Check `monitor.Enabled` flag before recording
   - Use `stats.RecordWithTags()` to record metrics
   - Automatically exported to Prometheus via registered exporter

2. **Kafka Events**:
   - Function `SendQueueEventAsync()` sends events to Kafka
   - Checks if `kafkaProducer != nil` (initialized)
   - Events are buffered and batched before sending
   - Independent of OpenCensus - no automatic forwarding

3. **Dual Reporting Pattern**:
   - Code must call both functions explicitly
   - Example: Some events send to both, some only to one system
   - No automatic correlation between the two systems

**How Monitor Package Routes Metrics**:

The monitor package provides **two independent systems**:

1. **OpenCensus Metrics System** (`monitor/census.go`):
   - Functions record metrics to OpenCensus
   - Check `monitor.Enabled` flag
   - Export to Prometheus automatically

2. **Kafka Events System** (`monitor/kafka.go`):
   - Function `SendQueueEventAsync()` sends events to Kafka
   - Checks if Kafka producer is initialized
   - Independent of OpenCensus

**They are NOT automatically connected** - code must call both explicitly if dual reporting is needed.

**OpenCensus Implementation Details**:

The codebase uses **OpenCensus** (not just "Census") for metrics collection. Here's the actual implementation:

**Imports** (`monitor/census.go`):
```go
import (
    "contrib.go.opencensus.io/exporter/prometheus"
    "go.opencensus.io/stats"
    "go.opencensus.io/stats/view"
    "go.opencensus.io/tag"
)
```

**Measure Definition** (example for AI metrics):
```go
// Define measures
census.mAIFirstSegmentDelay = stats.Int64(
    "ai_first_segment_delay_ms", 
    "Delay of the first live AI segment being processed", 
    "ms"
)

census.mAILiveAttempts = stats.Int64(
    "ai_live_attempts", 
    "AI Live stream attempted", 
    "tot"
)
```

**View Registration** (aggregation configuration):
```go
views := []*view.View{
    {
        Name:        "ai_first_segment_delay_ms",
        Measure:     census.mAIFirstSegmentDelay,
        Description: "Delay of the first live AI segment being processed",
        TagKeys:     baseTagsWithOrchInfo,
        Aggregation: view.Distribution(0, .10, .20, .50, .100, .150, .200, .500, .1000, .5000, 10.000),
    },
    {
        Name:        "ai_live_attempt",
        Measure:     census.mAILiveAttempts,
        Description: "AI Live stream attempted",
        TagKeys:     append([]tag.Key{census.kModelName}, baseTags...),
        Aggregation: view.Count(),
    },
}

// Register views
if err := view.Register(views...); err != nil {
    glog.Exitf("Failed to register views: %v", err)
}
```

**Prometheus Exporter Setup**:
```go
registry := rprom.NewRegistry()
registry.MustRegister(rprom.NewProcessCollector(rprom.ProcessCollectorOpts{}))
registry.MustRegister(rprom.NewGoCollector())

pe, err := prometheus.NewExporter(prometheus.Options{
    Namespace: "livepeer",
    Registry:  registry,
})

// Register Prometheus exporter
view.RegisterExporter(pe)
Exporter = pe  // Exported for /metrics endpoint
```

**Metric Recording** (example usage):
```go
// Record metric with tags
func AIFirstSegmentDelay(delayMs int64, orchInfo *lpnet.OrchestratorInfo) {
    if err := stats.RecordWithTags(
        census.ctx, 
        orchInfoTags(orchInfo), 
        census.mAIFirstSegmentDelay.M(delayMs)
    ); err != nil {
        glog.Errorf("Error recording metrics err=%q", err)
    }
}

// Record simple metric
func AILiveVideoAttempt(modelID string) {
    if err := stats.RecordWithTags(
        census.ctx,
        []tag.Mutator{tag.Insert(census.kModelName, modelID)},
        census.mAILiveAttempts.M(1),
    ); err != nil {
        glog.Errorf("Error recording metrics err=%q", err)
    }
}
```

**Dependencies** (`go.mod`):
```go
require (
    contrib.go.opencensus.io/exporter/prometheus v0.4.2
    go.opencensus.io v0.24.0
)
```

**Example: Dual Reporting in Code** (`server/ai_live_video.go`):

Here's an actual example where both OpenCensus and Kafka are called for the same event:

```go
// First processed segment received
if monitor.Enabled {
    // 1. Record to OpenCensus
    monitor.AIFirstSegmentDelay(delayMs, params.liveParams.sess.OrchestratorInfo)
    
    // 2. Send to Kafka (separate call, independent)
    monitor.SendQueueEventAsync("stream_trace", map[string]interface{}{
        "type":        "gateway_receive_first_processed_segment",
        "timestamp":   time.Now().UnixMilli(),
        "stream_id":   params.liveParams.streamID,
        "pipeline_id": params.liveParams.pipelineID,
        "request_id":  params.liveParams.requestID,
        "orchestrator_info": map[string]interface{}{
            "address": params.liveParams.sess.Address(),
            "url":     params.liveParams.sess.Transcoder(),
        },
    })
}
```

**Example: OpenCensus Only** (`server/ai_mediaserver.go`):

```go
// Only OpenCensus, no Kafka event
monitor.AILiveVideoAttempt(pipeline) // Records to OpenCensus only
```

**Example: Kafka Only** (`server/ai_live_video.go`):

```go
// Only Kafka, no OpenCensus metric
monitor.SendQueueEventAsync("stream_trace", map[string]interface{}{
    "type": "gateway_send_first_ingest_segment",
    ...
})
```

**Example: Conditional Reporting** (`server/ai_process.go`):

```go
// OpenCensus only if enabled
if monitor.Enabled {
    monitor.AIRequestError(errMsg, monitor.ToPipeline(capName), modelID, nil)
}
// Kafka always (if producer initialized)
monitor.SendQueueEventAsync("stream_trace", map[string]interface{}{
    "type": "gateway_no_orchestrators_available",
    ...
})
```

**Initialization** (`cmd/livepeer/starter/starter.go`):

```go
// Initialize OpenCensus (if monitoring enabled)
if *cfg.Monitor {
    lpmon.Enabled = true
    lpmon.InitCensus(nodeType, core.LivepeerVersion)  // Sets up OpenCensus + Prometheus
}

// Initialize Kafka (if monitoring enabled)
if *cfg.Monitor {
    if err := startKafkaProducer(cfg); err != nil {
        exit("Error while starting Kafka producer", err)
    }
}
```

**Key Observations**:

1. **Independent Systems**: OpenCensus and Kafka are completely separate
2. **Explicit Calls**: Code must call both functions if dual reporting is needed
3. **Different Checks**: 
   - OpenCensus: `monitor.Enabled` flag
   - Kafka: `kafkaProducer != nil` check
4. **No Automatic Forwarding**: OpenCensus metrics are NOT automatically sent to Kafka
5. **Selective Usage**: Some events use both, some use only one system

**Collection Points by Workflow Stage**:

1. **Request Initiation**:
   - OpenCensus: `AILiveVideoAttempt()` → `ai_live_attempt` metric
   - Kafka: `gateway_receive_stream_request`

2. **Orchestrator Selection**:
   - OpenCensus: `AINumOrchestrators()` → `ai_orchestrators_available_total` metric
   - Kafka: `gateway_no_orchestrators_available` (if none found)

3. **Segment Upload**:
   - Kafka: `gateway_send_first_ingest_segment`

4. **Segment Processing**:
   - OpenCensus: `AIFirstSegmentDelay()` → `ai_first_segment_delay_ms` metric (distribution)
   - Kafka: `gateway_receive_first_processed_segment`
   - Kafka: `gateway_receive_few_processed_segments` (after 3 segments)

5. **Ongoing Processing**:
   - OpenCensus: `AICurrentLiveSessions()` → `ai_current_live_pipelines` metric (last value)
   - Kafka: `stream_ingest_metrics` (every 5 seconds)
   - OpenCensus: `AIWhipTransportBytesReceived/Sent()` → `ai_whip_transport_bytes_received/sent` metrics

6. **Error Handling**:
   - OpenCensus: `AIRequestError()` → `ai_request_errors` metric (count)
   - Kafka: `orchestrator_swap` (if swap occurs)

7. **Stream Completion**:
   - Kafka: `gateway_ingest_stream_closed`

#### 10.4.5 Non-Live AI Workflow Monitoring

**For Non-Live AI Jobs** (text-to-image, image-to-video, etc.):

1. **Request Processing**:
   ```go
   // OpenCensus Metrics
   monitor.AIRequestFinished(ctx, pipeline, modelID, AIJobInfo{
       LatencyScore: latencyScore,
       PricePerUnit: pricePerUnit
   })
   
   // On Error
   monitor.AIRequestError(err.Error(), pipeline, modelID, orchestratorInfo)
   ```

2. **Result Upload/Download**:
   ```go
   // Upload metrics
   monitor.AIResultUploaded(ctx, uploadDuration, pipeline, modelID, orchestratorAddr)
   
   // Download metrics
   monitor.AIResultDownloaded(ctx, pipeline, modelID, downloadDuration)
   ```

3. **Metrics Collected**:
   - Request latency score (processing time vs. content duration)
   - Price per unit
   - Upload/download times
   - Error rates
   - Model usage counts

**Note**: Non-live AI workflows use OpenCensus metrics primarily, with fewer Kafka events compared to live video workflows.

#### 10.4.6 Metrics Tagging & Dimensions

**Common Tags** (for OpenCensus metrics):
- `pipeline`: Pipeline name (e.g., "live-video-to-video")
- `model_name`: Model ID (e.g., "runway-gen3")
- `orchestrator_uri`: Orchestrator URL
- `orchestrator_address`: Orchestrator ETH address
- `gateway`: Gateway ETH address
- `node_type`: Node type (broadcaster, orchestrator, etc.)

**Kafka Event Dimensions**:
- `stream_id`: Unique stream identifier
- `pipeline_id`: Unique pipeline instance identifier
- `request_id`: Unique request identifier
- `orchestrator_info`: Orchestrator address and URL
- `timestamp`: Event timestamp (Unix milliseconds)

#### 10.4.7 Metrics Aggregation & Views

**OpenCensus View Types**:

1. **Count Views**: Total occurrences
   - `ai_models_requested`
   - `ai_request_errors`
   - `ai_live_attempt`

2. **Distribution Views**: Percentile distributions
   - `ai_first_segment_delay_ms`: Distribution of delays (0, 10, 20, 50, 100, 150, 200, 500, 1000, 5000, 10000 ms)
   - `ai_request_latency_score`: Performance distribution

3. **Last Value Views**: Current state
   - `ai_current_live_pipelines`: Current active pipelines
   - `ai_container_idle`: Current idle containers
   - `live_ai_price_per_pixel`: Current price

4. **Sum Views**: Cumulative totals
   - `ai_result_downloaded_total`
   - `ai_result_uploaded_total`

**Kafka Event Aggregation**:
- Events are sent as-is to Kafka
- Aggregation happens downstream (ClickHouse, analytics tools)
- Events include full context for later analysis

**Note on OpenCensus vs OpenTelemetry**:
- The codebase currently uses **OpenCensus** (`go.opencensus.io v0.24.0`)
- OpenTelemetry packages are present in `go.mod` but are **indirect dependencies** (likely from other libraries)
- OpenCensus is the **active** metrics collection framework used throughout the codebase
- Metrics are exported to Prometheus via the OpenCensus Prometheus exporter (`contrib.go.opencensus.io/exporter/prometheus`)

### 10.5 ClickHouse Integration

**Current Status**: **Not directly implemented in codebase**

The codebase sends events to Kafka, but ClickHouse integration would typically be:
- **Kafka → ClickHouse**: Consumer reads from Kafka topic and writes to ClickHouse
- **Direct Export**: Could add ClickHouse writer similar to Kafka producer

**Recommended Approach**: Use Kafka as buffer, then Kafka→ClickHouse consumer for analytics.

---

## 11. Load Test Gateway Design

### 11.1 Architecture Overview

```
Load Test Gateway
   |
   +---> [Orchestrator Discovery] --> [All Orchestrators]
   |                                      |
   |                                      v
   |                              [Load Test Scheduler]
   |                                      |
   |                                      v
   |                              [Test Execution Engine]
   |                                      |
   |                                      +---> [Test Job 1] --> Orchestrator A
   |                                      +---> [Test Job 2] --> Orchestrator B
   |                                      +---> [Test Job N] --> Orchestrator N
   |                                      |
   |                                      v
   |                              [Metrics Collector]
   |                                      |
   |                                      +---> [Kafka Producer]
   |                                      +---> [ClickHouse Writer]
   |                                      +---> [Local Aggregation]
```

### 11.2 Core Components

#### A. Load Test Scheduler

**Responsibilities**:
- Maintains list of all registered orchestrators
- Schedules test jobs at configurable intervals
- Distributes load evenly across orchestrators
- Respects orchestrator capacity limits

**Implementation**:
```go
type LoadTestScheduler struct {
    orchestratorPool OrchestratorPool
    testInterval     time.Duration
    testConfig       LoadTestConfig
    jobQueue         chan LoadTestJob
    metricsCollector *MetricsCollector
}

func (s *LoadTestScheduler) Start() {
    ticker := time.NewTicker(s.testInterval)
    for {
        select {
        case <-ticker.C:
            s.scheduleTestRound()
        }
    }
}

func (s *LoadTestScheduler) scheduleTestRound() {
    orchs := s.orchestratorPool.GetInfos()
    for _, orch := range orchs {
        job := LoadTestJob{
            Orchestrator: orch,
            TestType:     s.testConfig.TestType,
            TestData:     s.testConfig.TestData,
        }
        s.jobQueue <- job
    }
}
```

#### B. Test Execution Engine

**Responsibilities**:
- Executes test jobs against orchestrators
- Measures performance metrics
- Handles errors and timeouts
- Tracks test results

**Implementation**:
```go
type TestExecutionEngine struct {
    jobQueue         chan LoadTestJob
    resultsQueue     chan TestResult
    maxConcurrency   int
    testTimeout      time.Duration
}

func (e *TestExecutionEngine) Run() {
    semaphore := make(chan struct{}, e.maxConcurrency)
    for job := range e.jobQueue {
        semaphore <- struct{}{}
        go func(job LoadTestJob) {
            defer func() { <-semaphore }()
            result := e.executeTest(job)
            e.resultsQueue <- result
        }(job)
    }
}

func (e *TestExecutionEngine) executeTest(job LoadTestJob) TestResult {
    startTime := time.Now()
    
    // Get job token
    token, err := e.getJobToken(job.Orchestrator)
    if err != nil {
        return TestResult{Error: err, Orchestrator: job.Orchestrator.URL}
    }
    
    // Submit test job
    resp, err := e.submitJob(job, token)
    if err != nil {
        return TestResult{Error: err, Orchestrator: job.Orchestrator.URL}
    }
    
    // Measure metrics
    duration := time.Since(startTime)
    return TestResult{
        Orchestrator:    job.Orchestrator.URL,
        Success:         err == nil,
        ResponseTime:    duration,
        ProcessingTime: resp.ProcessingTime,
        Error:           err,
    }
}
```

#### C. Metrics Collector

**Responsibilities**:
- Aggregates test results
- Calculates statistics (P50, P95, P99, mean, etc.)
- Sends metrics to Kafka/ClickHouse
- Maintains historical data

**Implementation**:
```go
type MetricsCollector struct {
    resultsQueue    chan TestResult
    kafkaProducer   *KafkaProducer
    clickhouseWriter *ClickHouseWriter
    aggregator      *MetricsAggregator
}

func (c *MetricsCollector) Collect() {
    for result := range c.resultsQueue {
        // Aggregate metrics
        c.aggregator.AddResult(result)
        
        // Send to Kafka
        c.sendToKafka(result)
        
        // Send to ClickHouse
        c.sendToClickHouse(result)
    }
}

func (c *MetricsCollector) sendToKafka(result TestResult) {
    event := LoadTestEvent{
        Type:          "load_test_result",
        Orchestrator:  result.Orchestrator,
        Success:       result.Success,
        ResponseTime:  result.ResponseTime.Milliseconds(),
        ProcessingTime: result.ProcessingTime.Milliseconds(),
        Timestamp:     time.Now().UnixMilli(),
    }
    monitor.SendQueueEventAsync("load_test_result", event)
}
```

### 11.3 Load Test Configuration

**Test Types**:

1. **Capability Test**: Verify orchestrator supports required capabilities
2. **Latency Test**: Measure response time for small jobs
3. **Throughput Test**: Measure maximum jobs per second
4. **Reliability Test**: Long-running test with failure tracking
5. **Capacity Test**: Test at advertised capacity limits

**Configuration Structure**:
```go
type LoadTestConfig struct {
    TestType        string        // "capability", "latency", "throughput", etc.
    TestInterval    time.Duration // How often to run tests
    TestDuration    time.Duration // How long each test runs
    TestData        TestData      // Test input data
    MaxConcurrency  int           // Max parallel tests
    TestTimeout     time.Duration // Per-test timeout
    TargetOrchestrators []string   // Specific orchestrators (empty = all)
}
```

### 11.4 Metrics Collected

**Per-Orchestrator Metrics**:

1. **Availability**:
   - Uptime percentage
   - Discovery success rate
   - Token request success rate

2. **Performance**:
   - Response time (P50, P95, P99)
   - Processing time
   - Throughput (jobs/second)
   - Latency score

3. **Reliability**:
   - Error rate
   - Error types
   - Timeout rate
   - Retry count

4. **Capacity**:
   - Advertised capacity
   - Available capacity
   - Utilization percentage
   - Queue depth

5. **Network**:
   - Discovery latency
   - Connection establishment time
   - Upload/download speeds

**Aggregated Metrics**:

1. **Network-Wide**:
   - Total orchestrators
   - Available orchestrators
   - Average response time
   - Network throughput
   - Overall error rate

2. **SLA Metrics**:
   - Availability SLA (target: 99.9%)
   - Latency SLA (target: P95 < threshold)
   - Error rate SLA (target: < 1%)

### 11.5 Data Schema

**Kafka Event Schema**:
```json
{
  "id": "uuid",
  "type": "load_test_result",
  "timestamp": "unix_ms",
  "gateway": "load_test_gateway_address",
  "data": {
    "orchestrator_url": "https://orch.example.com",
    "orchestrator_address": "0x...",
    "test_type": "latency",
    "success": true,
    "response_time_ms": 150,
    "processing_time_ms": 120,
    "error": null,
    "test_id": "test_uuid",
    "round_id": "round_uuid"
  }
}
```

**ClickHouse Table Schema**:
```sql
CREATE TABLE load_test_results (
    timestamp DateTime,
    gateway String,
    orchestrator_url String,
    orchestrator_address String,
    test_type String,
    success UInt8,
    response_time_ms UInt32,
    processing_time_ms UInt32,
    error String,
    test_id String,
    round_id String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, orchestrator_url);
```

### 11.6 Ensuring Normal Workload Protection

**Design Principles**:

1. **Rate Limiting**: Limit load test frequency to avoid overwhelming orchestrators
2. **Capacity Awareness**: Respect advertised capacity, don't exceed
3. **Time-Based Scheduling**: Run tests during low-traffic periods if possible
4. **Priority Queuing**: Load test jobs have lower priority than production
5. **Circuit Breaker**: Stop testing if orchestrator shows signs of stress

**Implementation**:

```go
type LoadTestThrottler struct {
    maxTestsPerOrch    int           // Max tests per orchestrator per interval
    testInterval       time.Duration // Minimum time between test rounds
    capacityThreshold  float64       // Don't test if capacity < threshold
    lastTestTime       map[string]time.Time
}

func (t *LoadTestThrottler) CanTest(orch OrchestratorInfo) bool {
    // Check capacity
    if orch.AvailableCapacity < t.capacityThreshold {
        return false
    }
    
    // Check rate limit
    lastTest := t.lastTestTime[orch.URL]
    if time.Since(lastTest) < t.testInterval {
        return false
    }
    
    return true
}
```

**Capacity-Aware Testing**:
```go
func (s *LoadTestScheduler) scheduleTestRound() {
    orchs := s.orchestratorPool.GetInfos()
    for _, orch := range orchs {
        // Only test if orchestrator has available capacity
        if orch.AvailableCapacity > 0.1 { // 10% threshold
            // Use small test jobs that don't consume much capacity
            job := LoadTestJob{
                Orchestrator: orch,
                TestType:     "lightweight_latency",
                TestData:     s.getLightweightTestData(),
            }
            s.jobQueue <- job
        }
    }
}
```

### 11.7 Load Test Architecture Recommendations

#### A. Distributed Load Testing

**Option 1: Single Load Test Gateway**
- **Pros**: Simple, centralized metrics
- **Cons**: Single point of failure, limited scale

**Option 2: Multiple Load Test Gateways**
- **Pros**: Redundancy, geographic distribution
- **Cons**: Need coordination, duplicate tests possible

**Recommendation**: Start with single gateway, scale to multiple if needed.

#### B. Test Data Strategy

**Test Data Types**:

1. **Synthetic Test Data**:
   - Small, standardized test videos
   - Predictable processing requirements
   - Fast execution

2. **Realistic Test Data**:
   - Representative of production workloads
   - Various resolutions, bitrates
   - More accurate performance measurement

3. **Stress Test Data**:
   - Large files, complex processing
   - Tests capacity limits
   - Use sparingly

**Recommendation**: Use lightweight synthetic data for regular tests, realistic data for periodic deep tests.

#### C. Test Scheduling Strategy

**Recommended Schedule**:

1. **Continuous Light Tests**: Every 30 seconds
   - Quick capability check
   - Latency measurement
   - Minimal resource usage

2. **Periodic Medium Tests**: Every 5 minutes
   - Throughput test
   - Reliability check
   - Moderate resource usage

3. **Daily Deep Tests**: Once per day
   - Full capacity test
   - Stress test
   - Comprehensive metrics

**Time-Based Scheduling**:
```go
type TestSchedule struct {
    LightTests  time.Duration // 30s
    MediumTests time.Duration // 5m
    DeepTests   time.Duration // 24h
}

func (s *LoadTestScheduler) schedule() {
    lightTicker := time.NewTicker(30 * time.Second)
    mediumTicker := time.NewTicker(5 * time.Minute)
    deepTicker := time.NewTicker(24 * time.Hour)
    
    for {
        select {
        case <-lightTicker.C:
            s.runLightTests()
        case <-mediumTicker.C:
            s.runMediumTests()
        case <-deepTicker.C:
            s.runDeepTests()
        }
    }
}
```

### 11.8 SLA Monitoring & Alerting

**SLA Definitions**:

1. **Availability SLA**: 99.9% uptime
   - Calculate: (Successful tests / Total tests) × 100
   - Alert if: < 99.9% over 1 hour window

2. **Latency SLA**: P95 < 500ms
   - Calculate: 95th percentile of response times
   - Alert if: P95 > 500ms over 5 minute window

3. **Error Rate SLA**: < 1%
   - Calculate: (Errors / Total requests) × 100
   - Alert if: > 1% over 5 minute window

**Alerting Implementation**:
```go
type SLAMonitor struct {
    aggregator    *MetricsAggregator
    alertChannel  chan SLAAlert
    thresholds    SLATresholds
}

func (m *SLAMonitor) CheckSLA() {
    metrics := m.aggregator.GetMetrics()
    
    // Check availability
    if metrics.Availability < m.thresholds.Availability {
        m.alertChannel <- SLAAlert{
            Type:        "availability",
            Current:      metrics.Availability,
            Threshold:   m.thresholds.Availability,
            Severity:    "critical",
        }
    }
    
    // Check latency
    if metrics.P95Latency > m.thresholds.MaxLatency {
        m.alertChannel <- SLAAlert{
            Type:        "latency",
            Current:     metrics.P95Latency,
            Threshold:   m.thresholds.MaxLatency,
            Severity:    "warning",
        }
    }
    
    // Check error rate
    if metrics.ErrorRate > m.thresholds.MaxErrorRate {
        m.alertChannel <- SLAAlert{
            Type:        "error_rate",
            Current:     metrics.ErrorRate,
            Threshold:   m.thresholds.MaxErrorRate,
            Severity:    "critical",
        }
    }
}
```

### 11.9 Integration with Existing Codebase

**Reuse Existing Components**:

1. **Orchestrator Discovery**: Use existing `OrchestratorPool` and `GetOrchestrators()`
2. **Job Submission**: Reuse `getJobOrchestrators()` and job submission logic from `job_rpc.go`
3. **Kafka Integration**: Use existing `monitor.SendQueueEventAsync()`
4. **Metrics Collection**: Extend `monitor` package with load test metrics

**New Components Needed**:

1. **Load Test Scheduler**: New component for test orchestration
2. **Test Execution Engine**: New component for running tests
3. **Metrics Aggregator**: New component for calculating statistics
4. **ClickHouse Writer**: New component (or use Kafka→ClickHouse consumer)

**Implementation Path**:

1. **Phase 1**: Basic load test gateway
   - Single test type (latency)
   - Kafka integration
   - Basic metrics

2. **Phase 2**: Enhanced metrics
   - Multiple test types
   - ClickHouse integration
   - SLA monitoring

3. **Phase 3**: Advanced features
   - Distributed testing
   - Predictive analytics
   - Auto-scaling recommendations

---

## 12. Design Recommendations Summary

### 12.1 Load Test Gateway Architecture

**Recommended Architecture**:
- **Single dedicated gateway** for load testing (initially)
- **Separate from production gateways** to avoid interference
- **Rate-limited testing** to respect orchestrator capacity
- **Multi-tier test schedule** (light/medium/deep tests)

### 12.2 Data Collection Strategy

**Metrics Pipeline**:
```
Load Test Gateway
   |
   v
[Local Aggregation] --> [Kafka] --> [ClickHouse Consumer] --> [ClickHouse]
   |                                                              |
   v                                                              v
[Real-time Alerts]                                          [Analytics/Reports]
```

**Data Retention**:
- **Kafka**: 7 days (buffer)
- **ClickHouse**: 90 days (detailed), 1 year (aggregated)

### 12.3 Test Data Recommendations

**Lightweight Test Data**:
- Small test videos (1-5 seconds)
- Low resolution (480p)
- Standard codec (H.264)
- File size: < 1MB

**Realistic Test Data**:
- Representative resolutions (720p, 1080p)
- Various bitrates
- Multiple codecs
- File size: 5-50MB

**Stress Test Data**:
- High resolution (4K)
- High bitrate
- Complex processing
- File size: 50-500MB

### 12.4 Capacity Protection Mechanisms

1. **Capacity Threshold**: Don't test if orchestrator capacity < 10%
2. **Rate Limiting**: Max 1 test per orchestrator per 30 seconds
3. **Lightweight Tests**: Use minimal resource test jobs
4. **Time Windows**: Prefer low-traffic periods for deep tests
5. **Circuit Breaker**: Stop testing if error rate spikes

### 12.5 SLA Monitoring Recommendations

**Key SLAs to Monitor**:
1. **Availability**: 99.9% target
2. **Latency**: P95 < 500ms target
3. **Error Rate**: < 1% target
4. **Throughput**: Maintain advertised capacity

**Alerting Strategy**:
- **Critical**: Immediate notification (availability < 99%, error rate > 5%)
- **Warning**: Periodic notification (latency > threshold, error rate > 1%)
- **Info**: Daily summary reports

### 12.6 Implementation Priority

**High Priority** (MVP):
1. Basic load test gateway
2. Latency testing
3. Kafka integration
4. Basic metrics collection

**Medium Priority**:
1. Multiple test types
2. ClickHouse integration
3. SLA monitoring
4. Alerting

**Low Priority** (Future):
1. Distributed testing
2. Predictive analytics
3. Auto-remediation
4. Advanced reporting

