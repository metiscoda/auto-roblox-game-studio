# Roblox — UI Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM knows Roblox GUI system well; main gaps are AutomaticSize behavior and newer UIFlexItem

---

## Overview

Roblox GUI systems:

| Class | Use Case |
|-------|---------|
| `ScreenGui` | HUD, menus, overlay UI (2D screen-space) |
| `BillboardGui` | World-space UI on a part, always faces camera |
| `SurfaceGui` | UI rendered on a part's surface (flat, world-space) |
| `GuiObject` subtypes | `Frame`, `TextLabel`, `TextButton`, `TextBox`, `ImageLabel`, `ImageButton`, `ScrollingFrame`, `VideoFrame` |

All UI **must be created and modified from LocalScripts** (client side only).
Never create or modify GUI from server scripts.

---

## ScreenGui Setup

```lua
--!strict
-- In StarterGui (Rojo: game-rojo/src/client/UI/)

-- ScreenGui properties
local gui = Instance.new("ScreenGui")
gui.Name = "HUD"
gui.ResetOnSpawn = false     -- IMPORTANT: keep UI across respawns
gui.IgnoreGuiInset = false   -- true = render behind topbar
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling  -- ZIndex is relative to siblings
gui.Parent = Players.LocalPlayer.PlayerGui
```

---

## Sizing: Scale vs Offset

Always prefer **Scale** for responsive layouts. Use **Offset** only for fixed pixel sizes.

```lua
-- ❌ Offset only — breaks on different screen sizes
frame.Size = UDim2.new(0, 400, 0, 200)  -- 400x200 pixels

-- ✅ Scale — responsive
frame.Size = UDim2.new(0.3, 0, 0.15, 0)  -- 30% screen width, 15% screen height

-- ✅ Mixed — fixed pixel padding with scale size
button.Size = UDim2.new(0.2, -16, 0, 44)  -- 20% width minus 8px each side

-- UDim2 constructor: UDim2.new(xScale, xOffset, yScale, yOffset)
```

---

## Layout Containers

### UIListLayout

```lua
local list = Instance.new("UIListLayout")
list.FillDirection = Enum.FillDirection.Vertical
list.HorizontalAlignment = Enum.HorizontalAlignment.Center
list.VerticalAlignment = Enum.VerticalAlignment.Top
list.Padding = UDim.new(0, 8)  -- 8px gap between items
list.SortOrder = Enum.SortOrder.LayoutOrder
list.Parent = container
```

### UIGridLayout

```lua
local grid = Instance.new("UIGridLayout")
grid.CellSize = UDim2.new(0, 80, 0, 80)
grid.CellPadding = UDim2.new(0, 4, 0, 4)
grid.FillDirection = Enum.FillDirection.Horizontal
grid.SortOrder = Enum.SortOrder.LayoutOrder
grid.Parent = inventoryFrame
```

### UIPadding

```lua
local padding = Instance.new("UIPadding")
padding.PaddingTop = UDim.new(0, 8)
padding.PaddingBottom = UDim.new(0, 8)
padding.PaddingLeft = UDim.new(0, 12)
padding.PaddingRight = UDim.new(0, 12)
padding.Parent = frame
```

### UIAspectRatioConstraint

```lua
-- Force a frame to maintain aspect ratio
local arc = Instance.new("UIAspectRatioConstraint")
arc.AspectRatio = 16/9
arc.AspectType = Enum.AspectType.FitWithinMaxSize
arc.DominantAxis = Enum.DominantAxis.Width
arc.Parent = frame
```

---

## AutomaticSize

```lua
-- Auto-size a frame to fit its children
frame.AutomaticSize = Enum.AutomaticSize.XY  -- or X, Y

-- Use with UIListLayout to make containers grow dynamically
-- NOTE: Set initial Size to UDim2.new(0,0,0,0) for full auto-size
frame.Size = UDim2.new(0, 0, 0, 0)
frame.AutomaticSize = Enum.AutomaticSize.XY
```

---

## TextLabel / TextButton

```lua
local label = Instance.new("TextLabel")
label.Text = "Credits: 1,500,000"
label.Font = Enum.Font.GothamBold        -- or GothamMedium, Gotham, RobotoMono
label.TextSize = 18
label.TextColor3 = Color3.fromRGB(255, 255, 255)
label.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
label.TextStrokeTransparency = 0.5
label.TextXAlignment = Enum.TextXAlignment.Left
label.TextScaled = false                  -- prefer explicit TextSize over TextScaled
label.BackgroundTransparency = 1
label.Size = UDim2.new(1, 0, 0, 24)
label.Parent = frame

-- Button
local button = Instance.new("TextButton")
button.Text = "DOCK"
button.Font = Enum.Font.GothamBold
button.TextSize = 16
button.TextColor3 = Color3.fromRGB(255, 255, 255)
button.BackgroundColor3 = Color3.fromRGB(30, 80, 200)
button.BorderSizePixel = 0
button.Size = UDim2.new(0, 120, 0, 40)

button.MouseButton1Click:Connect(function()
    onDockButtonClicked()
end)

button.Parent = frame
```

