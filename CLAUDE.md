# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**shootyepgp** is a World of Warcraft 1.12 (Vanilla/Classic) addon implementing an EPGP (Effort Points / Gear Points) loot distribution system for guilds. EPGP tracks raid participation (EP) and loot received (GP), calculating a Priority Ratio (PR = EP/GP) to determine loot priority.

- Interface target: `11200` (WoW 1.12 Vanilla)
- Language: Lua 5.0 (WoW embedded interpreter)
- No build system — the addon is loaded directly by the WoW client via `shootyepgp.toc`

## Lua 5.0 Constraints

The WoW 1.12 client embeds Lua 5.0, which lacks many 5.1+ features. Key pitfalls:
- `table.getn(t)` not `#t` for array length
- `string.find` with captures instead of `string.match` (doesn't exist)
- `math.mod(a, b)` not `a % b` for modulo
- No vararg as table: `arg` is implicit, not `...` spread into a table
- `table.foreach` / `table.foreachi` instead of generalized `for` in some patterns
- `unpack()` is global, not `table.unpack()`
- String library functions available both as `string.find(s, pat)` and `s:find(pat)` — codebase uses both forms

## Architecture

### Framework

Built on the **Ace2** addon framework. The main addon object is created as:
```lua
sepgp = AceLibrary("AceAddon-2.0"):new("AceConsole-2.0", "AceHook-2.1", "AceDB-2.0", "AceDebug-2.0", "AceEvent-2.0", "AceModuleCore-2.0", "FuBarPlugin-2.0")
```

Key Ace2 libraries used throughout:
- **Tablet-2.0** (`T`): All UI panels (standings, bids, loot, reserves, alts, logs)
- **Dewdrop-2.0** (`D`): Dropdown/right-click menus
- **Crayon-2.0** (`C`): Text coloring
- **Babble-Class/Zone-2.2** (`BC`/`BZ`): Localized class/zone names
- **Deformat-2.0** (`DF`): String pattern parsing
- **Gratuity-2.0** (`G`): Tooltip scanning
- **AceLocale-2.2** (`L`): Localization strings

### Module System

Each feature area is an Ace2 module with its own file:

| Module | File | Purpose |
|--------|------|---------|
| `sepgp` (core) | `shootyepgp.lua` | Main addon: events, EPGP logic, hooks, comms |
| `sepgp_standings` | `standings.lua` | EP/GP/PR rankings UI, CSV export/import |
| `sepgp_bids` | `bids.lua` | Bid tracking during loot calls |
| `sepgp_prices` | `prices.lua` | Item GP pricing lookup tables |
| `sepgp_loot` | `loot.lua` | Loot history UI |
| `sepgp_reserves` | `reserves.lua` | Standby/reserve list UI |
| `sepgp_alts` | `alts.lua` | Main-alt relationship UI |
| `sepgp_logs` | `logs.lua` | Activity log UI |

Modules are created via `sepgp:NewModule("name", mixins...)` and follow the `OnEnable()`/`OnDisable()`/`OnTooltipUpdate()` lifecycle. UI modules use `Toggle(forceShow)` to detach/attach Tablet windows, and `Refresh()` to update content.

### Initialization Sequence

The addon initializes in four phases — important when adding new hooks or event registrations:

1. **`OnInitialize()`** — SavedVariables defaults, DB registration
2. **`OnEnable()`** — Guild roster request, core event registration (GUILD_ROSTER_UPDATE, CHAT_MSG_RAID, CHAT_MSG_LOOT, etc.)
3. **`AceEvent_FullyInitialized()`** — Hook into game functions (GiveMasterLoot, SetItemRef, tooltips, LootFrameItem_OnClick), pfUI compatibility hooks
4. **`delayedInit()`** (2–3s after phase 3) — Reserves channel setup, v2→v3 migration, chat command registration (`/shooty`, `/sepgp`), addon message handler, version broadcast, settings sync

### Data Storage

EPGP data is stored in **guild officer notes** (not SavedVariables), formatted as `{EP:GP}` within the 31-character note limit. Example: `Raider{250:150}` means 250 EP, 150 GP.

Key accessor functions:
- `sepgp:get_ep_v3(name, officernote)` / `sepgp:get_gp_v3(name, officernote)` — read from `{EP:GP}` pattern
- `sepgp:update_epgp_v3(ep, gp, guild_index, name, officernote)` — write both EP and GP via `GuildRosterSetOfficerNote`
- `sepgp:update_ep_v3(name, ep)` / `sepgp:update_gp_v3(name, gp)` — write single value (finds guild index internally)

Alt characters store their main's name as `{MainName}` in their officer note, parsed by `sepgp:parseAlt(name, officernote)`.

SavedVariables (per-account and per-character) store addon settings like decay rate, progression tier, and loot history. These are declared in `shootyepgp.toc`.

### Key Constants

The `VARS` table in `shootyepgp.lua` defines defaults:
- `basegp = 1000` — Starting GP for new members
- `baseaward_ep = 100` — Suggested EP award per boss
- `decay = 0.9` — Decay multiplier (10% decay)
- `minlevel = 55` — Minimum level to receive EP
- `timeout = 60` — Reserves AFK check duration (seconds)
- `prefix = "SEPGP_PREFIX"` — Addon message channel prefix

### Load Order

Defined in `shootyepgp.toc`: Ace2 libs first, then `localization.lua` → `shootyepgp.lua` → `migrations.lua` → `prices.lua` → UI modules.

### Key Data Flows

1. **EP Awards**: `award_raid_ep()` iterates raid members, filters by level >= `minlevel` (default 55), calls `givename_ep()` per member which updates officer notes
2. **Loot/GP**: `GiveMasterLoot` hook → `processLoot()` → price lookup → StaticPopup dialog (Mainspec GP / Offspec GP / Bank-DE)
3. **Bidding**: Raid leader says item link in `/raid` → `captureLootCall()` opens 5-min window → whispers parsed by `captureBid()` → sorted by PR in bids tablet
4. **Decay**: `decay_epgp_v3()` multiplies all EP and GP by decay factor (default 0.9)

### Data Structures

**Loot entries** (`sepgp_looted` table) are 8-indexed arrays:
```lua
{timestamp, player_name, player_color, itemLink, bind_type, gp_price, offspec_price, action}
-- Indexed via: loot_index = {time=1, player=2, player_c=3, item=4, bind=5, price=6, off_price=7, action=8, update=9}
```

**Bid entries** (`sepgp.bids_main` / `sepgp.bids_off`) are 5-indexed arrays with optional 6th:
```lua
{name, class, ep, gp, pr}           -- or
{name, class, ep, gp, pr, main_name} -- if alt
```

### Addon Message Protocol

Messages sent via `SendAddonMessage(SEPGP_PREFIX, msg, channel)` with format `"WHO;WHAT;AMOUNT"`:

| WHO | WHAT | AMOUNT | Purpose |
|-----|------|--------|---------|
| PlayerName | EP | +/-value | EP award/penalty |
| PlayerName | GP | +value | GP charge |
| ALL | DECAY | percentage | Decay broadcast |
| RAID | AWARD | ep_value | Raid EP award notification |
| RESERVES | AWARD | ep_value | Reserve EP award notification |
| VERSION | vX.Y | major_ver | Version sync |
| SETTINGS | progress:discount:decay:minep:alts:altpct | 1 | Guild leader settings broadcast |
| BID_ITEM | itemString~quality~itemName~sender | 1 | Open bid window |
| BID_ADD | name:class:ep:gp:pr:main:spec | 1 | Add bid entry |
| BID_CLEAR | 0 | 1 | Clear all bids |

### Permission Model

`admin()` returns `CanEditOfficerNote()` — only officers can modify EPGP data. All write operations check this before proceeding. Guild leader has additional powers (GP reset, settings broadcast).

## Coding Conventions

- All user-facing strings wrapped in `L["..."]` (AceLocale). Translations live in `localization.lua` (enUS base, zhCN community translation).
- UI modules follow a consistent Tablet-2.0 pattern: `Register` → `Detach`/`Attach` for show/hide, `OnTooltipUpdate` for rendering.
- Officer note manipulation uses `string.find`/`string.gsub` with the `{(%d+):(%d+)}` pattern.
- Item prices in `prices.lua` are keyed by item ID with `{gp_cost, tier_string}` tuples, scaled by progression tier multipliers.
- `table.getn()` for array length (Lua 5.0 compatibility).
- Global comments at end of files: `-- GLOBALS: sepgp, sepgp_standings, ...`
- Static popup dialogs prefixed `SHOOTY_EPGP_` (e.g., `SHOOTY_EPGP_AUTO_GEARPOINTS` for loot GP assignment).
