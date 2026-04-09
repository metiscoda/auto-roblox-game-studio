# Game Architecture Reference

Reference for all hats that need to understand the Roblox project structure,
Rojo build system, and MCP live bridge to Studio.

## Rojo Project Structure

**Project root**: `game-rojo/`
**Project file**: `game-rojo/default.project.json`
**Version**: Rojo 7.6.1

### Directory-to-Studio Mapping

| Local Path | Studio Location | Purpose |
|------------|----------------|---------|
| `game-rojo/src/server/` | `ServerScriptService.Server` | Server scripts and modules |
| `game-rojo/src/client/` | `StarterPlayer.StarterPlayerScripts.Client` | Client scripts and modules |
| `game-rojo/src/shared/` | `ReplicatedStorage.Shared` | Shared modules (replicated to client) |

### Editing Workflow
1. Edit `.luau` files locally in `game-rojo/src/`
2. Rojo automatically syncs changes to Studio
3. Use MCP tools to verify in Studio if needed

**Always edit scripts locally via Rojo.** Only use MCP `multi_edit` for
scripts outside the Rojo-managed tree.

## Client/Server Architecture

### Responsibility Split

| Layer | Runs On | Responsibilities |
|-------|---------|-----------------|
| Server | ServerScriptService | Gameplay state, DataStores, validation, physics authority, economy, progression |
| Client | StarterPlayerScripts | Input handling, UI, camera, VFX, sound, local predictions |
| Shared | ReplicatedStorage | Type definitions, config modules, utility functions, constants |

### Data Flow Pattern

```
Client Input → RemoteEvent → Server Validation → State Update → Replication → Client Display
```

### Shared Module Categories
- **Config modules**: Gameplay values, constants, tuning knobs (`shared/Config/`)
- **Type definitions**: Shared types used by both client and server (`shared/Types/`)
- **Utility functions**: Math helpers, string formatting, etc. (`shared/Utils/`)
- **Enums/Constants**: Game-wide enumerations (`shared/Enums/`)

## RemoteEvent Architecture

### Naming Convention
PascalCase, verb-first: `FireProjectile`, `PurchaseItem`, `GetInventory`

### Organization
```
ReplicatedStorage/
  Shared/
    Remotes/
      init.luau          -- Module that creates/caches RemoteEvents
      -- or individual RemoteEvent instances via Rojo
```

### Server-Side Pattern
```luau
-- Always validate: type, range, rate, authorization
remote.OnServerEvent:Connect(function(player, ...)
    if not validateArgs(...) then return end
    if not rateLimiter:check(player) then return end
    -- process
end)
```

## DataStore Schema Patterns

### Player Data
```luau
type PlayerData = {
    version: number,         -- Schema version for migrations
    coins: number,
    inventory: {[string]: number},
    stats: {
        level: number,
        xp: number,
    },
    settings: {
        musicVolume: number,
        sfxVolume: number,
    },
}

local DEFAULT_DATA: PlayerData = {
    version = 1,
    coins = 0,
    inventory = {},
    stats = { level = 1, xp = 0 },
    settings = { musicVolume = 0.5, sfxVolume = 0.5 },
}
```

### Session Locking
- Save player's `JobId` in DataStore on join
- Check `JobId` on load — if different server, wait or reject
- Clear `JobId` on leave + `game:BindToClose`

## MCP Tools (Live Studio Bridge)

When Studio is running with the MCP plugin, these tools are available:

### Inspection
- `search_game_tree` — Browse instance hierarchy (path, className, keywords, depth)
- `inspect_instance` — Get properties, attributes, children of an instance

### Script Operations
- `script_read` — Read script source by dot-notation path
- `script_search` — Fuzzy-find scripts by name
- `script_grep` — Search all script contents for a pattern

### Execution & Testing
- `execute_luau` — Run Luau in Studio (plugin context, not game context)
- `start_stop_play` — Start/stop play mode
- `get_console_output` — Read console during play mode

### Player Simulation
- `character_navigation` — Move character to coordinates or instance
- `user_keyboard_input` — Send keyboard events
- `user_mouse_input` — Send mouse events

### Testing Workflow
1. Edit scripts locally (Rojo syncs)
2. `start_stop_play(is_start: true)`
3. `get_console_output` — check for errors
4. `execute_luau` — verify runtime state
5. `start_stop_play(is_start: false)`

### Path Format
All instance paths use dot notation from the service level:
`game.ServerScriptService.Server.MyScript`, `Workspace.Models.SpawnPad`

## Project Directory Structure

```
roblox-game-studio/
├── game-rojo/                 # Rojo project (syncs to Studio)
│   ├── default.project.json   # Rojo config
│   └── src/
│       ├── server/            # ServerScriptService scripts
│       ├── client/            # StarterPlayerScripts
│       └── shared/            # ReplicatedStorage modules
├── design/                    # Game design documents
│   ├── gdd/                   # Per-system GDDs
│   ├── narrative/             # Story, lore, dialogue
│   └── concepts/              # Game concept docs
├── assets/                    # Game assets
│   └── data/                  # Balance data, config JSON
├── tests/                     # Test suites
├── docs/                      # Technical documentation
│   └── engine-reference/      # Engine API snapshots
├── production/                # Sprint plans, milestones, releases
├── presets/                   # Ralph hat pipelines
└── skills/                    # Ralph domain knowledge
```
