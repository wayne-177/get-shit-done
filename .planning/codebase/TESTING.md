# Testing Patterns

**Analysis Date:** 2026-01-24

## Test Framework Status

**No automated test framework present.**

This is deliberate. The codebase prioritizes:
- Manual testing during development (shell testing hooks)
- Integration testing via real installation flows
- Hook verification through Claude Code/OpenCode runtime

**No files found:**
- No `jest.config.js`, `vitest.config.ts`, or test runners
- No `*.test.js` or `*.spec.js` files
- No assertion libraries (no Mocha, Chai, Vitest, etc.)

## Manual Testing Patterns

**Development setup for testing:**

```bash
npm link
npx get-shit-done-cc
```

Test locally before publishing (documented in CONTRIBUTING.md line 312).

**Hook validation:** Hooks (`gsd-check-update.js`, `gsd-statusline.js`) are tested via:
1. Installation to `.claude/hooks/` or `.config/opencode/command/`
2. Manual invocation in Claude Code/OpenCode sessions
3. Verification that output appears in statusline/console

**Path testing:** Windows path handling explicitly tested (mentioned in CONTRIBUTING.md, line 248):
```bash
# Works on Windows (test paths with backslashes)
```

## Installation Testing

**Test vectors for `bin/install.js`:**

1. **Interactive mode:**
   ```bash
   npx get-shit-done-cc
   # Select runtime, location, statusline options
   ```

2. **Non-interactive with flags:**
   ```bash
   npx get-shit-done-cc --claude --global
   npx get-shit-done-cc --opencode --local
   npx get-shit-done-cc --both --global
   ```

3. **Custom config directory:**
   ```bash
   npx get-shit-done-cc --claude --global --config-dir ~/.claude-bc
   ```

4. **Uninstall flow:**
   ```bash
   npx get-shit-done-cc --claude --global --uninstall
   ```

5. **Statusline handling:**
   - Fresh install: installs statusline
   - Existing statusline: prompts (or --force-statusline)
   - OpenCode: skips statusline (uses themes)

**Verification approach:**

Directory checks: `fs.existsSync()` and `fs.readdirSync()` verify files copied correctly
- Function `verifyInstalled(dirPath, description)` — checks directory exists and has entries
- Function `verifyFileInstalled(filePath, description)` — checks single file exists

JSON validation: `JSON.parse()` with try-catch in settings management

## What Gets Tested

### Code Paths Requiring Manual Verification

**Path handling across platforms:**
- Linux/Mac: `~/.claude/` and `~/.config/opencode/`
- Windows: UNC paths, backslash conversion
- Custom config directory expansion

**Runtime selection:**
- Claude Code (`.claude`) vs OpenCode (`.config/opencode`)
- Both runtimes installed side-by-side
- Flattened command structure for OpenCode

