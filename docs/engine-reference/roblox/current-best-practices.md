# Roblox — Current Best Practices

**Last verified:** 2026-03-29

Modern Roblox/Luau patterns for a multiplayer server-authoritative game.
Focused on patterns relevant to a space trading tycoon.

---

## Project Structure

### Rojo Directory Layout

```
spacetycoon-rojo/src/
├── server/          → ServerScriptService.Server     (*.server.luau)
├── client/          → StarterPlayer.StarterPlayerScripts.Client (*.client.luau)
└── shared/          → ReplicatedStorage.Shared       (*.luau modules)
```

Rules:
- Gameplay state, economy, DataStore, validation: **server only**
- Input, UI, visual FX, sounds: **client only**
- Pure data/config/utility: **shared** (visible to both sides)
- Server-only modules: keep in `ServerStorage` (not replicated)

---

## Luau Strict Mode

Every script must begin with:

```lua
--!strict
```

Strict mode catches type errors at edit time, not runtime. Combine with explicit
type annotations on all public functions and non-obvious locals.

```lua
--!strict

local MAX_CREDITS: number = 999_999_999
local STATION_TYPES: {string} = {"Trading", "Refinery", "Shipyard"}

type PlayerData = {
    credits: number,
    shipId: string,
    rank: number,
}

local function calculateTrade(base: number, modifier: number): number
    return base * modifier
end
```

---

## Module Pattern

All ModuleScripts return a single table. Use PascalCase for module names.

```lua
--!strict

local EconomyService = {}

-- Private
local DataStore = game:GetService("DataStoreService")
local creditsStore = DataStore:GetDataStore("PlayerCredits")

-- Public
function EconomyService.getCredits(userId: number): number
    -- implementation
    return 0
end

return EconomyService
```

---

## Service Caching

Cache service references at module scope — never inside functions, loops, or
per-frame callbacks.

```lua
--!strict

-- ✅ Cache at top of module
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local DataStoreService = game:GetService("DataStoreService")
local CollectionService = game:GetService("CollectionService")

-- ❌ Never do this
RunService.Heartbeat:Connect(function()
    local players = game:GetService("Players") -- expensive every frame
end)
```

---

## Task Library (Required)

The project bans `wait()`, `spawn()`, `delay()`. Use the `task` library exclusively.

```lua
-- Precise wait (does not throttle)
task.wait(0.5)

-- Spawn new thread immediately next resumption step
task.spawn(function()
    doSomething()
end)

-- Deferred: runs after current frame's work completes
task.defer(function()
    updateUI()
end)

-- Precise delay
task.delay(5, function()
    expireOffer()
end)
```

---

## Server-Authoritative Design

All gameplay-critical state lives on the server. Clients request actions; servers
validate and apply them.

```lua
-- Server (TradeHandler.server.luau)
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TradeRemote = ReplicatedStorage.Shared.Remotes.InitiateTrade

TradeRemote.OnServerEvent:Connect(function(player: Player, targetId: number, commodityId: string, quantity: number)
    -- 1. Validate types
    if type(targetId) ~= "number" then return end
    if type(commodityId) ~= "string" then return end
    if type(quantity) ~= "number" or quantity <= 0 or quantity > 1000 then return end

    -- 2. Validate game state
    local playerData = getPlayerData(player.UserId)
    if not playerData then return end
    if playerData.commodities[commodityId] < quantity then return end

    -- 3. Execute trade server-side
    applyTrade(player, targetId, commodityId, quantity)
end)
```

---

## DataStore Best Practices

```lua
--!strict

local DataStoreService = game:GetService("DataStoreService")
local playerStore = DataStoreService:GetDataStore("PlayerData_v1")

-- Load with pcall
local function loadPlayerData(userId: number): (boolean, any)
    local success, result = pcall(function()
        return playerStore:GetAsync(tostring(userId))
    end)
    return success, result
end

-- Save with UpdateAsync (atomic, race-condition safe)
local function savePlayerData(userId: number, newData: table): boolean
    local success, err = pcall(function()
        playerStore:UpdateAsync(tostring(userId), function(current)
            -- Merge strategy: preserve server-side fields
            if current == nil then return newData end
            for k, v in pairs(newData) do
                current[k] = v
            end
            return current
        end)
    end)
    if not success then
        warn("DataStore save failed for " .. userId .. ": " .. tostring(err))
    end
    return success
end

-- Auto-save loop (server script)
local AUTO_SAVE_INTERVAL = 60 -- seconds
task.spawn(function()
    while true do
        task.wait(AUTO_SAVE_INTERVAL)
        for _, player in Players:GetPlayers() do
            savePlayerData(player.UserId, getCurrentData(player))
        end
    end
end)

-- Save on leave
Players.PlayerRemoving:Connect(function(player: Player)
    savePlayerData(player.UserId, getCurrentData(player))
end)

-- Save on server close
game:BindToClose(function()
    for _, player in Players:GetPlayers() do
        savePlayerData(player.UserId, getCurrentData(player))
    end
end)
```

---

## RemoteEvent Patterns

### Fire-and-Forget (Action Request)

