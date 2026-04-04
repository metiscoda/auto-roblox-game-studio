# Roblox — Physics Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may suggest deprecated BodyMover classes (BodyVelocity, etc.) — use new Constraint system

---

## Overview

Roblox uses a proprietary physics engine (derived from Bullet Physics).

Key systems:
- **Rigid body simulation**: automatic for all non-Anchored parts
- **Constraints**: `HingeConstraint`, `BallSocketConstraint`, `WeldConstraint`, etc.
- **Movers (NEW)**: `LinearVelocity`, `AngularVelocity`, `AlignPosition`, `AlignOrientation`, `VectorForce`, `Torque`
- **Raycasting / Spatial Queries**: `Raycast`, `GetPartBoundsInRadius`, `GetPartBoundsInBox`
- **Collision Groups**: `PhysicsService` for per-group collision filtering

---

## Part Anchoring and Mass

```lua
--!strict

-- Anchored parts: no physics simulation (static)
part.Anchored = true

-- Unanchored parts: simulated by physics engine
part.Anchored = false

-- Mass is derived from Size and Density
part.CustomPhysicalProperties = PhysicalProperties.new(
    0.5,  -- density (default: 0.7)
    0.3,  -- friction
    0.5,  -- elasticity (bounciness)
    0.0,  -- frictionWeight
    0.0   -- elasticityWeight
)

-- Or use preset material physical properties
part.Material = Enum.Material.Metal  -- uses Metal's preset PhysicalProperties
```

---

## New Constraint Movers (REQUIRED — BodyMover Classes Deprecated)

### LinearVelocity (replaces BodyVelocity)

```lua
--!strict
local lv = Instance.new("LinearVelocity")
lv.Attachment0 = shipAttachment  -- Attachment on the part to move
lv.VectorVelocity = Vector3.new(0, 0, -50)  -- target velocity
lv.MaxForce = 100000             -- max force to apply
lv.RelativeTo = Enum.ActuatorRelativeTo.Attachment0  -- or World
lv.Parent = shipRootPart
```

### AngularVelocity (replaces BodyAngularVelocity)

```lua
local av = Instance.new("AngularVelocity")
av.Attachment0 = shipAttachment
av.AngularVelocity = Vector3.new(0, 1, 0)  -- rad/s
av.MaxTorque = 50000
av.ReactionTorqueEnabled = false
av.RelativeTo = Enum.ActuatorRelativeTo.World
av.Parent = shipRootPart
```

### AlignPosition (replaces BodyPosition)

```lua
local ap = Instance.new("AlignPosition")
ap.Attachment0 = shipAttachment
ap.Attachment1 = targetAttachment  -- or set Position directly
ap.Position = Vector3.new(100, 50, 200)
ap.MaxForce = 100000
ap.MaxVelocity = 50
ap.Responsiveness = 10  -- higher = stiffer
ap.Parent = shipRootPart
```

### AlignOrientation (replaces BodyGyro)

```lua
local ao = Instance.new("AlignOrientation")
ao.Attachment0 = shipAttachment
ao.Attachment1 = targetAttachment
ao.MaxTorque = 50000
ao.MaxAngularVelocity = 10
ao.Responsiveness = 10
ao.Parent = shipRootPart
```

### VectorForce (replaces BodyForce/BodyThrust)

```lua
local vf = Instance.new("VectorForce")
vf.Attachment0 = shipAttachment
vf.Force = Vector3.new(0, 0, -10000)  -- continuous force
vf.RelativeTo = Enum.ActuatorRelativeTo.Attachment0
vf.Parent = shipRootPart
```

---

## WeldConstraint (Rigid connection)

```lua
-- Weld two parts together rigidly (common for ship assembly)
local weld = Instance.new("WeldConstraint")
weld.Part0 = shipHull
weld.Part1 = engineNacelle
weld.Parent = shipModel

-- Or use Motor6D for animated joints (character rigs)
```

---

## Raycasting

```lua
--!strict

local params = RaycastParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.FilterDescendantsInstances = {playerShip}  -- ignore self

local origin = shipRootPart.Position
local direction = shipRootPart.CFrame.LookVector * 500

local result: RaycastResult? = workspace:Raycast(origin, direction, params)

if result then
    local hitPart = result.Instance
    local hitPos = result.Position
    local hitNormal = result.Normal
    local hitMaterial = result.Material
    print("Hit:", hitPart.Name, "at", hitPos)
end
```

---

## Spatial Queries

