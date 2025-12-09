# FPS Measurement & Reporting Summary

## Quick Answer

### How FPS is Measured:

The AI Runner measures FPS using two `FPSCounter` instances in the `ProcessGuardian` class:

1. **Input FPS Counter**: Incremented when frames enter from the Gateway
2. **Output FPS Counter**: Incremented when frames exit after AI processing

**Measurement Process:**
- Counters increment on each frame: `counter.inc()`
- Every 10 seconds, the monitor loop calculates: `fps = frame_count / elapsed_time`
- FPS is computed over sliding 10-second windows
- Separate measurements for input and output streams

**Code Location:** `runner/app/live/process/process_guardian.py`

### How FPS is Reported Upstream:

1. **Collection:** Every 10 seconds, `report_status_loop()` gets current status including FPS
2. **Packaging:** Status wrapped in monitoring event with timestamp
3. **Transport:** Sent via Trickle Protocol HTTP POST to `events_url`
4. **Routing:** Orchestrator → Gateway
5. **Format:** JSON with full pipeline status including both input and output FPS

**Code Location:** `runner/app/live/streamer/streamer.py`

---

## Detailed Breakdown

### 1. FPS Counter Class

**File:** `runner/app/live/process/process_guardian.py:343-372`

```python
class FPSCounter:
    def __init__(self):
        self.frame_count = 0
        self.start_time = None
    
    def inc(self):
        """Called when a frame is processed"""
        if not self.start_time:
            # First frame: set measurement window start
            self.start_time = time.time()
            self.frame_count = 0
        else:
            # Subsequent frames: increment counter
            self.frame_count += 1
    
    def fps(self, now=time.time()) -> float:
        """Calculate FPS and reset for next window"""
        if not self.frame_count or not self.start_time:
            return 0.0
        
        # Calculate FPS over the measurement window
        fps = self.frame_count / (now - self.start_time)
        
        # Reset for next measurement window
        self.start_time = now
        self.frame_count = 0
        
        return fps
```

**Key Characteristics:**
- Simple frame counting over time windows
- Sliding window (resets after each calculation)
- First frame only sets start time, doesn't count
- Thread-safe for single producer/consumer

### 2. Frame Counting Triggers

**Input FPS - File:** `runner/app/live/process/process_guardian.py:100-107`

```python
def send_input(self, frame: InputFrame):
    """Called when frame arrives from Gateway"""
    if not self.process:
        raise RuntimeError("Process not running")
    
    # Track timing
    self.status.input_status.last_input_time = time.time()
    
    # Increment input FPS counter
    self.input_fps_counter.inc()
    
    # Forward to pipeline process
    self.process.send_input(frame)
```

**Output FPS - File:** `runner/app/live/process/process_guardian.py:109-118`

```python
async def recv_output(self) -> OutputFrame | None:
    """Called when frame comes from AI pipeline"""
    if not self.process:
        raise RuntimeError("Process not running")
    
    output = await self.process.recv_output()
    
    # Only count non-loading frames
    if output and not output.is_loading_frame:
        # Track timing
        self.status.inference_status.last_output_time = time.time()
        
        # Increment output FPS counter
        self.output_fps_counter.inc()
    
    return output
```

### 3. FPS Calculation in Monitor Loop

**File:** `runner/app/live/process/process_guardian.py:263-297`

```python
async def _monitor_loop(self):
    """Runs continuously to monitor pipeline health and compute metrics"""
    last_fps_compute = time.time()
    
    while True:
        # Check status every second
        await asyncio.sleep(1)
        
        if not self.process:
            continue
        
        # ... error checking ...
        
        now = time.time()
        
        # Compute FPS every 10 seconds
        if now - last_fps_compute > FPS_LOG_INTERVAL:  # FPS_LOG_INTERVAL = 10.0
            # Calculate FPS for both streams
            input_fps = self.input_fps_counter.fps(now)
            output_fps = self.output_fps_counter.fps(now)
            
            # Update status object
            self.status.input_status.fps = input_fps
            self.status.inference_status.fps = output_fps
            
            # Reset timer for next calculation
            last_fps_compute = now
            
            # Log locally
            if self.streamer.is_stream_running():
                if input_fps > 0:
                    logging.info(f"Input FPS: {input_fps:.2f}")
                if output_fps > 0:
                    logging.info(f"Output FPS: {output_fps:.2f}")
        
        # ... state management ...
```

**Timing:**
- Monitor loop: Every 1 second
- FPS calculation: Every 10 seconds
- Provides smoothed FPS over 10-second windows

### 4. Status Reporting Loop

**File:** `runner/app/live/streamer/streamer.py:122-134`

```python
async def report_status_loop(self):
    """Reports status upstream every 10 seconds"""
    next_report = time.time() + status_report_interval  # 10 seconds
    
    while not self.stop_event.is_set():
        current_time = time.time()
        
        # Wait until next report time
        if next_report <= current_time:
            next_report = current_time + status_report_interval
        else:
            await asyncio.sleep(next_report - current_time)
            next_report += status_report_interval
        
        # Get current status (includes FPS)
        status = self.process.get_status(clear_transient=True)
        
        # Emit to upstream via Trickle protocol
        await self.emit_monitoring_event(status.model_dump())
```

