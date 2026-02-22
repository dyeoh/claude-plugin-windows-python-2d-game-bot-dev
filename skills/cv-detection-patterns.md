# Computer Vision & Detection Patterns

Domain knowledge for template matching, color filtering, frame differencing, and detection pipelines in Python 2D game bots.

## Template Matching Pipeline

The standard pipeline for detecting game UI elements and objects:

```
1. ROI Crop        → Never search full frame
2. dHash Pre-Screen → Skip unchanged frames (optional, for high-frequency checks)
3. matchTemplate   → cv2.TM_CCOEFF_NORMED
4. Threshold       → Per-template confidence cutoff
5. NMS             → Non-Maximum Suppression for overlapping detections
6. Sub-Pixel       → Parabolic interpolation for position-critical detection
```

### Core Function: `multi_match()`

Located in `src/common/utils.py`. Used across all modules for template detection.

```python
multi_match(frame, template, threshold=0.8, roi=None)
```

- Always provide ROI to avoid full-frame search
- Returns list of (x, y) match positions
- Applies NMS internally when multiple matches found

### ROI Guidelines

| Detection Target | ROI Source | Typical Size |
|-----------------|-----------|-------------|
| Minimap elements | Minimap bounds from capture | ~250x200 px |
| UI dialogs | Fixed screen region | ~400x300 px |
| Menu elements | Known menu location | ~200x100 px |
| Player marker | Minimap crop | ~250x200 px |
| Rune arrows | Fixed puzzle region | `frame[143:371, 381:957]` |

### Threshold Guidelines

| Confidence | Use When |
|-----------|----------|
| 0.85-0.95 | Clean, high-contrast, static UI elements |
| 0.75-0.85 | Standard detection (most game elements) |
| 0.65-0.75 | Variable backgrounds, small templates, animations |
| < 0.65 | Avoid — too many false positives |

Thresholds are stored centrally in `src/common/templates.py`.

## dHash Frame Differencing

64-bit perceptual hash for detecting unchanged frames. Essential for high-frequency checks.

```python
# Compute hash
hash_current = dhash(roi_crop)

# Compare (hamming distance)
diff = bin(hash_current ^ hash_previous).count('1')
if diff < threshold:  # typically 5-10 bits
    skip  # Frame hasn't changed enough
```

When to use:
- Rune detection (checked every frame)
- Other-player detection (checked every frame)
- Any detection that runs more than once per second

When NOT to use:
- One-time detection (revive dialog check after death)
- Template matching that already has a cooldown guard

## Color Filtering

For detecting color-coded game elements (rune symbols, player markers):

```python
# Convert to HSV
hsv = cv2.cvtColor(roi, cv2.COLOR_BGR2HSV)

# Apply mask (handle hue wrapping for red)
mask = cv2.inRange(hsv, lower_bound, upper_bound)

# Clean up noise
kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (3, 3))
mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel)

# Find contours
contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
```

Common HSV ranges in the codebase:
- **Rune symbol**: Orange-red hue range on minimap
- **Other players**: Red marker on minimap
- **Rune arrows**: Orange-to-green range for arrow detection

## Sub-Pixel Refinement

For position-critical detection (player tracking on ~238px minimap):

```python
def _subpixel_refine(result, py, px):
    # Fit parabola to 3-point correlation neighborhood
    left, center, right = result[py, px-1], result[py, px], result[py, px+1]
    denom = 2.0 * (left - 2.0 * center + right)
    sub_x = px - (left - right) / denom
    # Same for Y axis
```

- One minimap pixel ≈ 0.0042 normalized units
- Sub-pixel gives ~0.1-0.3 pixel accuracy
- Critical for movement system (dead-band is 1.5 pixels ≈ 0.0063)

## Rune Solver Backends

| Backend | Method | Dependencies | Accuracy |
|---------|--------|-------------|----------|
| API | POST screenshot to solver service | External service at `172.16.11.230:8001` | Highest |
| Local CV | 4-strategy cascade | OpenCV only | Good |
| TF Model | HSV filter → Canny → inference | TensorFlow | Good (legacy) |

### Local CV 4-Strategy Cascade

The local solver tries four independent strategies in order, advancing when fewer than 4 arrows are detected:

