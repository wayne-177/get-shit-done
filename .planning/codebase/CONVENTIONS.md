# Coding Conventions

**Analysis Date:** 2026-01-24

## Language & Philosophy

This codebase prioritizes **clarity and directness** following the principle: "The complexity is in the system, not in your workflow."

**Core principles:**
- Imperative voice (direct instructions)
- No filler language ("Let me", "Just", "Simply")
- No sycophancy ("Great!", "Awesome!")
- Brevity with substance
- Technical precision over politeness

## Naming Patterns

**Files:**
- Kebab-case for markdown and scripts: `execute-phase.md`, `gsd-check-update.js`
- Snake_case for XML step attributes: `name="load_project_state"`
- UPPERCASE.md for static reference files: `CONVENTIONS.md`, `TESTING.md`

**Functions:**
- camelCase for JavaScript functions: `getDirName()`, `getGlobalDir()`, `copyWithPathReplacement()`
- PascalCase for conceptual sections: `SessionStart`, `OpenCode`, `Claude Code`

**Variables:**
- const for immutable references: `const fs = require('fs');`
- let for state that changes: `let selectedRuntimes = [];`
- CAPS_UNDERSCORES for bash/shell variables: `HOOKS_DIR`, `COMMIT_PLANNING_DOCS`

**XML Tags:**
- Kebab-case for semantic containers: `<execution_context>`, `<execution_flow>`, `<step>`
- Semantic purpose required (no generic `<section>`, `<item>`, `<content>`)
- Examples: `<objective>`, `<process>`, `<step>`, `<role>`

## Code Style

**Formatting:**
- No formal linter (no .eslintrc, .prettierrc, or biome.json)
- Manual consistency following patterns observed in codebase
- 2-space indentation (observed in JSON, JavaScript)
- No trailing semicolons in shell/bash sections

**Comments:**
- JSDoc/block comments for complex functions
- Inline comments explain WHY, not WHAT (code shows WHAT)
- Single-line comments with descriptive purpose
- Example from `gsd-check-update.js`:
  ```javascript
  // Check for GSD updates in background, write result to cache
  // Called by SessionStart hook - runs once per session
  ```

**Error Handling:**
- Silent failures for non-critical operations: `try { ... } catch (e) {}`
- Explicit error console output with colored status: `console.error(${yellow}⚠${reset} ...)`
- fs operations include existence checks before operations
- JSON parsing wrapped in try-catch to prevent crashes

## Imports and Dependencies

**Node.js requires (no ES modules):**
- Grouped at file top: built-in modules, then local modules
- Examples: `const fs = require('fs')`, `const path = require('path')`

**No external dependencies** in core tools:
- `package.json` contains ZERO runtime dependencies
- Only `esbuild@^0.24.0` in devDependencies for building
- All code uses native Node.js APIs

**Import organization:**
1. Built-in Node modules (fs, path, os, child_process, readline)
2. Local module imports (package.json)
3. Constants and utility definitions
4. Main logic

## Logging & Output

**Framework:** `console` API only (no external logging library)

**Patterns:**
- `console.log()` for normal output and progress
- `console.error()` for errors
- `console.warn()` for warnings (rarely used)
- Color codes via ANSI escape sequences:
  - `\x1b[36m` = cyan (headers, commands)
  - `\x1b[32m` = green (checkmarks ✓)
  - `\x1b[33m` = yellow (warnings ⚠)
  - `\x1b[0m` = reset

**Silent failures:** File operations that fail non-critically (JSON parse, file reads) swallow errors with `catch (e) {}` comment

**Status indicators:**
- `✓` (green) = success
- `⚠` (yellow) = warning/skip
- `✗` (red, unused) = hard failure
- Spaces after indicators: `  ${green}✓${reset} Installed...`

## Markdown Structure

**Frontmatter (YAML):**
- Used in command/agent files only (`.md` files in `commands/`, `agents/`)
- Required fields: `name`, `description`
- Optional: `color`, `allowed-tools`, `argument-hint`, `tools`

**Section ordering in commands:**
1. `<objective>` — What/why/when
2. `<execution_context>` — @-references to required context
3. `<context>` — Dynamic content
4. `<process>` or `<step>` — Implementation steps
5. `<success_criteria>` — Verification checklist

**XML semantic containers only:**
- DO use: `<objective>`, `<process>`, `<step>`, `<reference>`
- DON'T use: `<section>`, `<item>`, `<content>`, `<subsection>`

## Function Design

**Size:** Generally under 100 lines
- Complex operations split into helper functions
- Single responsibility per function
- Example: `install()` delegates to `copyWithPathReplacement()`, `copyFlattenedCommands()`, etc.

**Parameters:** Documented with JSDoc
- Example:
  ```javascript
  /**
   * Copy commands to a flat structure for OpenCode
   * @param {string} srcDir - Source directory
   * @param {string} destDir - Destination directory
   * @param {string} prefix - Prefix for filenames
   */
  function copyFlattenedCommands(srcDir, destDir, prefix, ...) {}
  ```

**Return values:**
- Explicit returns (no implicit undefined)
- Objects returned for multiple values: `{ settingsPath, settings, statuslineCommand, runtime }`
- Arrays returned for lists

## Module Design

**Exports:** No formal export pattern (scripts are executable)
- Files in `bin/`, `hooks/`, `scripts/` are entry points
- No CommonJS exports (modules not meant for reuse)
- Direct execution via `#!/usr/bin/env node` shebang

**Barrel files:** Not used

**File-per-concept:** One concept per file
- `install.js` = installation logic
- `gsd-check-update.js` = update checking
- `gsd-statusline.js` = statusline rendering

## Temporal Language Rules

**Banned in implementation files:**
- "We changed X to Y"
- "Previously"
- "No longer"
- "Instead of"

**Allowed in:**
- CHANGELOG.md (historical record)
- MIGRATION.md
- Git commit messages

**Current state only:** Describe how things work NOW

## Conditional Logic

**Pattern in interactive scripts:**
```javascript
if (hasFlag) {
  // Handle flag case
} else if (otherCondition) {
  // Handle other
} else {
  // Default/interactive
}
```

**Early returns:** Used to reduce nesting in setup functions

**Ternary operators:** Allowed for simple conditions, if-else for complex logic

## Constants & Magic Values

**Color codes defined at top:**
```javascript
const cyan = '\x1b[36m';
const green = '\x1b[32m';
const yellow = '\x1b[33m';
const dim = '\x1b[2m';
const reset = '\x1b[0m';
```

**Feature detection objects:**
```javascript
const claudeToOpencodeTools = {
  AskUserQuestion: 'question',
  SlashCommand: 'skill',
  TodoWrite: 'todowrite',
};

const colorNameToHex = {
  cyan: '#00FFFF',
  red: '#FF0000',
  // ...
};
```

## Cross-File Conventions

**Path handling:**
- Absolute paths for file operations in Node.js
- Forward slashes in path output: `.replace(/\\/g, '/')`
- Path.join() for cross-platform safety
- Home directory expansion: `expandTilde()` helper

**File I/O:**
- UTF-8 encoding explicit: `fs.readFileSync(path, 'utf8')`
- JSON.stringify with 2-space indent: `JSON.stringify(obj, null, 2) + '\n'`
- Existence checks before reads: `fs.existsSync(path)`
- Recursive mkdir: `fs.mkdirSync(dir, { recursive: true })`

**Process spawning:**
- `child_process.spawn()` for background hooks
- `stdio: 'ignore'` to suppress output
- `.unref()` to allow parent exit without waiting

---

*Conventions analysis: 2026-01-24*
