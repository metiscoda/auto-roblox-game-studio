# Roblox — Navigation Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may not know `PathfindingModifier` or `PathfindingLink` for custom navigation

---

## Overview

Roblox navigation systems:

- **`PathfindingService`**: A* pathfinding for humanoid/custom agents
- **`PathfindingModifier`**: Mark parts as preferred/avoided/impassable
- **`PathfindingLink`**: Custom jump/teleport links between surfaces
- **`Humanoid:MoveTo()`**: Simple target-based character movement
- **`NavigationMesh`**: Auto-computed from geometry; no manual baking needed

For a **space tycoon**: Pathfinding is relevant for NPCs/crew moving inside stations,
and potentially for AI trader ships navigating between waypoints (though space
navigation usually uses direct movement + obstacle avoidance, not grid pathfinding).

---

## PathfindingService Quick Start

```lua
--!strict
local PathfindingService = game:GetService("PathfindingService")

-- Create a path
local path = PathfindingService:CreatePath({
    AgentRadius = 2,       -- character collision radius
    AgentHeight = 5,       -- character height
    AgentCanJump = true,
    AgentCanClimb = false,
    WaypointSpacing = 4,   -- distance between waypoints
    Costs = {              -- custom material costs (optional)
        Water = 20,
        Metal = 1,
    },
})
```

---

## Computing and Following a Path

```lua
--!strict
local PathfindingService = game:GetService("PathfindingService")

local function moveNPCToTarget(npc: Model, targetPosition: Vector3)
    local humanoid = npc:FindFirstChildOfClass("Humanoid")
    local rootPart = npc:FindFirstChild("HumanoidRootPart") :: BasePart
    if not humanoid or not rootPart then return end

    local path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        WaypointSpacing = 4,
    })

    -- Compute the path
    local success, err = pcall(function()
        path:ComputeAsync(rootPart.Position, targetPosition)
    end)

    if not success or path.Status ~= Enum.PathStatus.Success then
        warn("Pathfinding failed:", err or path.Status)
        return
    end

    local waypoints = path:GetWaypoints()
    local currentWaypointIndex = 1

    -- Move to each waypoint
    local reachedConnection: RBXScriptConnection
    local blockedConnection: RBXScriptConnection

    -- Handle path blocked (dynamic obstacle appeared)
    blockedConnection = path.Blocked:Connect(function(blockedWaypointIndex: number)
        if blockedWaypointIndex >= currentWaypointIndex then
            blockedConnection:Disconnect()
            reachedConnection:Disconnect()
            -- Recompute path
            moveNPCToTarget(npc, targetPosition)
        end
    end)

    -- Move through waypoints
    reachedConnection = humanoid.MoveToFinished:Connect(function(reached: boolean)
        if reached and currentWaypointIndex < #waypoints then
            currentWaypointIndex += 1
            local wp = waypoints[currentWaypointIndex]

            -- Handle jump waypoints
            if wp.Action == Enum.PathWaypointAction.Jump then
                humanoid.Jump = true
            end

            humanoid:MoveTo(wp.Position)
        else
            reachedConnection:Disconnect()
            blockedConnection:Disconnect()
        end
    end)

    -- Start moving to first waypoint
    humanoid:MoveTo(waypoints[2].Position)  -- index 1 is start position
end
```

---

## PathfindingModifier (Mark Areas)

Attach a `PathfindingModifier` to a part to influence pathfinding costs.

```lua
-- Mark a hazardous area as expensive to cross
local modifier = Instance.new("PathfindingModifier")
modifier.Label = "Hazard"          -- matches Costs table key in CreatePath
modifier.PassThrough = false        -- false = impassable barrier
modifier.Parent = hazardZonePart

-- Or use a label for a soft cost:
modifier.Label = "Water"            -- agents prefer to avoid but can cross
modifier.PassThrough = true
modifier.Parent = waterPart

-- In CreatePath:
path = PathfindingService:CreatePath({
    Costs = {
        Water = 20,   -- 20x more expensive than default ground
        Hazard = math.huge,  -- impassable
    }
})
```

---

## PathfindingLink (Custom Traversal)

Links let agents jump across gaps, use elevators, or teleport between areas.

