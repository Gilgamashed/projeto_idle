# MMORPG-Inspired Idle Game: Technical Architecture & Development Roadmap

---

## Executive Summary & Core Architectural Principle

As a Python developer transitioning into game development from a tabletop background, you have a massive advantage: **tabletop games and idle MMORPGs share the exact same foundation—clear mathematical rules, turn/tick resolution, state transitions, and deterministic mechanics.**

In idle game development, the most vital rule is:
> **Decouple the Engine (Data & Simulation) completely from the View (UI/Renderer).**

By keeping all game logic (ticks, combat, character stats, inventory, offline progression) in pure, headless Python classes, you can:
1. Run automated unit tests on combat balance in milliseconds.
2. Simulate hours of offline grinding instantly using fast-forward loops.
3. Switch or upgrade UI frameworks (from a Terminal TUI to Pygame, Arcade, or Web UI) without rewriting a single line of game logic.

```
+-----------------------------------------------------------------------+
|                             USER INTERFACE                            |
|        (Main Menu -> Character Select -> Hub Screen -> Grind Area)    |
+-----------------------------------------------------------------------+
                                   |
                          (Actions & Render Calls)
                                   v
+-----------------------------------------------------------------------+
|                              GAME ENGINE                              |
|   +-----------------------+ +--------------------+ +---------------+  |
|   |   Tick / Loop Manager  | |   Combat Resolver  | | Offline Sim   |  |
|   +-----------------------+ +--------------------+ +---------------+  |
+-----------------------------------------------------------------------+
                                   |
                            (State Mutation)
                                   v
+-----------------------------------------------------------------------+
|                            DATA / SAVE LAYER                          |
|   +---------------------------------------------------------------+   |
|   | Account Data (12 Character Slots + Global Achievements)       |   |
|   | Serialization / JSON Save Manager                             |   |
|   +---------------------------------------------------------------+   |
+-----------------------------------------------------------------------+
```

---

# Part 1: Step-by-Step Progress Roadmap

This roadmap takes you from zero to a fully functioning multi-character idle prototype with save persistence.

---

### Phase 0: Project Setup & Data Modeling (The Foundation)
*Before drawing any screens, define the shape of your data.*

1. **Project Directory Layout**:
   ```text
   idle_game/
   ├── main.py                     # Entry point
   ├── core/
   │   ├── config.py               # Constants (tick rates, max slots = 12)
   │   ├── engine.py               # Main tick orchestrator
   │   ├── combat.py               # Combat calculation formulas
   │   └── offline.py              # Offline time catch-up resolver
   ├── models/
   │   ├── account.py              # Account-wide data & 12 slots
   │   ├── character.py            # Character stats, inventory, active state
   │   ├── monster.py              # Monster templates & instances
   │   └── zone.py                 # Safe hub & grinding zones
   ├── storage/
   │   └── save_manager.py         # JSON serializer/deserializer with atomic writes
   ├── ui/
   │   ├── main_menu.py
   │   ├── character_select.py
   │   ├── character_create.py
   │   ├── hub_view.py
   │   └── grinding_view.py
   └── saves/
       └── account_save.json
   ```

2. **Data Model Definitions (`models/` using `dataclasses`)**:
   - `Account`: Stores global metadata, timestamps, total playtime, global unlocked achievements, and a fixed array/list of 12 `CharacterSlot` instances.
   - `Character`: Stores `id`, `name`, `class_name` ("Novice"), `level`, `current_exp`, `allocated_stats`, `inventory`, `equipped_gear`, `current_zone_id`, `is_idling`, and `last_tick_timestamp`.
   - `Zone`: Stores zone type (`SAFE_HUB` vs `HUNTING_GROUND`), monster encounter tables, drop tables, and level requirements.

3. **Save System Implementation (`storage/save_manager.py`)**:
   - Implement `save_account(account: Account, path: str)` using atomic writes (write to `.tmp` file first, then replace).
   - Implement `load_account(path: str) -> Account`: Initializes a default 12-slot empty account if no save exists.
   - Version tag inside the save file (e.g., `"version": 1`) for smooth future schema migrations.

