# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**shootyepgp** is a World of Warcraft 1.12 (Vanilla/Classic) addon implementing an EPGP (Effort Points / Gear Points) loot distribution system for guilds. EPGP tracks raid participation (EP) and loot received (GP), calculating a Priority Ratio (PR = EP/GP) to determine loot priority.

- Interface target: `11200` (WoW 1.12 Vanilla)
- Language: Lua 5.0 (WoW embedded interpreter — use `table.getn()` not `#`, `string.find` not `string.match` for captures, etc.)
- No build system — the addon is loaded directly by the WoW client via `shootyepgp.toc`

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

Modules are created via `sepgp:NewModule("name", mixins...)` and follow the `OnEnable()`/`OnDisable()`/`OnTooltipUpdate()` lifecycle.

### Data Storage

EPGP data is stored in **guild officer notes** (not SavedVariables), formatted as `{EP:GP}` within the 31-character note limit. Example: `Raider{250:150}` means 250 EP, 150 GP.

Key accessor functions:
- `sepgp:get_ep_v3(name, officernote)` / `sepgp:get_gp_v3(name, officernote)` — read from `{EP:GP}` pattern
- `sepgp:update_ep_v3(name, ep)` / `sepgp:update_gp_v3(name, gp)` — write via `GuildRosterSetOfficerNote`

SavedVariables (per-account and per-character) store addon settings like decay rate, progression tier, and loot history. These are declared in `shootyepgp.toc`.

### Load Order

Defined in `shootyepgp.toc`: Ace2 libs first, then `localization.lua` → `shootyepgp.lua` → `migrations.lua` → `prices.lua` → UI modules.

### Key Data Flows

1. **EP Awards**: `award_raid_ep()` iterates raid members, filters by level >= `minlevel` (default 55), calls `givename_ep()` per member which updates officer notes
2. **Loot/GP**: `GiveMasterLoot` hook → `processLoot()` → price lookup → StaticPopup dialog (Mainspec GP / Offspec GP / Bank-DE)
3. **Bidding**: Raid leader says item link in `/raid` → `captureLootCall()` opens 5-min window → whispers parsed by `captureBid()` → sorted by PR in bids tablet
4. **Decay**: `decay_epgp_v3()` multiplies all EP and GP by decay factor (default 0.9)

### Permission Model

`admin()` returns `CanEditOfficerNote()` — only officers can modify EPGP data. All write operations check this before proceeding.

### Communication

Addon-to-addon messaging via `SendAddonMessage` with prefix `SEPGP_PREFIX` over GUILD/RAID channels. Used for version checks, settings sync (guild leader broadcasts), and award notifications.

## Coding Conventions

- All user-facing strings wrapped in `L["..."]` (AceLocale). Translations live in `localization.lua` (enUS base, zhCN community translation).
- UI modules follow a consistent Tablet-2.0 pattern: `Register` → `Detach`/`Attach` for show/hide, `OnTooltipUpdate` for rendering.
- Officer note manipulation uses `string.find`/`string.gsub` with the `{(%d+):(%d+)}` pattern.
- Item prices in `prices.lua` are keyed by item ID with `{gp_cost, tier_string}` tuples, scaled by progression tier multipliers.
- `table.getn()` for array length (Lua 5.0 compatibility).
- Global comments at end of files: `-- GLOBALS: sepgp, sepgp_standings, ...`