**Frontmatter conversion** (`convertClaudeToOpencodeFrontmatter()`):
- YAML `allowed-tools:` array → YAML `tools:` object with tool names converted
- Color names (cyan) → hex (#00FFFF)
- Command paths `/gsd:` → `/gsd-` (flat structure)
- Preserves non-tool fields

**Settings.json manipulation:**
- Adding SessionStart hooks without duplication
- Removing orphaned hooks on cleanup
- Statusline configuration (command or URL)
- Preserving user's other settings

### Code Paths NOT Covered

**Negative test cases are manual:**
- Invalid JSON in settings.json (handled with try-catch, user warned)
- Missing directories mid-installation (caught by verify functions)
- Permission issues on write (Node errors surface naturally)
- Concurrent installations (undefined behavior, assume single user)

**Environment variations:**
- WSL2 stdin handling (detected via `process.stdin.isTTY`)
- Non-TTY environments (fall back to non-interactive defaults)
- Missing npm (affects update check only, fails gracefully)

## Build Testing

**Build script: `scripts/build-hooks.js`**

Copies hooks to dist for bundling:

```bash
npm run build:hooks
```

**What it verifies:**
- Source files exist: `gsd-check-update.js`, `gsd-statusline.js`
- Destination directory created: `hooks/dist/`
- Files actually copied (logs verification)

**Manual test:**
```bash
npm run build:hooks
ls hooks/dist/  # Should contain both hook files
```

## Hook Testing

### `gsd-check-update.js`

**What it does:**
1. Spawns background process (non-blocking)
2. Reads installed version from VERSION file (project or global)
3. Calls `npm view get-shit-done-cc version` with 10s timeout
4. Writes result to `~/.claude/cache/gsd-update-check.json`

**Manual verification:**
```bash
# After installation
ls ~/.claude/cache/gsd-update-check.json
cat ~/.claude/cache/gsd-update-check.json
# Should show: { update_available: bool, installed: "X.Y.Z", latest: "X.Y.Z", checked: timestamp }
```

**Test cases:**
- Installed < latest → `update_available: true`
- Installed == latest → `update_available: false`
- npm command fails → `latest: "unknown"`
- Cache file persists across runs

### `gsd-statusline.js`

**What it does:**
1. Reads JSON from stdin (provided by Claude Code)
2. Extracts model, directory, context window, session ID
3. Reads todos file from `~/.claude/todos/` for current task
4. Reads update cache from `~/.claude/cache/gsd-update-check.json`
5. Writes colored statusline to stdout

**Manual verification:**
```bash
# Simulate input
echo '{"model":{"display_name":"Claude 3.5 Sonnet"},"workspace":{"current_dir":"/home/user/project"},"session_id":"abc123","context_window":{"remaining_percentage":45}}' | node hooks/gsd-statusline.js
# Output: colored status line (model | task | dir | context bar)
```

**Test cases:**
- With task: `model | task | dir | context`
- Without task: `model | dir | context`
- Update available: `⬆ /gsd:update │ model | ...`
- No remaining_percentage: skips context display
- Missing todos dir: skips task display
- JSON parse error: silent fail (line 81)

## Testing Philosophy

**No unit tests because:**
1. Code is procedural (setup/installation scripts)
2. Heavy I/O and OS integration make mocking difficult
3. Real testing (actual installation) more valuable than mocked tests
4. Single-user, solo developer workflow (no regression risk)

**Real testing instead:**
- Maintainers test locally before merging
- Users test via `npm link` during development
- CI validates Windows path handling in PR checks
- Each release tested before publishing to npm

## Testing Checkpoints in GSD Artifacts

While the GSD codebase itself has no tests, GSD is used to *manage* test-driven projects. Reference files for test-driven projects:

- `.planning/references/tdd.md` — TDD cycle (RED → GREEN → REFACTOR)
- Test framework detection in other projects via `npm list jest` or `npm list vitest`
- Phase discovery confirms test setup before planning

## Continuous Integration

**GitHub Actions expected but not configured** (observed in `.github/` directory structure)

GSD release process (from CONTRIBUTING.md):
```bash
npm version minor
git add package.json CHANGELOG.md
git commit -m "chore: release v1.10.0"
git tag -a v1.10.0 -m "Release v1.10.0"
git push origin main --tags
npm publish
```

Pre-publish checklist (implicit):
1. Manual local testing with `npm link`
2. Test on Windows if touching paths
3. Verify both Claude Code and OpenCode installation
4. Check CHANGELOG.md updated
5. Tag and publish

## Verification Strategy

**Install verification approach:**

```javascript
// Function used after each component installation
function verifyInstalled(dirPath, description) {
  if (!fs.existsSync(dirPath)) {
    console.error(`  ${yellow}✗${reset} Failed to install ${description}: directory not created`);
    return false;
  }
  try {
    const entries = fs.readdirSync(dirPath);
    if (entries.length === 0) {
      console.error(`  ${yellow}✗${reset} Failed to install ${description}: directory is empty`);
      return false;
    }
  } catch (e) {
    console.error(`  ${yellow}✗${reset} Failed to install ${description}: ${e.message}`);
    return false;
  }
  return true;
}
```

**Failure collection:**
- Tracks installation failures in `failures` array
- Exits with error code 1 if any critical component fails (line 988)
- Provides helpful message: `Try running directly: node ~/.npm/_npx/*/node_modules/get-shit-done-cc/bin/install.js --global`

## Coverage

**No coverage tracking** (no coverage tool, no coverage reports)

**Implicit coverage areas:**
- Installation: 5+ code paths (interactive, flags, uninstall, both runtimes, custom config)
- Path handling: ~20+ code branches (global/local, tilde expansion, OpenCode vs Claude)
- JSON manipulation: settings read/write, hook registration cleanup
- File operations: copy, delete, find, verify

**Untested edge cases:**
- Corrupted VERSION file
- Out-of-disk during installation
- Permission denied on config directory
- Symlinks in installation path
- Very long file paths (Windows limitations)

---

*Testing analysis: 2026-01-24*
