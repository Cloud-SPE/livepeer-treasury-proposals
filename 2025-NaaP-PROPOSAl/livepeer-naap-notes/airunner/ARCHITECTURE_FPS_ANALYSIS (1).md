# AI Runner Architecture & FPS Measurement Analysis

## Overview

The Livepeer AI Runner is a containerized Python application that processes AI inference jobs on the Livepeer network. It measures inference FPS and reports metrics upstream to the Gateway through the Orchestrator.

## Architecture Components

### 1. **AI Worker (Go-Livepeer)**
- Embedded in go-livepeer
- Can run as Orchestrator or separate Worker node
- Manages runner containers lifecycle
- Polls `/health` endpoint every 5 seconds
- Handles container state: OK → keep running, IDLE → return to pool, ERROR → kill after 2 errors

### 2. **AI Runner Container**
Contains two main processes:

#### A. **Runner API (FastAPI)**
- Entry point: `runner/app/main.py`
- Exposes REST endpoints:
  - `/health` - Health check endpoint
  - `/live-video-to-video` - Start inference stream
  - Pipeline-specific routes (text-to-image, image-to-image, etc.)
- Loads pipeline based on `PIPELINE` and `MODEL_ID` environment variables

#### B. **Infer Process (`infer.py`)**
- Started when `/live-video-to-video` API is called
- Manages inference runtime and monitoring
- Runs as subprocess with its own HTTP server on port 8888
- Exposes internal endpoints:
  - `/api/status` - Returns pipeline status with FPS metrics
  - `/api/params` - Update inference parameters
  - `/api/live-video-to-video` - Start new stream

### 3. **Process Guardian**
Location: `runner/app/live/process/process_guardian.py`

**Responsibilities:**
- Keeps pipeline process alive
- Monitors process status
- Handles input/output frame streaming
- **Measures FPS using FPSCounter instances**

**Key Components:**
- `input_fps_counter`: Tracks input frame rate
- `output_fps_counter`: Tracks output (inference) frame rate
- `_monitor_loop()`: Runs every 1 second to compute metrics

### 4. **Pipeline Process**
Location: `runner/app/live/process/process.py`

- Separate subprocess using `torch.multiprocessing`
- Runs actual AI inference
- Communicates via multiprocessing Queues:
  - `input_queue`: Receives video/audio frames
  - `output_queue`: Sends processed frames
  - `param_update_queue`: Receives parameter updates
  - `error_queue`: Reports errors
  - `log_queue`: Logs for diagnostics

### 5. **Pipeline Streamer**
Location: `runner/app/live/streamer/streamer.py`

**Main Tasks (runs concurrently):**
- `run_ingress_loop()`: Receives frames from Trickle protocol
- `run_egress_loop()`: Sends processed frames back
- `report_status_loop()`: Reports status every 10 seconds
- `run_control_loop()`: Handles parameter updates

### 6. **Trickle Protocol**
Location: `runner/app/live/streamer/protocol/trickle.py`

**Purpose:** Low-latency streaming protocol

**Streams:**
- `subscribe_url`: Input video stream (Gateway → Runner via Orchestrator)
- `publish_url`: Output video stream (Runner → Gateway via Orchestrator)
- `control_url`: Control messages for parameter updates
- `events_url`: **Monitoring events including FPS metrics**

**Media Processing:**
- Uses FFmpeg for encoding/decoding
- Segments streamed incrementally (Trickle magic)
- Frames decoded and sent to Pipeline Process

## FPS Measurement Flow

### 1. **FPS Counter Implementation**
Location: `runner/app/live/process/process_guardian.py:343-372`

```python
class FPSCounter:
    def __init__(self):
        self.frame_count = 0
        self.start_time = None
    
    def inc(self):
        """Called when a frame is processed"""
        if not self.start_time:
            self.start_time = time.time()
            self.frame_count = 0
        else:
            self.frame_count += 1
    
    def fps(self, now=time.time()) -> float:
        """Calculate FPS over measurement window"""
        if not self.frame_count or not self.start_time:
            return 0.0
        
        fps = self.frame_count / (now - self.start_time)
        
        # Reset for next window
        self.start_time = now
        self.frame_count = 0
        
        return fps
```

**Characteristics:**
- Sliding window measurement (resets after each calculation)
- Separate counters for input and output
- First frame only sets start time, doesn't count

### 2. **FPS Computation in Monitor Loop**
Location: `runner/app/live/process/process_guardian.py:263-297`

```python
async def _monitor_loop(self):
    last_fps_compute = time.time()
    while True:
        await asyncio.sleep(1)  # Check every second
        
        now = time.time()
        if now - last_fps_compute > FPS_LOG_INTERVAL:  # 10 seconds
            input_fps = self.input_fps_counter.fps(now)
            output_fps = self.output_fps_counter.fps(now)
            
            # Update status
            self.status.input_status.fps = input_fps
            self.status.inference_status.fps = output_fps
            
            # Log locally
            if self.streamer.is_stream_running():
                if input_fps > 0:
                    logging.info(f"Input FPS: {input_fps:.2f}")
                if output_fps > 0:
                    logging.info(f"Output FPS: {output_fps:.2f}")
```