### 5. Monitoring Event Emission

**File:** `runner/app/live/streamer/streamer.py:136-144`

```python
async def emit_monitoring_event(self, event: dict, queue_event_type: str = "ai_stream_events"):
    """Protected method to emit monitoring event with lock"""
    # Add timestamp
    event["timestamp"] = timestamp_to_ms(time.time())
    
    logging.info(f"Emitting monitoring event: {event}")
    
    # Thread-safe emission
    async with self.emit_event_lock:
        try:
            await self.protocol.emit_monitoring_event(event, queue_event_type)
        except Exception as e:
            logging.error(f"Failed to emit monitoring event: {e}")
```

### 6. Trickle Protocol Transmission

**File:** `runner/app/live/streamer/protocol/trickle.py:101-109`

```python
async def emit_monitoring_event(self, event: dict, queue_event_type: str = "ai_stream_events"):
    """Send event via Trickle protocol to Orchestrator"""
    if not self.events_publisher:
        return
    
    try:
        # Wrap event with queue type
        event_json = json.dumps({
            "event": event,
            "queue_event_type": queue_event_type
        })
        
        # HTTP POST to events_url
        async with await self.events_publisher.next() as event:
            await event.write(event_json.encode())
    except Exception as e:
        logging.error(f"Error reporting status: {e}")
```

**File:** `runner/app/live/trickle/trickle_publisher.py`

The `TricklePublisher` class handles the actual HTTP POST:
- Uses `aiohttp` for async HTTP
- Posts to `events_url` provided by Orchestrator
- Content-Type: `application/json`
- Connection: close (new connection per event)

---

## Status Data Structure

### PipelineStatus

**File:** `runner/app/live/process/status.py`

```python
class PipelineStatus(BaseModel):
    """Complete pipeline status including FPS metrics"""
    type: str = "status"
    pipeline: str                    # e.g., "comfyui", "streamdiffusion"
    start_time: float                # When stream started
    state: str                       # LOADING, ONLINE, OFFLINE, DEGRADED_*, ERROR
    last_state_update_time: float | None
    
    input_status: InputStatus        # Input stream metrics
    inference_status: InferenceStatus # Inference metrics

class InputStatus(BaseModel):
    """Input stream metrics"""
    last_input_time: float | None    # Last frame received time
    fps: float = 0.0                 # Input FPS (frames from Gateway)

class InferenceStatus(BaseModel):
    """Inference process metrics"""
    last_output_time: float | None   # Last frame produced time
    fps: float = 0.0                 # Output FPS (frames after AI processing)
    
    last_params_update_time: float | None
    last_params: dict | None
    last_params_hash: str | None
    
    last_error_time: float | None
    last_error: str | None
    
    last_restart_time: float | None
    last_restart_logs: list[str] | None
    restart_count: int = 0
```

### Example Monitoring Event

```json
{
  "event": {
    "type": "status",
    "pipeline": "comfyui",
    "start_time": 1699876543000,
    "state": "ONLINE",
    "last_state_update_time": 1699876543000,
    "input_status": {
      "last_input_time": 1699876553000,
      "fps": 24.5
    },
    "inference_status": {
      "last_output_time": 1699876553000,
      "fps": 23.8,
      "last_params_update_time": 1699876543000,
      "last_params": {
        "strength": 0.8,
        "guidance_scale": 7.5,
        "width": 512,
        "height": 512
      },
      "last_params_hash": "a1b2c3d4e5f6",
      "last_error_time": null,
      "last_error": null,
      "last_restart_time": null,
      "last_restart_logs": null,
      "restart_count": 0
    }
  },
  "queue_event_type": "ai_stream_events",
  "timestamp": 1699876553000
}
```

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Frame Processing Flow                        │
└─────────────────────────────────────────────────────────────────┘

Gateway → Orchestrator → Trickle Subscribe → FFmpeg Decode
                                                    ↓
                                              VideoFrame
                                                    ↓
                                    ┌───────────────────────────┐
                                    │   PipelineStreamer        │
                                    │   run_ingress_loop()      │
                                    └───────────┬───────────────┘
                                                ↓
                                    ┌───────────────────────────┐
                                    │   ProcessGuardian         │
                                    │   send_input(frame)       │
                                    │   input_fps_counter.inc() │ ← Count input frame
                                    └───────────┬───────────────┘
                                                ↓
                                    ┌───────────────────────────┐
                                    │   PipelineProcess         │
                                    │   (multiprocessing)       │
                                    │   AI Model Inference      │
                                    └───────────┬───────────────┘
                                                ↓
                                    ┌───────────────────────────┐
                                    │   ProcessGuardian         │
                                    │   recv_output()           │
                                    │   output_fps_counter.inc()│ ← Count output frame
                                    └───────────┬───────────────┘
                                                ↓
                                    ┌───────────────────────────┐
                                    │   PipelineStreamer        │
                                    │   run_egress_loop()       │
                                    └───────────┬───────────────┘
                                                ↓
                                          VideoFrame
                                                ↓
