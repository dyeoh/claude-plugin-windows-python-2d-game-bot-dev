# API Server & MCP Dev Bridge Patterns

Domain knowledge for the embedded Bot API server, MCP Claude Code bridge, structured event system, and template management used for development and debugging.

## Architecture

```
Claude Code <--stdio--> MCP Server <--HTTP/WS--> Bot API Server (in-process)
                                                      |
                                                config.capture.frame
                                                config.state_manager
                                                vkeys.key_down/up
                                                etc.

Future:  MagicMirror <--WS--> Rust Relay <--WS--> Bot API Server
```

Three layers:
1. **Bot API Server** (`src/modules/api_server.py`) — aiohttp HTTP+WS server on daemon thread, port 8377. Direct access to all `config.*` internals. Localhost only.
2. **MCP Server** (`mcp_server.py`) — Separate Python process launched by Claude Code. Thin wrapper: MCP tool calls -> HTTP requests to Bot API.
3. **Event System** (`src/common/events.py`) — Thread-safe EventBus, JSONL logging, WebSocket broadcast.

## Bot API Server

Embedded aiohttp server running on its own asyncio event loop in a daemon thread. Zero impact on bot loop or tkinter.

### Endpoints

| Route | Method | Purpose |
|---|---|---|
| `/status` | GET | Bot state, player position, routine info, capture status |
| `/screenshot` | GET | Current frame as JPEG (`?region=x,y,w,h`, `?quality=80`) |
| `/minimap` | GET | Minimap image as PNG |
| `/logs` | GET | Last N lines from session log (`?lines=50`) |
| `/routine` | GET | Current routine as JSON (points and commands) |
| `/templates` | GET | List template files in `assets/` (`?subdir=ui/game`) |
| `/config` | GET/POST | Read or set runtime config values |
| `/input/key` | POST | Send key press via Interception driver |
| `/input/click` | POST | Send mouse click at screen coordinates |
| `/input/move` | POST | Move cursor to screen coordinates |
| `/focus` | POST | Bring game window to foreground |
| `/template/create` | POST | Crop region from live frame, save to `assets/` |
| `/template/match` | POST | Test template against current frame |
| `/bot/toggle` | POST | Toggle bot enabled/disabled |
| `/bot/reload` | POST | Reload routine CSV and recalibrate |
| `/ws` | WS | Real-time event stream (state changes, logs, position) |

### Thread Safety

- Frame access reads `config.capture.frame` (numpy reference swap is atomic under GIL)
- Input endpoints use `vkeys.press()` / `vkeys.key_down()` / `vkeys.key_up()` from the aiohttp thread
- WebSocket broadcast uses `asyncio.run_coroutine_threadsafe()` for cross-thread event delivery
- Bounded WS queue — drops oldest if client falls behind

### Key Design Patterns

```python
# Broadcasting events from any thread
api_server.broadcast_event('state_change', {'old': 'Idle', 'new': 'Enabled'})

# Broadcasting log lines (called from GUI log writer)
api_server.broadcast_log(formatted_line)

# Subscribing event bus to WS broadcast
from src.common.events import event_bus
event_bus.subscribe(lambda e: api_server.broadcast_event(e['type'], e['data']))
```

### Startup

```python
# In main.py, after Capture and before GUI
from src.modules.api_server import BotAPIServer
api_server = BotAPIServer()
config.api_server = api_server
api_server.start()
```

## MCP Server (Claude Code Bridge)

Thin stdio MCP server that translates Claude Code tool calls into HTTP requests to the bot API.

### Configuration (`.mcp.json`)

```json
{
    "mcpServers": {
        "funkytown": {
            "command": "python",
            "args": ["mcp_server.py"],
            "cwd": "c:/Users/Darren/Desktop/funkytown"
        }
    }
}
```

### MCP Tools (ranked by dev value)

| Tool | Purpose | API Route |
|------|---------|-----------|
| `capture_screenshot` | See what the bot sees | GET `/screenshot` |
| `read_logs` | Real-time log monitoring | GET `/logs` |
| `test_template` | Check template matching live | POST `/template/match` |
| `create_template` | Crop from live game frame | POST `/template/create` |
| `get_bot_status` | Player position, state, routine | GET `/status` |
| `send_key` / `send_click` | Test interactions | POST `/input/key`, `/input/click` |
| `move_cursor` | Move cursor without clicking | POST `/input/move` |
| `focus_game_window` | Bring game to foreground | POST `/focus` |
| `get_minimap` | Current minimap state | GET `/minimap` |
| `toggle_bot` / `reload_routine` | Remote control | POST `/bot/toggle`, `/bot/reload` |
| `get_routine` | Routine structure | GET `/routine` |
| `list_templates` | Browse `assets/` | GET `/templates` |
| `set_config` | Runtime config changes | POST `/config` |
| `read_file` / `write_file` | Read/write files on remote VM | GET/POST `/file` |
| `restart_bot` | Gracefully restart bot process | POST `/restart` |
| `run_action` | Execute game controller action | POST `/action` |
| `git_pull` / `git_status` | Git operations on remote VM | GET/POST `/git/*` |

