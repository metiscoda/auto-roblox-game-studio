# Roblox — Deprecated APIs

**Last verified:** 2026-03-29

Quick lookup table for deprecated APIs and their replacements.
Format: **Don't use X** → **Use Y instead**

---

## Scheduling / Coroutines

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `wait(n)` | `task.wait(n)` | Legacy wait is imprecise (throttled), can yield >1s |
| `spawn(fn)` | `task.spawn(fn)` | Legacy spawn deferred + throttled; task.spawn runs next step |
| `delay(n, fn)` | `task.delay(n, fn)` | Legacy delay throttled; task.delay is precise |
| `coroutine.wrap` for deferred calls | `task.spawn` / `task.defer` | Use task for Roblox scheduling; coroutines only for generators |

**Critical:** `wait()` with no argument is NOT equivalent to `task.wait()`. Legacy
`wait()` is throttled by the Roblox scheduler and can stall for multiple seconds under
load. Every script in this project **must** use the `task` library exclusively.

```lua
-- ❌ FORBIDDEN in this project
wait(1)
spawn(function() end)
delay(2, function() end)

-- ✅ REQUIRED
task.wait(1)
task.spawn(function() end)
task.delay(2, function() end)
```

---

## Player Data / Character

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `game.Players.LocalPlayer.Character` (direct, no guard) | `player.CharacterAdded:Wait()` or check nil | Character may not be loaded |
| `player:LoadCharacter()` on client | Server only via `Player:LoadCharacter()` | FilteringEnabled |
| `Player.CharacterAutoLoads = false` without cleanup | Pair with `CharacterRemoving` | Memory leak risk |

---

## Instance Queries

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Instance:FindFirstChild()` in hot loops | Cache the reference | Expensive per-call traversal |
| `Instance:WaitForChild()` without timeout | `WaitForChild(name, timeout)` | Infinite yield risk |
| `game.Workspace` | `workspace` (shorthand) or `game:GetService("Workspace")` | Both work; prefer cached service |
| Folder-based entity classification | `CollectionService` tags | Event-based, query-efficient |

---

## Networking

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `RemoteFunction` for fire-and-forget | `RemoteEvent` | RemoteFunctions block server if client yields/disconnects |
| Sending large tables over RemoteEvents | Chunk data or use `HttpService:JSONEncode` pattern | 4MB serialization limit |
| Unbounded `RemoteEvent:FireAllClients()` | Rate-limit or use `UnreliableRemoteEvent` for cosmetics | Bandwidth abuse vector |

---

## DataStore

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `DataStore:SetAsync()` for player data | `DataStore:UpdateAsync()` | SetAsync creates race conditions |
| DataStore calls outside `pcall` | Wrap every call in `pcall` | DataStore **will** error in Studio and under throttling |
| Storing `Instance` references in DataStore | Serialize to plain tables | Instances cannot be serialized |
| Storing functions or metatables | Plain data only | Not serializable |

---

## Events / Connections

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `event:connect()` | `event:Connect()` | Old lowercase method; still works but bad practice |
| Firing connections and never disconnecting | Store and `:Disconnect()` or use Maid/Trove | Memory leak in long-running games |
| `Instance.Changed:Connect()` for specific property | `Instance:GetPropertyChangedSignal("Prop"):Connect()` | `.Changed` fires on ANY property change |

---

## Rendering / Appearance

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `BrickColor` assignment for colors | `Color3` via `BasePart.Color` | BrickColor is a limited palette |
| `Part.Material = Enum.Material.SmoothPlastic` for custom look | `SurfaceAppearance` object | Full PBR texturing |
| Legacy `Sky` object only | `Atmosphere` + `Sky` + `ColorCorrectionEffect` | Modern sky system |

---

## Services

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `game:GetService()` inside loops | Cache at module scope: `local DataStore = game:GetService(...)` | Expensive per-call |
| `game.Players` (direct indexing) | `game:GetService("Players")` | Consistent service access pattern |
| `require(script.Parent.Module)` with relative paths | Canonical paths via `ReplicatedStorage.Shared` | Fragile on hierarchy changes |

---

## Physics / Movement

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `BodyVelocity`, `BodyPosition`, `BodyGyro`, `BodyAngularVelocity` | `LinearVelocity`, `AngularVelocity`, `AlignPosition`, `AlignOrientation` | Constraints deprecated |
| `BodyForce` | `VectorForce` | Same deprecation wave |
| `VehicleSeat.Steer`/`.Throttle` (scripted) | `VehicleSeat` input or custom `LinearVelocity` | Depends on use case |

---

## Input

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Mouse` object from `Player:GetMouse()` | `UserInputService` | `GetMouse()` still works; UIS preferred for new code |
| Polling input in `RunService.Heartbeat` unnecessarily | `UserInputService.InputBegan` events | Event-driven is more efficient |

---

## Quick Migration Patterns

### Scheduling
```lua
-- ❌ Deprecated
while true do
    wait(0.5)
    updateHUD()
end

-- ✅ Correct
while true do
    task.wait(0.5)
    updateHUD()
end
```

### DataStore
```lua
-- ❌ Race condition risk
dataStore:SetAsync(key, data)

-- ✅ Atomic update
dataStore:UpdateAsync(key, function(oldData)
    -- merge or replace
    return newData
end)
```

### Physics constraints
```lua
-- ❌ Deprecated constraint
local bv = Instance.new("BodyVelocity")
bv.Velocity = Vector3.new(0, 50, 0)
bv.Parent = part

-- ✅ New constraint
local lv = Instance.new("LinearVelocity")
lv.VectorVelocity = Vector3.new(0, 50, 0)
lv.RelativeTo = Enum.ActuatorRelativeTo.World
lv.Parent = part
```

---

**Sources:**
- https://create.roblox.com/docs/luau/task
- https://create.roblox.com/docs/cloud-services/data-stores
- https://create.roblox.com/docs/reference/engine/classes/BodyVelocity (deprecated notice)
