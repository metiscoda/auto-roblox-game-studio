# Roblox/Luau Coding Patterns

Reference for all implementation hats. Consolidated from the game studio's
roblox-specialist agent, technical preferences, and coding rules.

## Luau Strict Mode

Every `.luau` file MUST begin with `--!strict` on the first line.
Use strict type annotations: `local health: number = 100`.

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Classes/Modules | PascalCase | `PlayerManager`, `CombatSystem` |
| Variables/Functions | camelCase | `playerHealth`, `calculateDamage` |
| RemoteEvents/Functions | PascalCase | `FireProjectile`, `GetInventory` |
| Constants | UPPER_SNAKE_CASE | `MAX_HEALTH`, `SPAWN_DELAY` |
| Files (Rojo) | PascalCase `.luau` | `PlayerManager.luau`, `init.server.luau` |
| Folders | PascalCase | `CombatSystem/`, `UI/` |

## Rojo File Naming

| File Pattern | Roblox Class |
|-------------|-------------|
| `*.server.luau` | Script (server) |
| `*.client.luau` | LocalScript (client) |
| `*.luau` (plain) | ModuleScript (shared) |
| `init.server.luau` | Script (represents parent folder) |
| `init.client.luau` | LocalScript (represents parent folder) |
| `init.luau` | ModuleScript (represents parent folder) |

## Forbidden Patterns

These patterns MUST NOT appear in any `.luau` file:

| Forbidden | Replacement | Why |
|-----------|------------|-----|
| `wait()` | `task.wait()` | Legacy, imprecise timing |
| `spawn()` | `task.spawn()` | Legacy, no error propagation |
| `delay()` | `task.delay()` | Legacy, imprecise timing |
| Global variables | `local` keyword | Pollutes global scope, hard to trace |
| `game:GetService()` in loops | Cache at module scope | Performance: service lookup is not free |
| `SetAsync` for player data | `UpdateAsync` | Race condition prevention |
| `Instance:FindFirstChild()` in tight loops | Cache references | Performance: tree traversal per call |
| Hardcoded gameplay values | Config modules in `shared/` | Must be data-driven and tunable |
| `while true do wait() end` | Event-driven patterns | Wasteful polling |

## Service Caching Pattern

```luau
--!strict
-- Cache services at module scope, NEVER inside functions or loops
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local module = {}
-- ... module code using cached services
return module
```

## Client/Server Boundary

- **Server-authoritative**: ALL gameplay state (health, inventory, currency, position validation) lives on the server
- **Client**: Input handling, UI rendering, visual effects, sound, camera
- **Never trust client**: Validate all RemoteEvent/RemoteFunction arguments on the server
- **RemoteEvent**: Fire-and-forget actions (preferred)
- **RemoteFunction**: Only when return value is essential (avoid for gameplay — vulnerable to client yielding)
- Minimize RemoteEvent traffic: batch updates, use unreliable for cosmetics

## RemoteEvent Patterns

```luau
-- Server: validate everything
remote.OnServerEvent:Connect(function(player: Player, action: string, data: any)
    -- Type check
    if typeof(action) ~= "string" then return end
    -- Range check
    if typeof(data) ~= "number" or data < 0 or data > MAX_VALUE then return end
    -- Rate limit
    if not rateLimiter:check(player) then return end
    -- Process
    processAction(player, action, data)
end)
```

## DataStore Patterns

- Wrap every call in `pcall` — DataStores error in Studio and under throttling
- Use `UpdateAsync` over `SetAsync` to avoid race conditions
- Implement session locking to prevent multi-server data corruption
- Auto-save every 30-60 seconds + save on `PlayerRemoving` + `game:BindToClose`
- 4MB per-key limit — serialize efficiently, never store Instances
- Use `MemoryStoreService` for temporary cross-server data

## Performance Budgets

| Metric | Target |
|--------|--------|
| Client framerate | 60 FPS |
| Server heartbeat | < 20ms |
| RemoteEvent traffic | Minimize; batch updates |

### Performance Patterns
- Object pooling for frequently created/destroyed instances (projectiles, NPCs, VFX)
- `Workspace:GetPartBoundsInRadius()` for spatial queries, never iterate all parts
- Minimize `RunService.Heartbeat` / `RenderStepped` — only for truly per-frame work
- Use `StreamingEnabled` for large worlds — design around streaming in/out
- Profile with MicroProfiler (`Ctrl+F5`) and Developer Console (`F9`)
- Offload heavy computation across frames with `task.wait()`

## Replication Strategies

- **Attributes**: Lightweight replicated key-value data on instances
- **CollectionService tags**: Entity classification (avoid folder-based organization)
- **Value objects** (`IntValue`, `StringValue`): Only when you need `.Changed` events
- Large data transfers: chunk or use multiple RemoteEvents

## Module Pattern

```luau
--!strict
local MyModule = {}

-- Public API with doc comments
--- Calculates damage after applying armor reduction.
function MyModule.calculateDamage(baseDamage: number, armor: number): number
    return math.max(0, baseDamage - armor)
end

return MyModule
```

## Security Checklist

- [ ] No admin commands or debug tools in client scripts
- [ ] Rate-limiting on all RemoteEvent handlers
- [ ] Type/range/frequency validation on all incoming data
- [ ] No secrets (API keys, webhooks) in any script
- [ ] Server-side purchase validation via `MarketplaceService.ProcessReceipt`
- [ ] Memory leak prevention: disconnect event connections, use Maid/Trove cleanup
- [ ] Handle `StreamingEnabled` — don't assume all parts exist on client
- [ ] Timeout on `WaitForChild` — avoid infinite yield warnings

## Common Pitfalls

- Trusting client input for gameplay decisions
- Not wrapping DataStore calls in pcall
- Memory leaks from undisconnected connections
- Using `while true do wait() end` instead of event-driven patterns
- Storing non-serializable data in DataStores (Instances, functions, metatables)
- Not handling StreamingEnabled correctly
