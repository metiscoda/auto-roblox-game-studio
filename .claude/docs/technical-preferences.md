# Technical Preferences

<!-- Populated by engine setup. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Roblox Studio
- **Language**: Luau (`--!strict` mode required)
- **Rendering**: Roblox rendering engine (Voxel lighting configured)
- **Physics**: Roblox default physics engine
- **Sync Tool**: Rojo 7.6.1 (`game-rojo/` -> Studio)
- **Live Bridge**: Roblox Studio MCP (inspection, testing, runtime)

## Naming Conventions

- **Classes/Modules**: PascalCase (`PlayerManager`, `CombatSystem`)
- **Variables/Functions**: camelCase (`playerHealth`, `calculateDamage`)
- **RemoteEvents/Functions**: PascalCase (`FireProjectile`, `GetInventory`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_HEALTH`, `SPAWN_DELAY`)
- **Files (Rojo)**: PascalCase `.luau` (`PlayerManager.luau`, `init.server.luau`)
- **Folders**: PascalCase (`CombatSystem/`, `UI/`)

## Roblox Architecture

- **Server scripts**: `game-rojo/src/server/` -> `ServerScriptService.Server`
- **Client scripts**: `game-rojo/src/client/` -> `StarterPlayer.StarterPlayerScripts.Client`
- **Shared modules**: `game-rojo/src/shared/` -> `ReplicatedStorage.Shared`
- **Server-only modules**: Consider adding `ServerStorage` path to Rojo project as needed
- **Client/Server model**: Server-authoritative for all gameplay state

## Performance Budgets

- **Target Framerate**: 60 FPS (client)
- **Server Heartbeat**: < 20ms
- **Memory Ceiling**: [TO BE CONFIGURED — platform dependent]
- **RemoteEvent Budget**: Minimize traffic; batch updates; use unreliable for cosmetics

## Testing

- **Framework**: [TO BE CONFIGURED — TestEZ or similar]
- **Minimum Coverage**: [TO BE CONFIGURED]
- **Required Tests**: Balance formulas, gameplay systems, DataStore operations

## Forbidden Patterns

- `wait()`, `spawn()`, `delay()` — use `task.wait()`, `task.spawn()`, `task.delay()`
- Global variables — all variables must be `local`
- `game:GetService()` inside loops or per-frame callbacks — cache at module scope
- Client-side gameplay state authority — server-authoritative only
- `SetAsync` for player data — use `UpdateAsync` to avoid race conditions
- Hardcoded gameplay values — must be data-driven (config modules or Attributes)
- `Instance:FindFirstChild()` in tight loops — cache references
- Scripts without `--!strict` mode

## Allowed Libraries / Addons

- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

- [No ADRs yet — use /architecture-decision to create one]
