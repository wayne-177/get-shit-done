# Technology Stack

**Analysis Date:** 2026-01-24

## Languages

**Primary:**
- JavaScript (Node.js) - Installation and CLI infrastructure, hooks, and build scripts
- Markdown - Command definitions, agent definitions, reference documentation, templates

**Secondary:**
- YAML - Configuration format for command and agent frontmatter (Claude Code/OpenCode format)

## Runtime

**Environment:**
- Node.js 16.7.0 or later (see `package.json` engines field)

**Package Manager:**
- npm (used for dependency management)
- Lockfile: `package-lock.json` (present and version 3)

## Frameworks

**Core:**
- None (zero production dependencies) - GSD is built purely with Node.js standard library modules

**Build/Dev:**
- esbuild ^0.24.0 - Bundler for hook compilation (devDependency only)

## Key Dependencies

**Production Dependencies:**
- None - The package has zero production dependencies for maximum portability

**Dev Dependencies:**
- esbuild - For compiling hooks to distribution directory (`hooks/dist/`)

## Node.js Built-in Modules Used

**Installation (`bin/install.js`):**
- `fs` - File system operations (read/write config files, copy directories)
- `path` - Cross-platform path handling for config directories
- `os` - System home directory detection, platform detection
- `readline` - Interactive prompts for runtime and install location selection

**Hooks (`hooks/gsd-*.js`):**
- `fs` - File I/O for cache and version files
- `path` - Path resolution for global and local config
- `os` - Home directory access
- `child_process` - Subprocess spawning for background checks, running npm commands

**Build Scripts (`scripts/build-hooks.js`):**
- `fs` - Copy operations
- `path` - Path manipulation

## Configuration

**Environment:**
- CLI-driven configuration (interactive install prompts or command-line flags)
- Environment variables read:
  - `CLAUDE_CONFIG_DIR` - Override default Claude Code config location (`~/.claude`)
  - `OPENCODE_CONFIG_DIR` - Override OpenCode config location
  - `OPENCODE_CONFIG` - Alternative OpenCode config spec (uses its directory)
  - `XDG_CONFIG_HOME` - XDG standard for config directory

**Config Directories (Runtime):**
- Claude Code: `~/.claude/` (or `CLAUDE_CONFIG_DIR` override)
- OpenCode: `~/.config/opencode/` (or `OPENCODE_CONFIG_DIR` override, respects XDG_CONFIG_HOME)
- Local project install: `./.claude/` or `./.opencode/` (created in project root)

**Settings Files:**
- `~/.claude/settings.json` - Claude Code settings (hooks, statusline configuration)
- `~/.config/opencode/opencode.json` - OpenCode permissions and settings
- `.claude/settings.local.json` - Example local settings file

**Build Configuration:**
- `scripts/build-hooks.js` - Custom build script (copies hooks to `hooks/dist/`)
- `package.json` - Defines `build:hooks` script and `prepublishOnly` hook

## Platform Requirements

**Development:**
- Any OS with Node.js 16.7.0+ (Mac, Windows, Linux supported)
- bash or shell for interactive prompts
- TTY for interactive mode (falls back to non-interactive on environments like WSL2)

**Installation/Deployment:**
- npm (for `npx get-shit-done-cc` installation)
- Node.js 16.7.0+ on target system
- Write access to config directory (`~/.claude/` or `~/.config/opencode/`)

**Distribution:**
- Published to npm as `get-shit-done-cc` package
- Installed globally via `npx get-shit-done-cc`
- See `package.json` files entry for what gets distributed (excludes node_modules, `.git`)

## Notable Technical Decisions

**Zero Production Dependencies:**
- Reduces security surface and installation size
- Uses only Node.js standard library (available since Node 16)
- Maximizes portability across environments

**Cross-Platform Path Handling:**
- Forward slashes used in hook commands for Windows compatibility
- Path expansion for `~` handled manually (shells don't expand in env vars)
- XDG Base Directory spec compliance for OpenCode

**Background Process Management:**
- Update checks run in detached background process (non-blocking)
- Version comparison uses npm registry query (`npm view get-shit-done-cc version`)
- Cache-based approach avoids repeated network calls

---

*Stack analysis: 2026-01-24*