---

## UICorner (Rounded Corners)

```lua
local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)  -- 8px radius
corner.Parent = frame
```

---

## UIStroke (Border)

```lua
local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(0, 150, 255)
stroke.Thickness = 2
stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
stroke.Parent = frame
```

---

## BillboardGui (World-Space, Camera-Facing)

```lua
-- Attach to a part for a floating label (e.g., station name)
local billboard = Instance.new("BillboardGui")
billboard.Adornee = stationPart
billboard.Size = UDim2.new(0, 200, 0, 40)
billboard.StudsOffset = Vector3.new(0, 5, 0)  -- offset above part
billboard.AlwaysOnTop = false
billboard.LightInfluence = 1                   -- 0=fully lit, 1=unlit
billboard.MaxDistance = 150                    -- hide beyond 150 studs
billboard.Parent = workspace  -- or PlayerGui

local label = Instance.new("TextLabel")
label.Text = "Alpha Station"
label.TextColor3 = Color3.fromRGB(255, 255, 255)
label.BackgroundTransparency = 1
label.Size = UDim2.new(1, 0, 1, 0)
label.Parent = billboard
```

---

## SurfaceGui (Part Surface UI)

```lua
-- UI rendered onto a part's surface (info panels, screens)
local surfGui = Instance.new("SurfaceGui")
surfGui.Adornee = screenPart
surfGui.Face = Enum.NormalId.Front
surfGui.SizingMode = Enum.SurfaceGuiSizingMode.FixedXY
surfGui.CanvasSize = Vector2.new(800, 400)
surfGui.LightInfluence = 0    -- 0=bright even in darkness
surfGui.AlwaysOnTop = false
surfGui.Parent = workspace

local frame = Instance.new("Frame")
frame.Size = UDim2.new(1, 0, 1, 0)
frame.BackgroundColor3 = Color3.fromRGB(10, 20, 40)
frame.Parent = surfGui
```

---

## TweenService for UI Animation

```lua
--!strict
local TweenService = game:GetService("TweenService")

-- Slide panel in
local function showPanel(panel: Frame)
    panel.Visible = true
    local tween = TweenService:Create(
        panel,
        TweenInfo.new(0.3, Enum.EasingStyle.Quart, Enum.EasingDirection.Out),
        {Position = UDim2.new(0, 0, 0, 0)}
    )
    tween:Play()
end

-- Fade in
local function fadeIn(element: GuiObject)
    element.BackgroundTransparency = 1
    element.Visible = true
    TweenService:Create(element, TweenInfo.new(0.2), {BackgroundTransparency = 0}):Play()
end
```

---

## ScrollingFrame (Commodity List)

```lua
local scroll = Instance.new("ScrollingFrame")
scroll.Size = UDim2.new(1, 0, 1, 0)
scroll.CanvasSize = UDim2.new(0, 0, 0, 0)  -- AutomaticSize handles this
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.ScrollBarThickness = 6
scroll.ScrollBarImageColor3 = Color3.fromRGB(100, 150, 255)
scroll.BackgroundTransparency = 1
scroll.Parent = frame

local list = Instance.new("UIListLayout")
list.SortOrder = Enum.SortOrder.LayoutOrder
list.Padding = UDim.new(0, 4)
list.Parent = scroll
```

---

## TextBox (Input Fields)

```lua
local input = Instance.new("TextBox")
input.PlaceholderText = "Enter quantity..."
input.PlaceholderColor3 = Color3.fromRGB(150, 150, 150)
input.Text = ""
input.ClearTextOnFocus = true
input.Font = Enum.Font.Gotham
input.TextSize = 16
input.TextColor3 = Color3.fromRGB(255, 255, 255)
input.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
input.Parent = frame

input.FocusLost:Connect(function(enterPressed: boolean)
    if enterPressed then
        local qty = tonumber(input.Text)
        if qty then
            submitOrder(qty)
        end
    end
end)
```

---

## Z-Index Management

```lua
-- ZIndexBehavior.Sibling: ZIndex is relative to siblings (default — recommended)
-- ZIndexBehavior.Global: ZIndex is global (legacy)

-- Set on ScreenGui:
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

-- Then set ZIndex on elements to control layering within a container
overlay.ZIndex = 10
background.ZIndex = 1
```

---

## Performance Tips

- Set `Visible = false` on hidden UI — Roblox still renders invisible GuiObjects
- Avoid creating/destroying UI objects every frame — pool or toggle visibility
- Use `BackgroundTransparency = 1` instead of creating transparent frames (still renders)
- Limit deep nesting — each layer adds a render pass
- Use `GuiObject.Visible` not `GuiObject.Parent = nil` for toggling (Parent nil re-creates)

---

**Sources:**
- https://create.roblox.com/docs/ui
- https://create.roblox.com/docs/reference/engine/classes/ScreenGui
- https://create.roblox.com/docs/reference/engine/classes/BillboardGui
- https://create.roblox.com/docs/reference/engine/classes/UIListLayout
