# Automated r/PvZH Garden

> A Discord bot cog for [Red-DiscordBot](https://github.com/Cog-Creators/Red-DiscordBot) that simulates a real-time Zen Garden game inspired by *Plants vs. Zombies: Fusion*, complete with passive plant growth, a recipe-based fusion crafting system, a player-driven economy, peer-to-peer trading, and PIL-generated garden visuals.

---

## Team

| No. | Full Name | Student Code | Role |
|-----|-----------|--------------|------|
| 1 | Lê Thái Minh Tín | ITCSIU24086 | Team Leader |
| 2 | Phạm Gia Hân | ITITWE24009 | Team Member |
| 3 | Phạm Ngọc Yến Nhi | ITDSIU24037 | Team Member |
| 4 | Vũ Đức Thiên Ân | ITDSIU24002 | Team Member |

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Data Structures & Algorithms](#data-structures--algorithms)
- [Installation](#installation)
- [Configuration](#configuration)
- [Commands Reference](#commands-reference)
- [Game Mechanics](#game-mechanics)
- [Project Structure](#project-structure)

---

## Overview

**Automated r/PvZH Garden** (ARG) is a persistent, server-wide idle garden game delivered entirely through Discord. Players spend Sun (the in-game currency) to plant seedlings, wait for them to grow into randomized plants, combine plants and materials via fusion recipes to craft rare hybrids, and compete on a leaderboard for the highest Sun balance.

The cog is built as a course project for a **Data Structures & Algorithms** class, demonstrating practical use of counters/multisets, recursive tree traversal, priority queues, hash maps, and greedy algorithms within a real production system.

---

## Features

| Category | Feature |
|---|---|
| **Economy** | Per-user Sun balance, daily reward, tiered sale prices, mastery bonuses |
| **Garden** | 12 garden plots, passive real-time growth loop (~60 s ticks), seedling categories |
| **Fusion** | Recipe-based crafting, recursive ingredient deconstruction, first-discovery bonus |
| **Storage** | 8-slot shed, move plants between garden and storage |
| **Shops** | Crazy Dave (hourly), Penny's Treasures (hourly), Rux's Bazaar (permanent upgrades) |
| **Trading** | Plant-for-Sun and material-for-Sun peer-to-peer trades with 60 s confirmation window |
| **Almanac** | Full plant encyclopedia with regex filtering, availability check, and discovery tracking |
| **Mastery** | Sun Mastery (+10% sell/level) and Time Mastery (+10% growth speed/level) |
| **Visuals** | PIL-composited 4×3 garden image rendered on demand |
| **Backgrounds** | Cosmetic garden backgrounds unlocked by completing fusion sets |
| **Admin** | Owner-only commands for Sun grants, item injection, speed control, and data management |
| **Persistence** | Write-through in-memory cache; full state flushed to Red's Config every ~60 seconds |

---

## Tech Stack

| Technology | Version | Purpose |
|---|---|---|
| Python | 3.10+ | Core language |
| Red-DiscordBot | ≥ 3.3.0 | Bot framework, Config system, cog loader |
| discord.py | (via Red) | Discord API integration |
| Pillow (PIL) | Latest | Garden image generation |
| pytz | Latest | EST timezone handling |
| asyncio | stdlib | Async runtime, background growth loop |
| dataclasses | stdlib | Typed immutable/mutable data models |
| collections.Counter | stdlib | Multiset operations for fusion matching |

---

## Architecture

The codebase follows a **layered composition** pattern — the main cog class instantiates and orchestrates 12 single-responsibility helper classes, keeping each domain isolated and testable.

```
┌─────────────────────────────────────────────────────────┐
│                    ARG Cog (arg.py)                      │
│   Discord commands · background growth loop · routing   │
└───────────────────────┬─────────────────────────────────┘
                        │ composes
        ┌───────────────┼───────────────────────┐
        ▼               ▼                       ▼
┌──────────────┐ ┌─────────────┐       ┌──────────────────┐
│ GameState    │ │ Garden      │  ...  │ Image / Trade /  │
│ Helper       │ │ Helper      │       │ Fusion / Shop    │
│ (persistence)│ │ (profiles)  │       │ Helpers (×9)     │
└──────────────┘ └─────────────┘       └──────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│              Models Layer (arg/models/)                 │
│  BasePlant · FusionRecipe · Background · UserProfile   │
│  (frozen dataclasses for assets, mutable for state)    │
└────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────┐
│              Data Layer (arg/data/)                     │
│  JSON files: plants · fusions · shops · backgrounds    │
└────────────────────────────────────────────────────────┘
```

### Key Design Decisions

- **Write-through cache** — all mutations hit the in-memory dict immediately, then batch-flush to disk every 60 seconds. This keeps Discord's 3-second response window while bounding data loss to ~1 minute.
- **Immutable / mutable boundary** — `UserProfileView` is a read-only snapshot; external code cannot accidentally corrupt a live profile.
- **Per-user concurrency locks** — `LockHelper` prevents two concurrent operations from racing on the same user's data without database transactions.
- **Composition over inheritance** — the cog wires helpers together at `__init__` time; no complex inheritance chain.

---

## Data Structures & Algorithms

### Recursive Plant Deconstruction — `FusionHelper`

Fusion trees can be arbitrarily deep (a tier-12 plant may require tier-6 ingredients that themselves require tier-3 ingredients…). The helper walks the tree recursively, flattening any `FusionRecipe` node into its primitive `BasePlant` leaves.

```
deconstruct(plant):
    if plant has no recipe → return {plant: 1}          # base case
    result = {}
    for ingredient in recipe.ingredients:
        sub = deconstruct(ingredient)                    # recurse
        merge sub into result
    return result
```

**Complexity:** O(d × b) where d = tree depth, b = branching factor.

### Counter-Based Multiset Matching — `FusionHelper`

Fusion recipes are *order-independent*: `["Wall-nut", "Wall-nut", "Peashooter"]` requires two Wall-nuts regardless of inventory slot order. Python's `collections.Counter` provides O(1) set-difference checks.

```python
required  = Counter(recipe.ingredients)          # {"Wall-nut": 2, "Peashooter": 1}
available = Counter(user.plants)                 # from recursive deconstruction
can_fuse  = not (required - available)           # True if all counts satisfied
```

### Greedy Crafting Planner

When a user requests a fusion, the planner determines the minimum set of intermediate fusions needed, resolving dependencies greedily from the deepest tier upward.

**Complexity:** O(n × d) where n = number of required ingredients, d = max recipe depth.

### Growth Simulation Loop

Every ~60 seconds, one background task iterates all active users and all 12 garden slots:

```
for each user U:
    for each seedling slot S in U.garden:
        elapsed = now - S.last_tick
        delta   = (100 / duration_hours) × elapsed × time_mastery_bonus × growth_multiplier
        S.progress += delta
        if S.progress >= 100:
            replace S with random plant from S.category_pool
```

**Complexity per tick:** O(U × 12) — linear in user count, constant slots per user.

### Leaderboard

Uses Python's built-in Timsort (`sorted(users, key=lambda u: u.sun, reverse=True)`) for O(n log n) ranking, with O(k) pagination for display.

### Almanac Filtering

Supports arbitrary filter combinations: `type:vanilla tier:>=6 name:Wall`. Each token is parsed with a small regex dispatcher, then applied as a linear scan over the plant catalogue — O(n × f) where f = number of active filters.

---

## Installation

### Prerequisites

- Python 3.10 or higher
- A running [Red-DiscordBot](https://docs.discord.red/en/stable/install_guides/index.html) instance (v3.3.0+)
- Pillow and pytz installed in the bot's environment

### Steps

```bash
# 1. Clone the repository into Red's cog path
git clone https://github.com/<your-repo>/Automated-rPvZH-Garden.git

# 2. Install Python dependencies
pip install Pillow pytz

# 3. Load the cog from inside Discord (bot owner only)
[p]load arg

# 4. Verify the cog loaded
[p]cog list
```

> Replace `[p]` with your bot's command prefix.

---

## Configuration

All persistent data is managed automatically by Red's `Config` system — no manual database setup is required. Admin-only settings can be adjusted via the `[p]gardenadmin` command group after loading the cog.

| Admin Command | Description |
|---|---|
| `gardenadmin setsun <user> <amount>` | Grant or set a user's Sun balance |
| `gardenadmin additem <user> <item>` | Add a material to a user's inventory |
| `gardenadmin addplant <user> <plant>` | Add a plant directly to a user's garden/storage |
| `gardenadmin speed <multiplier>` | Multiply the global growth speed (e.g. `10` for 10×) |
| `gardenadmin setsunmastery <user> <level>` | Set a user's Sun Mastery level |
| `gardenadmin settimemastery <user> <level>` | Set a user's Time Mastery level |

---

## Commands Reference

### Profile & Economy

| Command | Description |
|---|---|
| `profile [@user]` | View your (or another user's) Sun balance, mastery levels, and stats |
| `daily` | Claim 1,000 Sun (once per EST calendar day) |
| `leaderboard` | Display the top Sun earners on the server |
| `gardenhelp` | Show the full command guide |

### Garden

| Command | Description |
|---|---|
| `plant <seedling>` | Spend 100 Sun to plant a seedling in an empty slot |
| `sell <slot/plant>` | Sell a mature plant for Sun |
| `shovel <slot>` | Remove a seedling before it matures (no refund) |
| `reorder <slot> <slot>` | Swap two garden slots |
| `store <slot>` | Move a plant to the storage shed |
| `unstore <slot>` | Move a plant from storage to the garden |
| `storage` | View your storage shed |

### Fusion

| Command | Description |
|---|---|
| `fuse <recipe>` | Fuse plants (and materials) following a recipe |
| `almanac` | Browse the plant encyclopaedia |
| `almanac info <plant>` | Detailed info on a specific plant |
| `almanac available` | Show plants you can currently fuse |
| `almanac discover` | Show your discovery progress |

### Shops

| Command | Description |
|---|---|
| `daveshop` | Browse Crazy Dave's shop (refreshes hourly) |
| `davebuy <item>` | Buy from Crazy Dave |
| `pennyshop` | Browse Penny's Treasures (refreshes hourly) |
| `pennybuy <item>` | Buy from Penny |
| `ruxshop` | Browse Rux's Bazaar (permanent upgrades) |
| `ruxbuy <item>` | Buy a permanent upgrade from Rux |

### Trading

| Command | Description |
|---|---|
| `trade <@user> <plant> <price>` | Propose a plant trade |
| `tradeitem <@user> <item> <price>` | Propose a material trade |
| `accept` | Accept an incoming trade proposal |
| `decline` | Decline an incoming trade proposal |

### Backgrounds

| Command | Description |
|---|---|
| `background list` | View available cosmetic backgrounds and unlock requirements |
| `background set <name>` | Equip an unlocked background |

---

## Game Mechanics

### Sun Economy

- **Starting balance:** 0 Sun
- **Daily reward:** 1,000 Sun (resets at midnight EST)
- **Seedling cost:** 100 Sun per plot
- **Sale prices (base):**

| Tier | Sun value |
|------|-----------|
| Base plant | 1,000 |
| Tier 3 | 5,000 |
| Tier 6 | 20,000 |
| Tier 9 | 60,000 |
| Tier 12 | 144,000 |
| Tier ∞ | 2,000,000 |

- **Sun Mastery:** +10% sell price per level, earned by selling tier-∞ plants.

### Plant Growth

1. User plants a seedling (costs 100 Sun).
2. The background loop advances progress each tick: `Δ = (100 / duration_h) × elapsed_h × time_mastery_bonus × growth_multiplier`.
3. Default duration: **4 hours** to 100%.
4. At 100% progress, the seedling is replaced with a random plant drawn from its category pool.
5. **Time Mastery** grants +10% growth speed per level.

### Fusion System

- Each fusion recipe specifies a list of required ingredients (plants + materials).
- The `FusionHelper` recursively deconstructs the user's current plants to check if the recipe is satisfiable.
- Discovering a fusion for the first time grants a permanent **+50% sale price bonus** for that plant.
- Certain fusion sets unlock **cosmetic garden backgrounds**.

### Trading

- Either player proposes a trade (plant or material → Sun price).
- The counterparty has **60 seconds** to accept or decline.
- Both users are locked during the window to prevent concurrent inventory mutations.
- All balances and inventories are validated atomically at acceptance time.

---

## Project Structure

```
Automated-rPvZH-Garden/
├── arg/
│   ├── __init__.py               # Red-DiscordBot cog entry point
│   ├── arg.py                    # Main ARG cog (~2,950 lines)
│   ├── info.json                 # Cog metadata & dependency declarations
│   ├── decorators/
│   │   └── checks.py             # is_cog_ready, is_not_locked guards
│   ├── helpers/
│   │   ├── game_state_helper.py  # Persistent state (in-memory + disk flush)
│   │   ├── garden_helper.py      # User profiles & garden/storage operations
│   │   ├── plant_helper.py       # Plant definitions & random selection
│   │   ├── fusion_helper.py      # Recursive deconstruction & recipe matching
│   │   ├── shop_helper.py        # Shop stock generation & refresh scheduling
│   │   ├── trade_helper.py       # Peer-to-peer trade management
│   │   ├── sales_helper.py       # Tiered sale price calculation
│   │   ├── image_helper.py       # PIL garden image compositor
│   │   ├── background_helper.py  # Background unlock management
│   │   ├── lock_helper.py        # Per-user concurrency locks
│   │   ├── data_helper.py        # JSON data loading & dataclass parsing
│   │   ├── logging_helper.py     # Discord + console dual logging
│   │   └── time_helper.py        # EST timezone & Unix timestamp utilities
│   ├── models/
│   │   ├── assets.py             # Frozen dataclasses (BasePlant, FusionRecipe, …)
│   │   └── user_data.py          # Mutable user state (UserProfile, UserProfileView)
│   └── data/
│       ├── backgrounds.json
│       ├── materials.json
│       ├── sales_price.json
│       ├── seedlings.json
│       ├── base_plants/          # vanilla.json · snow.json · mystery.json · shop.json
│       ├── fusions/              # vanilla · snow · std · dave_arg · deltarune · …
│       ├── shop/                 # dave.json · penny.json · rux.json
│       ├── font/
│       │   └── Roboto-Medium.ttf
│       └── images/               # 128×128 RGBA PNG sprites
└── info.json                     # Top-level cog metadata
```

---

## Academic Context

This project was developed as part of a **Data Structures & Algorithms** course. The core learning objectives demonstrated are:

- **Hash maps / dictionaries** — O(1) user lookups, recipe indexing, inventory management
- **Multisets (Counter)** — order-independent ingredient matching for fusion recipes
- **Recursive tree traversal** — flattening arbitrarily deep fusion ingredient trees
- **Greedy algorithms** — minimum-step crafting plan generation
- **Sorting & pagination** — Timsort leaderboard ranking, O(k) page slicing
- **Concurrent state management** — per-user locks to emulate atomic transactions without a database

---

*Built with Red-DiscordBot · Pillow · discord.py*