FFmpeg Encode → Trickle Publish → Orchestrator → Gateway


┌─────────────────────────────────────────────────────────────────┐
│                   FPS Measurement Flow                           │
└─────────────────────────────────────────────────────────────────┘

Every 1 second:
    ┌───────────────────────────┐
    │   ProcessGuardian         │
    │   _monitor_loop()         │
    └───────────┬───────────────┘
                ↓
    Every 10 seconds:
                ↓
    ┌───────────────────────────┐
    │   FPSCounter              │
    │   input_fps_counter.fps() │ → Calculate: frames / elapsed_time
    │   output_fps_counter.fps()│ → Calculate: frames / elapsed_time
    └───────────┬───────────────┘
                ↓
    ┌───────────────────────────┐
    │   PipelineStatus          │
    │   input_status.fps = X    │ ← Store input FPS
    │   inference_status.fps = Y│ ← Store output FPS
    └───────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│                   Upstream Reporting Flow                        │
└─────────────────────────────────────────────────────────────────┘

Every 10 seconds:
    ┌───────────────────────────┐
    │   PipelineStreamer        │
    │   report_status_loop()    │
    └───────────┬───────────────┘
                ↓
    ┌───────────────────────────┐
    │   ProcessGuardian         │
    │   get_status()            │ → Returns PipelineStatus with FPS
    └───────────┬───────────────┘
                ↓
    ┌───────────────────────────┐
    │   PipelineStreamer        │
    │   emit_monitoring_event() │
    └───────────┬───────────────┘
                ↓
    ┌───────────────────────────┐
    │   TrickleProtocol         │
    │   emit_monitoring_event() │
    └───────────┬───────────────┘
                ↓
    ┌───────────────────────────┐
    │   TricklePublisher        │
    │   HTTP POST to events_url │
    └───────────┬───────────────┘
                ↓
           Orchestrator
                ↓
             Gateway
                ↓
    ┌───────────────────────────┐
    │   Metrics Aggregation     │
    │   - Input FPS: 24.5       │
    │   - Output FPS: 23.8      │
    │   - State: ONLINE         │
    └───────────────────────────┘
```

---

## Key Timing Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `FPS_LOG_INTERVAL` | 10 seconds | How often FPS is calculated |
| `status_report_interval` | 10 seconds | How often status is sent upstream |
| Monitor loop interval | 1 second | How often status is checked internally |
| Health check interval | 5 seconds | How often worker checks runner health |
| Stream timeout | 60 seconds | No input → triggers shutdown |
| Error timeout | 90 seconds | No input → ERROR state |

---

## FPS Interpretation

### Input FPS
- **What it measures:** Frames received from Gateway per second
- **Indicates:** Network throughput, upstream performance
- **Healthy range:** Should match source video FPS (e.g., 24-30 fps)
- **Low FPS causes:** Network issues, Gateway overload, bandwidth limits

### Output FPS (Inference FPS)
- **What it measures:** Frames processed by AI model per second
- **Indicates:** Inference performance, GPU utilization
- **Healthy range:** Should be close to input FPS (e.g., 0.8-1.0x input)
- **Low FPS causes:** GPU overload, complex model, insufficient VRAM

### FPS Comparison
- **input_fps ≈ output_fps:** Healthy, keeping up with input
- **output_fps < 0.8 * input_fps:** Degraded inference (state changes)
- **output_fps << input_fps:** Serious performance issue
- **input_fps < 15:** Degraded input (state changes)

---

## Pipeline States Based on FPS

The system uses FPS metrics to determine pipeline health:

- **ONLINE:** Both FPS healthy, processing normally
- **DEGRADED_INPUT:** `input_fps < 15` or gaps > 2s
- **DEGRADED_INFERENCE:** `output_fps < min(10, 0.8 * input_fps)` or recent errors
- **ERROR:** No output for > 5s, or other fatal conditions
- **OFFLINE:** No active stream

State transitions are logged and reported in monitoring events.

---

## Summary

**FPS Measurement:**
1. Frame counters increment on each frame
2. Every 10 seconds, calculate: `fps = frame_count / elapsed_time`
3. Store in `PipelineStatus` object
4. Reset counters for next window

**Upstream Reporting:**
1. Every 10 seconds, collect current status
2. Package as JSON monitoring event
3. HTTP POST via Trickle protocol to `events_url`
4. Orchestrator forwards to Gateway
5. Gateway aggregates metrics from all runners

**Key Files:**
- FPS counting: `runner/app/live/process/process_guardian.py`
- Status reporting: `runner/app/live/streamer/streamer.py`
- Trickle protocol: `runner/app/live/streamer/protocol/trickle.py`
- Status structure: `runner/app/live/process/status.py`