### Dependencies

- `mcp>=1.0` — MCP SDK for stdio server
- `httpx>=0.27` — Async HTTP client for MCP -> Bot API calls

## Run Actions

The `run_action` tool executes high-level game controller actions that may take 10-60 seconds.

| Action | Purpose |
|--------|---------|
| `store_meso` | Open storage, deposit meso |
| `store_pets` / `retrieve_pets` | Pet management via cash shop |
| `swap_to_first` / `swap_character` | Character switching |
| `start_dailies` / `complete_dailies` | Daily task automation |
| `collect_union_coins` | Union coin collection |
| `travel_henesys` / `travel_to_bookmark` | Map navigation |
| `ensure_ardent_closed` | Close Ardent Mill if open |

## Structured Event System

Thread-safe EventBus for machine-readable logging alongside existing `print()` output.

### EventBus API

```python
from src.common.events import event_bus

# Emit (from any thread)
event_bus.emit('rune_detected', {'x': 0.5, 'y': 0.3})
event_bus.emit('state_change', {'old': 'Idle', 'new': 'Enabled'})

# Subscribe (e.g., API server WebSocket broadcast)
event_bus.subscribe(callback)

# Query recent events
recent = event_bus.query(event_type='rune_detected', limit=10)
```

### Storage

- **Ring buffer** (1000 events) — in-memory, queryable via API
- **JSONL log** (`logs/events.jsonl`) — persistent, one JSON object per line
- **WebSocket broadcast** — real-time push to connected clients

### Events Emitted

| Source | Event Type | Data |
|--------|-----------|------|
| `bot_state.py` | `state_change` | `{old, new}` |
| `notifier.py` | `death`, `rune`, `player` | Detection details |
| `bot.py` | `routine_step`, `rune_solve` | Step/solve details |
| `capture.py` | `calibration`, `backend_switch` | Backend info |
| `cash_shop.py` | `cash_shop_enter/exit` | Operation details |

## Template Management

### Creating Templates (via MCP)

```
1. capture_screenshot → see the game
2. create_template(name="ui/my_button.png", x=100, y=200, w=50, h=30) → crop and save
3. test_template(template="ui/my_button.png", threshold=0.8) → verify match
```

### Hot Reload

```python
from src.common.templates import reload_template
reload_template('ui/my_button.png')  # Reload without restart
```

## Window Close Verification

Game windows now close with verification instead of blind ESC presses.

### Pattern: Verified Close

```python
# WindowInterface.close_with_verification()
# Presses ESC, checks is_open() after each press, stops when confirmed closed

window.close_with_verification(max_presses=5)
```

### Pattern: Cascade Cleanup

```python
# Module-level utility for multiple windows
from src.common.ui_interfaces import close_all_windows

close_all_windows([cash_shop, ardent_mill, game_menu], max_rounds=3)
```

## Remote Code Push Pattern

For rapid iteration, push code directly to the VM without git:

```python
# Push file to VM
write_file(path="src/modules/bot.py", content="...")

# Restart to pick up changes
restart_bot(reason="Updated bot.py")

# IMPORTANT: Direct writes dirty the VM working tree, blocking git pull --rebase.
# To sync git after direct pushes, commit on dev machine, then on VM:
#   git stash && git pull --rebase && git stash drop
```

### HWID Persistence Pattern

Interception device IDs can shift across restarts (especially on Hyper-V VMs where device enumeration order changes). The persistence system:

1. `save_devices()` in `src/gui/settings/devices.py` saves device IDs + HWIDs to `.settings/devices.json`
2. `_apply_frozen_devices()` loads saved IDs at startup, verifies HWIDs match, rescans if mismatch
3. **Critical**: pyinterception may be installed as `.egg` in site-packages (not from project tree). Custom functions in `pyinterception/src/interception/inputs.py` won't be available. The fix in `devices.py` is self-contained — uses only basic `interception.inputs` globals.

## Key Files

| File | Purpose |
|------|---------|
| `src/modules/api_server.py` | Embedded aiohttp HTTP+WS server |
| `mcp_server.py` | MCP stdio server (Claude Code bridge) |
| `.mcp.json` | Claude Code MCP configuration |
| `src/common/events.py` | EventBus, JSONL logging, ring buffer |
| `src/common/templates.py` | Template registry + `reload_template()` |
| `src/common/ui_interfaces.py` | `close_with_verification()`, `close_all_windows()` |