**Timing:**
- Monitor loop runs every 1 second
- FPS computed every 10 seconds (`FPS_LOG_INTERVAL`)
- Provides smoothed FPS over 10-second windows

### 3. **Frame Counting Triggers**

**Input FPS:**
```python
def send_input(self, frame: InputFrame):
    self.status.input_status.last_input_time = time.time()
    self.input_fps_counter.inc()  # Increment counter
    self.process.send_input(frame)
```

**Output FPS:**
```python
async def recv_output(self) -> OutputFrame | None:
    output = await self.process.recv_output()
    if output and not output.is_loading_frame:
        self.status.inference_status.last_output_time = time.time()
        self.output_fps_counter.inc()  # Increment counter
    return output
```

## Upstream Reporting Flow

### 1. **Status Report Loop**
Location: `runner/app/live/streamer/streamer.py:122-134`

```python
async def report_status_loop(self):
    next_report = time.time() + status_report_interval  # 10 seconds
    while not self.stop_event.is_set():
        await asyncio.sleep(next_report - current_time)
        next_report += status_report_interval
        
        # Get current status (includes FPS)
        status = self.process.get_status(clear_transient=True)
        
        # Emit to upstream
        await self.emit_monitoring_event(status.model_dump())
```

**Timing:** Every 10 seconds

### 2. **Status Structure**
Location: `runner/app/live/process/status.py`

```python
class PipelineStatus(BaseModel):
    type: str = "status"
    pipeline: str
    start_time: float
    state: str  # LOADING, ONLINE, OFFLINE, DEGRADED_INPUT, DEGRADED_INFERENCE, ERROR
    
    input_status: InputStatus
    inference_status: InferenceStatus

class InputStatus(BaseModel):
    last_input_time: float | None
    fps: float = 0.0  # Input FPS

class InferenceStatus(BaseModel):
    last_output_time: float | None
    fps: float = 0.0  # Output/Inference FPS
    last_error: str | None
    restart_count: int
    # ... other fields
```

### 3. **Emit Monitoring Event**
Location: `runner/app/live/streamer/streamer.py:136-144`

```python
async def emit_monitoring_event(self, event: dict, queue_event_type: str = "ai_stream_events"):
    event["timestamp"] = timestamp_to_ms(time.time())
    logging.info(f"Emitting monitoring event: {event}")
    
    async with self.emit_event_lock:
        await self.protocol.emit_monitoring_event(event, queue_event_type)
```

### 4. **Trickle Protocol Emission**
Location: `runner/app/live/streamer/protocol/trickle.py:101-109`

```python
async def emit_monitoring_event(self, event: dict, queue_event_type: str = "ai_stream_events"):
    if not self.events_publisher:
        return
    
    event_json = json.dumps({"event": event, "queue_event_type": queue_event_type})
    async with await self.events_publisher.next() as event:
        await event.write(event_json.encode())
```

### 5. **Trickle Publisher**
Location: `runner/app/live/trickle/trickle_publisher.py`

- Uses aiohttp to POST events to `events_url`
- Events flow: Runner → Orchestrator → Gateway
- JSON format with event type and data

## Complete Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         AI Worker (Go)                          │
│  - Manages container lifecycle                                  │
│  - Polls /health every 5s                                       │
│  - Provides Trickle URLs (subscribe, publish, control, events)  │
└────────────────┬────────────────────────────────▲───────────────┘
                 │                                │
                 │ HTTP                           │ Trickle Events
                 ▼                                │
