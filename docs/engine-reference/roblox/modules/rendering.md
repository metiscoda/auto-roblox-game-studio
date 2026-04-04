# Roblox — Rendering Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may not know Future lighting mode details, Atmosphere system, or SurfaceAppearance PBR workflow

---

## Overview

Roblox uses a proprietary renderer. The key rendering systems are:

- **Lighting technology**: Voxel (default), Shadow Map, Future (path-traced-like)
- **Materials**: Enum-based + SurfaceAppearance (PBR textures)
- **Sky/Atmosphere**: Sky object + Atmosphere object
- **Post-processing**: ColorCorrectionEffect, BloomEffect, BlurEffect, SunRaysEffect, DepthOfFieldEffect
- **Streaming**: StreamingEnabled for large worlds (required for space tycoon scale)

---

## Lighting Technology

Set via `Lighting.Technology` (Enum.Technology):

| Mode | Quality | Performance | Notes |
|------|---------|-------------|-------|
| `Voxel` | Low | Best | Default; global illumination per voxel |
| `ShadowMap` | Medium | Good | Hard shadows from directional light |
| `Future` | High | Heavy | Soft shadows, ambient occlusion; recommended for quality builds |

```lua
-- Set lighting technology (Studio only, not scriptable at runtime)
-- Set in Studio: Lighting > Technology property in Properties panel
-- Project uses Voxel (configured in technical-preferences.md)
```

**For this project:** Voxel lighting is configured. Future lighting can be evaluated
for space scenes if performance allows.

---

## Lighting Service Properties

```lua
--!strict
local Lighting = game:GetService("Lighting")

-- Key properties
Lighting.Ambient = Color3.fromRGB(100, 100, 130)  -- ambient light color
Lighting.OutdoorAmbient = Color3.fromRGB(30, 30, 60)
Lighting.Brightness = 2                             -- global light multiplier
Lighting.ClockTime = 14                             -- 0-24 hour
Lighting.GeographicLatitude = 0
Lighting.FogEnd = 1000                              -- fog distance
Lighting.FogColor = Color3.fromRGB(10, 10, 30)

-- For space: set dark ambient, bright stars
Lighting.Ambient = Color3.fromRGB(5, 5, 10)
Lighting.OutdoorAmbient = Color3.fromRGB(5, 5, 10)
Lighting.Brightness = 0
```

---

## Atmosphere Object

`Atmosphere` is a child of `Lighting`. Creates volumetric haze/glow.

```lua
-- Create or access Atmosphere
local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
if not atmosphere then
    atmosphere = Instance.new("Atmosphere")
    atmosphere.Parent = Lighting
end

atmosphere.Density = 0.3       -- 0-1; thickness of atmosphere
atmosphere.Offset = 0.25       -- 0-1; where density starts
atmosphere.Color = Color3.fromRGB(199, 170, 140)
atmosphere.Decay = Color3.fromRGB(106, 112, 125)
atmosphere.Glare = 0           -- 0-1; sun glare intensity
atmosphere.Haze = 1.5          -- 0-10; haze scattering
```

**For space:** Low density, dark decay, subtle glare from stars.

---

## Sky Object

```lua
-- Custom skybox
local sky = Instance.new("Sky")
sky.SkyboxBk = "rbxassetid://123456" -- back face
sky.SkyboxDn = "rbxassetid://123456" -- down face
sky.SkyboxFt = "rbxassetid://123456" -- front face
sky.SkyboxLf = "rbxassetid://123456" -- left face
sky.SkyboxRt = "rbxassetid://123456" -- right face
sky.SkyboxUp = "rbxassetid://123456" -- up face
sky.StarCount = 3000               -- visible stars
sky.Parent = Lighting
```

---

## SurfaceAppearance (PBR Materials)

Attach a `SurfaceAppearance` to a `BasePart` for PBR texturing.

```lua
local sa = Instance.new("SurfaceAppearance")
sa.ColorMap = "rbxassetid://123456"    -- albedo/diffuse
sa.NormalMap = "rbxassetid://123457"   -- normal map
sa.MetalnessMap = "rbxassetid://123458"
sa.RoughnessMap = "rbxassetid://123459"
sa.TextureSizeX = 4                    -- repeat scale
sa.TextureSizeY = 4
sa.Parent = myPart
```

Use `SurfaceAppearance` for ship hulls, station panels, metal structures.

---

## Post-Processing Effects

Children of `Lighting`. All are scriptable at runtime.

