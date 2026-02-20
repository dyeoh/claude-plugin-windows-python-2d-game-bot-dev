# Input & Key Management Patterns

Domain knowledge for keyboard/mouse simulation, key safety, mobbing keys, cursor movement, and click verification in Python 2D game bots.

## Key Safety Rules (CRITICAL)

These rules prevent the most dangerous failure mode — hung keys that cause the character to walk off-screen.

1. **Every `key_down()` MUST have a matching `key_up()`**
2. **Release keys in `finally` blocks** — exception safety
3. **Never hold direction keys across function boundaries** without try/finally
4. **Use `press()` for single key presses** — it handles down/up automatically
5. **Avoid conflicting simultaneous key presses**

### Pattern: Safe Key Hold

```python
key_down('left')
try:
    do_flash_jump(Key.FLASH_JUMP)
    press(Key.ATTACK, 1)
finally:
    key_up('left')
```

### Pattern: Direction Update Without Double-Tap

Some classes (e.g., Mechanic) trigger a dash on double-tap. This pattern avoids that by only changing keys when the direction actually changes:

```python
def _update_direction(self, new_direction):
    if self._held_direction != new_direction:
        if self._held_direction:
            key_up(self._held_direction)
        if new_direction:
            key_down(new_direction)
        self._held_direction = new_direction
```

### Pattern: Safe Vertical Movement

```python
def _move_down(self):
    try:
        key_down('down')
        time.sleep(timing.get('input_registration'))
        press(Key.JUMP, 1, down_time=0.1)
    finally:
        key_up('down')
```

## Interception Driver

Kernel-level input simulation via `pyinterception`. Input appears as physical hardware events — below the Windows input stack.

### Key Functions

| Function | Purpose |
|----------|---------|
| `key_down(key)` | Press and hold a key |
| `key_up(key)` | Release a held key |
| `press(key, n, down_time)` | Press key n times with specified hold duration |
| `click(x, y)` | Click at screen position |
| `operational_press(key)` | Press during non-combat operations |
| `operational_click(x, y)` | Click during non-combat operations |

### Extended Key Scan Codes

Extended keys (arrows, Insert, Delete, Home, End, Page Up/Down, numpad Enter) use the `KEY_E0` flag:

```python
# Scan codes > 1024 indicate extended keys
if scan_code > 1024:
    actual_scan = scan_code - 1024
    flags |= interception.KEY_E0
```

## Mobbing Key Registry

For classes that hold keys during normal combat (e.g., Wild Hunter's arrow blast, Lynn's mobbing attack). The registry ensures these keys are safely released before operations that conflict (buffs, rune solving, adjust).

### Registration

```python
# Module-level in command book file
MOBBING_KEYS = [Key.MAIN_ATTACK]
```

### API

| Function | Purpose |
|----------|---------|
| `hold_mobbing_key(key)` | Start holding a mobbing key |
| `release_mobbing_key(key)` | Stop holding a specific mobbing key |
| `release_all_mobbing_keys()` | Release all held mobbing keys (returns True if any released) |
| `restore_all_mobbing_keys()` | Re-press all previously held mobbing keys |

### Nesting Safety

Supports nested release/restore. Keys are only physically released on first `release_all` and re-pressed on last `restore_all`:

```python
release_all_mobbing_keys()  # Physical release
try:
    release_all_mobbing_keys()  # No-op (already released)
    try:
        # ... inner operation
    finally:
        restore_all_mobbing_keys()  # No-op (outer still active)
finally:
    restore_all_mobbing_keys()  # Physical re-press
```

### Auto-Release Triggers

Mobbing keys are automatically released by `Command.execute()` for all ACTION_TYPEs except RAW, FLASH_JUMP, and DROP.

## Smooth Cursor Movement

Bezier curve interpolation for human-like cursor movement (`src/common/smooth_cursor.py`).

### Presets

| Preset | Duration | Steps | Easing | Use When |
|--------|---------|-------|--------|----------|
| `DEFAULT` | 0.15-0.35s | 15-40 | ease-out-quad | Standard movement |
| `FAST` | 0.08-0.20s | 10-25 | ease-out-cubic | Quick UI interactions |
| `PRECISE` | 0.20-0.40s | 20-50 | ease-in-out-quad | Precise clicking |

### Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `duration_range` | (0.15, 0.35) | Min/max move duration |
| `steps_range` | (15, 40) | Interpolation points |
| `control_point_variance` | 0.3 | Bezier curve randomness |
| `jitter_amplitude` | 0.5 | Pixel-level noise |
| `end_jitter` | 1.5 | Final position variation |

## Click Verification

Pixel-diff detection to verify mouse clicks actually registered (`src/common/click_verify.py`).

```python
verified_interception_click(x, y, config=None)
```

1. Capture 80x80 ROI around target before click
2. Perform click
3. Wait, capture ROI after click
4. Compute MAD (Mean Absolute Difference) excluding 12px cursor region
5. If MAD > 6.0 → click registered
6. If MAD < 6.0 → retry (up to MAX_RETRIES)

Used for: PIN buttons, pet storage/retrieval, tab selection, menu interactions.

## Timing Helpers

From `src/common/command_timing.py`:

| Helper | Approximate Value | Purpose |
|--------|------------------|---------|
| `t.sleep_fast()` | ~35ms | Shortest meaningful delay |
| `t.sleep_normal()` | ~100ms | Standard operation delay |
| `t.skill_delay()` | ~40ms | Key hold for skill activation |
| `t.buff_delay()` | ~150ms | Key hold for buff activation |
| `t.key_hold(base)` | scaled base | Scaled key hold time |

From `src/common/timing.py`:

| Key | Value | Purpose |
|-----|-------|---------|
| `input_registration` | 50ms | Game input registration delay |
| `position_update` | 30ms | Position update after movement |
| `frame_stabilize` | 100ms | Frame stabilization wait |

## Key Files

| File | Purpose |
|------|---------|
| `src/common/vkeys.py` | Key press/release, mobbing key registry, scan codes |
| `src/common/key_state.py` | `GetAsyncKeyState` polling |
| `src/common/smooth_cursor.py` | Bezier curve cursor movement |
| `src/common/click_verify.py` | ROI pixel-change click verification |
| `src/common/command_timing.py` | Timing helpers |
| `src/common/timing.py` | Fixed timing constants |
| `docs/input.md` | Input system documentation |
