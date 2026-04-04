# Directory Structure

```text
/
├── CLAUDE.md                    # Master configuration
├── .claude/                     # Agent definitions, skills, hooks, rules, docs
├── game-rojo/            # Rojo project (syncs to Roblox Studio)
│   ├── default.project.json     # Rojo project config
│   └── src/
│       ├── server/              # ServerScriptService scripts (.server.luau)
│       ├── client/              # StarterPlayerScripts scripts (.client.luau)
│       └── shared/              # ReplicatedStorage modules (.luau)
├── src/                         # Non-Rojo source (tooling, utilities)
├── assets/                      # Game assets (art, audio, vfx, shaders, data)
├── design/                      # Game design documents (gdd, narrative, levels, balance)
├── docs/                        # Technical documentation (architecture, api, postmortems)
│   └── engine-reference/        # Curated engine API snapshots (version-pinned)
├── tests/                       # Test suites (unit, integration, performance, playtest)
├── tools/                       # Build and pipeline tools (ci, build, asset-pipeline)
├── prototypes/                  # Throwaway prototypes (isolated from src/)
└── production/                  # Production management (sprints, milestones, releases)
    ├── session-state/           # Ephemeral session state (active.md — gitignored)
    └── session-logs/            # Session audit trail (gitignored)
```
