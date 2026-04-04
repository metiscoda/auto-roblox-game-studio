# Roblox — Input Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may suggest `Player:GetMouse()` — prefer `UserInputService` for new code

---

## Overview

Roblox input systems (client-side only):

- **`UserInputService`** — primary API for keyboard, mouse, gamepad, touch (RECOMMENDED)
- **`ContextActionService`** — action-binding system with priority layers (good for layered controls)
- **`Player:GetMouse()`** — legacy mouse API; still works but UIS preferred

All input handling **must be in LocalScripts** (client side).

---

## UserInputService Quick Reference

```lua
--!strict
local UserInputService = game:GetService("UserInputService")
```

### Input Events

```lua
-- InputBegan: key/button pressed, touch started
UserInputService.InputBegan:Connect(function(input: InputObject, gameProcessed: boolean)
    -- gameProcessed = true if Roblox UI consumed the input (chat box, etc.)
    if gameProcessed then return end

    if input.KeyCode == Enum.KeyCode.F then
        openMap()
    end

    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        handleLeftClick(input.Position)
    end
end)

-- InputEnded: key/button released
UserInputService.InputEnded:Connect(function(input: InputObject, gameProcessed: boolean)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.W then
        stopThrusters()
    end
end)

-- InputChanged: mouse movement, scroll, gamepad stick
UserInputService.InputChanged:Connect(function(input: InputObject, gameProcessed: boolean)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Delta  -- Vector3: x=deltaX, y=deltaY
        rotateCameraBy(delta.X, delta.Y)
    end

    if input.UserInputType == Enum.UserInputType.MouseWheel then
        zoomCamera(input.Position.Z)  -- Z = scroll direction (+1/-1)
    end
end)
```

### Polling Input State

```lua
-- Check if a key is currently held
local isThrusting = UserInputService:IsKeyDown(Enum.KeyCode.W)
local isShiftHeld = UserInputService:IsKeyDown(Enum.KeyCode.LeftShift)

-- Check mouse button state
local isRightHeld = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)

-- Mouse position on screen
local mousePos: Vector2 = UserInputService:GetMouseLocation()

-- Gamepad state (if connected)
local gamepadConnected = UserInputService:GetGamepadConnected(Enum.UserInputType.Gamepad1)
if gamepadConnected then
    local state = UserInputService:GetGamepadState(Enum.UserInputType.Gamepad1)
    for _, inputObject in state do
        if inputObject.KeyCode == Enum.KeyCode.Thumbstick1 then
            local stick: Vector3 = inputObject.Position
            -- stick.X = left/right, stick.Y = up/down
        end
    end
end
```

### Device Detection

```lua
-- Detect input method for UI hints
local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
local isGamepad = UserInputService.GamepadEnabled
local isPC = UserInputService.KeyboardEnabled and UserInputService.MouseEnabled

UserInputService.LastInputTypeChanged:Connect(function(lastInputType: Enum.UserInputType)
    if lastInputType == Enum.UserInputType.Gamepad1 then
        showGamepadHints()
    elseif lastInputType == Enum.UserInputType.Keyboard then
        showKeyboardHints()
    end
end)
```

---

## ContextActionService

Better than raw UserInputService when you need **priority layers** or **mobile
buttons**. Useful for: UI overlays that temporarily rebind keys, docking mode vs.
free-flight mode, menu layer blocking game input.

```lua
--!strict
local ContextActionService = game:GetService("ContextActionService")

-- Bind an action with a priority
local DOCK_ACTION = "DockAtStation"

ContextActionService:BindAction(
    DOCK_ACTION,
    function(actionName: string, inputState: Enum.UserInputState, inputObject: InputObject)
        if inputState == Enum.UserInputState.Begin then
            startDocking()
        end
        return Enum.ContextActionResult.Sink  -- consume input (prevent lower-priority handlers)
    end,
    true,   -- createTouchButton (shows button on mobile)
    Enum.KeyCode.E,          -- keyboard
    Enum.KeyCode.ButtonX     -- gamepad
)

-- Set button title/image for mobile
ContextActionService:SetTitle(DOCK_ACTION, "Dock")
ContextActionService:SetImage(DOCK_ACTION, "rbxassetid://123456")

-- Unbind when leaving docking range
ContextActionService:UnbindAction(DOCK_ACTION)
```

