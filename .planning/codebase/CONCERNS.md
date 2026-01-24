# Codebase Concerns

**Analysis Date:** 2026-01-24

## Tech Debt

**Fork Divergence and Update Safety:**
- Issue: This codebase is a fork with agentic additions merged from `wayne-177/get-shit-done-agentic`, but the `/gsd:update` command still runs `npx get-shit-done-cc@latest` from upstream npm, which would overwrite local modifications
- Files: `commands/gsd/update.md`, `hooks/gsd-check-update.js`
- Impact: Running `/gsd:update` silently wipes 30+ custom workflows and 8+ commands that are not in upstream. Users lose data without warning
- Fix approach: Either (1) disable update checking in forks, (2) add explicit confirmation dialog showing what would be lost, or (3) provide fork-aware version checking that compares against local package.json instead of npm registry

**Large Agent Files Approach Context Limits:**
- Issue: Several agent files exceed 1,300 lines (`gsd-planner.md` at 1,386 lines, `gsd-debugger.md` at 1,203 lines)
- Files: `agents/gsd-planner.md`, `agents/gsd-debugger.md`, `agents/gsd-executor.md`
- Impact: Difficult to modify, refactor, or test individual concerns within these agents. At >1000 lines, single-file agents become monoliths that violate separation of concerns
- Fix approach: Split large agents into smaller specialized agents or move common logic to reusable `.planning/lib/` files. For example, `gsd-planner.md` could delegate phase research to `gsd-phase-researcher.md`

**Hardcoded Directory Assumptions:**
- Issue: Installation scripts and hooks assume `.claude` directory structure hardcoded in multiple places. For OpenCode, code still references both `~/.config/opencode` and `~/.opencode` in different places
- Files: `bin/install.js` (lines 12, 16-17, 64, 88, 186-190), `hooks/gsd-check-update.js` (lines 12, 16-17), `hooks/gsd-statusline.js` (lines 46, 64)
- Impact: Custom config directories are partially supported (via `--config-dir`) but paths are still scattered across codebase. If users relocate config, some hooks may fail
- Fix approach: Create `lib/paths.js` that exports config directory resolution functions, use it consistently across all files. This eliminates path hardcoding duplication

**No Centralized Error Handling:**
- Issue: Hooks silently fail with `catch (e) {}` blocks (`gsd-check-update.js`, `gsd-statusline.js`). Installation failure handling prints to stderr/stdout but doesn't log to persistent location
- Files: `bin/install.js` (line 812), `hooks/gsd-check-update.js` (lines 41, 46), `hooks/gsd-statusline.js` (lines 58, 71, 81)
- Impact: Users cannot debug why updates aren't checking or statusline isn't showing. Errors are swallowed silently
- Fix approach: Create `lib/logging.js` that writes to `~/.claude/logs/gsd-*.log`. Log all hook operations and errors there instead of to console

## Known Bugs

**Statusline Context Window Display Edge Case:**
- Symptoms: Context window percentage can show >100% if `remaining_percentage` is negative or malformed
- Files: `hooks/gsd-statusline.js` (lines 24-25)
- Trigger: When context window data contains NaN or negative remaining_percentage values
- Workaround: None - status line will display incorrect percentage but won't crash
- Root cause: `Math.max(0, Math.min(100, 100 - rem))` clamps output but `rem` can be undefined, leading to `100 - undefined = NaN`

**Installation Race Condition on Windows:**
- Symptoms: When installing globally on Windows during a Claude Code session, hooks directory may be locked by running processes
- Files: `bin/install.js` (lines 966-981)
- Trigger: Running installer while Claude Code has sessions active, hooks are being read
- Workaround: Close all Claude Code instances before installing
- Root cause: No file lock detection or retry logic in `fs.copyFileSync()` calls

**Version File Not Guaranteed to Exist:**
- Symptoms: Update checker silently fails if VERSION file doesn't exist at either location
- Files: `hooks/gsd-check-update.js` (lines 34-40)
- Trigger: Clean installation or manual file deletion
- Workaround: Reinstall GSD
- Root cause: No fallback to package.json version as last resort

