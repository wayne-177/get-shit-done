# External Integrations

**Analysis Date:** 2026-01-24

## APIs & External Services

**NPM Registry:**
- Service: NPM public registry
- Purpose: Version checking and package distribution
- SDK/Client: Node.js `execSync('npm view get-shit-done-cc version')`
- Auth: Public (no authentication required)
- Trigger: `gsd-check-update.js` hook runs on SessionStart
- Timeout: 10 seconds (hardcoded, prevents hanging)

**Discord Community:**
- Service: Discord server
- Purpose: Community support and collaboration
- URL: https://discord.gg/5JJgD5svVS
- Integration: Link shown after installation, referenced in README and help
- Type: External community link (no programmatic integration)

## Code Platforms

**GitHub:**
- Repository: https://github.com/glittercowboy/get-shit-done
- Purpose: Source control and distribution
- Integration: Package links to GitHub issues and repository
- Files: `.git/` (git repository), `bin/install.js` has remote URL

**Claude Code:**
- IDE: Anthropic's Claude Code
- Purpose: Runtime host for all commands and agents
- Integration: Tight coupling
  - Commands register via `~/.claude/commands/gsd/`
  - Agents spawn via Task tool
  - Settings via `~/.claude/settings.json`
  - Hooks via `~/.claude/hooks/`
  - Todos via `~/.claude/todos/` (session-specific)
  - Statusline via stdin/stdout JSON exchange
  - MCP tools: `mcp__context7__*` for research capabilities

**OpenCode:**
- IDE: Open source code editor
- Purpose: Alternative runtime (free models support)
- Integration: Compatibility layer
  - Commands register via `~/.config/opencode/command/`
  - Path: `~/.config/opencode/` (XDG Base Directory spec)
  - Permission system: `~/.config/opencode/opencode.json` (read/external_directory)
  - Frontmatter conversion: Claude -> OpenCode format (allowed-tools -> tools object)
  - Tool mapping: AskUserQuestion->question, SlashCommand->skill, etc.

## Data Storage

**Filesystem Only (No External Databases):**
- Configuration: JSON files in config directories
- Project state: Markdown files in `.planning/`
- Todo lists: Session-specific JSON in `~/.claude/todos/`
- Cache: Update check results in `~/.claude/cache/gsd-update-check.json`
- File storage: Local filesystem only, no cloud backend

## Authentication & Identity

**Auth Provider:**
- Custom: No external auth provider
- Implementation: File system based
  - Session identification via Claude Code session_id (from statusline JSON input)
  - User identification via workspace directory (from statusline JSON input)
  - No login/logout mechanism

## Monitoring & Observability

**Error Tracking:**
- None detected - No external error tracking service

**Logs:**
- Approach: Console stdout/stderr to Claude Code interface
- Log files: None (real-time output to IDE)
- Cache analysis: Update check results written to local cache file
- Debug: Manual inspection of `.planning/` artifacts

## CI/CD & Deployment

**Hosting:**
- NPM registry (public package)
- GitHub (source repository)
- Distribution: `npx get-shit-done-cc` (via NPM)

**CI Pipeline:**
- Not detected in codebase (no GitHub Actions, GitLab CI, etc. files)
- Manual build: `npm run build:hooks` (prepublishOnly script)
- Hook bundling: esbuild (optional, hooks are currently pure Node.js)

## Environment Configuration

**Required env vars:**
- CLAUDE_CONFIG_DIR - (Optional) Override default `~/.claude` config directory
- OPENCODE_CONFIG_DIR - (Optional) Override default `~/.config/opencode` directory
- OPENCODE_CONFIG - (Optional) Explicit opencode.json file path
- XDG_CONFIG_HOME - (Optional) XDG Base Directory spec compliance
- npm view command requires NPM in PATH (for update checks)

**Secrets location:**
- Not applicable - No API keys or secrets required
- Package uses public APIs only
- All configuration is local file-based

## Webhooks & Callbacks

**Incoming:**
- None detected

**Outgoing:**
- None detected (statusline is request-response, not webhook)

## IDE Integration Points

**Claude Code Integration:**

1. **Settings.json Hooks:**
   - `SessionStart` - Triggers update check hook
   - Hook type: command (executes Node.js script)
   - Command: `node ~/.claude/hooks/gsd-check-update.js`

2. **Statusline:**
   - Type: command
   - Command: `node ~/.claude/hooks/gsd-statusline.js`
   - Input: stdin JSON with model, session_id, workspace dir, context_window
   - Output: stdout text (statusline display)
   - Reads: `~/.claude/todos/` for current task, `~/.claude/cache/` for update status

3. **Slash Commands:**
   - Location: `~/.claude/commands/gsd/*.md` (nested structure)
   - Format: Markdown with YAML frontmatter
   - Invocation: `/gsd:command-name`
   - Tools allowed: Read, Write, Bash, Glob, Grep, Task, WebFetch, mcp__context7__*

4. **Task Tool (Agent Spawning):**
   - Agents are markdown prompts in `~/.claude/agents/gsd-*.md`
   - Spawned by commands via Task tool
   - Parallel execution: up to 3 concurrent agents (configurable)
   - Communication: Return structured text (confirmations, PLAN.md content, etc.)

5. **MCP Integration:**
   - Tools: `mcp__context7__*` (Model Context Protocol)
   - Used by: Researcher agents for project research
   - Capabilities: Web search, documentation access, context retrieval

**OpenCode Integration:**

1. **Command Structure:**
   - Location: `~/.config/opencode/command/gsd-*.md` (flat structure)
   - Invocation: `/gsd-command-name` (replaces colons with hyphens)
   - Tool references: Converted to lowercase (read, write, bash, etc.)

2. **Permission System:**
   - Config: `~/.config/opencode/opencode.json`
   - Granted paths:
     - `~/.config/opencode/get-shit-done/*` - read permission
     - `~/.config/opencode/get-shit-done/*` - external_directory permission
   - Purpose: Prevent permission prompts during execution

3. **Frontmatter Conversion:**
   - Input: Claude Code format (allowed-tools array)
   - Output: OpenCode format (tools object with true values)
   - Color conversion: Named colors (cyan, red) -> Hex (#00FFFF, #FF0000)
   - Command references: `/gsd:help` -> `/gsd-help`
   - Path references: `~/.claude/` -> `~/.config/opencode/`

## External Tool Dependencies

**None in production:**
- No external SDKs or clients required
- No API libraries (axios, node-fetch, etc.)
- Pure Node.js: fs, path, os, child_process, readline
- Build-only: esbuild (dev dependency)

## Installation & Uninstall Integrations

**NPX Install Command:**
- Entry point: `bin/install.js`
- Target directories auto-detected or configurable
- Performs path replacement in markdown content
- Creates hooks/, commands/, agents/, get-shit-done/ directories
- Updates settings.json with hook configurations

**Uninstall Process:**
- Removes only GSD-specific files and directories
- Preserves user content in config directories
- Cleans up settings.json hook registrations
- Cleans up orphaned hook entries from previous versions

---

*Integration audit: 2026-01-24*
