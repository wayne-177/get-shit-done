# External Integrations

**Analysis Date:** 2026-01-24

## APIs & External Services

**NPM Registry:**
- Service: npm package registry (npmjs.org)
- What it's used for: Version checking for GSD updates, installing get-shit-done-cc package
- Client: `npm view` command via `child_process.execSync()`
- Auth: None (public read)
- Location: `hooks/gsd-check-update.js` line 45
- Timeout: 10 seconds

**GitHub:**
- Service: GitHub repository (glittercowboy/get-shit-done)
- What it's used for: Release management, source repository
- Client: GitHub Actions workflow
- Auth: GitHub Actions context (no explicit credentials needed)
- Location: `.github/workflows/release.yml`

## Data Storage

**Databases:**
- None - GSD is a stateless CLI tool

**File Storage:**
- Local filesystem only
- Installation directories:
  - `~/.claude/` - Claude Code global install
  - `~/.config/opencode/` - OpenCode global install
  - `./.claude/` or `./.opencode/` - Local project install
- Configuration: `settings.json` (Claude Code) or `opencode.json` (OpenCode)
- Cache: `~/.claude/cache/gsd-update-check.json` - Version check cache

**Caching:**
- Local file-based cache only
- Update check cache location: `~/.claude/cache/gsd-update-check.json`
- Cache structure: JSON with `update_available`, `installed`, `latest`, `checked` fields
- No external cache services

## Authentication & Identity

**Auth Provider:**
- None - GSD is a CLI installer with no authentication

**User Context:**
- Operates within current user's shell context
- Reads/writes to user's home directory
- No user accounts or authentication tokens

## Authorization & Permissions

**System-Level:**
- Requires write access to config directories
- OpenCode: GSD configures read permissions in `opencode.json` to allow access to GSD reference docs
- Claude Code: Stores settings in `.claude/settings.json`

**OpenCode Permission Configuration:**
- Location: `~/.config/opencode/opencode.json`
- Permissions set during install:
  - `permission.read` - Allow reading GSD docs
  - `permission.external_directory` - Allow external directory access for GSD paths
- Format: Path-based allowlists in `opencode.json`

## Monitoring & Observability

**Error Tracking:**
- None - No external error tracking

**Logs:**
- Console output only (colored terminal output)
- No file-based logging
- No external log aggregation

**Health Checks:**
- Version check via npm registry (background, non-blocking)
- Cache-based to avoid repeated checks

## CI/CD & Deployment

**Hosting:**
- npm registry (npmjs.org)
- GitHub repository (github.com/glittercowboy/get-shit-done)

**CI Pipeline:**
- GitHub Actions only
- Workflow: `.github/workflows/release.yml`
- Trigger: Git tag push matching `v[0-9]+.[0-9]+.[0-9]+` pattern
- Steps:
  1. Checkout code
  2. Extract version from git tag
  3. Extract changelog section for version
  4. Create GitHub Release with release notes

**Release Automation:**
- Uses `softprops/action-gh-release@v2` to publish releases
- Releases to GitHub with changelog extracted from CHANGELOG.md

## Build & Package Distribution

**Package Manager:**
- npm registry
- Package: `get-shit-done-cc`
- Installation method: `npx get-shit-done-cc`
- Distribution files (in `package.json` files field):
  - `bin/` - Installation script
  - `commands/` - GSD commands
  - `agents/` - GSD agent definitions
  - `get-shit-done/` - Reference docs and templates
  - `hooks/dist/` - Bundled hook scripts
  - `scripts/` - Build scripts

## Environment Configuration

**Required env vars:**
- None (all optional, with defaults)

**Optional env vars:**
- `CLAUDE_CONFIG_DIR` - Override Claude Code config location (default: `~/.claude`)
- `OPENCODE_CONFIG_DIR` - Override OpenCode config location (respects XDG)
- `OPENCODE_CONFIG` - Alternative way to specify OpenCode config directory
- `XDG_CONFIG_HOME` - XDG Base Directory spec (used for OpenCode if set)

**Secrets location:**
- None - GSD stores no secrets
- Settings stored in user's config directory with restricted file permissions

## Webhooks & Callbacks

**Incoming:**
- None - GSD is not a server

**Outgoing:**
- None - No outgoing webhook calls

## Claude Code / OpenCode Integration

**Claude Code:**
- Installs to `~/.claude/`
- Registers hooks in `settings.json` under `hooks.SessionStart`
- Configures statusline in `settings.json` with command reference
- Commands invoked as `/gsd:command-name` (colon-based)
- Agents deployed to `agents/` directory

**OpenCode:**
- Installs to `~/.config/opencode/`
- Uses flat command structure: `command/gsd-*.md` (invoked as `/gsd-command`)
- Permissions configured in `opencode.json`
- Frontmatter converted from Claude Code format to OpenCode format:
  - `allowed-tools:` array → `tools:` object with boolean values
  - Color names (e.g., "cyan") → Hex colors (e.g., "#00FFFF")
  - Tool names normalized to lowercase
- Path references replaced during installation for compatibility

**Cross-Runtime Compatibility:**
- Single source tree with conversion during installation
- `convertClaudeToOpencodeFrontmatter()` handles format translation
- Tool name mapping: `AskUserQuestion` → `question`, `SlashCommand` → `skill`, etc.

## Installation & Setup Flow

**Interactive Setup:**
- Prompts for runtime selection (Claude Code / OpenCode / Both)
- Prompts for install location (Global or Local)
- Non-TTY fallback to defaults
- Path expansion for `~` and environment variables

**Installation Steps:**
1. Clean previous GSD files
2. Copy commands (flattened for OpenCode, nested for Claude Code)
3. Copy `get-shit-done/` reference directory
4. Copy agents (preserves user-created agents)
5. Copy CHANGELOG.md and VERSION file
6. Copy/bundle hooks to `hooks/dist/`
7. Configure settings.json (hooks, statusline)
8. Configure opencode.json permissions (OpenCode only)

**Uninstall:**
- Removes GSD-specific files and directories
- Preserves user's custom content
- Cleans orphaned hooks from previous versions
- Cleans permission entries from opencode.json

---

*Integration audit: 2026-01-24*
