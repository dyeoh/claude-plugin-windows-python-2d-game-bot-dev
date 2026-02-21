# Screen Capture Optimization

Domain knowledge for the 3-tier capture backend chain, frame management, adaptive FPS, and VM compatibility in Python 2D game bots.

## 3-Tier Backend Chain

Backend selection happens at import time with automatic fallback. Runtime validation at startup, and automatic failover during operation.

| Priority | Backend | API | FPS | Best For | Limitation |
|----------|---------|-----|-----|----------|------------|
| 1 | **bettercam** | DXGI Desktop Duplication | Highest | Bare metal | Stale duplicator on Hyper-V |
| 2 | **windows-capture** | Windows Graphics Capture (WGC) | High | VMs, DWM | Requires Windows 10 1903+ |
| 3 | **mss** | GDI BitBlt | ~75 | Universal | Highest CPU, lowest FPS |

### Selection Flow

```
Import time:
  try bettercam → apply monkey-patches → set as primary
  except → try windows-capture → set as primary
  except → fall back to mss

Runtime startup:
  _create_backend() validates with test grabs (3 retries for bettercam)
  If validation fails → fall through to next backend

During operation:
  If 3 consecutive screenshot failures → _next_backend() switches automatically
```

## Hyper-V VM Patches (bettercam)

Four monkey-patches handle DXGI `E_INVALIDARG` errors on Hyper-V GPU-PV:

| Patch | Target | Recovery |
|-------|--------|----------|
| `Duplicator.__post_init__` | Construction `E_INVALIDARG` | Retry 5x with output desc refresh |
| `Duplicator.update_frame` | Grab `E_INVALIDARG` | Return False → trigger `_on_output_change` |
| `BetterCam._on_output_change` | Infinite retry loop | Cap at 5 attempts, raise on failure |
| `BetterCam.__del__` | Shutdown `AttributeError` | Suppress cleanup errors |

### DXFactory Singleton Reset

Between bettercam creation retries, the DXFactory singleton is destroyed and recreated to obtain fresh DXGI adapters and D3D devices. Simply clearing the camera cache reuses stale COM pointers.

## WGC Camera Adapter

Wraps the windows-capture callback API into bettercam's synchronous interface:

```python
class _WGCCamera:
    def __init__(self):
        self._frame = None
        self._lock = threading.Lock()
        # WindowsCapture(monitor_index=1)

    def grab(self, region=None):
        with self._lock:
            frame = self._frame  # Latest frame from callback
        if frame is None:
            return None
        if region:
            left, top, right, bottom = region
            return frame[top:bottom, left:right]
        return frame
```

- Uses monitor capture (not window capture) for coordinate consistency with bettercam
- Frames arrive via callback thread, copied to avoid Rust buffer invalidation
- Thread-safe retrieval via lock

## Adaptive FPS

| Bot State | FPS | Rationale |
|-----------|-----|-----------|
| ENABLED (active) | 60 | Real-time detection needed |
| IDLE / other | 15 | Reduce CPU while monitoring |

## Frame Management Rules

### One Capture Per Loop

The capture thread grabs one frame per iteration and stores it in `config.capture.frame`. All detection consumers read from this shared frame. Never re-capture within a consumer.

### BGRA→BGR Conversion

Done once at capture time. All downstream `cv2` calls expect BGR format:

```python
frame_bgr = cv2.cvtColor(frame_bgra, cv2.COLOR_BGRA2BGR)
config.capture.frame = frame_bgr  # Share BGR result
```

### Thread Safety

The capture thread writes `config.capture.frame`. All other threads (Notifier, Bot, detection) read. Copy before multi-step processing:

```python
frame = config.capture.frame
if frame is None:
    return
frame = frame.copy()  # Safe for multi-step processing
```

### Frame Validity

Guard against corrupted frames:

```python
try:
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
except cv2.error:
    return  # Skip corrupted frame
```

## High-Resolution Timer

Windows system timer set to 1ms resolution at startup:

```python
# In main.py
ctypes.windll.winmm.timeBeginPeriod(1)  # 1ms resolution
# cleanup via atexit:
ctypes.windll.winmm.timeEndPeriod(1)
```

This makes `time.sleep()` accurate to ~1ms instead of the default ~15ms. Critical for all timing in capture, movement, and input systems.

## Runtime Backend Failover

If a backend fails repeatedly during operation (GPU driver reset, display disconnect):

1. `screenshot()` tracks consecutive failures via `_screenshot_failures`
2. After 3 consecutive failures, `_next_backend()` is called
3. Old camera released, next backend activated
4. Failure counter resets, capture continues seamlessly

## Performance Notes

- bettercam: Lowest CPU, highest FPS, but requires DXGI duplicator (bare metal)
- WGC: Good FPS, works on VMs, but callback model adds small latency
- mss: Universal but ~75 FPS max and highest CPU usage
- DPI awareness: `SetProcessDPIAware()` called at startup to get correct coordinates
- Frame format: Always BGRA from capture → convert to BGR once → share BGR

## Key Files

| File | Purpose |
|------|---------|
| `src/modules/capture.py` | Capture module, backend selection, `_WGCCamera`, adaptive FPS |
| `main.py` | `timeBeginPeriod(1)` timer setup, cleanup handlers |