### Action Priority

```lua
-- Higher priority = processed first; use to layer UI over gameplay
ContextActionService:BindActionAtPriority(
    "MenuAction",
    handler,
    false,
    Enum.ContextActionPriority.High.Value, -- or a number; higher wins
    Enum.KeyCode.Escape
)
```

---

## Mouse Lock / Cursor Control

```lua
--!strict
local UserInputService = game:GetService("UserInputService")

-- Lock cursor to center (FPS/flight mode)
UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
-- Or lock to current position
UserInputService.MouseBehavior = Enum.MouseBehavior.LockCurrentPosition
-- Release cursor (menu mode)
UserInputService.MouseBehavior = Enum.MouseBehavior.Default

-- Hide/show cursor
UserInputService.MouseIconEnabled = false  -- hide for flight mode
UserInputService.MouseIconEnabled = true   -- show for menus
```

---

## Touch Input (Mobile)

```lua
UserInputService.TouchStarted:Connect(function(touch: InputObject, gameProcessed: boolean)
    if gameProcessed then return end
    local pos: Vector2 = touch.Position  -- screen position
end)

UserInputService.TouchMoved:Connect(function(touch: InputObject, gameProcessed: boolean)
    local delta: Vector3 = touch.Delta
end)

UserInputService.TouchEnded:Connect(function(touch: InputObject, gameProcessed: boolean)
end)

-- Pinch gesture (zoom)
UserInputService.TouchPinch:Connect(function(
    touchPositions: {Vector2},
    scale: number,
    velocity: number,
    state: Enum.UserInputState,
    gameProcessed: boolean
)
    if gameProcessed then return end
    zoomCameraByScale(scale)
end)
```

---

## Raycasting from Mouse/Touch

For clicking on 3D objects in the world (selecting ships, stations):

```lua
--!strict
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local camera = workspace.CurrentCamera

local function screenPointToWorld(screenPos: Vector2): RaycastResult?
    local ray = camera:ScreenPointToRay(screenPos.X, screenPos.Y)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {Players.LocalPlayer.Character}
    return workspace:Raycast(ray.Origin, ray.Direction * 1000, params)
end

UserInputService.InputBegan:Connect(function(input: InputObject, gameProcessed: boolean)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local hit = screenPointToWorld(UserInputService:GetMouseLocation())
        if hit then
            local part = hit.Instance
            handlePartClick(part, hit.Position)
        end
    end
end)
```

---

## Key Codes Quick Reference

| Action | KeyCode |
|--------|---------|
| Move forward | `W` |
| Move backward | `S` |
| Move left | `A` |
| Move right | `D` |
| Boost/sprint | `LeftShift` |
| Interact | `E` |
| Map | `M` |
| Inventory | `I` / `Tab` |
| Cancel/menu | `Escape` |
| Gamepad confirm | `ButtonA` |
| Gamepad cancel | `ButtonB` |
| Gamepad menu | `ButtonStart` |
| Left stick | `Thumbstick1` |
| Right stick | `Thumbstick2` |

---

## Common Patterns for Space Tycoon

### Flight Mode vs Menu Mode Input Toggle

```lua
local inMenu = false

local function setMenuMode(active: boolean)
    inMenu = active
    UserInputService.MouseBehavior = active
        and Enum.MouseBehavior.Default
        or Enum.MouseBehavior.LockCenter
    UserInputService.MouseIconEnabled = active
end

-- In-flight: lock mouse center, hide cursor
setMenuMode(false)

-- Opening trade menu
setMenuMode(true)
```

### Throttle Input Polling

```lua
-- ✅ Event-driven — preferred
UserInputService.InputBegan:Connect(onInput)

-- ✅ When polling is necessary (continuous movement), use Heartbeat
local RunService = game:GetService("RunService")
RunService.Heartbeat:Connect(function(dt: number)
    local thrustInput = UserInputService:IsKeyDown(Enum.KeyCode.W)
    if thrustInput then
        applyThrust(dt)
    end
end)
```

---

**Sources:**
- https://create.roblox.com/docs/reference/engine/classes/UserInputService
- https://create.roblox.com/docs/reference/engine/classes/ContextActionService
- https://create.roblox.com/docs/input/mobile