```lua
local Lighting = game:GetService("Lighting")

-- Bloom (glow on bright areas)
local bloom = Instance.new("BloomEffect")
bloom.Intensity = 0.5
bloom.Size = 24
bloom.Threshold = 0.95
bloom.Parent = Lighting

-- Color correction (tone mapping)
local cc = Instance.new("ColorCorrectionEffect")
cc.Brightness = 0.05
cc.Contrast = 0.1
cc.Saturation = -0.1  -- slightly desaturate for space feel
cc.TintColor = Color3.fromRGB(210, 210, 255)
cc.Parent = Lighting

-- Sun rays
local sunRays = Instance.new("SunRaysEffect")
sunRays.Intensity = 0.1
sunRays.Spread = 0.5
sunRays.Parent = Lighting

-- Depth of field (blur background)
local dof = Instance.new("DepthOfFieldEffect")
dof.FarIntensity = 0.25
dof.FocusDistance = 50
dof.InFocusRadius = 30
dof.NearIntensity = 0.5
dof.Parent = Lighting
```

---

## Part Rendering Properties

```lua
-- Key rendering properties on BasePart
part.Material = Enum.Material.SmoothPlastic  -- or Metal, Neon, Glass, etc.
part.Color = Color3.fromRGB(80, 80, 100)
part.Reflectance = 0.3   -- 0-1; affects shininess (pre-SurfaceAppearance)
part.Transparency = 0    -- 0-1; 0=opaque, 1=invisible

-- CastShadow: disable for small parts to improve shadow performance
part.CastShadow = false

-- Neon material emits light (use for engine glows, station lights)
engineGlow.Material = Enum.Material.Neon
engineGlow.Color = Color3.fromRGB(0, 150, 255)
```

---

## StreamingEnabled

Critical for large space worlds. Set in `Workspace.StreamingEnabled = true`.

```lua
-- Workspace streaming properties (set in Studio)
-- Workspace.StreamingEnabled = true
-- Workspace.StreamingMinRadius = 64   -- always loaded radius
-- Workspace.StreamingTargetRadius = 1024  -- target loaded radius
-- Workspace.StreamingPauseMode = Enum.StreamingPauseMode.Default

-- ALWAYS guard part access when streaming is enabled
local part = workspace:FindFirstChild("ShipPart")
if part then
    -- safe to use
end
```

---

## SelectionBox / Highlight (UI Overlay)

```lua
-- Highlight an instance (no external texture needed)
local highlight = Instance.new("SelectionBox")
highlight.Adornee = targetPart
highlight.Color3 = Color3.fromRGB(0, 200, 255)
highlight.LineThickness = 0.05
highlight.Parent = workspace

-- Or use Highlight (newer, cleaner)
local hl = Instance.new("Highlight")
hl.Adornee = targetModel
hl.FillColor = Color3.fromRGB(0, 200, 255)
hl.FillTransparency = 0.7
hl.OutlineColor = Color3.fromRGB(0, 200, 255)
hl.OutlineTransparency = 0
hl.Parent = workspace
```

---

## Particle Effects

```lua
local emitter = Instance.new("ParticleEmitter")
emitter.Texture = "rbxassetid://123456"
emitter.Rate = 50           -- particles per second
emitter.Speed = NumberRange.new(5, 15)
emitter.Lifetime = NumberRange.new(0.5, 2)
emitter.Size = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0.2),
    NumberSequenceKeypoint.new(1, 0),
})
emitter.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 200, 50)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 50, 0)),
})
emitter.Parent = thrusterPart

-- Burst (one-shot, e.g. explosion)
emitter:Emit(100) -- emit 100 particles instantly
```

---

## Billboards and Beams

```lua
-- BillboardGui: world-space UI always facing camera
local billboard = Instance.new("BillboardGui")
billboard.Adornee = stationPart
billboard.Size = UDim2.new(0, 200, 0, 50)
billboard.AlwaysOnTop = false
billboard.Parent = playerGui

-- Beam: line between two Attachments
local beam = Instance.new("Beam")
beam.Attachment0 = att0
beam.Attachment1 = att1
beam.Width0 = 0.5
beam.Width1 = 0.5
beam.Color = ColorSequence.new(Color3.fromRGB(0, 200, 255))
beam.Transparency = NumberSequence.new(0.3)
beam.Parent = workspace
```

---

## Performance Tips

- Disable `CastShadow` on small parts and interior geometry
- Pool `ParticleEmitter` objects — reuse instead of create/destroy
- Use `Neon` material instead of PointLight for engine glow at distance
- Limit `PointLight` count — expensive in Voxel and ShadowMap modes
- Use `SpecialMesh` + `TextureId` for flat decal textures on parts
- Keep total part count per ship/station manageable; use `UnionOperation` to merge static geometry

---

**Sources:**
- https://create.roblox.com/docs/environment/lighting
- https://create.roblox.com/docs/environment/atmosphere
- https://create.roblox.com/docs/parts/surface-appearance
- https://create.roblox.com/docs/workspace/streaming
