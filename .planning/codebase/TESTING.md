# Testing Patterns

**Analysis Date:** 2026-01-24

## Overview

Get Shit Done uses a multi-layered testing strategy appropriate to its architecture as a meta-prompting system:

1. **Specification-as-test** — PLAN.md files contain expected behavior; verification against PLAN is the test
2. **Workflow verification** — Checkpoints and human gates verify critical operations
3. **Automated verification** — Shell scripts in `/gsd:verify-phase` check for TODO comments, stubs, incomplete patterns
4. **Integration testing** — End-to-end phase execution with checkpoint gates
5. **Manual testing** — Installation verification, path handling on Windows/Mac/Linux

No traditional unit test framework (Jest, Vitest) is configured. Testing happens through:
- Bash scripts that verify file states
- Human checkpoints for critical decisions
- Phase verification workflows
- Installation testing with multiple CLI flag combinations

---

## Testing Philosophy

**Core principle:** GSD optimizes for solo developer + Claude workflow. Testing serves to:
- Catch incomplete implementations (stubs, TODOs, placeholders)
- Verify file system state (directories created, files written)
- Validate command execution (proper return codes, output format)
- Gate critical operations until human verification
- Provide confidence before merging to main

**What's NOT tested:**
- Individual helper functions (they're trivial)
- Node.js builtin modules (fs, path, etc.)
- npm packages (rely on their tests)

**What IS tested:**
- End-to-end workflows (create-roadmap → plan-phase → execute-phase)
- Installation across runtimes (Claude, OpenCode) and locations (global, local)
- Phase state (ROADMAP.md, PLAN.md, SUMMARY.md artifacts created)
- Verification gates (TODO comments flagged, stubs detected)

---

## Test Organization

### Test Files Location

Tests exist as shell scripts and verification workflows, not separate test files:

**Verification workflows:**
- `get-shit-done/workflows/verify-phase.md` — Phase completion verification
- `get-shit-done/references/verification-patterns.md` — Pattern detection guide

**Hook tests (manual):**
- Test paths with Windows backslashes: `bin/install.js` handles `C:\Users\...`
- Test XDG compliance: OpenCode uses `~/.config/opencode/opencode.json`
- Test config directory precedence: `CLAUDE_CONFIG_DIR` > `~/.claude`

**Installation tests:**
- Global install: `npx get-shit-done-cc --claude --global`
- Local install: `npx get-shit-done-cc --claude --local`
- OpenCode: `npx get-shit-done-cc --opencode --global`
- Both runtimes: `npx get-shit-done-cc --both --global`
- Uninstall: `npx get-shit-done-cc --claude --global --uninstall`

### No Test Runner

No Jest, Vitest, Mocha, or other test framework is configured. Instead:
- Verification happens in running `npm test` (currently not configured but expected to run installation)
- Phase verification uses bash script patterns in workflows
- Acceptance is manual checkpoint gates

---

## Test Structure

### Verification Workflow Pattern

Location: `get-shit-done/workflows/verify-phase.md`

Verification follows this structure for each phase:

**1. Check for incomplete code:**
```bash
# Look for stubs
grep -c -E "TODO|FIXME|placeholder|not implemented" "$path"

# Check for inline console.log
grep -c "console\.log" "$file"

# Search for empty implementations
grep -E "return null|return \[\]|return {}" "$file"
```

**2. Check for comments:**
```bash
# TODO/FIXME detection
grep -n -E "TODO|FIXME|XXX|HACK" "$file"

# Output pattern (for tables)
echo "| $file | $(line_number) | TODO/FIXME | ⚠️ Warning |"
```

**3. Verify file structure:**
```bash
# Check if required files exist
[ -f .planning/ROADMAP.md ] || exit 1
[ -f .planning/phases/NN-name/PLAN.md ] || exit 1
[ -f .planning/phases/NN-name/SUMMARY.md ] || exit 1
```

**4. Verify execution artifacts:**
```bash
# Check commit history
git log --oneline | head -5

# Verify phase directory
ls -la .planning/phases/NN-name/
```

### Test Categories

**Type 1: File existence checks**
- Verify PLAN.md created
- Verify SUMMARY.md created
- Verify phase directory created
- Verify CHANGELOG updated

Pattern from `verify-phase.md`:
```bash
if [ ! -f ".planning/phases/$PHASE_NUM-$PHASE_SLUG/PLAN.md" ]; then
  echo "ERROR: Plan not created"
  exit 1
fi
```

**Type 2: Content validation**
- Detect TODO/FIXME comments (⚠️ warning)
- Detect placeholder text
- Detect stubs and incomplete functions
- Check for excessive console.log (debugging leftover)

Pattern:
```bash
stubs=$(grep -c -E "TODO|FIXME|placeholder|not implemented|coming soon" "$path")
if [ "$stubs" -gt 0 ]; then
  echo "⚠️ Warning: $stubs stubs found"
fi
```

**Type 3: State verification**
- ROADMAP.md contains all phases
- STATE.md tracks progress
- Git history has commits with correct format
- Artifacts match expected structure

**Type 4: Integration verification**
- Phase execution completes without errors
- Hooks installed correctly
- Commands available in runtime (Claude/OpenCode)
- Installation can be uninstalled cleanly

---

## Mocking

**Pattern:** Not applicable to GSD's architecture

GSD uses real file I/O and bash execution. "Mocking" happens through:
- Checkpoints that pause execution for human review
- Conditional logic based on file existence
- Safe guards that prevent destructive operations

Example from `bin/install.js` (verification before deletion):
```javascript
// Clean install: remove existing destination to prevent orphaned files
if (fs.existsSync(destDir)) {
  fs.rmSync(destDir, { recursive: true });
}
```

Verification before use:
```javascript
function verifyInstalled(dirPath, description) {
  if (!fs.existsSync(dirPath)) {
    console.error(`  ${yellow}✗${reset} Failed to install ${description}`);
    return false;
  }
  // ... check contents
  return true;
}
```

---

## Fixtures and Test Data

### Project Test Data

For manual testing, create a test project:
```bash
mkdir test-gsd-project
cd test-gsd-project
npm link get-shit-done-cc

# Test interactive mode
npx get-shit-done-cc

# Test specific runtimes
npx get-shit-done-cc --claude --global
npx get-shit-done-cc --opencode --global --config-dir ~/.opencode-test

# Test local install
npx get-shit-done-cc --claude --local
```

### Installation Cleanup

After testing, verify cleanup works:
```bash
# Test uninstall
npx get-shit-done-cc --claude --global --uninstall

# Verify directories removed
ls ~/.claude/commands/gsd/  # should not exist
ls ~/.claude/get-shit-done/  # should not exist
```

---

## Coverage

### What's Covered

**Installation logic** (primary test surface):
- Argument parsing (--global, --local, --uninstall, --config-dir)
- Runtime selection (Claude, OpenCode, both)
- Interactive prompts (TTY detection, readline handling)
- Directory creation and file copying
- Path replacement in markdown files
- Settings.json hook configuration
- Orphaned file cleanup
- Permissions configuration (OpenCode)

**Command execution:**
- Slash command parsing (`/gsd:command-name`)
- Step execution order
- Bash script execution and output capture
- File references (`@.planning/PROJECT.md`)

**Verification:**
- TODO/FIXME comment detection
- Stub identification (empty returns, placeholders)
- State file validation

### What's NOT Covered

**Helper functions:**
- `expandTilde()` — trivial string manipulation
- `parseConfigDirArg()` — tested implicitly through arg parsing
- `buildHookCommand()` — tested implicitly through installation

**Node.js builtins:**
- `fs.readFileSync()`, `fs.writeFileSync()` — rely on Node.js
- `path.join()` — rely on Node.js
- `child_process.spawn()` — tested implicitly through hook execution

**No specific coverage targets** — GSD prioritizes end-to-end flow over unit test coverage

---

## Test Types

### Installation Tests (Manual)

**What:** Verify installation works across configurations

**How:** Run install script with different flags

**Test cases:**
```bash
# Test 1: Claude global install
npx get-shit-done-cc --claude --global
# Verify: ~/.claude/commands/gsd/ exists
# Verify: ~/.claude/get-shit-done/ exists
# Verify: ~/.claude/hooks/ has gsd-statusline.js and gsd-check-update.js

# Test 2: OpenCode global install
npx get-shit-done-cc --opencode --global
# Verify: ~/.config/opencode/command/ has gsd-*.md files
# Verify: ~/.config/opencode/opencode.json has permissions configured

# Test 3: Local install
npx get-shit-done-cc --claude --local
# Verify: .claude/commands/gsd/ exists (local)
# Verify: .claude/get-shit-done/ exists (local)

# Test 4: Uninstall removes all GSD files
npx get-shit-done-cc --claude --global --uninstall
# Verify: ~/.claude/commands/gsd/ doesn't exist
# Verify: ~/.claude/get-shit-done/ doesn't exist
# Verify: ~/.claude/hooks/gsd-*.js removed
# Verify: settings.json hooks cleaned up

# Test 5: Windows path handling
# Test with paths containing backslashes
# Verify: Forward slashes used in Node.js calls
```

### Phase Execution Tests

**What:** Verify phase planning and execution workflow

**How:** Run `/gsd:plan-phase 1` and `/gsd:execute-phase 1` with test project

**Test flow:**
1. Create test project with PROJECT.md
2. Create ROADMAP.md with phases
3. Run `/gsd:plan-phase 1` → verify PLAN.md created
4. Run `/gsd:execute-phase 1` → verify tasks created
5. Complete tasks (manual or scripted)
6. Run `/gsd:verify-phase 1` → check for stubs/TODOs

### Verification Tests

**What:** Detect incomplete/stub implementations

**How:** Run bash patterns from `verify-phase.md`

**Patterns tested:**
```bash
# Pattern 1: TODO/FIXME comments
grep -E "TODO|FIXME|XXX|HACK" "$file"
# Expected: Empty output after implementation

# Pattern 2: Empty implementations
grep "return null\|return \[\]\|return {}" "$file"
# Expected: Only legitimate empty returns

# Pattern 3: Placeholder text
grep -i "placeholder\|not implemented\|coming soon" "$file"
# Expected: Empty output

# Pattern 4: Debug console.log
grep "console\.log" "$file"
# Expected: Empty (or explained in comments)
```

---

## Common Test Patterns

### Bash Validation Pattern

Used in all command steps:
```bash
# Validation check
[ -f .planning/PROJECT.md ] || {
  echo "ERROR: PROJECT.md not found"
  exit 1
}

# Result captured for use
STATUS=$([ -f .planning/ROADMAP.md ] && echo "EXISTS" || echo "MISSING")
```

### File Verification Pattern

Check before reading/modifying:
```javascript
// Read and validate
if (!fs.existsSync(filePath)) {
  console.error(`File not found: ${filePath}`);
  process.exit(1);
}

// Parse with error handling
try {
  const data = JSON.parse(fs.readFileSync(filePath, 'utf8'));
} catch (e) {
  console.error(`Failed to parse: ${e.message}`);
  process.exit(1);
}
```

### State Verification Pattern

Check artifacts after execution:
```bash
# After creating roadmap
[ -f .planning/ROADMAP.md ] || exit 1
[ -f .planning/STATE.md ] || exit 1
[ -d .planning/phases/01-name ] || exit 1

# After executing phase
[ -f .planning/phases/01-name/PLAN.md ] || exit 1
[ -f .planning/phases/01-name/SUMMARY.md ] || exit 1
[ -n "$(git log --oneline | head -1)" ] || exit 1
```

---

## Regression Prevention

### Code Review Checklist

Before merging feature branches, verify:

**From CONTRIBUTING.md:**
- [ ] Follows GSD style (no enterprise patterns, no filler)
- [ ] Updates CHANGELOG.md for user-facing changes
- [ ] Doesn't add unnecessary dependencies
- [ ] Works on Windows (test paths with backslashes)

**Installation testing:**
- [ ] Test `npx get-shit-done-cc --claude --global`
- [ ] Test `npx get-shit-done-cc --opencode --global`
- [ ] Test `npx get-shit-done-cc --local`
- [ ] Test uninstall: `npx get-shit-done-cc --claude --global --uninstall`
- [ ] Verify settings.json hooks cleaned up

**Path handling:**
- [ ] Use `path.join()` not string concatenation
- [ ] Use forward slashes in Node.js commands
- [ ] Test `~` expansion on Mac/Linux
- [ ] Test `%USERPROFILE%` equivalent on Windows

**Commands:**
- [ ] No temporal language ("we changed", "previously")
- [ ] No filler language ("let me", "basically")
- [ ] Imperative voice ("do X", not "X should be done")

---

## Test Execution

### How Tests Run

**Manual installation testing:**
```bash
npm link
npx get-shit-done-cc --help  # Verify help works
npx get-shit-done-cc --claude --global  # Test installation
```

**Verification in phase:**
```bash
/gsd:verify-phase 1  # Runs bash patterns from verify-phase.md
```

**Expected output:**
```
Phase Verification Report
========================

[Files Created]
✓ .planning/phases/01-name/PLAN.md
✓ .planning/phases/01-name/SUMMARY.md
✓ Commit history recorded

[Code Quality]
⚠️ Warning: 2 TODO comments found
✓ No stubs detected
✓ No console.log leftover

[Summary]
Status: ⚠️ Ready for review (minor warnings)
```

### Continuous Integration

**Current:** No automated CI configured

**Recommended CI checks (if adding):**
- Syntax check: `node -c bin/install.js`
- Path validation: Verify no hardcoded backslashes
- YAML validation: Check command frontmatter
- Installation test: Run `npm link && npx get-shit-done-cc --help`

---

## Best Practices for Testing

### When Writing Tests

1. **Test the interface, not implementation**
   - Don't test `expandTilde()` internals
   - Test that install script properly handles `~/.claude/` paths

2. **Use bash exit codes**
   - Exit 0 on success
   - Exit 1 on failure
   - Check with: `$?` or `|| exit 1`

3. **Verify file state, not functions**
   - Don't mock file I/O
   - Actually create files and verify they exist
   - Clean up after tests

4. **Check both happy path and error cases**
   - Test `--uninstall` removes files
   - Test missing PROJECT.md fails validation
   - Test invalid JSON fails gracefully

### When Debugging Tests

1. **Enable verbose output**
   - Add `set -x` to bash scripts
   - Use `console.log()` before throwing

2. **Check file state**
   ```bash
   ls -la ~/.claude/
   cat .planning/PLAN.md
   git log --oneline
   ```

3. **Isolate the failure**
   - Test specific flag combination
   - Create minimal test project
   - Run single step in isolation

---

## Acceptance Criteria

A feature is considered "tested" when:

- [ ] Installation succeeds across all runtime/location combinations
- [ ] Uninstallation cleanly removes all GSD files
- [ ] ROADMAP.md can be created and verified
- [ ] Phase planning produces valid PLAN.md
- [ ] Phase execution produces SUMMARY.md with commits
- [ ] `/gsd:verify-phase` detects any incomplete code
- [ ] Windows paths are handled correctly
- [ ] No new TODO comments introduced without justification
- [ ] Code follows GSD style (no enterprise patterns, imperative voice)
- [ ] CHANGELOG.md updated for user-facing changes

---

## Summary

GSD testing prioritizes:
1. **Real file I/O** — Test against actual filesystem
2. **End-to-end flows** — Verify complete workflows work
3. **Verification gates** — Detect incomplete implementations
4. **Multi-platform validation** — Test Windows/Mac/Linux paths
5. **Manual checkpoints** — Human reviews for critical operations

Testing is lightweight, focused, and integrated into the development workflow—no separate test suite to maintain.

---

*Testing analysis: 2026-01-24*