---

### Phase 1: Main Menu & Account Initialization
*Goal: A boot screen that initializes the account data and presents primary pathways.*

1. **State Flow**:
   ```text
   App Launch ──> Load/Verify Save File ──> Render Main Menu
                                              ├── [1] Play (Go to Character Select)
                                              ├── [2] Account Achievements
                                              ├── [3] Settings / Backup Save
                                              └── [4] Exit Game
   ```
2. **Key Tasks**:
   - On launch, load `account_save.json`. If missing, auto-create a clean account with 12 empty slots.
   - Display account summary header: Total Account Level, Global Achievement count, Active characters count.
   - Handle navigation input cleanly with a unified UI State Machine (`MAIN_MENU`, `CHAR_SELECT`, `CHAR_CREATE`, `HUB`, `GRIND`).

---

### Phase 2: Character Select Menu (12 Save Slots)
*Goal: A visual grid or list displaying all 12 slots, their status, and actionable controls.*

1. **Slot Representation**:
   - **Empty Slot**: Displays `[Slot #X: Empty]` + button/prompt `[Create Character]`.
   - **Populated Slot**: Displays summary card:
     - Character Name & Title
     - Class (`Novice`) & Level
     - Current Location (e.g., `Town Square` or `Slime Meadows [Idling]`)
     - HP / EXP Progress bar preview
     - Last active timestamp
2. **User Actions per Slot**:
   - **Select / Play**: Loads active character context into the engine and transitions to their current zone (Hub or Grinding).
   - **Delete**: Requires double-confirmation (e.g., typing character name or confirmation prompt). Empties the slot without corrupting other slots.
   - **Inspect**: Shows quick gear and attribute overview without entering the game loop.

---

### Phase 3: Character Creation Menu
*Goal: An interactive form to name and configure a new Novice character.*

1. **Creation Flow**:
   ```text
   Select Empty Slot ──> Enter Name ──> Allocate Starter Stats ──> Confirm ──> Save & Launch Hub
   ```
2. **Form Elements & Validation**:
   - **Name Input**:
     - Length constraints (e.g., 3–16 characters).
     - Character set restrictions (letters, numbers, underscores).
     - Unique check across the other 11 slots on the account.
   - **Class Picker**: Defaults to `"Novice"` (future-proofed to display locked tier-1 branches like Warrior/Mage/Rogue).
   - **Starter Stat Allocation**: Give the player an initial pool of unspent points (e.g., 10 points to distribute among STR, AGI, INT, VIT).
   - **Confirmation Action**: Instantiates `Character`, places it into the target slot in `Account`, executes immediate save to disk, and transitions to the Safe Hub.

---

### Phase 4: Safe Hub Area (Sanctuary / Town)
*Goal: A low-stress interface where the character rests, manages gear, allocates level-up stats, and chooses where to deploy.*

1. **Hub UI Components**:
   - **Character Status Panel**:
     - HP/MP bars (at 100% or auto-regenerating rapidly in the Hub).
     - Current Level & EXP to next level.
     - Attributes (Base + Gear bonuses) and unspent stat points with `[+]` allocation buttons.
     - Inventory grid & equipped item slots.
   - **Town Navigation Options**:
     - **[Rest / Inn]**: Instantly replenish HP/MP and clear any debuffs.
     - **[Storage / Account Vault]**: Stash items accessible by all 12 characters.
     - **[Adventure Gate / Map]**: Opens zone selection to start idle grinding.
     - **[Save & Return to Character Select]**.
2. **Engine Rules in Safe Hub**:
   - No hostile combat ticks.
   - Regeneration rate boosted (e.g., +10% max HP per second).

---

### Phase 5: The "Idling" Area (The Core Combat & Grinding Engine)
*Goal: An automated, tick-based battle loop where the character fights monsters, earns EXP, loots items, and tracks combat statistics.*

