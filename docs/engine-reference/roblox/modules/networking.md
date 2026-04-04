# Roblox — Networking Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may not know UnreliableRemoteEvent details; may suggest RemoteFunction for patterns that should use RemoteEvent

---

## Overview

Roblox networking is built on **FilteringEnabled** (always on). The client and server
are strictly separated. All communication goes through:

- **`RemoteEvent`**: fire-and-forget (client → server, server → client, server → all clients)
- **`RemoteFunction`**: request-response (client → server only for gameplay; avoid server → client)
- **`UnreliableRemoteEvent`**: fire-and-forget, can be dropped, no ordering guarantee (cosmetics only)
- **`BindableEvent`** / **`BindableFunction`**: same-side communication (server↔server or client↔client)

All Remote instances live in `ReplicatedStorage` so both sides can access them.

---

## Remote Organization

Recommended structure in `ReplicatedStorage.Shared.Remotes`:

```
ReplicatedStorage/
└── Shared/
    └── Remotes/
        ├── Economy/
        │   ├── RequestTrade        (RemoteEvent)
        │   ├── TradeResult         (RemoteEvent, server→client)
        │   └── GetMarketPrices     (RemoteFunction, client→server)
        ├── Ships/
        │   ├── SetThrust           (RemoteEvent, client→server)
        │   ├── DockRequest         (RemoteEvent, client→server)
        │   └── ShipStateUpdate     (RemoteEvent, server→client)
        └── FX/
            └── ThrusterEffect      (UnreliableRemoteEvent, server→client)
```

---

## RemoteEvent

### Client → Server (Action Request)

```lua
--!strict
-- Client (LocalScript)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RequestTrade = ReplicatedStorage.Shared.Remotes.Economy.RequestTrade

RequestTrade:FireServer(targetUserId, commodityId, quantity, offeredPrice)

-- Server (Script)
RequestTrade.OnServerEvent:Connect(function(
    player: Player,
    targetUserId: number,
    commodityId: string,
    quantity: number,
    offeredPrice: number
)
    -- ALWAYS validate on server
    if type(targetUserId) ~= "number" then return end
    if type(commodityId) ~= "string" or #commodityId > 50 then return end
    if type(quantity) ~= "number" or quantity < 1 or quantity > 10000 then return end
    if type(offeredPrice) ~= "number" or offeredPrice < 0 then return end

    processTrade(player, targetUserId, commodityId, quantity, offeredPrice)
end)
```

### Server → Specific Client

```lua
-- Server
local TradeResult = ReplicatedStorage.Shared.Remotes.Economy.TradeResult

TradeResult:FireClient(player, {
    success = true,
    creditsGained = 500,
    commodityLost = "IronOre",
    quantity = 50,
})

-- Client
TradeResult.OnClientEvent:Connect(function(result: {success: boolean, creditsGained: number})
    if result.success then
        updateCreditDisplay(result.creditsGained)
    end
end)
```

### Server → All Clients (Market Broadcast)

```lua
-- Server: broadcast market price update
-- WARNING: expensive at 100+ players — consider per-client or region-based updates
MarketPriceUpdate:FireAllClients(priceTable)
```

---

## RemoteFunction (Client → Server Only)

Use `RemoteFunction` only when the client needs a **synchronous return value** from the
server and the client will stay connected. Do NOT use `InvokeClient` — it stalls the
server if the client disconnects.

```lua
-- Server
local GetMarketPrices = ReplicatedStorage.Shared.Remotes.Economy.GetMarketPrices

GetMarketPrices.OnServerInvoke = function(player: Player, stationId: string): {[string]: number}
    if type(stationId) ~= "string" then return {} end
    return getStationPrices(stationId)
end

-- Client
local prices = GetMarketPrices:InvokeServer(stationId)
-- prices is now {[commodityId]: price}
```

**Caution:** The client thread yields until the server returns. If the server is busy,
the client UI will freeze. For data that updates regularly, prefer server-push via
`RemoteEvent` instead of client-poll via `RemoteFunction`.

---

## UnreliableRemoteEvent (Cosmetic FX Only)

Messages can be dropped and arrive out of order. Use for visual effects that are
acceptable to miss occasionally.

```lua
--!strict
-- Server: broadcast thruster FX (can be dropped — not gameplay-critical)
local ThrusterFX: UnreliableRemoteEvent = ReplicatedStorage.Shared.Remotes.FX.ThrusterEffect

ThrusterFX:FireAllClients(shipId, thrustLevel, color)

-- Client: play effect if received
ThrusterFX.OnClientEvent:Connect(function(shipId: string, thrustLevel: number, color: Color3)
    playThrusterEffect(shipId, thrustLevel, color)
end)
```

**Never use UnreliableRemoteEvent for:**
- Economy/trade results
- Inventory changes
- Position corrections
- Any state that must be guaranteed delivered

