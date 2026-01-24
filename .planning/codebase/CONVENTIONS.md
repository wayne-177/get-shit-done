# Coding Conventions

**Analysis Date:** 2026-01-24

## Overview

Get Shit Done (GSD) is a meta-prompting system that prioritizes clarity and specification over traditional code. The codebase consists primarily of:
- **Markdown command files** with YAML frontmatter (`commands/gsd/*.md`)
- **Workflow orchestration files** (`get-shit-done/workflows/*.md`)
- **Node.js utilities** for installation, hooks, and build tasks
- **Reference documentation** and templates

All code is designed as both implementation and specification—files teach Claude how to build systematically.

---

## Naming Patterns

### Files

**Markdown files (commands, workflows, references):**
- Convention: `kebab-case`
- Examples: `execute-phase.md`, `create-roadmap.md`, `verify-phase.md`
- Location pattern: `commands/gsd/{command-name}.md`, `get-shit-done/workflows/{workflow-name}.md`

**JavaScript files (utilities, hooks):**
- Convention: `kebab-case` for executables
- Examples: `gsd-statusline.js`, `gsd-check-update.js`, `build-hooks.js`
- Shebang: Always include `#!/usr/bin/env node` for executable files

**Phase directories:**
- Pattern: `.planning/phases/{NN}-{phase-slug}/` where NN is 2-digit number
- Example: `.planning/phases/01-auth-setup/`, `.planning/phases/02-api-endpoints/`

**Config files:**
- `ROADMAP.md` — Phase breakdown (exists at project root, `.planning/ROADMAP.md`)
- `PROJECT.md` — Project specification (`.planning/PROJECT.md`)
- `STATE.md` — Living memory across sessions (`.planning/STATE.md`)
- `PLAN.md` — Phase plan files (`.planning/phases/NN-name/PLAN.md`)
- `SUMMARY.md` — Execution summary (`.planning/phases/NN-name/SUMMARY.md`)

### XML Tags and Elements

**Convention:** `kebab-case` with underscores for attributes

**Semantic containers** (not generic `<section>` or `<content>`):
- `<objective>` — What/why/when (primary goal)
- `<execution_context>` — @-references to workflows, templates, references
- `<context>` — Dynamic content: `$ARGUMENTS`, bash output, @file refs
- `<process>` — Container for steps (uses nested `<step>` elements)
- `<step>` — Individual execution step
- `<task>` — Structured work unit with type attribute
- `<success_criteria>` — Measurable completion checklist
- `<output>` — Files/artifacts produced
- `<purpose>` — What a workflow accomplishes
- `<when_to_use>` — Decision criteria for using a workflow
- `<verification>` — How to verify completion

### Step Names

**Convention:** `snake_case`

Examples from `bin/install.js`:
- `name="validate"` — Validation step
- `name="check_existing"` — Check for existing resources
- `name="create_roadmap"` — Create the roadmap
- `name="done"` — Completion step

### Functions and Variables (JavaScript)

**Naming:**
- Function names: `camelCase`
- Variables: `camelCase`
- Constants: `SCREAMING_SNAKE_CASE`
- Private functions: prefix with underscore (optional, rarely used)

Examples from `bin/install.js`:
- `function expandTilde(filePath)` — camelCase function
- `const selectedRuntimes = []` — camelCase variable
- `const HOOKS_TO_COPY = [...]` — SCREAMING_SNAKE_CASE constant
- `const cyan = '\x1b[36m'` — camelCase for color variables

**Parameters:**
- Use descriptive names: `runtime`, `filePath`, `targetDir`, `isGlobal`
- Avoid single-letter names except loop indices: `for (const file of files)`

### Command References

**Convention:** `/gsd:{command-name}` or `/gsd-{command-name}` (for OpenCode)

Examples:
- `/gsd:create-roadmap` — Main GSD command
- `/gsd:plan-phase` — Phase planning
- `/gsd:execute-phase` — Phase execution
- `/gsd-help` — OpenCode equivalent (flat structure)