1. **The Tick Loop Architecture**:
   - **Fixed Tick Rate**: e.g., 10 ticks per second (100ms per tick) or 1 tick per second (1000ms).
   - **Tick Processing Steps**:
     ```text
     Every Tick:
       1. Accumulate character and monster attack timers (Action Gauge / ATB).
       2. If Character Attack Gauge Full -> Execute Player Attack -> Reset Gauge.
       3. If Monster Attack Gauge Full -> Execute Monster Attack -> Reset Gauge.
       4. Check HP thresholds:
          - Monster HP <= 0 -> Trigger Kill Event:
              * Award EXP & Gold to Character.
              * Roll Drop Table for Loot (put into inventory).
              * Check Level Up condition (award stat points).
              * Check Global Achievement triggers.
              * Reset / Spawn next monster after a brief delay.
          - Character HP <= 0 -> Trigger Death / Defeat Event:
              * Auto-retreat character to Safe Hub.
              * Apply death penalty (if any) and pause idling.
     ```

2. **Idling Screen UI Components**:
   - **Battle Viewport**: Character vs Current Monster (Name, Level, Animated/ASCII HP bar, Status effects).
   - **Live Combat Log**: Real-time ticker showing damage dealt, damage taken, drops, and EXP gains.
   - **Grind Analytics Widget**:
     - Kills per minute (KPM)
     - EXP gained per hour
     - Gold gained per hour
     - Estimated time to next level
   - **Controls**:
     - `[Stop Idling & Return to Hub]`
     - `[Change Target Zone]`
     - `[Potion / Auto-Consume Settings]`

3. **Offline Progress Calculation (The "Catch-Up" Engine)**:
   - When loading a character who was left in an idle zone:
     $$\Delta t = \text{Current Time} - \text{last\_tick\_timestamp}$$
   - Run a deterministic mathematical simulation (or batch tick execution) for $\Delta t$.
   - Display an **"Offline Gains Summary"** modal on login:
     *"While you were away for 4h 12m, you defeated 1,240 Goblins, gained 45,200 EXP (Level 12 -> 14), and found 18 items."*

---

### Phase 6: Account-Wide Achievements & Synergy
*Goal: Incentivize using multiple character slots by making achievements and bonuses account-wide.*

1. **Global Achievement Triggers**:
   - Global milestones (e.g., *Total Account Level = 100*, *100,000 Total Monsters Slain*, *All 12 Slots Populated*).
2. **Account Bonuses**:
   - Passive multipliers unlocked by achievements (e.g., `+1% Global EXP`, `+5 Shared Vault Slots`, `+2% Gold Find`).

---

# Part 2: Deep System Analysis & Design Questionnaire

To turn this roadmap into exact code specifications, review these critical game-design questions. Your answers will establish the exact formulas and constants for your implementation.

---

### Category A: Tick Rate, Time & Offline Progression

1. **Tick Resolution**:
   - *Question*: How fast should the simulation loop tick?
     - Option 1: **Fast Action-Gauge (10 Hz / 100ms)**: Smooth combat timers, attacks happen every e.g. 1.8 seconds.
     - Option 2: **Turn-Based Tick (1 Hz / 1000ms)**: Every second represents 1 turn; cleaner math and lower CPU usage.
2. **Offline Progress Model**:
   - *Question*: How should offline time be rewarded?
     - **Exact Simulation**: Run full combat loops in headless mode up to a cap (e.g., max 12 or 24 hours). Accurate drop rates, but requires fast loop execution.
     - **Statistical Extrapolation**: Calculate average kills/min and multiply by offline seconds. Instantaneous, zero lag on load.
     - **Offline Cap**: Is there a maximum offline accumulation time (e.g., 8 hours, 24 hours, or uncapped)?
3. **Multi-Character Idling**:
   - *Question*: In your 12-slot vision, do **all 12 characters idle simultaneously in the background**, or **only the currently active character**?
     *(Simultaneous idle for 12 characters adds incredible MMORPG guild-building depth, similar to games like IdleOn).*

---

### Category B: Character Attributes & Core Combat Math

