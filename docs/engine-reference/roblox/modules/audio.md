# Roblox — Audio Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may not know the AudioAPI (wiring system) introduced ~2024–2025; may only know legacy Sound/SoundService

---

## Overview

Roblox audio systems:

- **Legacy Audio**: `Sound` instances with `SoundService` — simple, production-ready
- **New Audio API** (beta → stabilizing ~2024+): node-based wiring system with `AudioPlayer`, `AudioEmitter`, `AudioListener`, `AudioEffect` nodes

**For this project:** Use the **legacy Sound/SoundService** system. The new AudioAPI
is still maturing. Revisit when it graduates to production.

---

## SoundService Configuration

```lua
--!strict
local SoundService = game:GetService("SoundService")

-- Global volume settings
SoundService.MusicVolume = 0.5     -- 0-1
SoundService.AmbientReverb = Enum.ReverbType.NoReverb  -- or SmallRoom, LargeHall, etc.
SoundService.DistanceFactor = 1    -- studs-to-meters conversion (affects 3D audio)
SoundService.DopplerScale = 1      -- doppler effect intensity
SoundService.RolloffScale = 1      -- distance falloff rate
```

---

## Sound Instance

### 2D Sound (UI, Music, Ambient)

```lua
-- Parented to a service/script — no 3D spatialization
local sound = Instance.new("Sound")
sound.SoundId = "rbxassetid://123456"
sound.Volume = 0.8     -- 0-1
sound.Looped = true
sound.Parent = SoundService  -- or script, workspace, etc.
sound:Play()
```

### 3D Positional Sound (Ship Engines, Explosions)

```lua
-- Parented to a BasePart or Attachment for 3D positioning
local engineSound = Instance.new("Sound")
engineSound.SoundId = "rbxassetid://123456"
engineSound.Volume = 1
engineSound.RollOffMode = Enum.RollOffMode.InverseTapered  -- distance falloff
engineSound.RollOffMinDistance = 10   -- full volume within this distance
engineSound.RollOffMaxDistance = 200  -- silent beyond this distance
engineSound.Looped = true
engineSound.Parent = enginePart  -- follows part's position

engineSound:Play()
```

### RollOffMode Options

| Mode | Behavior |
|------|---------|
| `Inverse` | Classic inverse square falloff |
| `Linear` | Linear falloff |
| `LinearSquare` | Linear-squared falloff |
| `InverseTapered` | Inverse with smooth taper at max distance (RECOMMENDED) |

---

## Sound Control

```lua
-- Play / Stop / Pause
sound:Play()
sound:Stop()
sound:Pause()
sound:Resume()

-- Check state
local isPlaying: boolean = sound.IsPlaying
local isPaused: boolean = sound.IsPaused

-- Seek
sound.TimePosition = 2.5  -- jump to 2.5 seconds

-- Pitch shift
sound.PlaybackSpeed = 1.5  -- 1 = normal, 2 = octave up, 0.5 = octave down

-- Fade volume (manual)
local TweenService = game:GetService("TweenService")
local tween = TweenService:Create(sound, TweenInfo.new(2), {Volume = 0})
tween:Play()
tween.Completed:Connect(function()
    sound:Stop()
end)
```

---

## SoundGroup (Volume Categories)

Group sounds so players can control music vs. SFX vs. voice independently.

```lua
-- Create SoundGroups (usually in SoundService via Studio)
-- SoundService > SoundGroup "Music"
-- SoundService > SoundGroup "SFX"
-- SoundService > SoundGroup "UI"

-- Assign Sound to group
sound.SoundGroup = SoundService:FindFirstChild("Music")

-- Adjust group volume
local musicGroup = SoundService:FindFirstChild("Music") :: SoundGroup
musicGroup.Volume = 0.7
```

---

## Sound Events

```lua
-- Trigger on completion
sound.Ended:Connect(function()
    playNextTrack()
end)

-- Trigger when playback starts
sound.Playing:Connect(function()
    updateNowPlayingUI()
end)

-- Trigger on loop
sound.DidLoop:Connect(function()
    -- sound looped
end)
```

---

## Sound Effects (Filters)

Attach `SoundEffect` instances as children of a `Sound`.