┌─────────────────────────────────────────────────┴───────────────┐
│                    AI Runner Container                          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Runner API (FastAPI - Port 8000)              │ │
│  │  - /health → checks infer.py status                        │ │
│  │  - /live-video-to-video → starts infer.py subprocess       │ │
│  └──────────────────────────┬─────────────────────────────────┘ │
│                             │ subprocess.Popen                   │
│                             ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         Infer Process (infer.py - Port 8888)               │ │
│  │                                                             │ │
│  │  ┌───────────────────────────────────────────────────────┐ │ │
│  │  │              ProcessGuardian                          │ │ │
│  │  │  - input_fps_counter.inc() on each input frame       │ │ │
│  │  │  - output_fps_counter.inc() on each output frame     │ │ │
│  │  │  - _monitor_loop() every 1s                          │ │ │
│  │  │    * Computes FPS every 10s                          │ │ │
│  │  │    * Updates status.input_status.fps                 │ │ │
│  │  │    * Updates status.inference_status.fps             │ │ │
│  │  └───────────────┬───────────────────────────────────────┘ │ │
│  │                  │                                          │ │
│  │  ┌───────────────▼───────────────────────────────────────┐ │ │
│  │  │           PipelineStreamer                            │ │ │
│  │  │                                                        │ │ │
│  │  │  Concurrent Tasks:                                    │ │ │
│  │  │  1. run_ingress_loop() ─┐                            │ │ │
│  │  │     Trickle Subscribe   │                            │ │ │
│  │  │     ↓ FFmpeg Decode     │                            │ │ │
│  │  │     ↓ send_input()      │                            │ │ │
│  │  │                         │                            │ │ │
│  │  │  2. run_egress_loop() ──┤                            │ │ │
│  │  │     recv_output()       │                            │ │ │
│  │  │     ↓ FFmpeg Encode     │                            │ │ │
│  │  │     ↓ Trickle Publish   │                            │ │ │
│  │  │                         │                            │ │ │
│  │  │  3. report_status_loop()│ ← Every 10s               │ │ │
│  │  │     get_status()        │   Includes FPS            │ │ │
│  │  │     ↓ emit_monitoring_event()                        │ │ │
│  │  │     ↓ Trickle Events    │                            │ │ │
│  │  │                         │                            │ │ │
│  │  │  4. run_control_loop()  │                            │ │ │
│  │  │     Trickle Control     │                            │ │ │
│  │  │     ↓ update_params()   │                            │ │ │
│  │  └─────────────────────────┼───────────────────────────┘ │ │
│  │                            │                              │ │
│  │  ┌─────────────────────────▼───────────────────────────┐ │ │
│  │  │         PipelineProcess (multiprocessing)           │ │ │
│  │  │  - Separate Python process                          │ │ │
│  │  │  - Loads AI model (ComfyUI, StreamDiffusion, etc.)  │ │ │
│  │  │  - Runs actual inference                            │ │ │
│  │  │  - Communicates via Queues:                         │ │ │
│  │  │    * input_queue: receives frames                   │ │ │
│  │  │    * output_queue: sends processed frames           │ │ │
│  │  │    * param_update_queue: parameter changes          │ │ │
│  │  │    * error_queue: error reporting                   │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ Trickle Protocol
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Orchestrator                             │
│  - Proxies Trickle streams                                      │
│  - Forwards monitoring events to Gateway                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Gateway                                │
│  - Receives monitoring events with FPS metrics                  │
│  - Aggregates metrics from multiple runners                     │
└─────────────────────────────────────────────────────────────────┘
```

## FPS Measurement Summary

### How FPS is Measured:

1. **Frame Counting:**
   - `input_fps_counter.inc()` called when frame enters ProcessGuardian
   - `output_fps_counter.inc()` called when frame exits Pipeline Process
   - Counters track frames over time windows

2. **Calculation:**
   - Every 10 seconds in `_monitor_loop()`
   - Formula: `fps = frame_count / elapsed_time`
   - Sliding window (resets after each calculation)
   - Separate measurements for input and output

3. **Storage:**
   - Stored in `PipelineStatus` object:
     - `input_status.fps`: Input frame rate
     - `inference_status.fps`: Output/inference frame rate

### How FPS is Reported Upstream:

1. **Collection:**
   - `report_status_loop()` runs every 10 seconds
   - Calls `process.get_status()` to get current metrics

2. **Emission:**
   - `emit_monitoring_event()` sends status as JSON
   - Event type: `"ai_stream_events"`
   - Includes full `PipelineStatus` with FPS values

3. **Transport:**
   - Uses Trickle Protocol via `events_url`
   - HTTP POST to Orchestrator
   - Orchestrator forwards to Gateway

4. **Event Structure:**
   ```json
   {
     "event": {
       "type": "status",
       "pipeline": "comfyui",
       "state": "ONLINE",
       "start_time": 1699876543000,
       "input_status": {
         "last_input_time": 1699876553000,
         "fps": 24.5
       },
       "inference_status": {
         "last_output_time": 1699876553000,
         "fps": 23.8,
         "last_error": null,
         "restart_count": 0
       }
     },
     "queue_event_type": "ai_stream_events",
     "timestamp": 1699876553000
   }
   ```

## Key Configuration Constants

- `FPS_LOG_INTERVAL = 10.0` seconds - How often FPS is computed
- `status_report_interval = 10` seconds - How often status is reported upstream
- Health check interval: 5 seconds (from worker)
- Monitor loop interval: 1 second
- Stream timeout: 60 seconds of no input → triggers shutdown

## Pipeline States

The system uses these states to indicate health:

- **LOADING**: Pipeline initializing (model loading)
- **ONLINE**: Healthy, processing frames normally
- **OFFLINE**: No active stream
- **DEGRADED_INPUT**: Input FPS < 15 or gaps > 2s
- **DEGRADED_INFERENCE**: Output FPS < min(10, 0.8 * input_fps) or recent errors
- **ERROR**: Fatal error, container should be restarted

State transitions are logged and included in monitoring events.

## Additional Metrics Reported

Beyond FPS, the system reports:

- **Timestamps:** Last input time, last output time, state change times
- **Errors:** Last error message and time
- **Restarts:** Pipeline process restart count and logs
- **Parameters:** Current inference parameters (hash for change detection)
- **State:** Current pipeline state with transition history

All metrics flow through the same Trickle events channel to the Gateway.