1. **Primary Attributes**:
   - What primary stats define your Novice? Common MMORPG paradigms:
     - **STR (Strength)**: Flat physical damage / inventory carry weight?
     - **AGI / DEX (Agility / Dexterity)**: Attack speed, evasion, accuracy, or critical hit chance?
     - **INT (Intelligence)**: Magic damage, mana pool, mana regen?
     - **VIT (Vitality / Stamina)**: Max HP, HP regen rate, physical defense?
     - **LUK (Luck)**: Drop rate multiplier, rare loot tier chance?
2. **Derived / Secondary Stats**:
   - What are the conversion formulas?
     - Example: $\text{Max HP} = 100 + (\text{VIT} \times 15) + (\text{Level} \times 10)$
     - Example: $\text{Attack Speed (seconds per hit)} = \max\left(0.5, 3.0 - (\text{AGI} \times 0.05)\right)$
3. **Damage & Mitigation Formula**:
   - When Player attacks Monster (or vice versa):
     - **Flat Reduction**: $\text{Damage} = \max(1, \text{Attack} - \text{Defense})$
     - **Percentage Mitigation**: $\text{Damage} = \text{Attack} \times \left(\frac{100}{100 + \text{Defense}}\right)$
     - **RNG Spread**: Does damage have a roll range (e.g., $\pm 10\%$ min/max damage)?
4. **Accuracy & Evasion**:
   - Can attacks miss? Or is hit chance always 100% with varying damage numbers?
5. **Critical Strike**:
   - Is there a base Crit Chance (e.g., 5%) and Crit Multiplier (e.g., 150%)?

---

### Category C: Monster Design & Idling Zones

1. **Monster Attributes**:
   - What attributes does a monster have?
     - Name, Level, Max HP, Attack Power, Defense, Attack Interval (seconds), EXP yield, Gold yield, Loot Table.
2. **Zone Structure & Encounter Logic**:
   - Are monsters fought **1-on-1 sequentially**, or can the player engage multiple enemies (AoE)?
   - How does a player advance zones?
     - Example: Defeat 50 Goblins $\rightarrow$ Unlock Goblin Chief (Boss) $\rightarrow$ Defeat Boss $\rightarrow$ Unlock Dark Forest.
3. **Death & Defeat Penalty**:
   - What happens if the character's HP reaches 0 while idling?
     - **Safe Failure**: Auto-teleport to Hub, idle stops, no item/EXP loss.
     - **MMORPG Classic**: Lose 1–5% of current level's EXP (no de-leveling) and wake up in Town.
     - **Auto-Revive**: Respawn in the zone with a 30-second penalty cooldown.

---

### Category D: Class Progression & Leveling Curves

1. **Experience Curve**:
   - What mathematical formula dictates EXP to level up?
     - Example (Power curve): $\text{EXP Required}(L) = 50 \times L^{2.2}$
     - Example (Compound curve): $\text{EXP Required}(L) = 100 \times (1.15)^L$
2. **Level-Up Rewards**:
   - How many unallocated stat points are awarded per level? (e.g., 5 points per level).
   - Does HP/MP fully restore upon leveling up?
3. **The "Novice" Class & Branching**:
   - At what level does a Novice promote to their first specialization (e.g., Level 10)?
   - What classes are planned (e.g., Warrior, Mage, Archer, Thief, Cleric, Merchant)?
   - Does class promotion reset stats, grant a big stat multiplier, or unlock active/passive skill trees?

---

### Category E: Economy, Inventory & Drops

1. **Inventory Capacity**:
   - Is inventory limited by **Slot Count** (e.g., 20 distinct slots) or **Weight / Carry Capacity**?
   - What happens when the inventory fills up while idling?
     - Auto-discard common loot? Auto-convert to gold? Drop to floor?
2. **Item Categories & Equipment Slots**:
   - Equipment layout: Main Hand (Weapon), Off-Hand (Shield), Head, Body (Armor), Legs, Accessory 1, Accessory 2.
   - Item Rarities: Common (White) $\rightarrow$ Uncommon (Green) $\rightarrow$ Rare (Blue) $\rightarrow$ Epic (Purple) $\rightarrow$ Legendary (Gold).
3. **Drop Table Math**:
   - Weighted pool vs Independent percentage rolls:
     - *Independent Roll*: Slime Jelly (80%), Broken Sword (15%), Slime Card (0.5%).