### Bash Variables in Commands

**Convention:** `SCREAMING_SNAKE_CASE`

Examples from command files:
- `$PHASE_ARG` — Phase argument
- `$PLAN_START_TIME` — Timestamp
- `$ROADMAP_EXISTS` — Boolean-like check result
- `$TODO_COUNT` — Counter variable

---

## YAML Frontmatter (Commands)

**Location:** `commands/gsd/*.md`

**Required fields:**
```yaml
---
description: One-line description (required)
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---
```

**Field purposes:**
- `description`: What the command does (appears in `/gsd:help`)
- `allowed-tools`: Which tools this command is permitted to use

**Allowed tool values:**
- `Read` — Read files
- `Write` — Write files
- `Bash` — Execute bash commands
- `Glob` — File pattern matching
- `Grep` — Text search with regex
- `AskUserQuestion` — Prompt user with options
- `SlashCommand` — Call other commands
- `WebFetch` — HTTP requests (if needed)

**Note:** No `name:` field in modern GSD (command name derives from filename)

---

## Import Organization

### JavaScript Imports

**Order:**
1. Built-in Node.js modules (`fs`, `path`, `os`, `child_process`)
2. External packages (rarely used in main codebase)
3. Internal requires (relative paths)

Example from `bin/install.js`:
```javascript
const fs = require('fs');
const path = require('path');
const os = require('os');
const readline = require('readline');

// ... much later ...
const pkg = require('../package.json');
```

### Markdown References (@-patterns)

**Convention:** `@{path/to/file.md}`

**Types:**
- Static references (always load): `@~/.claude/get-shit-done/workflows/execute-phase.md`
- Conditional references (based on existence): `@.planning/DISCOVERY.md (if exists)`

**Order in execution_context blocks:**
1. Workflow references
2. Template references
3. Reference documentation

Example from `commands/gsd/create-roadmap.md`:
```xml
<execution_context>
@~/.claude/get-shit-done/workflows/create-roadmap.md
@~/.claude/get-shit-done/templates/roadmap.md
@~/.claude/get-shit-done/templates/state.md
</execution_context>
```

---

## Code Style

### Formatting

**JavaScript:**
- No linter configured (conventionally formatted)
- Indentation: 2 spaces
- Line length: No strict limit, but prefer readability
- Semicolons: Used throughout

Example style from `gsd-statusline.js`:
```javascript
const fs = require('fs');
const path = require('path');

let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', chunk => input += chunk);
```

**Markdown:**
- Indentation: 2 spaces in lists/code blocks
- Line length: No strict limit, but keep readable
- Code blocks: Use triple backticks with language identifier

### Comments

**JSDoc/TypeDoc:** Not used in this codebase (no type annotations)

**Inline comments:**
- Explain *why*, not *what*
- Use // for single line
- Use /* */ for multi-line
- Minimal use—code should be self-explaining

Example from `bin/install.js`:
```javascript
// Parse args (explains why we're doing this)
const args = process.argv.slice(2);

// Build a hook command path using forward slashes for cross-platform compatibility.
// On Windows, $HOME is not expanded by cmd.exe/PowerShell, so we use the actual path.
function buildHookCommand(claudeDir, hookName) {
```

**Documentation comments:**
- Use JSDoc for public functions
- Always include purpose, parameters, and return value

Example from `bin/install.js`:
```javascript
/**
 * Get the global config directory for OpenCode
 * OpenCode follows XDG Base Directory spec and uses ~/.config/opencode/
 * Priority: OPENCODE_CONFIG_DIR > dirname(OPENCODE_CONFIG) > XDG_CONFIG_HOME/opencode > ~/.config/opencode
 */
function getOpencodeGlobalDir() {
```

---

## Error Handling

### JavaScript Error Patterns

**Silent failures for non-critical operations:**
```javascript
try {
  const data = JSON.parse(input);
  // process data
} catch (e) {
  // Silent fail - don't break statusline on parse errors
}
```