---

## Rate Limiting (Server-Side)

Always rate-limit RemoteEvent calls to prevent client spam exploits.

```lua
--!strict
-- Simple rate limiter per player
local lastFired: {[number]: number} = {}
local RATE_LIMIT = 0.1 -- minimum seconds between calls

local function checkRateLimit(player: Player): boolean
    local now = os.clock()
    local last = lastFired[player.UserId] or 0
    if now - last < RATE_LIMIT then
        return false  -- too fast
    end
    lastFired[player.UserId] = now
    return true
end

SetThrust.OnServerEvent:Connect(function(player: Player, ...)
    if not checkRateLimit(player) then return end
    processThrust(player, ...)
end)

-- Clean up on player leave
game:GetService("Players").PlayerRemoving:Connect(function(player: Player)
    lastFired[player.UserId] = nil
end)
```

---

## Server Validation Pattern

Every `OnServerEvent` handler must validate all arguments — clients are never trusted.

```lua
local function validateTrade(
    commodityId: unknown,
    quantity: unknown,
    price: unknown
): boolean
    if type(commodityId) ~= "string" then return false end
    if type(quantity) ~= "number" then return false end
    if type(price) ~= "number" then return false end

    if quantity < 1 or quantity > MAX_TRADE_QUANTITY then return false end
    if price < 0 or price > MAX_PRICE then return false end
    if not VALID_COMMODITIES[commodityId] then return false end

    return true
end

TradeEvent.OnServerEvent:Connect(function(player: Player, commodityId: unknown, quantity: unknown, price: unknown)
    if not validateTrade(commodityId, quantity, price) then
        warn("Invalid trade from:", player.Name)
        return
    end
    -- safe to proceed
    executeTrade(player, commodityId :: string, quantity :: number, price :: number)
end)
```

---

## Replication: What Auto-Replicates

| Item | Replicates? | Notes |
|------|-------------|-------|
| Instance hierarchy (server creates) | YES → client | Automatic |
| BasePart properties (Anchored, Color, CFrame) | YES → client | Automatic |
| `Attributes` on instances | YES → client | Lightweight KV |
| Module state (Lua tables) | NO | Must use Remotes or Attributes |
| `RemoteEvent:FireClient` | YES | Manual trigger |
| Server Script local variables | NO | Server only |
| `BindableEvent` | NO | Same side only |

---

## MemoryStoreService (Cross-Server)

For data shared across multiple server instances (global market prices, leaderboards):

```lua
--!strict
local MemoryStoreService = game:GetService("MemoryStoreService")
local marketMap = MemoryStoreService:GetSortedMap("GlobalMarket")

-- Write (server)
local function setGlobalPrice(commodityId: string, price: number)
    local success, err = pcall(function()
        marketMap:SetAsync(commodityId, price, 300) -- expires in 5 minutes
    end)
    if not success then
        warn("MemoryStore write failed:", err)
    end
end

-- Read (server)
local function getGlobalPrice(commodityId: string): number?
    local success, value = pcall(function()
        return marketMap:GetAsync(commodityId)
    end)
    if success then
        return value
    end
    return nil
end
```

---

## Batching Market Updates

For a space tycoon with 100+ commodity prices updating frequently, batch updates into
a single RemoteEvent call rather than one event per commodity:

```lua
-- ✅ Batch: one event, entire price table
local priceUpdate = {
    IronOre = 12.5,
    Fuel = 8.3,
    Electronics = 45.0,
}
MarketPriceUpdate:FireAllClients(priceUpdate)

-- ❌ Unbatched: 100+ events per tick
for commodity, price in allPrices do
    MarketPriceUpdate:FireAllClients(commodity, price)
end
```

---

## Common Patterns for Space Tycoon

| Use Case | Pattern |
|----------|---------|
| Player requests to dock | `RemoteEvent` (client→server) with position validation |
| Server confirms dock | `RemoteEvent` (server→client) with dock state |
| Trade initiation | `RemoteEvent` (client→server) |
| Trade result notification | `RemoteEvent` (server→client) |
| Get station commodity prices | `RemoteFunction` (client→server) or server-push on approach |
| Thruster particle effects | `UnreliableRemoteEvent` (server→all nearby clients) |
| Ship position sync | `Attributes` on ship part + optional correction `RemoteEvent` |
| Global leaderboard | `MemoryStoreService` sorted map |
| Player credits display | `Attribute` on player Character or explicit `RemoteEvent` |

---

**Sources:**
- https://create.roblox.com/docs/scripting/events/remote
- https://create.roblox.com/docs/reference/engine/classes/RemoteEvent
- https://create.roblox.com/docs/reference/engine/classes/UnreliableRemoteEvent
- https://create.roblox.com/docs/cloud-services/memory-stores
