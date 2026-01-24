# Technology Stack

**Analysis Date:** 2026-01-24

## Languages

**Primary:**
- JavaScript (Node.js) - All runtime execution, installation, hooks, configuration
- Markdown - Command definitions, agent specifications, workflow templates, documentation

**Secondary:**
- YAML - Command frontmatter metadata (name, description, allowed-tools)
- JSON - Configuration, templates, templates index, workflow schema

## Runtime

**Environment:**
- Node.js >= 16.7.0

**Package Manager:**
- npm
- Lockfile: Present (`package-lock.json` v3)

## Frameworks

**Core:**
- No application frameworks - Standalone system for meta-prompting and context engineering
- GSD is a system that runs AS COMMANDS within Claude Code and OpenCode (not a framework itself)

**Build/Development:**
- esbuild 0.24.0 - Bundling hooks for distribution

## Key Dependencies

**Critical:**
- esbuild 0.24.0 - Bundles pre-publish hooks (gsd-check-update.js, gsd-statusline.js)
  - Only dev dependency - hooks are pure Node.js, no bundling actually needed currently
  - Optional platform packages for cross-platform support (darwin-arm64, linux-x64, etc.)

## Configuration

**Environment:**
- Installation targets: `~/.claude/` (Claude Code) or `~/.config/opencode/` (OpenCode)
- Configuration via: `settings.json` at runtime config dir
- Mode configuration: `.planning/config.json` (interactive/yolo modes, workflow options)

**Build:**
- Build script: `scripts/build-hooks.js` - copies hooks to `hooks/dist/` for npm publish
- Hook distribution: Pre-built hooks included in npm package under `hooks/dist/`

## Platform Requirements

**Development:**
- Node.js 16.7.0+
- Git (for version control and hooks)
- Bash/shell access (installation and execution contexts)
- Cross-platform: macOS, Windows (via WSL2 detection), Linux

**Production:**
- Deployment target: NPM registry (npx get-shit-done-cc)
- Installation: Global (`~/.claude/` or `~/.config/opencode/`) or Local (`./.claude/` or `./.opencode/`)
- Runtime: Claude Code v1.0+ or OpenCode

## Version Management

**Current Version:** 1.9.13

**Update Check:**
- Hook: `gsd-check-update.js` (installed to hooks/)
- Trigger: SessionStart lifecycle hook in Claude Code
- Method: `npm view get-shit-done-cc version` command
- Cache: `~/.claude/cache/gsd-update-check.json` (checked for staleness)
- Notification: Statusline shows update available indicator

## Core Components

**Installation:**
- `bin/install.js` - Interactive/non-interactive installer
  - Supports Claude Code and OpenCode runtime targets
  - Handles global and local installations
  - Custom config directory support via `--config-dir`
  - Path replacement for cross-platform compatibility
  - Frontmatter conversion for OpenCode compatibility

**Hooks (Runtime Integrations):**
- `hooks/gsd-statusline.js` - Statusline display in Claude Code showing model, task, context usage
- `hooks/gsd-check-update.js` - Background update checker (spawns detached Node process)
- Both hooks read stdin JSON and write text output

**Commands:**
- 40+ slash commands in `commands/gsd/` (nested markdown files)
- Each command: frontmatter metadata + execution objectives
- Tools: Read, Write, Bash, Glob, Grep, Task, WebFetch, mcp__context7__* (Claude Code MCP)

**Agents:**
- 11 specialized markdown agents in `agents/`
- Orchestration: Commands spawn agents via Task tool
- Subagent pattern: agents are prompts executed by Claude, not code modules

**Workflows:**
- Reference templates in `get-shit-done/workflows/` (markdown specifications)
- Loaded by commands and agents for context
- Workflow execution: Sequential or parallel depending on mode

## Data Storage

**Local Storage:**
- Project state: `.planning/` directory structure
  - `config.json` - Mode and workflow settings
  - `STATE.md` - Project memory and context
  - `PROJECT.md` - Vision and requirements
  - `ROADMAP.md` - Phase breakdown
  - `REQUIREMENTS.md` - Scoped requirements with REQ-IDs
  - `research/` - Domain research documents
  - `codebase/` - Codebase analysis documents (STACK.md, ARCHITECTURE.md, etc.)

**Session Storage:**
- `~/.claude/todos/` - Session todo lists (JSON format)
- `~/.claude/cache/` - Update check cache
- `settings.json` - Claude Code settings (hooks, statusline config)

**OpenCode Specific:**
- `~/.config/opencode/opencode.json` - OpenCode configuration
- `~/.config/opencode/permission/` - Path-based permission rules
- `command/gsd-*.md` - Flattened command structure (vs nested `commands/gsd/`)

## Cross-Platform Considerations

**Windows Support:**
- WSL2 detection in installer (non-interactive fallback)
- Forward-slash path normalization in hook commands
- `windowsHide` flag on spawned processes (prevents console flash)
- `npm` command availability check via `execSync`

**Path Handling:**
- Tilde expansion (`~`) for home directory references
- Environment variable support: `CLAUDE_CONFIG_DIR`, `OPENCODE_CONFIG_DIR`, `XDG_CONFIG_HOME`
- Path prefix replacement in markdown files during installation

---

*Stack analysis: 2026-01-24*