```lua
-- Parts within a radius (spherical)
local overlapParams = OverlapParams.new()
overlapParams.FilterType = Enum.RaycastFilterType.Include
overlapParams.FilterDescendantsInstances = {workspace.Ships, workspace.Stations}

local nearbyParts = workspace:GetPartBoundsInRadius(
    shipRootPart.Position,
    200,  -- radius in studs
    overlapParams
)

for _, part in nearbyParts do
    print("Nearby:", part.Name)
end

-- Parts in a box
local cframe = CFrame.new(0, 0, 0)
local halfSize = Vector3.new(100, 100, 100)
local boxParts = workspace:GetPartBoundsInBox(cframe, halfSize * 2, overlapParams)

-- Check if any part overlaps a box
local hasCollision = workspace:ArePartsTouchingOthers(
    {shipPart},
    0.001  -- overlap tolerance
)
```

---

## Collision Groups

```lua
--!strict
local PhysicsService = game:GetService("PhysicsService")

-- Register collision groups (server, usually in init script)
PhysicsService:RegisterCollisionGroup("PlayerShips")
PhysicsService:RegisterCollisionGroup("EnemyProjectiles")
PhysicsService:RegisterCollisionGroup("Stations")

-- Configure which groups collide
PhysicsService:CollisionGroupSetCollidable("PlayerShips", "PlayerShips", false)  -- ships don't collide with each other
PhysicsService:CollisionGroupSetCollidable("PlayerShips", "Stations", true)
PhysicsService:CollisionGroupSetCollidable("EnemyProjectiles", "PlayerShips", true)

-- Assign parts to groups
part.CollisionGroupId = PhysicsService:GetRegisteredCollisionGroups()
-- Or by name:
part.CollisionGroup = "PlayerShips"  -- direct string assignment (newer API)
```

---

## Physics Events

```lua
-- Touched: fires when parts make contact
part.Touched:Connect(function(otherPart: BasePart)
    -- NOTE: fires very frequently; debounce required
    handleCollision(part, otherPart)
end)

-- TouchEnded: fires when parts separate
part.TouchEnded:Connect(function(otherPart: BasePart)
    handleSeparation(part, otherPart)
end)

-- Debounce pattern (prevent rapid-fire collision events)
local lastTouched: {[string]: number} = {}
local DEBOUNCE_TIME = 0.5

part.Touched:Connect(function(otherPart: BasePart)
    local key = otherPart:GetFullName()
    local now = os.clock()
    if (lastTouched[key] or 0) + DEBOUNCE_TIME > now then return end
    lastTouched[key] = now
    handleCollision(part, otherPart)
end)
```

---

## Simulation Rate

```lua
-- Workspace.PhysicsSteppingMethod controls the simulation update method
-- Default: "Adaptive" (adjusts based on load)
-- "Fixed" (fixed timestep, more deterministic)
-- Set in Studio: Workspace properties

-- Access physics step time
local RunService = game:GetService("RunService")
RunService.Stepped:Connect(function(time: number, deltaTime: number)
    -- deltaTime is physics step time
    updateShipPhysics(deltaTime)
end)
```

---

## Ship Movement Pattern (Space Tycoon)

Recommended pattern for player-controlled spacecraft using the new constraint system:

```lua
--!strict
-- Server-side ship controller module

local Ships = {}

type ShipState = {
    linearVelocity: LinearVelocity,
    angularVelocity: AngularVelocity,
    targetSpeed: number,
    targetAngular: Vector3,
}

function Ships.createShipMover(rootPart: BasePart): ShipState
    local att = Instance.new("Attachment")
    att.Parent = rootPart

    local lv = Instance.new("LinearVelocity")
    lv.Attachment0 = att
    lv.VectorVelocity = Vector3.zero
    lv.MaxForce = 200000
    lv.RelativeTo = Enum.ActuatorRelativeTo.Attachment0
    lv.Parent = rootPart

    local av = Instance.new("AngularVelocity")
    av.Attachment0 = att
    av.AngularVelocity = Vector3.zero
    av.MaxTorque = 100000
    av.ReactionTorqueEnabled = false
    av.Parent = rootPart

    return { linearVelocity = lv, angularVelocity = av, targetSpeed = 0, targetAngular = Vector3.zero }
end

function Ships.setThrust(state: ShipState, forward: number, up: number, right: number)
    state.linearVelocity.VectorVelocity = Vector3.new(right, up, -forward) * state.targetSpeed
end

return Ships
```

---

## Performance Tips

- `Anchored = true` on all static parts (stations, asteroids) — no physics simulation cost
- Disable `CanCollide` on parts that don't need collision detection (visual-only geometry)
- Disable `CanQuery` on parts that should not be raycasted against
- Use `CollisionGroup` to prevent unnecessary collision detection between allied ships
- Pool physics constraints; reuse instead of destroy/recreate
- Avoid `part.Touched` in hot paths — use spatial queries for proximity detection instead

---

**Sources:**
- https://create.roblox.com/docs/physics/constraints
- https://create.roblox.com/docs/physics/raycasting
- https://create.roblox.com/docs/reference/engine/classes/PhysicsService
- https://create.roblox.com/docs/reference/engine/classes/LinearVelocity