---

### Category F: Safe Hub & Town Features

1. **Hub Services**:
   - **Innkeeper**: Instant heal / set respawn point.
   - **Blacksmith**: Upgrade weapons/armor using gathered crafting materials and gold.
   - **General Merchant**: Buy healing potions, sell junk loot.
   - **Shared Account Stash**: A bank with 50+ slots shared across all 12 characters.
2. **Passive Hub Activities**:
   - Can characters idle in the Hub? (e.g., "Training Dummy" for slow safe EXP, or "Crafting Station" to passively smelt ore).

---

### Category G: Account-Wide Systems & Save Architecture

1. **Shared vs Character-Specific Assets**:
   - *Character-Specific*: Level, Stats, Equipped Gear, Bag Inventory, Active Quest.
   - *Account-Wide*: Shared Gold Bank, Shared Storage Vault, Unlocked Achievements, Monster Lore Bestiary, Global Prestige Multipliers.
2. **Save File Integrity**:
   - How to protect against corruption during crashes:
     - Automated rolling backups (`account_save.bak1`, `account_save.bak2`).
     - Save intervals: Auto-save every 30 seconds and on every zone transition / menu change.

---

# Part 3: Python Architecture Starter Code

Below is a clean, modular starting foundation implementing the core dataclasses and serialization for the 12-slot account system.

```python
"""
core/models.py - Core data structures for the MMORPG Idle Game
"""
from dataclasses import dataclass, field, asdict
from typing import List, Dict, Optional
import time
import json
import os

MAX_SAVE_SLOTS = 12

@dataclass
class CharacterAttributes:
    strength: int = 5
    agility: int = 5
    vitality: int = 5
    intelligence: int = 5
    unspent_points: int = 0

@dataclass
class Character:
    slot_id: int
    name: str
    class_name: str = "Novice"
    level: int = 1
    current_exp: int = 0
    current_hp: int = 100
    max_hp: int = 100
    gold: int = 0
    attributes: CharacterAttributes = field(default_factory=CharacterAttributes)
    current_zone: str = "safe_hub"
    is_idling: bool = False
    last_active_timestamp: float = field(default_factory=time.time)
    inventory: List[Dict] = field(default_factory=list)

    @property
    def exp_to_next_level(self) -> int:
        return int(100 * (self.level ** 1.8))

@dataclass
class Account:
    account_name: str = "Player"
    created_timestamp: float = field(default_factory=time.time)
    global_gold_bank: int = 0
    shared_storage: List[Dict] = field(default_factory=list)
    unlocked_achievements: List[str] = field(default_factory=list)
    slots: List[Optional[Character]] = field(
        default_factory=lambda: [None] * MAX_SAVE_SLOTS
    )

    def to_dict(self) -> dict:
        return {
            "account_name": self.account_name,
            "created_timestamp": self.created_timestamp,
            "global_gold_bank": self.global_gold_bank,
            "shared_storage": self.shared_storage,
            "unlocked_achievements": self.unlocked_achievements,
            "slots": [asdict(c) if c is not None else None for c in self.slots]
        }

    @classmethod
    def from_dict(cls, data: dict) -> "Account":
        account = cls(
            account_name=data.get("account_name", "Player"),
            created_timestamp=data.get("created_timestamp", time.time()),
            global_gold_bank=data.get("global_gold_bank", 0),
            shared_storage=data.get("shared_storage", []),
            unlocked_achievements=data.get("unlocked_achievements", [])
        )
        loaded_slots = []
        for s in data.get("slots", [None] * MAX_SAVE_SLOTS):
            if s is None:
                loaded_slots.append(None)
            else:
                attrs = CharacterAttributes(**s.pop("attributes", {}))
                loaded_slots.append(Character(attributes=attrs, **s))
        
        # Ensure exactly MAX_SAVE_SLOTS
        while len(loaded_slots) < MAX_SAVE_SLOTS:
            loaded_slots.append(None)
        account.slots = loaded_slots[:MAX_SAVE_SLOTS]
        return account
```