## Security Considerations

**JSON.parse() Without Validation in Hooks:**
- Risk: `gsd-statusline.js` parses stdin JSON without schema validation. If malformed input is sent, unexpected values propagate
- Files: `hooks/gsd-statusline.js` (line 15)
- Current mitigation: Try-catch block silently fails (line 81)
- Recommendations:
  1. Add strict object shape validation (`if (!data.model || typeof data.model !== 'object')`)
  2. Create `.planning/lib/types.js` with TypeScript-style validators for stdin schemas
  3. Log parsing errors to persistent log file instead of silent fail

**execSync() Without Timeout or Bounds:**
- Risk: `gsd-check-update.js` calls `npm view get-shit-done-cc version` with 10-second timeout. Network failures or npm registry slowness could still block statusline
- Files: `hooks/gsd-check-update.js` (line 45)
- Current mitigation: 10-second timeout is specified
- Recommendations:
  1. Reduce timeout to 5 seconds (npm registry should respond in <1 second normally)
  2. Add explicit network error handling and fallback to cached version if fetch fails
  3. Cache results for 24 hours instead of checking every session

**File Path Traversal Risk in Frontmatter Conversion:**
- Risk: `bin/install.js` processes path references with regex replacements that don't validate results. Malicious frontmatter could construct paths like `../../sensitive`
- Files: `bin/install.js` (lines 274-371)
- Current mitigation: Source files come from trusted npm package
- Recommendations:
  1. Validate all replacement paths with `path.resolve()` and ensure they're within expected directories
  2. Add unit tests for path replacement with adversarial inputs

## Performance Bottlenecks

**Synchronous File Reads in Statusline:**
- Problem: `gsd-statusline.js` does synchronous file I/O (`fs.readdirSync`, `fs.statSync`, `fs.readFileSync`) which blocks hook execution
- Files: `hooks/gsd-statusline.js` (lines 48-60, 65-72)
- Cause: Statusline is called every keystroke/model response. With large todos directories (100+ files), directory scan takes milliseconds
- Improvement path:
  1. Cache todo file metadata for 5 minutes instead of scanning on every keystroke
  2. Use async file I/O but return cached result immediately
  3. Limit directory scan to last 10 modified files instead of all files

**Background Process Management:**
- Problem: `gsd-check-update.js` spawns Node process every session via `spawn()`. With thousands of users, this creates many orphaned processes on slow systems
- Files: `hooks/gsd-check-update.js` (lines 25-59)
- Cause: `child.unref()` releases parent but child still runs
- Improvement path:
  1. Check cache age before spawning: only spawn if cache is >1 hour old
  2. Write PID file to prevent concurrent checks
  3. Use `timeout` command on Unix to prevent runaway processes

**Installation Directory Traversal:**
- Problem: `bin/install.js` recursively copies entire `agents/` directory with in-memory path replacement on every file
- Files: `bin/install.js` (lines 905-940)
- Cause: No parallel processing, no streaming file copies
- Improvement path: For large installations (100+ agents), batch file copies or use `cp -r` shell command with post-processing instead

## Fragile Areas

**Frontmatter Conversion Logic:**
- Files: `bin/install.js` (lines 224-371)
- Why fragile: Line-by-line YAML parsing is brittle. Complex frontmatter with:
  - Nested objects (`metadata: { key: value }`)
  - Comments (`# comment after value`)
  - Multi-line strings (YAML `|` blocks)
  - Mixed tool formats (some `allowed-tools:`, some `tools:`)

  Will either fail silently or produce invalid output
- Safe modification: Add YAML parser library (js-yaml) instead of regex/line parsing. Update `package.json` dependencies
- Test coverage: No tests for edge cases like multiline descriptions or nested config

**Hook Command Registration in settings.json:**
- Files: `bin/install.js` (lines 991-1026)
- Why fragile: Code assumes specific hook structure in settings.json (SessionStart array with hooks array). If Claude Code changes settings.json format or adds nested validation, installation breaks silently
- Common failures: Users with custom hook configurations lose them during upgrade because cleanup logic (lines 506-527) may over-match patterns
- Safe modification: Only add/remove specific hook entries by UUID/command pattern, don't wipe entire SessionStart array
- Test coverage: No integration tests with real Claude Code settings files

