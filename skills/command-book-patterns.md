# Command Book Development Patterns

Domain knowledge for creating and modifying command book skills in Python 2D game bots.

## Quick Reference: Creating a New Skill

### Step 1: Choose Base Class

| If the skill... | Use base class | Set these variables |
|-----------------|---------------|-------------------|
| Flash jumps + attacks | `JumpAttack` | `ATTACK_KEY`, `FJ_KEY` |
| Flash jumps only | `FlashJump` | `FJ_KEY` |
| Attacks in a direction | `DirectionalAttack` | `ATTACK_KEY` |
| Teleports | `Teleport` | `JUMP_KEY` |
| Has a cooldown | `CooldownSkill` | `SKILL_KEY`, `COOLDOWN` |
| Drops through platform | `JumpDown` | `JUMP_KEY` |
| Presses up | `PressUp` | — |
| Jumps up (teleport) | `UpJump` | `JUMP_KEY` |
| Holds a key for mobbing | `KeyDownMob` | `MOB_KEY` |
| Releases mobbing key | `KeyUpMob` | `MOB_KEY` |
| Needs full manual control | `Command` | `ACTION_TYPE = ActionType.RAW` |

All base classes are in `src/routine/components.py`.

### Step 2: Choose ACTION_TYPE

Inherited from base class — only override if behavior differs from the base.

| Category | Pre-Settle | Post-Delay | Grounded | Use When |
|----------|-----------|------------|----------|----------|
| `SKILL` | 15ms | 150ms | No | Generic skill cast (default) |
| `ATTACK` | 15ms | 120ms | No | Stationary attack, key-hold |
| `BUFF` | 15ms | 400ms | No | Buff activation, long animation |
| `JUMP_ATTACK` | 0ms | 350ms | **Yes** | Flash jump + attack mid-air |
| `FLASH_JUMP` | 0ms | 120ms | **Yes** | Flash jump movement only |
| `SUMMON` | 50ms | 300ms | No | Summon/placement skill |
| `COMBO` | 15ms | 200ms | No | Multi-input sequence |
| `DROP` | 15ms | 100ms | No | Platform drop-down |
| `UTILITY` | 10ms | 100ms | No | Non-combat (potions, toggles) |
| `RAW` | 0ms | 0ms | No | Full manual control |

All timing values are base values scaled by performance profile (1.0x bare metal → 3.5x VM). ±15% jitter applied automatically for anti-detection.

### Step 3: Override POST_DELAY (if needed)

Only override when the default for your ACTION_TYPE is wrong:

```python
class SlowSkill(CooldownSkill):
    SKILL_KEY = Key.MY_SKILL
    COOLDOWN = 60
    POST_DELAY = 0.50  # Longer than default SUMMON 0.30s
```

### Step 4: Override Hooks (if needed)

JumpAttack provides three hooks for class-specific behavior:

```python
class JumpAttack:
    def _pre_flash_jump(self):
        """Hook: after direction held, before flash jump."""
        t.sleep_fast()  # default

    def _after_flash_jump(self):
        """Hook: after flash jump, before attack. E.g., direction change."""
        pass  # default

    def _do_flash_jump(self):
        """Hook: complete FJ override. E.g., manual press timing."""
        do_flash_jump(self.FJ_KEY, presses=self.FJ_PRESSES, gap=self.FJ_GAP)
```

### Step 5: Register Mobbing Keys

If the class holds keys during normal combat (keydown attack classes):

```python
# Module-level in command book file — auto-released during buffs/rune/adjust
MOBBING_KEYS = [Key.MAIN_ATTACK]
```

## Execution Flow

```
Command.execute()
  ├── Check cooldown (skip if not ready)
  ├── Pre-settle delay OR grounded check (per ACTION_TYPE)
  ├── Release mobbing keys (if ACTION_TYPE not RAW/FLASH_JUMP/DROP)
  ├── Call main()  ← subclass implements this
  ├── Restore mobbing keys
  └── Post-delay with ±15% jitter
```

## Templates

### Minimal Skill

```python
class MySkill(JumpAttack):
    """Jump + MySkill mid-air."""
    ATTACK_KEY = Key.MY_SKILL
    FJ_KEY = Key.FLASH_JUMP
```

### Skill with Follow-Up

```python
class JumpSplashF(JumpAttack):
    """Flash jump + Splash-F with Homing Missile follow-up."""
    ATTACK_KEY = Key.SPLASH_F
    FJ_KEY = Key.FLASH_JUMP
    FOLLOW_UP_KEY = Key.HOMING_MISSILE
    FJ_GAP = 0.02
```

### Skill with Custom Flash Jump

```python
class JumpShowdown(JumpAttack):
    """Flash jump + Showdown with manual FJ timing."""
    ATTACK_KEY = Key.SHOWDOWN
    FJ_KEY = Key.FLASH_JUMP

    def _pre_flash_jump(self):
        t.sleep_normal()

    def _do_flash_jump(self):
        press(Key.FLASH_JUMP, 1, down_time=0.22)
        t.sleep_fast()
        press(Key.FLASH_JUMP, 1, down_time=0.22)
```

### Cooldown Skill

```python
class ErdaShower(CooldownSkill):
    """Erda Shower (56s cooldown)."""
    SKILL_KEY = Key.ERDA_SHOWER
    COOLDOWN = 56
```

### Buff with Multiple Skills

```python
class Buff(Command):
    """Buff management with cooldown tracking."""
    ACTION_TYPE = ActionType.RAW

    def __init__(self):
        super().__init__(locals())
        self.buff_time = 0

    def main(self):
        now = time.time()
        if self.buff_time == 0 or now - self.buff_time > settings.buff_cooldown:
            press(Key.BUFF_1, 2, down_time=t.buff_delay())
            press(Key.BUFF_2, 2, down_time=t.buff_delay())
            self.buff_time = now
```

### New Command Book (Skeleton)

```python
import time
from src.common import config, settings, utils
from src.common.action_types import ActionType
from src.common.vkeys import press, key_down, key_up
from src.common.command_timing import t
from src.common.movement import flash_jump_step_factory, rope_with_monitoring
from src.routine.components import (
    Command, Adjust as BaseAdjust,
    FlashJump as BaseFlashJump, JumpAttack, DirectionalAttack,
    CooldownSkill, JumpDown as BaseJumpDown,
)

class Key:
    JUMP = 'space'
    FLASH_JUMP = 'space'
    ROPE = 'c'
    MAIN_ATTACK = 'a'
    # ... add keybindings

step = flash_jump_step_factory(Key.FLASH_JUMP)

class Adjust(BaseAdjust):
    def _move_up(self, target_y=None):
        rope_with_monitoring(Key.ROPE, target_y, cancel_key=Key.ROPE)

class FlashJump(BaseFlashJump):
    FJ_KEY = Key.FLASH_JUMP

# ... add skills
```

## Key Files

| File | Purpose |
|------|---------|
| `src/routine/components.py` | 11 base classes, Command.execute(), Adjust, Move |
| `src/common/action_types.py` | ActionType enum, ACTION_TIMING table |
| `resources/command_books/_template.py` | Template for new command books |
| `resources/command_books/*.py` | Per-class skill definitions (16 files) |
| `docs/command-books.md` | Command book documentation |
