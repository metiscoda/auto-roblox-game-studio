# Claude Code Game Studios -- Game Studio Agent Architecture

Indie game development managed through 48 coordinated Claude Code subagents.
Each agent owns a specific domain, enforcing separation of concerns and quality.

## Technology Stack

- **Engine**: Roblox Studio
- **Language**: Luau (--!strict mode)
- **Version Control**: Git with trunk-based development
- **Build System**: Rojo 7.6.1 (syncs `game-rojo/` to Studio)
- **Asset Pipeline**: Roblox Studio (models, meshes, images managed in Studio)
- **Live Bridge**: Roblox Studio MCP (inspection, testing, runtime interaction)

> **Engine specialist**: Use `roblox-specialist` agent for all Roblox-specific decisions.

## Project Structure

@.claude/docs/directory-structure.md

## Roblox Studio Tooling

@.claude/docs/roblox-studio-mcp.md

## Technical Preferences

@.claude/docs/technical-preferences.md

## Coordination Rules

@.claude/docs/coordination-rules.md

## Collaboration Protocol

**User-driven collaboration, not autonomous execution.**
Every task follows: **Question -> Options -> Decision -> Draft -> Approval**

- Agents MUST ask "May I write this to [filepath]?" before using Write/Edit tools
- Agents MUST show drafts or summaries before requesting approval
- Multi-file changes require explicit approval for the full changeset
- No commits without user instruction

See `docs/COLLABORATIVE-DESIGN-PRINCIPLE.md` for full protocol and examples.

> **First session?** If the project has no engine configured and no game concept,
> run `/start` to begin the guided onboarding flow.

## Coding Standards

@.claude/docs/coding-standards.md

## Context Management

@.claude/docs/context-management.md
