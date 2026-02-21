# Asset Structure & Template Management

Domain knowledge for the organized asset directory, template loading conventions, and how to add new templates to the bot.

## Asset Directory Structure

```
assets/
├── alerts/                  # Sound files
│   ├── ding.mp3
│   ├── rune_appeared.mp3
│   └── siren.mp3
├── app/                     # Application icons
│   ├── funky.ico
│   ├── icon.ico
│   └── icon.png
├── detection/               # Game state detection templates
│   ├── minimap/             # Minimap overlays
│   │   ├── player.png       # Player marker
│   │   ├── rune.png         # Rune marker
│   │   ├── other_player.png # Other player marker
│   │   ├── elite.jpg        # Elite monster marker
│   │   ├── tl_corner.png    # Minimap top-left corner
│   │   ├── br_corner.png    # Minimap bottom-right corner
│   │   ├── tl_corner_back.png
│   │   ├── br_corner_a.png
│   │   ├── br_corner_back.png
│   │   └── br_corner_c.png
│   ├── buffs/               # Buff icon monitoring
│   │   └── rune_buff.jpg    # Rune buff active indicator
│   └── status/              # Status effects
│       ├── exposed.png      # Exposed debuff
│       ├── graveyard.png    # Death/graveyard state
│       ├── petrify_skull.png
│       └── unablemush.png
├── login/                   # Login flow templates
├── models/                  # TF rune solver model
├── pin/                     # PIN entry templates
├── skills/                  # Skill icons for detection
│   ├── demon_frenzy.png
│   └── serpent_screw.png
└── ui/                      # UI element templates
    ├── buttons/             # Generic UI buttons
    │   ├── ok.png
    │   ├── yes.png
    │   ├── add_slots.png
    │   └── exit_cash_storage.png
    ├── cash_shop/           # Cash shop elements
    │   ├── storage_header.png
    │   ├── storage_button.png
    │   ├── inventory_cash_tab.png
    │   ├── pet_selected.png
    │   └── pet_unselected.png
    ├── character/           # Character switch
    │   ├── switch_header.png
    │   └── now_loading.png
    ├── daily/               # Scheduler & daily gift
    │   ├── daily_gift_claim.png
    │   ├── maple_admin.png
    │   ├── scheduler_complete.png
    │   ├── scheduler_header.png
    │   ├── scheduler_icon.png
    │   └── scheduler_text.png
    ├── dialog/              # Popup dialogs
    │   ├── end_chat.png
    │   ├── revive.png
    │   └── unable_cc.png
    ├── game/                # In-game HUD
    │   ├── quickslot/
    │   │   └── menu_icon.png
    │   └── window/
    │       └── ardent/
    │           ├── ardent.png
    │           └── ardent-okay.png
    ├── inventory/           # Inventory UI
    │   ├── header.png
    │   └── expand.png
    ├── menu/                # ESC game menu items
    │   ├── character_header.png
    │   ├── adventure_header.png
    │   ├── cs.png
    │   ├── change_character.png
    │   ├── daily_gift.png
    │   ├── equip.png
    │   ├── guide.png
    │   ├── inventory.png
    │   └── union.png
    ├── navigation/          # World map & bookmarks
    │   ├── bookmark_header.png
    │   ├── bookmark_icon.png
    │   ├── guide_henesys.png
    │   └── world_map_icon.png
    ├── storage/             # Meso storage
    │   ├── icon.png
    │   ├── store_meso.png
    │   ├── max_meso.png
    │   └── no_meso.png
    ├── town/                # Town menu
    │   └── expand_arrow.png
    └── union/               # Union UI
        └── receive_coins.png
```

## Template Loading

### Centralized: `load_template()`

All templates should be loaded via `src/common/templates.py`:

```python
from src.common.templates import load_template, asset_path

# Load grayscale (default — used for matchTemplate)
template = load_template('ui/buttons/ok.png')

# Load color (for color filtering)
template = load_template('detection/minimap/rune.png', grayscale=False)

# Suppress warning if template is optional
template = load_template('detection/buffs/wealth.png', quiet=True)
```

### Module-Level Constants

Templates used frequently should be loaded once at module level:

```python
# In src/game/game_menu.py
from src.common.templates import load_template

MENU_ICON = load_template('ui/game/quickslot/menu_icon.png')
MENU_CS = load_template('ui/menu/cs.png')
```

### Direct cv2.imread (legacy)

Some modules still use `cv2.imread()` directly for color-filtered templates:

```python
# In src/modules/notifier.py — needs color for HSV filtering
rune_filtered = utils.filter_color(cv2.imread('assets/detection/minimap/rune.png'), RUNE_RANGES)
```

These work because the bot always runs from the project root directory.

## Adding a New Template

### Via MCP (live game)

1. Take a screenshot: `capture_screenshot`
2. Identify the region of interest
3. Crop and save: `create_template(name="ui/my_element.png", x=100, y=200, w=50, h=30)`
4. Test match: `test_template(template="ui/my_element.png", threshold=0.8)`
5. Adjust threshold until it matches reliably

### Manual

1. Save PNG to the appropriate `assets/` subdirectory
2. Templates are grayscale by default (saved as grayscale PNGs)
3. Register in the consuming module with `load_template()`
4. For shared/critical templates, add to `src/common/templates.py`
5. Set threshold in `Thresholds` class if non-default

### Naming Conventions

- Lowercase with underscores: `scheduler_header.png`
- Descriptive of what the template shows, not where it's used
- No `_template` suffix (the directory context provides that)
- Use subdirectories for grouping, not filename prefixes

## Thresholds

Centralized in `src/common/templates.py`:

```python
class Thresholds:
    REVIVE: float = 0.75
    ARDENT_BUTTON: float = 0.75
    MENU_ICON: float = 0.90
    END_CHAT: float = 0.75
    UNABLE_CC: float = 0.50
    MAX_MESO: float = 0.80
    RUNE: float = 0.90
    OTHER_PLAYER: float = 0.50
    RUNE_BUFF: float = 0.90
    DEFAULT: float = 0.80
    STRICT: float = 0.95
    LOOSE: float = 0.60
```

## Hot Reload

For iterating on templates without restarting:

```python
from src.common.templates import reload_template
new_template = reload_template('ui/my_element.png')
```

## Key Files

| File | Purpose |
|------|---------|
| `src/common/templates.py` | `load_template()`, `Thresholds`, pre-loaded templates, dHash cache |
| `src/common/utils.py` | `multi_match()` for template detection |
| `assets/` | All template images organized by category |
