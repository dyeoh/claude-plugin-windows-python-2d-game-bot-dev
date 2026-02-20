# Movement & Pathfinding Patterns

Domain knowledge for the movement system: Move (A* path traversal), Adjust (fine-grained positioning), step factories, rope climbing, and velocity prediction.

## Architecture

```
Routine Point (x, y, adjust=True)
  └── Move(x, y)         → A* shortest path → step()/walk per node
        └── Adjust(x, y)  → priority-based micro-walk/rope to exact position
```

| Component | File | Responsibility |
|-----------|------|---------------|
| `Move` | `src/routine/components.py` | Path-based traversal between distant points |
| `Adjust` | `src/routine/components.py` | Fine-grained position adjustment (X and Y) |
| `step()` | Factory-generated | Single movement action (flash jump or teleport) |
| `MovementConfig` | `src/common/movement.py` | Per-class movement parameters |
| Layout | `src/routine/layout.py` | Platform graph, A* shortest path |

## Movement Factories

Choose based on the class's movement skill:

```python
# Flash jump classes (most classes)
step = flash_jump_step_factory(Key.FLASH_JUMP)
step = flash_jump_step_factory(Key.FLASH_JUMP, stage_fright_enabled=False)
step = flash_jump_step_factory(Key.FLASH_JUMP, flash_jump_gap=0.09)  # VM timing

# Teleport classes (Kanna, Bishop, Luminous, Ice/Lightning)
step = teleport_step_factory(Key.TELEPORT)
step = teleport_step_factory(Key.TELEPORT, teleport_presses=2)  # Kanna

# Custom vertical movement (e.g., Shadow Leap)
step = flash_jump_step_factory(Key.FLASH_JUMP, custom_vertical_up=shadow_leap_up)
```

## MovementConfig

```python
@dataclass
class MovementConfig:
    skill_distance: float = 0.12      # Use movement skill above this distance
    walk_threshold: float = 0.7       # Walk fraction (below this × skill_distance)
    brake_after_flash: bool = True    # Hold opposite direction after flash jump
    brake_duration: float = 0.12      # Brake hold time (seconds)
    post_skill_delay: float = 0.12    # Settle time after movement skill
```

## Move System

`Move` handles long-distance traversal between routine points:

1. **A* shortest path** on layout graph (200-iteration cap, visited set)
2. **For each path node**:
   - If distance < `WALK_THRESHOLD` (0.12): continuous walk (`_continuous_walk_to`)
   - If distance >= threshold: call `step()` (flash jump / teleport)
3. **Continuous walk**: Hold direction key with real-time position polling at ~10ms intervals
4. **Edge detection**: Abort walk if Y drops (fell off platform)
5. **Vertical movement**: step('up'/'down') for platform changes

## Adjust System

`Adjust` positions the character at exact `(x, y)` using a priority-based loop:

```
Priority 1: Y Recovery     → If Y drifted (fell off platform)
Priority 2: X Adjustment   → Horizontal micro-walk to target X
Priority 3: Y Adjustment   → Rope up or jump down to target Y
```

### X Micro-Walk (Velocity Prediction)

Walks smoothly with real-time edge detection and momentum compensation:

- **Long distance (>0.06)**: Fast continuous walk, skip probes
- **Medium distance (0.03-0.06)**: Probe with opposite-direction braking, then walk
- **Short distance (<0.03)**: Fine-tuning with small cautious steps

**Velocity prediction** uses EMA estimation to predict where the character will stop after key release:

```python
# Time-adaptive EMA: frame drops get lower weight
alpha = base_alpha * min(1.0, expected_dt / actual_dt)
ema_velocity = alpha * instantaneous_v + (1 - alpha) * ema_velocity

# Predictive stop: release key when predicted stop position passes target
coast_time = calibrated_decel + measured_latency
predicted_stop = current_x + ema_velocity * coast_time
```

### Self-Calibrating Deceleration

Actual coast distance is measured after each predictive stop and used to EMA-update the deceleration estimate. Persists across Adjust calls via class variable `_shared_calibrated_decel`.

### Dead-Band Quantization

One minimap pixel ≈ 0.0042 normalized units. Errors below 1.5× pixel quantum (0.0063) are sub-pixel noise and cannot be corrected. Attempting correction causes oscillation.

## Adjust Class Variables

Override in command book subclasses for per-class behavior:

| Variable | Default | Purpose |
|----------|---------|---------|
| `VERTICAL_METHOD` | `'rope'` | `'rope'` or `'teleport'` |
| `ROPE_KEY` | `'alt'` | Key for rope/ladder |
| `ROPE_CANCEL_KEY` | `None` | Key to cancel rope (None → use ROPE_KEY) |
| `JUMP_KEY` | `'space'` | Key for jumping |
| `_DECEL_TIME` | 0.022s | Initial deceleration coast estimate (self-calibrates) |
| `_MARGIN_VELOCITY_SCALE` | 0.012 | Stop margin per unit velocity |
| `_EMA_ALPHA` | 0.35 | Base EMA smoothing factor |
| `_POSITION_LATENCY` | 0.015s | Initial capture pipeline lag estimate |
| `_PIXEL_QUANTUM` | 0.0042 | One minimap pixel in normalized units |
| `_DEAD_BAND` | 0.0063 | Sub-pixel noise threshold (1.5× quantum) |

### Command Book Patterns

```python
# Standard rope (most classes)
class Adjust(BaseAdjust):
    pass  # Defaults: ROPE_KEY='alt', JUMP_KEY='space'

# Non-default keys
class Adjust(BaseAdjust):
    ROPE_KEY = Key.ROPE    # e.g., 'f'
    JUMP_KEY = Key.JUMP    # e.g., 'x'

# Teleport classes
class Adjust(BaseAdjust):
    VERTICAL_METHOD = 'teleport'
```

## Rope Climbing

`rope_with_monitoring()` in `src/common/movement.py`:

1. Press rope key, start monitoring
2. Direction-aware movement check (only count correct direction)
3. Adaptive polling: 15ms short distances, 30ms standard
4. Position-based cancel when within reach tolerance
5. Timeout cancel with rope key press to dismount

### Rope Failure Recovery

Three-layer recovery when rope fails at certain X positions:

1. **Displacement retry** — Suppress X correction, retry rope from displaced position (up to 3)
2. **Layout-guided search walk** — After 3 local failures, walk to known-good X from layout graph (up to 2 search walks)
3. **Exhaustion** — After 8 total rope failures, accept current Y and stop

## Key Files

| File | Purpose |
|------|---------|
| `src/routine/components.py` | Adjust, Move, base classes |
| `src/common/movement.py` | MovementConfig, factories, rope_with_monitoring |
| `src/routine/layout.py` | Platform graph, A* shortest path |
| `src/common/utils.py` | distance(), position utilities |
| `src/common/timing.py` | Performance profiles, adaptive_sleep |
| `docs/movement.md` | Movement system documentation |
