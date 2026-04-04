# Roblox — DataStores Module Reference

**Last verified:** 2026-03-29
**Knowledge Gap:** LLM may suggest SetAsync (race condition) or missing pcall patterns; may not know DataStore2 vs. official API nuances

---

## Overview

Roblox data persistence services:

| Service | Use Case | Scope |
|---------|---------|-------|
| `DataStoreService` | Persistent player data (credits, ships, progress) | Per-key, cross-server persistent |
| `MemoryStoreService` | Ephemeral cross-server data (market prices, lobbies) | Expires, fast read/write |
| `OrderedDataStore` | Sorted leaderboards | Ranked, persistent |

All DataStore calls **must be server-side** and **wrapped in `pcall`**.
DataStore calls can and do fail in production. Plan for failure.

---

## DataStoreService: Key Concepts

- **Key limit**: 50 characters, only alphanumeric, underscore, dash
- **Value limit**: 4MB per key (JSON-serialized)
- **Throughput**: 60 + (player count × 10) read requests/min per DataStore
- **Write throttle**: 60 + (player count × 10) writes/min
- **`UpdateAsync` is atomic** — use for all player data writes
- **`SetAsync` has race conditions** — avoid for player data

---

## Basic Setup

```lua
--!strict
local DataStoreService = game:GetService("DataStoreService")

-- Use versioned store names so schema migrations don't corrupt old data
local PLAYER_STORE_VERSION = "v2"
local playerStore = DataStoreService:GetDataStore("PlayerData_" .. PLAYER_STORE_VERSION)
```

---

## Loading Player Data

```lua
--!strict

type PlayerData = {
    credits: number,
    shipId: string,
    rank: number,
    inventory: {[string]: number},
    lastSaved: number,
}

local DEFAULT_DATA: PlayerData = {
    credits = 500,
    shipId = "starter_ship",
    rank = 1,
    inventory = {},
    lastSaved = 0,
}

local function loadPlayerData(userId: number): PlayerData
    local success, result = pcall(function()
        return playerStore:GetAsync(tostring(userId))
    end)

    if not success then
        warn(string.format("[DataStore] Load failed for %d: %s", userId, tostring(result)))
        -- Return default data; player can still play but data isn't persisted
        return table.clone(DEFAULT_DATA)
    end

    if result == nil then
        -- New player
        return table.clone(DEFAULT_DATA)
    end

    -- Migrate / fill missing fields from DEFAULT_DATA
    for key, value in DEFAULT_DATA do
        if result[key] == nil then
            result[key] = value
        end
    end

    return result :: PlayerData
end
```

---

## Saving Player Data (UpdateAsync — Required)

```lua
--!strict

local function savePlayerData(userId: number, data: PlayerData): boolean
    -- Sanitize: strip non-serializable fields
    data.lastSaved = os.time()

    local success, err = pcall(function()
        playerStore:UpdateAsync(tostring(userId), function(current: any)
            -- current is the existing stored value (or nil for new player)
            -- Return the new value to store
            -- Returning nil cancels the update without writing
            return data
        end)
    end)

    if not success then
        warn(string.format("[DataStore] Save failed for %d: %s", userId, tostring(err)))
        return false
    end

    return true
end
```

---

## Auto-Save + Save on Leave + BindToClose

```lua
--!strict
-- PlayerDataService.server.luau

local Players = game:GetService("Players")
local AUTO_SAVE_INTERVAL = 60  -- seconds

-- In-memory cache (never trust client, never expose directly)
local cache: {[number]: PlayerData} = {}

-- Load on join
Players.PlayerAdded:Connect(function(player: Player)
    local data = loadPlayerData(player.UserId)
    cache[player.UserId] = data
    -- Push initial state to client via RemoteEvent if needed
end)

-- Save and clean up on leave
Players.PlayerRemoving:Connect(function(player: Player)
    local data = cache[player.UserId]
    if data then
        savePlayerData(player.UserId, data)
        cache[player.UserId] = nil
    end
end)

-- Auto-save loop
task.spawn(function()
    while true do
        task.wait(AUTO_SAVE_INTERVAL)
        for userId, data in cache do
            savePlayerData(userId, data)
        end
    end
end)

-- Save all on server shutdown (critical for game:BindToClose)
game:BindToClose(function()
    -- BindToClose gives 30 seconds max; must save synchronously
    local saveThreads: {thread} = {}
    for userId, data in cache do
        table.insert(saveThreads, task.spawn(function()
            savePlayerData(userId, data)
        end))
    end
    -- Wait for all saves to finish (up to 29 seconds)
    -- Note: BindToClose blocks server shutdown while this function runs
end)
```

---

## Session Locking (Advanced — Prevent Multi-Server Corruption)

Session locking prevents two servers from simultaneously writing the same player's
data (edge case: player joins server B before server A has saved their data after
a crash or transfer).

```lua
--!strict
-- Simplified session lock pattern

local sessionStore = DataStoreService:GetDataStore("SessionLocks")
local SERVER_ID = game.JobId  -- unique per server instance

local function acquireSessionLock(userId: number): boolean
    local key = "lock_" .. tostring(userId)
    local acquired = false

    local success = pcall(function()
        sessionStore:UpdateAsync(key, function(current: any)
            local now = os.time()
            -- Release lock if it's older than 60 seconds (crashed server)
            if current ~= nil and current.serverId ~= SERVER_ID then
                if now - (current.timestamp or 0) < 60 then
                    return nil  -- lock still held by another server; cancel update
                end
            end
            acquired = true
            return { serverId = SERVER_ID, timestamp = now }
        end)
    end)

    return success and acquired
end

local function releaseSessionLock(userId: number)
    local key = "lock_" .. tostring(userId)
    pcall(function()
        sessionStore:UpdateAsync(key, function(current: any)
            if current and current.serverId == SERVER_ID then
                return nil  -- delete the lock
            end
            return current
        end)
    end)
end
```