**Explicit errors for critical operations:**
```javascript
if (!fs.existsSync(settingsPath)) {
  console.error(`  ${yellow}⚠${reset} Directory does not exist: ${locationLabel}`);
  console.log(`  Nothing to uninstall.\n`);
  return;
}
```

**Process exit codes:**
- `process.exit(0)` — Success
- `process.exit(1)` — Error (invalid arguments, missing files, installation failure)

### Markdown Error Handling

No traditional error handling in markdown files. Instead:
- Use `<success_criteria>` to verify completion
- Use checkpoint tasks for human verification
- Use `@-references` to abort if prerequisites missing

---

## Logging

### Console Output (JavaScript)

**Color codes (ANSI):**
```javascript
const cyan = '\x1b[36m';
const green = '\x1b[32m';
const yellow = '\x1b[33m';
const dim = '\x1b[2m';
const reset = '\x1b[0m';
```

**Patterns:**
- Status messages: `console.log(`  ${green}✓${reset} Installed commands`)
- Warnings: `console.log(`  ${yellow}⚠${reset} Skipping statusline`)
- Errors: `console.error()` (reserved for failures)
- Progress: Prefix with two spaces for indentation

### Markdown Logging

Commands output structured text:
- Status symbols: ✓ (success), ⚠ (warning), ✗ (error)
- Indentation: Use 2 spaces for sub-items
- Block formatting: Use `---` separators for visual breaks

Example from command files:
```markdown
---

## ▶ Next Up

**{identifier}: {name}** — {one-line description}

`{copy-paste command}`

---
```

---

## Validation

### Input Validation (JavaScript)

**File existence:**
```javascript
if (!fs.existsSync(filePath)) {
  console.error(`  ${yellow}Error: File not found: ${filePath}${reset}`);
  process.exit(1);
}
```

**Argument parsing:**
```javascript
if (configDirIndex !== -1) {
  const nextArg = args[configDirIndex + 1];
  if (!nextArg || nextArg.startsWith('-')) {
    console.error(`  ${yellow}--config-dir requires a path argument${reset}`);
    process.exit(1);
  }
}
```

**JSON validation:**
```javascript
try {
  const settings = JSON.parse(fs.readFileSync(settingsPath, 'utf8'));
} catch (e) {
  return {}; // Return empty object on parse error
}
```

### Markdown Validation

Validation happens at execution time through bash commands in steps:
```bash
[ -f .planning/PROJECT.md ] || { echo "ERROR: No PROJECT.md found."; exit 1; }
```

---

## Module Design

### JavaScript Modules

**Pattern:** No explicit module exports (most files are executable scripts)

Entry point patterns:
- `bin/install.js` — Installation script with no exports
- `hooks/gsd-*.js` — Hook scripts (executable)
- `scripts/build-hooks.js` — Build script (executable)

**Function organization in large files:**
1. Constants and initialization
2. Helper functions (small, focused)
3. Main logic functions (large, complex)
4. Execution block at bottom

Example from `bin/install.js`:
```javascript
// 1. Constants
const cyan = '\x1b[36m';
const pkg = require('../package.json');

// 2. Helper functions
function expandTilde(filePath) { ... }
function parseConfigDirArg() { ... }
function getGlobalDir(runtime, explicitDir) { ... }

// 3. Main logic
function install(isGlobal, runtime) { ... }
function uninstall(isGlobal, runtime) { ... }

// 4. Execution
if (hasHelp) {
  // show help
}
```

### Markdown Structure

**Command structure:**
1. YAML frontmatter (metadata)
2. `<objective>` (purpose)
3. `<execution_context>` (@-references)
4. `<context>` (dynamic content)
5. `<process>` (steps with names)
6. `<output>` (artifacts)
7. `<success_criteria>` (completion checklist)

**Workflow structure:**
Varies by workflow, but typically:
1. Purpose/description
2. Decision gates
3. Numbered or named steps
4. References to templates

---

## Interdependencies and Cross-References

### Path References