```lua
-- Reverb
local reverb = Instance.new("ReverbSoundEffect")
reverb.DecayTime = 2.5
reverb.Density = 0.5
reverb.WetLevel = -6   -- dB
reverb.Parent = sound

-- Equalizer
local eq = Instance.new("EqualizerSoundEffect")
eq.LowGain = 0     -- dB
eq.MidGain = 3
eq.HighGain = -3
eq.Parent = sound

-- Pitch shift (for doppler, voice modulation)
local pitchShift = Instance.new("PitchShiftSoundEffect")
pitchShift.Octave = 0.8  -- 0-2, 1 = normal
pitchShift.Parent = sound

-- Distortion
local distort = Instance.new("DistortionSoundEffect")
distort.Level = 0.3
distort.Parent = sound

-- Compressor
local compressor = Instance.new("CompressorSoundEffect")
compressor.Threshold = -20  -- dB
compressor.Ratio = 4
compressor.Parent = sound
```

---

## Music System Pattern (Space Tycoon)

```lua
--!strict
-- MusicController.client.luau (LocalScript)

local SoundService = game:GetService("SoundService")
local RunService = game:GetService("RunService")

local TRACKS: {[string]: string} = {
    space_ambient = "rbxassetid://111111",
    docking_theme = "rbxassetid://222222",
    combat_tension = "rbxassetid://333333",
}

local currentTrack: Sound? = nil

local function playTrack(trackId: string)
    if currentTrack then
        -- Crossfade out
        local tween = game:GetService("TweenService"):Create(
            currentTrack,
            TweenInfo.new(2),
            {Volume = 0}
        )
        tween:Play()
        tween.Completed:Connect(function()
            currentTrack:Stop()
        end)
    end

    local sound = Instance.new("Sound")
    sound.SoundId = TRACKS[trackId]
    sound.Looped = true
    sound.Volume = 0
    sound.Parent = SoundService
    sound:Play()

    -- Fade in
    game:GetService("TweenService"):Create(sound, TweenInfo.new(2), {Volume = 0.6}):Play()
    currentTrack = sound
end

-- Zone-based music
local inStation = false

RunService.Heartbeat:Connect(function()
    -- Check player zone and swap music accordingly
    local nearStation = checkNearStation()
    if nearStation ~= inStation then
        inStation = nearStation
        playTrack(inStation and "docking_theme" or "space_ambient")
    end
end)
```

---

## Engine Sound Pattern (3D, Pitch-Responsive)

```lua
--!strict
-- Attach to ship root part; adjust pitch based on thrust

local engineSound: Sound = shipRoot:WaitForChild("EngineSound") :: Sound

local function updateEngineSFX(thrustLevel: number)
    -- thrustLevel: 0-1
    engineSound.PlaybackSpeed = 0.8 + thrustLevel * 0.6  -- 0.8 to 1.4x
    engineSound.Volume = 0.3 + thrustLevel * 0.5
end
```

---

## Performance Tips

- Pool `Sound` instances — reuse rather than create/destroy for frequent SFX
- Set `RollOffMaxDistance` appropriately — large values mean more sounds computed per frame
- Use `Sound.Paused` (not `:Stop()`) for sounds you'll resume shortly
- Avoid parenting sounds to parts that are frequently destroyed/recreated
- Use `SoundGroup.Volume` for global category control rather than tweening individual sounds

---

## Space Tycoon Sound Checklist

| Audio Layer | Sound Type | Notes |
|-------------|-----------|-------|
| Space ambient | 2D, looped, low volume | Subtle background hum |
| Ship engines | 3D, looped, pitch-scaled | Parent to engine part |
| Thruster bursts | 3D, one-shot | Pool 4–8 instances |
| Warp/jump | 2D, one-shot | Full-screen event |
| Station ambience | 3D, looped | Activate when docked |
| Trade success SFX | 2D, one-shot | UI event |
| Alert/warning | 2D, one-shot | High priority |
| Music | 2D, crossfaded | Zone-based |

---

**Sources:**
- https://create.roblox.com/docs/reference/engine/classes/Sound
- https://create.roblox.com/docs/reference/engine/classes/SoundService
- https://create.roblox.com/docs/reference/engine/classes/SoundGroup
- https://create.roblox.com/docs/audio