**Installation Verification Logic:**
- Files: `bin/install.js` (lines 800-816)
- Why fragile: `verifyInstalled()` only checks directory existence and entry count. Doesn't verify:
  - Files have correct content
  - Permissions are readable
  - Symlinks are valid (on Unix)

  Installation could appear successful but fail at runtime
- Safe modification: Add checksum validation (`md5(file) === expected`) for critical files
- Test coverage: No regression tests for corrupted installations

## Scaling Limits

**Single Cache File for Update Checks:**
- Current capacity: Unlimited sessions share one `~/.claude/cache/gsd-update-check.json` file
- Limit: If multiple Claude Code sessions write simultaneously, race condition corrupts cache file
- Scaling path: Use file locking (flock on Unix, LockFile on Windows) or write-to-temp + rename pattern

**Todos Directory Scan in Statusline:**
- Current capacity: Scales linearly with number of todo files (typically <100)
- Limit: With 1000+ todo files (large projects with many sessions), statusline scan becomes noticeable (>50ms)
- Scaling path: Index todo files by session ID in separate subdirectories, or use SQLite instead of filesystem enumeration

## Dependencies at Risk

**No Production Dependencies:**
- Risk: Package has zero dependencies (by design). But build system needs esbuild (dev dependency) to work. If esbuild disappears or breaks, build fails
- Impact: Cannot produce dist/ hooks without esbuild
- Migration plan: Add `postinstall` script that regenerates hooks, or commit pre-built hooks to git

**Node.js Version Requirement:**
- Risk: Codebase specifies `"engines": { "node": ">=16.7.0" }` but code uses modern JavaScript (const, arrow functions, nullish coalescing)
- Impact: Works on Node 16.7.0 but not older versions. Constraint is loose (could be >=16.14.0 safely)
- Migration plan: Test on Node 16.7.0 to confirm, or bump minimum to 16.14.0 and document requirement

## Test Coverage Gaps

**Installation Flow Not Tested End-to-End:**
- What's not tested: Full interactive installation flow across platforms (Windows with WSL2, macOS with custom config dirs, Linux with XDG)
- Files: `bin/install.js`, `commands/gsd/*.md`, agents
- Risk: Bugs in installation only discovered after users report issues (happened with WSL2 in v1.6.4)
- Priority: High - installation is critical path. Breaking it blocks all usage

**Hook Behavior Not Tested Across Context States:**
- What's not tested: Statusline behavior with:
  - Missing todos directory
  - Corrupted todo JSON files
  - Context window at exactly 100% / 0%
  - Very long task names (>80 characters)
  - Rapid successive calls (stress test)
- Files: `hooks/gsd-statusline.js`, `hooks/gsd-check-update.js`
- Risk: Edge cases silently fail, users see blank statusline without knowing why
- Priority: High - statusline is visible every interaction

**OpenCode Compatibility Not Verified:**
- What's not tested: Actual OpenCode installation and command execution with converted frontmatter
- Files: `bin/install.js` (frontmatter conversion), `agents/`, `commands/`
- Risk: OpenCode support added in v1.9.6 but only tested with regex replacements, not actual OpenCode runtime
- Priority: Medium - affects new users choosing OpenCode, but smaller user base than Claude Code

**Cleanup Logic for Orphaned Files Not Regression Tested:**
- What's not tested: Installing v1.9.13 over previous broken installs. Cleanup (lines 476-534) only tests specific filenames, doesn't handle:
  - Custom user files accidentally in GSD directories
  - Files from future versions (backward compatibility)
  - Permission errors during deletion
- Files: `bin/install.js` (cleanupOrphanedFiles, cleanupOrphanedHooks)
- Risk: Cleanup silently fails, leaving corrupt state for next upgrade
- Priority: Medium - affects upgrade path, impacts each user ~quarterly

---

*Concerns audit: 2026-01-24*