```lua
-- Create a link between two Attachments
local link = Instance.new("PathfindingLink")
link.Attachment0 = att0  -- start of the link
link.Attachment1 = att1  -- end of the link
link.Label = "JumpToLedge"
link.IsBidirectional = false  -- one-way: att0 → att1 only
link.Parent = workspace
```

---

## Humanoid:MoveTo (Simple NPC Movement)

For simple NPC movement without full pathfinding (e.g., move to a docking port):

```lua
local humanoid: Humanoid = npc.Humanoid

-- Move to position (humanoid navigates around parts automatically if simple)
humanoid:MoveTo(targetPosition)

-- Wait for arrival (with timeout)
local arrived = humanoid.MoveToFinished:Wait()
-- arrived = true if reached, false if timed out (8 second default)

-- Or with explicit timeout:
local reachedEvent = humanoid.MoveToFinished
task.delay(8, function()
    -- timeout handler
end)
humanoid.MoveToFinished:Wait()
```

---

## Space Station Interior Navigation

For crew/NPC movement inside stations:

```lua
--!strict
-- Waypoint-based patrol for station crew (simpler than full pathfinding)

type PatrolAgent = {
    model: Model,
    humanoid: Humanoid,
    waypoints: {BasePart},
    currentIndex: number,
}

local function createPatrol(npcModel: Model, waypointParts: {BasePart}): PatrolAgent
    return {
        model = npcModel,
        humanoid = npcModel:FindFirstChildOfClass("Humanoid") :: Humanoid,
        waypoints = waypointParts,
        currentIndex = 1,
    }
end

local function tickPatrol(agent: PatrolAgent)
    local target = agent.waypoints[agent.currentIndex]
    agent.humanoid:MoveTo(target.Position)

    agent.humanoid.MoveToFinished:Connect(function()
        agent.currentIndex = (agent.currentIndex % #agent.waypoints) + 1
        tickPatrol(agent)
    end)
end
```

---

## Space Ship Navigation (Non-Pathfinding)

For spacecraft, classic A* pathfinding is NOT used. Use waypoint/steering approach:

```lua
--!strict
-- Waypoint-based ship navigation for AI trader ships

type ShipNavigator = {
    rootPart: BasePart,
    waypoints: {Vector3},
    currentIndex: number,
    speed: number,
}

local ARRIVAL_THRESHOLD = 20  -- studs

local function tickShipNav(nav: ShipNavigator, dt: number)
    if nav.currentIndex > #nav.waypoints then return end

    local target = nav.waypoints[nav.currentIndex]
    local pos = nav.rootPart.Position
    local direction = (target - pos)
    local distance = direction.Magnitude

    if distance < ARRIVAL_THRESHOLD then
        nav.currentIndex += 1
        return
    end

    -- Move toward waypoint
    local velocity = direction.Unit * nav.speed
    nav.rootPart.AssemblyLinearVelocity = velocity
end
```

---

## Character Movement Properties

```lua
--!strict
-- Configure humanoid movement (NPC or player)
local humanoid: Humanoid = character.Humanoid

humanoid.WalkSpeed = 16          -- studs per second (default: 16)
humanoid.JumpPower = 50          -- jump velocity (default: 50)
humanoid.MaxSlopeAngle = 89      -- max climbable slope in degrees

-- Disable jumping for station crew
humanoid.JumpEnabled = false

-- Disable climbing
humanoid.AutoJumpEnabled = false
```

---

## Performance Tips

- `PathfindingService:CreatePath()` is cheap; reuse path objects for the same agent
- `ComputeAsync` is the expensive call — don't call it every frame
- Recompute paths only when blocked or target moves significantly
- For large stations, consider dividing into zones with precalculated connection graphs
- NPC patrols within a small area: use `MoveTo` + simple waypoints, not full pathfinding
- Disable `CanQuery` on decorative parts to reduce pathfinding mesh complexity

---

**Sources:**
- https://create.roblox.com/docs/characters/pathfinding
- https://create.roblox.com/docs/reference/engine/classes/PathfindingService
- https://create.roblox.com/docs/reference/engine/classes/PathfindingModifier
- https://create.roblox.com/docs/reference/engine/classes/PathfindingLink