**Absolute paths in markdown:**
- Always use tilde-relative: `~/.claude/` for Claude Code config
- Or relative: `.planning/` for project planning directory

**Relative paths in code:**
- Use `path.join()` in Node.js (handles OS differences)
- Never hardcode backslashes

Example from `bin/install.js`:
```javascript
const globalDir = path.join(os.homedir(), '.claude');
const targetDir = path.join(process.cwd(), dirName);
```

### Tool Dependencies

No external packages required in main bundle (zero dependencies).

Hook scripts only use Node.js builtins: `fs`, `path`, `os`, `child_process`, `readline`

---

## Patterns to Follow

### File I/O Pattern

```javascript
// Read and validate
if (!fs.existsSync(filePath)) {
  return null; // or throw/exit
}

// Parse (with error handling)
try {
  const data = JSON.parse(fs.readFileSync(filePath, 'utf8'));
  return data;
} catch (e) {
  return {}; // Fallback
}

// Write with formatting
fs.writeFileSync(filePath, JSON.stringify(data, null, 2) + '\n');
```

### Command Pattern (Markdown)

Every command has this structure:
```markdown
---
description: [one-liner]
allowed-tools: [tools]
---

<objective>[what/why]</objective>

<execution_context>
@path/to/workflow.md
</execution_context>

<context>
@.planning/PROJECT.md (if exists)
</context>

<process>
<step name="step-name">
[action or bash code]
</step>
</process>

<success_criteria>
- [ ] Criterion 1
- [ ] Criterion 2
</success_criteria>
```

### Step Naming Pattern

Use one of these standard names:
- `validate` — Check prerequisites
- `check_existing` — Check if resource exists
- `create_{resource}` — Create something
- `update_{resource}` — Update something
- `parse_{data}` — Parse structured data
- `write_{file}` — Write output
- `done` — Final status/next steps

---

## Anti-Patterns to Avoid

### Banned Language Patterns

**DO NOT use temporal language in code/docs:**
- ✗ "We changed X to Y"
- ✗ "Previously did X"
- ✗ "No longer uses X"
- ✓ "X uses Y" (current state only)

**Exception:** CHANGELOG.md, MIGRATION.md, git commits

**DO NOT use enterprise patterns:**
- ✗ Story points
- ✗ Sprint ceremonies
- ✗ RACI matrices
- ✗ Change management processes
- ✓ Direct implementation + checkpoints

**DO NOT use filler language:**
- ✗ "Let me", "Just", "Simply", "Basically", "I'd be happy to"
- ✗ "Great!", "Awesome!", "Excellent!"
- ✓ Direct, technical, precise

### Banned XML Patterns

**DO NOT use generic XML tags:**
- ✗ `<section>`, `<item>`, `<content>`, `<data>`
- ✓ `<objective>`, `<verification>`, `<action>`

**DO NOT nest XML deeply:**
- ✗ `<section><subsection><item><content>`
- ✓ `<objective>` with markdown headers inside

### Banned Code Patterns

**DO NOT use:**
- vague function names: `doStuff()`, `process()`
- unused variables: `let unused = 5`
- bare except blocks: `catch (e) { }` without comment
- console.log in production code (except hooks that output data)

**DO use:**
- descriptive names: `parseVersionFile()`, `validateProjectDir()`
- commented catch blocks: `catch (e) { /* expected: file may not exist */ }`
- early returns for validation

---

## Summary of Core Patterns

1. **Markdown + XML** — Commands and workflows use semantic XML + markdown
2. **Zero dependencies** — No external packages (except esbuild for build)
3. **Bash for automation** — Steps contain actual bash code, not pseudo-code
4. **@-references** — Load files lazily via reference notation
5. **Imperative voice** — "Do X", not "X is done"
6. **Current state only** — No temporal language in specs
7. **Explicit errors** — Failed validation exits immediately
8. **ANSI colors** — Structured output with visual status
9. **JSDoc for functions** — Document public functions even without types
10. **Sequential execution** — Steps run in order, each can validate or abort

---

*Convention analysis: 2026-01-24*
