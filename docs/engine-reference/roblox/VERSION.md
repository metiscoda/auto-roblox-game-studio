# Roblox Engine — Version Reference

| Field | Value |
|-------|-------|
| **Platform** | Roblox Studio |
| **Language** | Luau (strict mode) |
| **Luau Version** | 0.647+ (rolling release) |
| **Rojo Version** | 7.6.1 |
| **Project Pinned** | 2026-03-29 |
| **Last Docs Verified** | 2026-03-29 |
| **LLM Knowledge Cutoff** | August 2025 |

## Knowledge Gap Warning

Roblox ships updates on a rolling release schedule — there is no fixed version number
pinned to a project. The platform is always the current live version. The LLM's
training data covers Roblox APIs up to approximately mid-2025. Always cross-reference
this directory before suggesting any Roblox API call, especially for:

- Task library (`task.spawn`, `task.wait`, `task.delay`)
- DataStore and MemoryStore APIs
- Physics solver configuration
- Streaming (content streaming / LoD)
- New rendering features (Future lighting, Sky/Atmosphere)

## Post-Cutoff Roblox Changes (Approximate Timeline)

| Period | Risk Level | Key Theme |
|--------|------------|-----------|
| 2024 Q1 | MEDIUM | `task` library fully mature, legacy scheduling officially deprecated |
| 2024 Q2 | HIGH | `StreamingEnabled` `PauseOutsideDistance` model introduced; `StreamingPauseMode` added |
| 2024 Q3 | MEDIUM | Luau type inference improvements; `buffer` type added |
| 2024 Q4 | HIGH | Universal `UnreliableRemoteEvent` stabilized for production use |
| 2025 Q1 | HIGH | New `DataModel` replication guarantees; `ParallelLuau` actor model stabilized |
| 2025 Q2 | MEDIUM | `EditableMesh`/`EditableImage` APIs graduated from beta |
| 2025 Q3 | MEDIUM | Geometry/CSG improvements; `MaterialService` surface appearance updates |

## Luau Language Status

Luau is a superset of Lua 5.1 with additions:
- `--!strict` type checking (required for this project)
- `--!nonstrict` (partial checking)
- `--!nocheck` (disabled)
- Generic functions and type aliases
- String interpolation: `` `Hello {name}!` ``
- `buffer` built-in type (efficient binary buffers)
- `continue` in loops
- Compound assignment: `+=`, `-=`, `*=`, `/=`

## Active Engine Features for This Project

| Feature | Status | Notes |
|---------|--------|-------|
| `task` library | Production | Replaces `spawn()`, `wait()`, `delay()` — REQUIRED |
| `UnreliableRemoteEvent` | Production | Use for non-critical cosmetic network traffic |
| `StreamingEnabled` | Active | Must design around part streaming for space world |
| `MemoryStoreService` | Production | Cross-server leaderboard/session data |
| `DataStoreService` | Production | Player persistence — wrap all calls in `pcall` |
| `ParallelLuau` (Actors) | Beta/Stable | CPU-heavy workloads, experimental for gameplay |
| `EditableMesh` | Beta | Not recommended for production yet |

## Verified Sources

- Creator docs: https://create.roblox.com/docs
- Luau reference: https://luau.org/
- Luau changelog: https://luau.org/changelog
- Roblox engine changelog: https://devforum.roblox.com/c/updates/release-notes/36
- Rojo docs: https://rojo.space/docs/
- DataStore guide: https://create.roblox.com/docs/cloud-services/data-stores
- MemoryStore guide: https://create.roblox.com/docs/cloud-services/memory-stores