```lua
-- Client fires, server validates
-- ReplicatedStorage/Shared/Remotes/DockAtStation (RemoteEvent)

-- Client
DockAtStation:FireServer(stationId)

-- Server
DockAtStation.OnServerEvent:Connect(function(player: Player, stationId: string)
    -- validate stationId type and range
    -- validate player is close enough
    -- execute docking
end)
```

### Server-to-Client State Update

```lua
-- Server pushes state to specific client
UpdateMarketPrices:FireClient(player, priceTable)

-- Or broadcast to all (use sparingly — consider batching)
UpdateMarketPrices:FireAllClients(priceTable)
```

### UnreliableRemoteEvent (Cosmetics Only)

```lua
-- For visual effects, particle bursts, sound triggers that can be dropped
-- Do NOT use for state-critical data
local thrusterFX: UnreliableRemoteEvent = ReplicatedStorage.Remotes.ThrusterFX
thrusterFX:FireAllClients(shipId, thrustLevel)
```

---

## CollectionService Tags (Entity Classification)

Prefer CollectionService tags over folder hierarchies for runtime entity queries.

```lua
--!strict
local CollectionService = game:GetService("CollectionService")

-- Tag instances in Studio or via script
CollectionService:AddTag(stationPart, "SpaceStation")
CollectionService:AddTag(shipModel, "PlayerShip")

-- Query tagged instances
local stations = CollectionService:GetTagged("SpaceStation")

-- React to tags being added (useful for plug-in pattern)
CollectionService:GetInstanceAddedSignal("SpaceStation"):Connect(function(instance)
    initializeStation(instance)
end)
```

---

## Attributes for Lightweight Instance Data

Use `Attributes` for small, replicated key-value data on instances. Avoids
`Value` object proliferation.

```lua
-- Set on server (replicates to client automatically)
stationPart:SetAttribute("StationType", "Trading")
stationPart:SetAttribute("Tier", 2)
stationPart:SetAttribute("OwnerUserId", player.UserId)

-- Read on either side
local stationType: string = stationPart:GetAttribute("StationType")
local tier: number = stationPart:GetAttribute("Tier")

-- Listen for changes (client)
stationPart:GetAttributeChangedSignal("Tier"):Connect(function()
    updateStationUI(stationPart)
end)
```

---

## Event Cleanup (Memory Leak Prevention)

Always disconnect event connections when the listening object is destroyed.

```lua
--!strict

-- Simple cleanup pattern
local connections: {RBXScriptConnection} = {}

table.insert(connections, someEvent:Connect(function()
    doSomething()
end))

-- On cleanup (e.g., when player leaves or UI closes)
local function cleanup()
    for _, conn in connections do
        conn:Disconnect()
    end
    table.clear(connections)
end

-- Or use a Maid/Trove-style utility if adopted by the project
```

---

## Performance Patterns

### Avoid Per-Frame GetService

```lua
-- ❌ Expensive every frame
RunService.Heartbeat:Connect(function()
    local ws = game:GetService("Workspace")
end)

-- ✅ Cached
local Workspace = game:GetService("Workspace")
RunService.Heartbeat:Connect(function()
    -- use Workspace directly
end)
```

### Cache FindFirstChild Results

```lua
-- ❌ Traversal every heartbeat
RunService.Heartbeat:Connect(function()
    local dock = workspace:FindFirstChild("Station"):FindFirstChild("Dock")
end)

-- ✅ Cache on setup
local dock = workspace:WaitForChild("Station", 10):WaitForChild("Dock", 10)
RunService.Heartbeat:Connect(function()
    if dock then -- guard for StreamingEnabled
        -- use dock
    end
end)
```

### Spatial Queries Instead of Part Iteration

```lua
-- ❌ Iterates all descendants
for _, part in workspace:GetDescendants() do
    if part:IsA("BasePart") and isNearShip(part) then ... end
end

-- ✅ Spatial query
local overlapParams = OverlapParams.new()
overlapParams.FilterType = Enum.RaycastFilterType.Include
overlapParams.FilterDescendantsInstances = {workspace.Ships}
local parts = workspace:GetPartBoundsInRadius(shipPos, 100, overlapParams)
```

---

## Summary: Roblox Tech Stack for This Project

| Feature | Use This | Avoid This |
|---------|----------|------------|
| **Scheduling** | `task.wait/spawn/delay/defer` | `wait()`, `spawn()`, `delay()` |
| **DataStore writes** | `UpdateAsync` | `SetAsync` |
| **Networking** | `RemoteEvent` | `RemoteFunction:InvokeClient` |
| **Physics movement** | `LinearVelocity`, `AlignPosition` | `BodyVelocity`, `BodyPosition` |
| **Entity tagging** | `CollectionService` tags | Folder hierarchies |
| **Instance data** | `Attributes` | `IntValue`/`StringValue` objects |
| **Colors** | `Color3` | `BrickColor` |
| **Cosmetic FX network** | `UnreliableRemoteEvent` | `FireAllClients` for every FX |
| **Type safety** | `--!strict` + annotations | `--!nocheck` or untyped |

---

**Sources:**
- https://create.roblox.com/docs/luau/task
- https://create.roblox.com/docs/cloud-services/data-stores
- https://create.roblox.com/docs/scripting/events/remote
- https://create.roblox.com/docs/reference/engine/classes/CollectionService