1. **Hue gradient** — HSV hue channel gradient analysis for arrow direction
2. **Brightness gradient** — Grayscale intensity gradient for arrow tips
3. **Contour moment** — Shape contour analysis with cv2.moments()-based direction
4. **Edge hull** — Canny edge detection with convex hull direction vectors

Each strategy independently identifies arrow directions in the crop region `frame[143:371, 381:957]`. The cascade provides resilience against variable backgrounds and lighting.

### Spinning Runes

A spinning rune (rotating instead of stationary) indicates the game server has detected bot-like behavior. Usually caused by:
- Virtual input methods instead of Interception driver
- Suspicious movement patterns (perfect timing, identical paths)
- Being in the same channel too long without natural behavior variance

## MCP Template Testing

When MCP tools are available, test detection pipelines against live game state:

```
test_template(template="ui/game/death_ui.png", threshold=0.8)  # Test match
create_template(name="ui/new_element.png", x, y, w, h)         # Capture new template
capture_screenshot(region="381,143,576,228")                     # Capture rune region
list_templates(subdir="ui/game")                                 # Browse templates
```

Use `test_template` to validate threshold choices across different maps and game states.

## Performance Rules

1. **Never full-frame search** — always provide ROI
2. **Cheap checks first** — cooldown timestamp before template match
3. **One capture per loop** — share frame via `config.capture.frame`
4. **Copy before processing** — `frame.copy()` for thread safety
5. **dHash for high-frequency** — skip unchanged minimap regions
6. **BGRA→BGR once** — convert at capture time, share BGR result

## Adding New Templates

1. Save template in organized `assets/` subdirectory:
   - `assets/detection/minimap/` — minimap overlays (player, rune, corners)
   - `assets/detection/buffs/` — buff icon monitoring
   - `assets/detection/status/` — status effects (exposed, graveyard, petrify)
   - `assets/ui/buttons/` — UI buttons (ok, yes, add_slots)
   - `assets/ui/menu/` — ESC game menu items
   - `assets/ui/dialog/` — popup dialogs (revive, end_chat, unable_cc)
   - `assets/ui/cash_shop/` — cash shop elements
   - `assets/ui/navigation/` — world map, bookmarks
   - `assets/ui/storage/` — storage UI elements
   - `assets/ui/daily/` — scheduler, daily gift
   - `assets/ui/inventory/` — inventory headers
   - `assets/ui/character/` — character switch
   - `assets/ui/town/` — town menu
   - `assets/ui/union/` — union UI
   - `assets/skills/` — skill icons for detection
   - `assets/ui/game/quickslot/` — quickslot HUD elements
2. Register in `src/common/templates.py` with threshold value
3. Use via `multi_match(frame, template, threshold, roi=region)`
4. Consider dHash pre-screen if detection runs every frame

## Stray Window Detection

Detects when a game window/dialog is blocking input by checking for menu icon absence:

```python
# In notifier.py — 3 consecutive misses trigger ESC press
menu_icon = multi_match(frame, MENU_ICON, threshold=0.90, roi=quickslot_roi)
if not menu_icon:
    miss_count += 1
    if miss_count >= 3:
        press('esc')
        miss_count = 0
```

The menu icon (quickslot area) is always visible during normal gameplay. Its absence indicates a window/dialog is open. This handles:
- ESC menu stuck open after failed operations
- Post-revive cleanup (ESC cascade with menu icon verification)
- Unexpected dialog popups

## Buff Icon Detection

For monitoring active buffs (wealth potion, union meso, EXP):

1. Capture buff icons from the buff bar (top-right area) using `create_template`
2. Save to `assets/detection/buffs/` (e.g., `wealth.png`, `union_meso.png`)
3. Check ROI of buff bar area after pressing potion key
4. If buff icon not found after press → potion depleted

Templates: `assets/detection/buffs/rune_buff.jpg` (existing), add wealth/union/exp as needed.

## Key Files

| File | Purpose |
|------|---------|
| `src/common/utils.py` | `multi_match()`, `_subpixel_refine()`, NMS, color filtering |
| `src/common/templates.py` | Template images and threshold configs |
| `src/modules/capture.py` | Capture pipeline, frame sharing |
| `src/modules/rune_solver.py` | Rune solver backends |
| `src/modules/notifier.py` | Detection consumers |
| `src/detection/detection.py` | TF model pipeline |
