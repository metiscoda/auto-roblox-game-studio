# Roblox — Breaking Changes

**Last verified:** 2026-03-29

Tracks breaking API changes and behavioral differences that the LLM may not know about.
Roblox uses a rolling release; all changes are cumulative. Organized by risk level.

---

## HIGH RISK — Will Break Existing Code

### Legacy Scheduling Functions: `wait()`, `spawn()`, `delay()`

**Status:** Functionally broken under load; officially deprecated in documentation.

These functions still exist but behave poorly:
- `wait()` is throttled by the legacy scheduler and can stall for **several seconds**
  when the game is under load
- `spawn()` has an unpredictable execution order
- `delay()` is imprecise under throttling

```lua
-- ❌ DO NOT USE — broken under load
wait(1)
spawn(myFunction)
delay(2, myFunction)

-- ✅ REQUIRED in this project
task.wait(1)
task.spawn(myFunction)
task.delay(2, myFunction)
```

**This project enforces `task` library exclusively via coding standards.**

---

### BodyMover Constraints Deprecated

**Status:** `BodyVelocity`, `BodyPosition`, `BodyGyro`, `BodyAngularVelocity`,
`BodyForce`, `BodyThrust` — all deprecated in favor of the new constraint system.

The old classes still exist but are flagged deprecated in the API reference.
The new constraint classes (`LinearVelocity`, `AngularVelocity`, `AlignPosition`,
`AlignOrientation`, `VectorForce`, `Torque`) are the production path.

```lua
-- ❌ Deprecated
local bv = Instance.new("BodyVelocity")
bv.Velocity = Vector3.new(0, 50, 0)

-- ✅ New constraint system
local lv = Instance.new("LinearVelocity")
lv.VectorVelocity = Vector3.new(0, 50, 0)
lv.RelativeTo = Enum.ActuatorRelativeTo.World
lv.MaxForce = 10000
lv.Parent = part
```

---

### DataStore `SetAsync` Race Condition

**Status:** Not a recent change, but frequently misused.

`SetAsync` can silently overwrite data written by another server in the same tick.
In a multiplayer tycoon game with multiple server instances, this **will** cause data
loss. `UpdateAsync` with a merge function is the only safe pattern.

```lua
-- ❌ Race condition — DO NOT USE for player data
DataStore:SetAsync(userId, newData)

-- ✅ Safe atomic update
DataStore:UpdateAsync(userId, function(current)
    if current == nil then return newData end
    -- merge fields
    current.credits = (current.credits or 0) + deltaCredits
    return current
end)
```

---

### `RemoteFunction` Client Yield Risk

**Status:** Long-standing behavioral issue; frequently causes server stalls.

If a client invoking `RemoteFunction:InvokeServer()` disconnects, or a server calling
`RemoteFunction:InvokeClient()` has the client disconnect, the thread stalls
**indefinitely** on the server. For gameplay-critical calls, always prefer
`RemoteEvent` + callback pattern.

```lua
-- ❌ Server stalls if client disconnects
local result = RemoteFunction:InvokeClient(player, data)

-- ✅ Fire-and-forget with server-side timeout
RemoteEvent:FireClient(player, data)
-- Server waits for acknowledgment RemoteEvent with task.delay timeout
```

---

## MEDIUM RISK — Behavioral Changes

### `StreamingEnabled` PauseOutsideDistance Mode

**Added:** ~2024

`Workspace.StreamingEnabled` now supports `StreamingPauseMode`:
- `Default` — old behavior: parts simply don't exist on client outside stream range
- `Disabled` — streaming but no pause (manual handling)
- `ClientPhysicsPause` — pauses physics for parts outside range

For the space tycoon, space stations and distant ships may not be loaded on the client.
Scripts must **never** assume a part exists. Always guard with `if part then`.

```lua
-- ❌ Assumes part exists
local station = workspace.Station.Dock

-- ✅ Streaming-safe
local station = workspace:FindFirstChild("Station")
if station then
    local dock = station:FindFirstChild("Dock")
    if dock then
        -- safe to use
    end
end
```

---

### `WaitForChild` Infinite Yield is a Warning

**Status:** Now emits a console warning after ~5 seconds without a timeout argument.

Always provide a timeout. If the instance doesn't appear in time, handle the nil.

```lua
-- ❌ Infinite yield warning
local module = script.Parent:WaitForChild("Config")

-- ✅ With timeout
local module = script.Parent:WaitForChild("Config", 10)
if not module then
    warn("Config module not found in time")
    return
end
```

---

### `UnreliableRemoteEvent` Ordering

**Added:** Stabilized ~2024–2025

`UnreliableRemoteEvent` messages are not ordered and can be dropped. Do NOT use for
state-critical data. Use only for cosmetic, best-effort traffic (particle FX, sound
triggers, UI animations that can be missed).

---

### Parallel Luau (`Actor` Model)

**Status:** Beta → Stabilizing (~2025)

Scripts can run in parallel on separate threads using `Actor` instances in the
DataModel. This is powerful for CPU-heavy simulation (NPC pathfinding batches,
physics pre-computation) but has strict rules:
- Cannot read/write shared Instance properties from parallel context without
  `task.synchronize()`
- No RemoteEvent firing from parallel context
- Use `task.desynchronize()` / `task.synchronize()` to move between parallel/serial

Do NOT use Parallel Luau for core game loop logic yet. Consult `roblox-specialist`
before adopting.

---

## LOW RISK — Deprecations (Still Functional)

### `BrickColor` for Part Colors

`BrickColor` still works but is a 64-color palette. New code should use
`BasePart.Color = Color3.fromRGB(...)` for full-range color control. `BrickColor`
properties are automatically converted.

---

### `event:connect()` (lowercase)

Lowercase `:connect()` still works but `:Connect()` (uppercase) is the canonical
form. All new code must use `:Connect()`.

---

### `game.Players` vs `game:GetService("Players")`

Both work. Project standard is `game:GetService("Players")` cached at module scope.

---

## Space Tycoon-Specific Watchpoints

| Risk | Pattern | Why It Matters |
|------|---------|----------------|
| HIGH | `DataStore:SetAsync` for ship/station data | Multiple servers can write simultaneously in MMO-style games |
| HIGH | `RemoteFunction:InvokeClient` for trade confirmation | Server stalls if buyer disconnects during trade |
| HIGH | Unguarded part access with StreamingEnabled | Ships and stations may be unloaded on clients |
| MEDIUM | `wait()` in economy calculation loops | Throttle lag compounds in high-player scenarios |
| MEDIUM | Unbounded `FireAllClients` for market updates | 100+ players means 100+ messages per price tick |

---

## Migration Checklist

When reviewing or writing new code:

- [ ] All `wait()` → `task.wait()`, all `spawn()` → `task.spawn()`, all `delay()` → `task.delay()`
- [ ] All DataStore writes use `UpdateAsync` (not `SetAsync`)
- [ ] No `RemoteFunction:InvokeClient()` for gameplay-critical paths
- [ ] All `WaitForChild` calls have a timeout argument
- [ ] All part access guarded against nil (StreamingEnabled)
- [ ] No `BodyVelocity`/`BodyPosition` etc. — use new constraint system
- [ ] All event connections tracked and disconnected on cleanup

---

**Sources:**
- https://create.roblox.com/docs/luau/task
- https://create.roblox.com/docs/cloud-services/data-stores
- https://create.roblox.com/docs/workspace/streaming
- https://devforum.roblox.com/c/updates/release-notes/36