---

## MemoryStoreService (Cross-Server, Ephemeral)

For the global commodity market — prices must be shared across all server instances.

```lua
--!strict
local MemoryStoreService = game:GetService("MemoryStoreService")
local marketMap = MemoryStoreService:GetSortedMap("MarketPrices_v1")

-- Write global price (server)
local function setMarketPrice(commodityId: string, price: number): boolean
    local success, err = pcall(function()
        -- expires in 3600 seconds (1 hour); re-write on tick to keep alive
        marketMap:SetAsync(commodityId, price, 3600)
    end)
    if not success then
        warn("[MemoryStore] Price write failed:", err)
    end
    return success
end

-- Read global price (server)
local function getMarketPrice(commodityId: string): number?
    local success, value = pcall(function()
        return marketMap:GetAsync(commodityId)
    end)
    if success then
        return value
    end
    return nil
end

-- Batch update prices (use UpdateAsync for safe increment/decrement)
local function adjustMarketPrice(commodityId: string, delta: number): number?
    local newPrice: number? = nil
    local success = pcall(function()
        newPrice = marketMap:UpdateAsync(commodityId, function(current: number?)
            local base = current or 100
            return math.clamp(base + delta, MIN_PRICE, MAX_PRICE)
        end, 3600)
    end)
    if success then return newPrice end
    return nil
end
```

---

## OrderedDataStore (Leaderboards)

```lua
--!strict
local leaderboardStore = DataStoreService:GetOrderedDataStore("WealthLeaderboard_v1")

-- Write score
local function updateLeaderboard(userId: number, totalCredits: number)
    local success, err = pcall(function()
        leaderboardStore:SetAsync(tostring(userId), totalCredits)
    end)
    if not success then
        warn("[Leaderboard] Write failed:", err)
    end
end

-- Read top 100
local function getTopPlayers(count: number): {{userId: string, credits: number}}
    local success, pages = pcall(function()
        return leaderboardStore:GetSortedAsync(
            false,  -- descending (highest first)
            count,  -- max entries
            1,      -- min value filter
            math.huge  -- max value filter
        )
    end)

    if not success then
        warn("[Leaderboard] Read failed")
        return {}
    end

    local results: {{userId: string, credits: number}} = {}
    while true do
        local pageData = pages:GetCurrentPage()
        for _, entry in pageData do
            table.insert(results, {
                userId = entry.key,
                credits = entry.value,
            })
        end
        if pages.IsFinished then break end
        local ok = pcall(function() pages:AdvanceToNextPageAsync() end)
        if not ok then break end
    end

    return results
end
```

---

## Schema Migration Pattern

When you change the data schema (add/remove fields), bump the store version.
Never modify data in a versioned store to fix bugs — create a new version.

```lua
-- Version bump strategy:
-- "PlayerData_v1" → "PlayerData_v2"
-- On first load from v2: check if v1 data exists and migrate

local function migrateFromV1(userId: number): PlayerData?
    local v1Store = DataStoreService:GetDataStore("PlayerData_v1")
    local success, oldData = pcall(function()
        return v1Store:GetAsync(tostring(userId))
    end)
    if not success or oldData == nil then return nil end

    -- Transform v1 schema to v2 schema
    return {
        credits = oldData.gold or 500,         -- renamed field
        shipId = oldData.ship or "starter_ship",
        rank = 1,
        inventory = oldData.items or {},
        lastSaved = os.time(),
    }
end
```

---

## Error Budget and Retry

DataStore calls are throttled. Under heavy load, calls may fail.

```lua
local MAX_RETRIES = 3
local RETRY_DELAY = 2  -- seconds

local function saveWithRetry(userId: number, data: PlayerData): boolean
    for attempt = 1, MAX_RETRIES do
        local success, err = pcall(function()
            playerStore:UpdateAsync(tostring(userId), function()
                return data
            end)
        end)

        if success then
            return true
        end

        warn(string.format("[DataStore] Save attempt %d/%d failed: %s", attempt, MAX_RETRIES, tostring(err)))

        if attempt < MAX_RETRIES then
            task.wait(RETRY_DELAY * attempt)  -- exponential backoff
        end
    end

    warn(string.format("[DataStore] All save attempts failed for %d", userId))
    return false
end
```

---

## Space Tycoon Data Schema (Reference)

```lua
-- Suggested PlayerData schema for space tycoon

type InventoryItem = {
    commodityId: string,
    quantity: number,
}

type ShipData = {
    shipModelId: string,
    cargoCapacity: number,
    speed: number,
    upgrades: {[string]: number},  -- upgradeId → level
}

type PlayerData = {
    -- Economy
    credits: number,
    lifetimeEarnings: number,

    -- Ship
    activeShipId: string,
    ships: {[string]: ShipData},

    -- Inventory (cargo hold)
    inventory: {[string]: number},  -- commodityId → quantity

    -- Progression
    rank: number,
    xp: number,
    unlockedStations: {string},

    -- Meta
    version: number,
    lastSaved: number,
    createdAt: number,
}
```

---

**Sources:**
- https://create.roblox.com/docs/cloud-services/data-stores
- https://create.roblox.com/docs/cloud-services/memory-stores
- https://create.roblox.com/docs/reference/engine/classes/DataStoreService
- https://create.roblox.com/docs/reference/engine/classes/MemoryStoreService
- https://create.roblox.com/docs/reference/engine/classes/OrderedDataStore
