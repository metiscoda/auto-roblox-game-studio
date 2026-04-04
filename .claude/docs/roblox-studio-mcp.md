# Roblox Studio — Tooling Reference

This project uses two complementary tools to interact with Roblox Studio:

1. **Rojo** (primary) — syncs local `.luau` files to Studio. Edit scripts locally.
2. **MCP** (secondary) — live bridge to Studio for inspection, testing, and non-script operations.

## Rojo Setup

- **Project root**: `game-rojo/`
- **Project file**: `game-rojo/default.project.json`
- **Version**: Rojo 7.6.1
- **Sync**: Rojo serve is running and connected to Studio

### Directory Mapping

| Local Path | Studio Location |
|------------|----------------|
| `game-rojo/src/server/` | `game.ServerScriptService.Server` |
| `game-rojo/src/client/` | `game.StarterPlayer.StarterPlayerScripts.Client` |
| `game-rojo/src/shared/` | `game.ReplicatedStorage.Shared` |

### File Naming Conventions (Rojo)

| File Pattern | Roblox Class |
|-------------|-------------|
| `*.server.luau` | Script |
| `*.client.luau` | LocalScript |
| `*.luau` (plain) | ModuleScript |
| `init.server.luau` | Script (represents the parent folder) |
| `init.client.luau` | LocalScript (represents the parent folder) |
| `init.luau` | ModuleScript (represents the parent folder) |

### Workflow: Editing Scripts

1. Edit `.luau` files locally using `Read`, `Edit`, `Write` tools
2. Rojo automatically syncs changes to Studio
3. Use MCP tools to verify the result in Studio if needed

**Always edit scripts locally via Rojo.** Only use MCP `multi_edit` for scripts that
are not managed by Rojo (e.g., scripts created directly in Studio outside the Rojo tree).

## MCP Tools Reference

The Roblox Studio MCP bridge connects Claude Code directly to a running Roblox Studio
instance. Use these tools for inspection, testing, and interacting with non-script
instances.

### Prerequisites

- Roblox Studio must be running with the MCP plugin installed
- Always call `list_roblox_studios` first to confirm connection and verify the active
  Studio instance before making any changes

## Tool Reference

### Session Management

| Tool | Purpose |
|------|---------|
| `mcp__Roblox_Studio__list_roblox_studios` | List all connected Studio instances (name, id, active status) |
| `mcp__Roblox_Studio__set_active_studio` | Set which Studio instance receives subsequent calls (requires `studio_id`) |

### Exploring the DataModel

| Tool | Purpose |
|------|---------|
| `mcp__Roblox_Studio__search_game_tree` | Browse the instance hierarchy with optional filters (path, className, keywords, depth) |
| `mcp__Roblox_Studio__inspect_instance` | Get all properties, attributes, and children of a specific instance (dot-notation path) |

### Script Operations

| Tool | Purpose |
|------|---------|
| `mcp__Roblox_Studio__script_read` | Read a script's full source (dot-notation path, e.g. `game.ServerScriptService.MyScript`) |
| `mcp__Roblox_Studio__script_search` | Fuzzy-find scripts by name keywords (comma-separated, case-insensitive) |
| `mcp__Roblox_Studio__script_grep` | Search all script contents for a string/pattern (capped at 50 matches) |
| `mcp__Roblox_Studio__multi_edit` | Edit or create scripts with batch edits (atomic, sequential). Supports creating new scripts with `className` |

### Execution & Testing

| Tool | Purpose |
|------|---------|
| `mcp__Roblox_Studio__execute_luau` | Run arbitrary Luau code in Studio and return the result |
| `mcp__Roblox_Studio__start_stop_play` | Start (`is_start: true`) or stop (`is_start: false`) play mode |
| `mcp__Roblox_Studio__get_console_output` | Read console/output log during play mode |

### Player Simulation (Play Mode)

| Tool | Purpose |
|------|---------|
| `mcp__Roblox_Studio__character_navigation` | Move the character to coordinates (x,y,z) or an instance path |
| `mcp__Roblox_Studio__user_keyboard_input` | Send keyboard events (keyDown, keyUp, keyPress, textInput) |
| `mcp__Roblox_Studio__user_mouse_input` | Send mouse events (moveTo, click, rightClick, scrollUp, scrollDown) |

## Common Workflows

### Edit a script (Rojo — primary)
1. Find the file locally: `Glob` / `Grep` in `game-rojo/src/`
2. `Read` the file, `Edit` to make changes
3. Rojo syncs automatically — verify in Studio with `inspect_instance` or `script_read` if needed

### Inspect non-script instances
1. `search_game_tree` to find the area of interest
2. `inspect_instance` to see properties and children

### Test a change
1. Edit scripts locally (Rojo syncs them)
2. `start_stop_play` (start)
3. `get_console_output` to check for errors
4. `execute_luau` to verify runtime state
5. `start_stop_play` (stop)

### Find and fix a bug
1. Search locally first: `Grep` in `game-rojo/src/`
2. If needed, also `script_grep` via MCP to search non-Rojo scripts in Studio
3. `Read` + `Edit` the local file to fix
4. Test with play mode

### Edit non-Rojo scripts (fallback)
1. `script_read` to read the script in Studio
2. `multi_edit` to make changes directly in Studio
3. Only use this for scripts outside the Rojo-managed tree

## Important Notes

- **Path format**: All instance paths use dot notation starting from the service level
  (e.g., `game.ServerScriptService.MyScript`, `Workspace.Models.SpawnPad`)
- **Rojo is the primary editing path**: Edit `.luau` files locally in `game-rojo/src/`.
  Rojo syncs them to Studio automatically. Only use MCP `multi_edit` for non-Rojo scripts.
- **multi_edit is atomic**: All edits in a batch succeed or fail together. `old_string`
  must match exactly. Each edit operates on the result of the previous one.
- **Script creation**: Prefer creating new `.luau` files locally (Rojo syncs them).
  For non-Rojo scripts, use `multi_edit` with `className` and empty `old_string`.
- **execute_luau runs in edit context**: Code runs as a plugin/command bar context, not
  inside a running game. Use `start_stop_play` + `get_console_output` for runtime testing.
- **Console output**: Only available during play mode. Call `get_console_output` after
  starting play to capture prints, warnings, and errors.
