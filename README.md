# PVEZ Reloaded — Community Fix

**PVE/PVP Zone System for DayZ Standalone Servers**

A community-maintained fork of [PVEZ Reloaded](https://steamcommunity.com/sharedfiles/filedetails/?id=2831742849) with critical bug fixes for modern DayZ modded servers. The original mod by [Ermiq](https://steamcommunity.com/sharedfiles/filedetails/?id=1878060278) is no longer actively maintained — this fork keeps it alive.

---

## Why This Fork Exists

PVEZ Reloaded is one of the most widely-used PVE/PVP zone mods for DayZ, but it hasn't received updates in some time. As DayZ and its modding ecosystem evolved, several critical issues emerged:

- **Server crashes** when running alongside Expansion AI (eAI) — the most popular AI mod for DayZ
- **Map zone circles stopped displaying** for many users, with no visual indication of PVP/PVE boundaries
- **Expansion map integration broken** — zone markers never refresh after initial map open

These issues affect thousands of servers. Since PVEZ is [MIT licensed](#license), this fork provides the fixes the community needs.

---

## Fixes

### 1. Expansion AI (eAI) Compatibility — Server Crash Fix

**Severity:** Critical (server crash)

**The Problem:**
When Expansion AI bots fire weapons and hit a player, the server crashes with:

```
NULL pointer to instance
Class: 'PVEZ_PlayerStatus'
Function: 'PVEZ_IsPvpAttackAllowed'
Stack trace:
  PVEZ/4_World/pvez_playerstatus.c:162
  PVEZ/4_World/pvez_damageredistributor.c:93
  ...
  DayZExpansion/AI/Scripts/4_World/.../weapon_base.c:157 (eAI_Fire)
```

**Root Cause:**
eAI bots inherit from `PlayerBase` (they pass `IsPlayer()` checks), but they don't have a `PlayerIdentity`. PVEZ initializes `pvez_PlayerStatus` only for entities with a valid identity:

```c
// In PlayerBase.OnPlayerLoaded():
if (GetGame().IsServer() && GetIdentity()) {    // eAI bots fail this check
    pvez_PlayerStatus = new PVEZ_PlayerStatus(this);  // Never created for AI
}
```

When an AI bot shoots a player, PVEZ's damage handler tries to check the **attacker's** `pvez_PlayerStatus` to determine PVP permissions — but it's `NULL`, causing a crash. This happens at multiple points in the call chain:

| File | Function | Line | Issue |
|------|----------|------|-------|
| `pvez_playerstatus.c` | `PVEZ_IsPvpAttackAllowed()` | 156, 160, 162 | Accesses attacker's `pvez_PlayerStatus` without NULL check |
| `pvez_damageredistributor.c` | `RegisterDeath()` | 104 | Accesses victim's `pvez_PlayerStatus` without NULL check |
| `pvez_killmanager.c` | `OnPlayerKilled()` | 9-10 | Accesses both victim and killer `pvez_PlayerStatus` |
| `pvez_killmanager.c` | `RegisterMurder()` | 51 | Accesses killer's `pvez_PlayerStatus` |
| `playerbase.c` | `PVEZ_ShouldBePardonedOnDeath()` | 107 | Accesses killer's `pvez_PlayerStatus` |
| `missionserver.c` | `OnUpdate()` | 65 | Iterates all players including AI for lawbreaker updates |

**The Fix:**
Added NULL safety checks across all six crash points. When the attacker is an AI bot (no `pvez_PlayerStatus`), damage is allowed through normally — AI follows its own rules, not PVEZ PVP permissions.

```c
// Before (crashes):
else if (PlayerBase.Cast(attacker).pvez_PlayerStatus.GetIsLawbreaker() && ...)

// After (safe):
PlayerBase attackerPlayer = PlayerBase.Cast(attacker);
if (!attackerPlayer || !attackerPlayer.pvez_PlayerStatus)
    return true;  // AI bots bypass PVP checks — their damage passes through
```

---

### 2. Map Zone Circles Not Displaying

**Severity:** High (broken UI)

**The Problem:**
Zone circles are invisible on the in-game map. Reported by multiple community members across different server configurations.

**Root Cause:**
In `pvez_zones.c`, the `Init()` function skips loading zones when the server mode is set to full PVE (Mode 2):

```c
// Original code:
if (g_Game.pvez_Config.GENERAL.Mode != PVEZ_MODE_PVE) {
    UpdateStaticZones();   // Skipped in Mode 2!
    UpdateDynamicZones();  // Skipped in Mode 2!
}
PushUpdateToClients();     // Sends empty zone list to clients
```

The developer's original logic was: "If it's full PVE everywhere, zones don't matter." But this also means:
- Zone circles never render on the map
- Players have no visual reference for zone boundaries
- Servers using Mode 2 with informational zones get nothing displayed

**The Fix:**
Zones now always load regardless of mode. PVP enforcement is handled separately in the damage logic — zone display and zone enforcement are independent concerns.

```c
// Fixed code:
activeZones = new array<ref PVEZ_Zone>;
UpdateStaticZones();
UpdateDynamicZones();
PushUpdateToClients();
```

---

### 3. Expansion Map Zone Refresh Broken

**Severity:** Medium (broken UI on Expansion maps)

**The Problem:**
When using DayZ Expansion's map (which most modded servers use), zone circles only appear when the map is first opened. Scrolling, zooming, or panning causes zones to disappear and never come back.

**Root Cause:**
The Expansion map menu integration had the refresh call **commented out**:

```c
// In plugins/expansion/5_mission/gui/mapmenu.c:
override void UpdateMarkers() {
    super.UpdateMarkers();
    //PVEZ_MapMarkersDrawer.LoadPVEZMarkers(m_MapWidget);  // DISABLED!
}
```

The `UpdateMarkers()` function is called by Expansion whenever the map viewport changes. With the refresh disabled, zone markers are static artifacts from the initial `Init()` call only.

**The Fix:**
Re-enabled the refresh call with the `isUpdating=true` parameter, which activates viewport-aware rendering (only draws zone points visible on screen, with proper scale adaptation):

```c
override void UpdateMarkers() {
    super.UpdateMarkers();
    PVEZ_MapMarkersDrawer.LoadPVEZMarkers(m_MapWidget, true);
}
```

---

## Configuration Reference

All configuration files are stored in `$profile/PVEZ/` (typically `profiles/PVEZ/`).

### Config.json

#### GENERAL

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Mode` | int | `0` | Server PVP mode (see [Modes](#modes) below) |
| `Update_Frequency` | float | `5.0` | How often (seconds) player zone status is checked |
| `Show_Notifications` | bool | `true` | Show enter/exit zone notifications |
| `Use_UI_Notifications` | bool | `true` | Use PVEZ's custom UI (`true`) or chat messages (`false`) |
| `Add_Zone_Name_To_Message` | bool | `true` | Include zone name in enter/exit notifications |
| `Exit_Zone_Countdown` | float | `10.0` | Delay (seconds) before PVP status changes when leaving a zone. `0` = instant |
| `Force1stPersonInPVP` | bool | `false` | Force first-person camera in PVP zones |
| `Week_Starts_On_Sunday` | bool | `false` | Affects zone schedule day numbering |
| `Custom_Enter_Zone_Message` | string | `""` | Custom enter message (empty = use default) |
| `Custom_Exit_Zone_Message` | string | `""` | Custom exit message |
| `Custom_Exit_Zone_Countdown_Message` | string | `""` | Custom countdown message |

#### Modes

| Mode | Value | Description |
|------|-------|-------------|
| `PVP_ZONES` | `0` | PVE everywhere, PVP only inside defined zones |
| `PVE_ZONES` | `1` | PVP everywhere, PVE only inside defined zones |
| `PVE` | `2` | Full PVE — no player damage anywhere |
| `PVP` | `3` | Full PVP — PVEZ damage protection disabled |

#### DAMAGE

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Restore_Target_Health` | bool | `true` | Heal damage dealt in PVE zones |
| `Protect_Clothing_And_Cargo` | bool | `true` | Protect gear from PVE damage |
| `Allow_Damage_Between_PVP_and_PVE` | bool | `false` | Allow a PVP-zone player to damage a PVE-zone player |
| `Damage_Types_Sent_Back_To_The_Attacker` | object | all `false` | Reflect damage back (Weapon, Explosive, Vehicle, Fist Fight) |

#### LAWBREAKERS_SYSTEM

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Declare_a_Lawbreaker_When_Killed_a_Player_in_PVE_Area` | object | — | Per-type kill triggers (Weaponary, Explosive, Vehicle, Fist Fight) |
| `Server_Wide_Message_About_Lawbreaker` | bool | `true` | Announce lawbreaker status to all players |
| `Auto_Clear_Lawbreakers_Data` | bool | `false` | Automatically remove lawbreaker status |
| `Autoclear_Period_Amount` | int | `14` | Auto-clear duration amount |
| `Autoclear_Period_Mode` | int | `2` | `0` = minutes, `1` = hours, `2` = days |
| `Allow_Lawbreakers_To_Attack_Anywhere` | bool | `false` | Lawbreakers can PVP outside PVP zones |
| `Pardon_On_Death_From_Any_Source` | bool | `false` | Clear lawbreaker status on death |

#### MAP

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Show_Player_Marker` | bool | `true` | Show "You are here" marker |
| `Show_Zones_In_VPPA_Teleport_Manager` | bool | `true` | Show zones in VPP Admin teleport UI |
| `Zones_Border_Color` | RGB | `255, 0, 0` | Zone circle border color |
| `Lawbreakers_Markers.Show_Markers_On_Map` | bool | `true` | Show lawbreaker positions on map |
| `Lawbreakers_Markers.Update_Frequency` | float | `800` | Marker update interval (seconds) |
| `Lawbreakers_Markers.Approximate_Location` | bool | `true` | Offset marker from real position |
| `Lawbreakers_Markers.Approximate_Location_Max_Offset` | int | `200` | Max offset distance (meters) |

#### Dynamic Zone Settings

Each dynamic zone type (AIRDROP_ZONES, TERRITORYFLAG_ZONES, HELICRASH_ZONES) has:

| Setting | Type | Description |
|---------|------|-------------|
| `Radius` | float | Zone radius in meters |
| `Name` | string | Display name |
| `ShowBorderOnMap` | bool | Draw circle on map |
| `ShowNameOnMap` | bool | Show name label on map |
| `OnlyWhenFlagIsRaised` | bool | (Territory flags only) Only active when flag is raised |
| `Activity_Schedule.Days` | string | Active days (`"1 2 3 4 5 6 7"` = all week) |
| `Activity_Schedule.StartHour` | int | Active start hour (0-24) |
| `Activity_Schedule.EndHour` | int | Active end hour (0-24) |

### Zones.json

Array of static zone definitions. Each zone:

```json
{
    "Type": 0,
    "Name": "Tisy military base",
    "X": 1576.0,
    "Z": 14000.0,
    "Radius": 600,
    "ShowBorderOnMap": true,
    "ShowNameOnMap": false,
    "Activity_Schedule": {
        "Days": "1 2 3 4 5 6 7",
        "StartHour": 0,
        "EndHour": 24
    }
}
```

Default zones are auto-generated for **Chernarus**, **Livonia**, and **Namalsk** on first run.

### Admins.txt

One Steam ID per line (hashed 44-char or plain 64-bit). Admin players can access the in-game PVEZ admin console.

### Bounties.json

Configure item rewards for killing lawbreakers:

```json
{
    "Enabled": true,
    "Items": [
        { "className": "Money_Dollar100", "displayName": "$100", "amount": 1 }
    ]
}
```

---

## Compatibility

| Mod | Status | Notes |
|-----|--------|-------|
| **DayZ Expansion (Core)** | Compatible | Required for Expansion map zones |
| **DayZ Expansion AI (eAI)** | Compatible (Fixed) | Original crashes — this fork fixes it |
| **DayZ Expansion Navigation** | Compatible | Zone markers integrate with Expansion map |
| **VPP Admin Tools** | Compatible | Admin console + teleport zone display |
| **CF (Community Framework)** | Compatible | Required dependency |
| **BasicMap** | Compatible | Plugin included for BasicMap map menu |
| **Carim Map** | Compatible | Plugin included for Carim map menu |

### Mod Load Order

PVEZ should load **after** CF and Expansion mods in your `-mod=` parameter:
```
-mod=@CF;@DayZExpansion;@DayZExpansionAI;...;@PVEZ_Reloaded
```

---

## Project Structure

```
PVEZ/
├── common/
│   └── defines.c               # Global defines (#define PVEZ)
├── 3_game/
│   ├── configmanagement/
│   │   ├── pvez_configcontents.c  # Config data classes
│   │   └── pvez_configmanager.c   # Config load/save logic
│   ├── constant/
│   │   └── constants.c            # Mode/zone/damage type constants
│   ├── enums/
│   │   └── enums.c                # RPC and notification enums
│   ├── dayzgame.c                 # Core init, RPC handling
│   ├── hud.c                      # HUD icon integration
│   ├── pvez_bounties.c            # Bounty system
│   ├── pvez_lawbreakersmarkers.c  # Lawbreaker map markers
│   ├── pvez_lawbreakersroster.c   # Lawbreaker tracking/storage
│   ├── pvez_mapmarkersdrawer.c    # Zone circle rendering
│   ├── pvez_notificationgui.c     # UI notification widget
│   ├── pvez_notifications.c       # Notification logic
│   ├── pvez_static.c              # Utility functions
│   └── pvez_zones.c               # Zone management + schedules
├── 4_world/
│   ├── classes/.../actionunpin.c  # Grenade thrower tracking
│   ├── entities/
│   │   ├── building/wrecks/       # Helicrash zone triggers
│   │   ├── creatures/infected/    # Zombie (debug mode only)
│   │   ├── dayzplayerimplement.c  # 1st person PVP enforcement
│   │   ├── itembase.c             # Clothing/cargo protection
│   │   └── manbase/playerbase.c   # Core damage handler + status init
│   ├── pvez_adminconsolegui.c     # Admin console UI
│   ├── pvez_bountiesspawner.c     # Bounty item spawning
│   ├── pvez_damageredistributor.c # Damage source tracking + healing
│   ├── pvez_killmanager.c         # Kill processing + lawbreaker declaration
│   └── pvez_playerstatus.c        # Per-player PVP/PVE status
├── 5_mission/
│   ├── gui/
│   │   ├── ingamehud.c            # HUD zone icon
│   │   └── mapmenu.c             # Vanilla map zone markers
│   ├── missiongameplay.c          # Client-side init
│   └── missionserver.c            # Server tick + zone updates
├── plugins/
│   ├── basicmap/                  # BasicMap compatibility
│   ├── carim/                     # Carim map compatibility
│   ├── expansion/                 # Expansion map compatibility
│   └── vppadmintools/             # VPP teleport zone display
├── gui/                           # UI layouts + textures
├── languagecore/                  # Localization strings (CSV)
├── data/                          # Input bindings
├── config.bin                     # CfgPatches + CfgMods
├── LICENSE                        # MIT License
└── README.md                      # This file
```

---

## How PVEZ Works (Technical Overview)

### Damage Flow

```
Player Hit by Weapon/Explosive/Vehicle/Fists
  └─> PlayerBase.EEOnDamageCalculated()
        └─> PVEZ_DamageRedistributor.RegisterHit()
              ├─> GetDamageInitiator() — traces damage source to root entity
              ├─> IsHitByAnotherPlayer() — filters non-player damage
              └─> PVEZ_PlayerStatus.PVEZ_IsPvpAttackAllowed() — PVP permission check
                    ├─> Lawbreaker? → Allow
                    ├─> Both in PVP zone? → Allow
                    ├─> Cross-zone enabled? → Check either zone
                    ├─> AI bot (no status)? → Allow  ← NEW IN THIS FORK
                    └─> Otherwise → Deny + heal damage
```

### Zone Update Cycle

```
MissionServer.OnUpdate() — every frame
  ├─> Every {Update_Frequency} seconds:
  │     └─> Check each player's position against active zones
  │           └─> Update PVP/PVE status + send RPC to client
  └─> Every ~60 seconds:
        └─> Re-evaluate zone schedules (day/hour filtering)
        └─> Check lawbreaker auto-clear timers
```

### Player Status Initialization

```
PlayerBase.OnPlayerLoaded()
  ├─> Has GetIdentity()? (real player)
  │     └─> Create PVEZ_PlayerStatus, PVEZ_DamageRedistributor, PVEZ_BountiesSpawner
  └─> No identity? (AI bot, singleplayer pre-init)
        └─> Skip — these entities are not tracked by PVEZ
```

---

## Changelog

### v1.1.0 — Community Fix (2026-03-02)

- **FIXED:** Server crash (NULL pointer) when Expansion AI bots hit players — 6 crash points patched
- **FIXED:** Map zone circles not displaying in PVE mode (Mode 2)
- **FIXED:** Expansion map zone markers not refreshing on scroll/zoom
- **IMPROVED:** All `pvez_PlayerStatus` access points are now NULL-safe for AI compatibility

### v1.0.0 — PVEZ Reloaded (Original)

- Original release by Ermiq (PVEZ) and Reloaded maintainer
- Supports Chernarus, Livonia, Namalsk default zones
- VPP Admin Tools integration
- Expansion, BasicMap, Carim map plugins
- Lawbreaker system with bounties
- Dynamic zones (airdrops, territory flags, helicrashes)

---

## Contributing

Contributions are welcome. If you find a bug or want to add a feature:

1. Fork the repository
2. Create a branch for your fix
3. Test on a local/dev server
4. Submit a pull request with a clear description

Please include the DayZ version and mod list when reporting issues.

---

## License

MIT License — Copyright (c) 2021 Ermiq

See [LICENSE](LICENSE) for full text. This fork is published under the same MIT license as the original.

---

## Credits

- **Ermiq** — Original PVEZ mod
- **PVEZ Reloaded maintainer** — Reloaded fork with VPP/Expansion plugins
- **inkihh** — VPP Admin Tools teleport zone circle support
- **Hell's Domain** — eAI compatibility fix, map zone fixes, this community fork
