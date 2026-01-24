# Codebase Structure

**Analysis Date:** 2026-01-24

## Directory Layout

```
get-shit-done/
├── bin/                      # Installation & setup
├── commands/gsd/             # User-facing slash commands (35 files)
├── agents/                   # Subagent specifications (11 files)
├── get-shit-done/            # Core knowledge base
│   ├── workflows/            # Multi-step orchestration workflows
│   ├── templates/            # Output templates (project, plan, research, etc.)
│   ├── references/           # Reusable guidance documents
│   ├── lib/                  # Utility & failure analysis docs
│   ├── tests/                # System verification tests
│   └── workflow-templates/   # Auto-generated workflow references
├── hooks/                    # Node.js hooks (statusline, update check)
├── scripts/                  # Build scripts
├── docs/                     # User documentation
├── .github/workflows/        # CI/CD (GitHub Actions)
├── .planning/codebase/       # Architecture analysis (this layer)
├── assets/                   # SVG, images
├── GSD-STYLE.md             # Comprehensive style guide
├── package.json             # NPM metadata
└── README.md                # User entry point
```

## Directory Purposes

**`bin/`:**
- Purpose: Installation entry point
- Contains: `install.js` (44KB Node.js script)
- Key files: `install.js`
- Responsibilities: Parse flags, detect runtime (Claude Code / OpenCode), detect install location (global/local), copy files to destination, create rules directory

**`commands/gsd/`:**
- Purpose: User-facing command definitions
- Contains: 35 markdown files (each defines one `/gsd:command`)
- Examples: `new-project.md`, `plan-phase.md`, `execute-phase.md`, `debug.md`
- Pattern: YAML frontmatter + `<objective>` + `<execution_context>` + `<context>` + `<process>`
- Responsibilities: Validate environment, parse arguments, spawn workflows/agents, present results

**`agents/`:**
- Purpose: Subagent specifications (spawned by orchestrators)
- Contains: 11 agent markdown files
- Key agents:
  - `gsd-executor.md` — Executes PLAN.md files with atomic commits
  - `gsd-planner.md` — Creates PLAN.md from phase requirements
  - `gsd-verifier.md` — Verifies phase goal achieved
  - `gsd-debugger.md` — Diagnoses execution failures
  - `gsd-plan-checker.md` — Validates plan quality
  - `gsd-phase-researcher.md` — Domain research before planning
  - `gsd-project-researcher.md` — Initial project discovery
  - `gsd-integration-checker.md` — External integration validation
  - `gsd-roadmapper.md` — Creates project ROADMAP
  - `gsd-research-synthesizer.md` — Synthesizes research
- Pattern: No YAML, includes `<role>`, `<execution_flow>`, `<step>` elements
- Responsibilities: Autonomous execution with full context, state management, error handling

**`get-shit-done/workflows/`:**
- Purpose: Multi-step orchestration coordination
- Contains: 20+ workflow markdown files
- Key workflows:
  - `execute-phase.md` — Wave-based parallel plan execution
  - `plan-phase.md` — Phase planning with research & verification loop
  - `execute-plan.md` — Single plan execution
  - `research-phase.md` — Domain research before planning
  - `retry-orchestration.md` — Failure retry logic
  - `create-roadmap.md` — Roadmap creation
- Pattern: No YAML, includes `<purpose>`, `<executive_summary>`, `<process>`, `<step>` elements
- Responsibilities: Coordinate multiple agents, handle state transitions, implement decision logic

**`get-shit-done/templates/`:**
- Purpose: Output structure templates
- Contains: 2 directories (codebase, research-project) + markdown templates
- Key templates:
  - `project.md` — Initial PROJECT.md template
  - `requirements.md` — REQUIREMENTS.md template
  - `plan.md` — PLAN.md template with task structure
  - `research.md` — RESEARCH.md template
  - `summary.md` — SUMMARY.md template with frontmatter
  - `roadmap.md` — ROADMAP.md template
- Responsibilities: Define structure for output artifacts

**`get-shit-done/references/`:**
- Purpose: Reusable guidance for patterns, concepts, verification
- Contains: 18 markdown files
- Key references:
  - `principles.md` — Core GSD philosophy
  - `ui-brand.md` — Visual formatting standards (banners, symbols)
  - `checkpoints.md` — Checkpoint protocol (how to structure checkpoints)
  - `tdd.md` — TDD pattern (RED → GREEN → REFACTOR)
  - `git-integration.md` — Git commit conventions
  - `verification-guide.md` — How to verify work
  - `pattern-schema.md` — Pattern metadata format
  - `model-profiles.md` — Model selection (quality/balanced/budget)
  - `config-schema.md` — config.json structure
  - `questioning.md` — Question templates for discovery
  - `monorepo-patterns.md` — Monorepo handling
  - `workflow-safety.md` — Safety checks in orchestration
- Responsibilities: @-referenced by workflows/agents for guidance

**`get-shit-done/lib/`:**
- Purpose: Utility & failure analysis
- Contains: 7 markdown files
- Key files:
  - `failure-analysis.md` — Post-mortem analysis
  - `failure-detection.md` — Recognizing failures
  - `failure-taxonomy.md` — Categorizing failure types
  - `regression-detection.md` — Test regression detection
  - `rollback-strategy.md` — Recovery strategies
- Responsibilities: @-referenced by debugging workflows

**`get-shit-done/tests/`:**
- Purpose: System verification
- Contains: Test markdown files
- Examples: `verify-transcendence-system.md`, `verify-retry-system.md`
- Responsibilities: Define verification test cases

**`hooks/`:**
- Purpose: Node.js runtime hooks
- Contains: 2 JavaScript files
- Files:
  - `gsd-statusline.js` — Formats Claude Code statusline (model | task | context usage)
  - `gsd-check-update.js` — Checks for GSD updates
- Responsibilities: Integrate with Claude Code UI

**`scripts/`:**
- Purpose: Build automation
- Contains: `build-hooks.js`
- Responsibilities: Build esbuild bundles for hooks/dist/

**`.planning/codebase/`:**
- Purpose: Architecture analysis (GSD codebase mapping)
- Generated by: `/gsd:map-codebase` agent
- Contains: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, STACK.md, INTEGRATIONS.md, CONCERNS.md
- Responsibilities: Guide future codebase changes

## Key File Locations

**Entry Points:**

- `bin/install.js`: Initial installation
- `commands/gsd/help.md`: Help listing
- `commands/gsd/new-project.md`: Project initialization
- `commands/gsd/plan-phase.md`: Phase planning
- `commands/gsd/execute-phase.md`: Phase execution

**Configuration:**

- `package.json`: NPM metadata, bin entry, build scripts
- `GSD-STYLE.md`: Comprehensive style guide for contributors
- `get-shit-done/references/config-schema.md`: config.json structure

**Core Logic:**

- `agents/gsd-executor.md`: Plan execution with deviation handling
- `agents/gsd-planner.md`: Task decomposition with goal-backward methodology
- `agents/gsd-verifier.md`: Goal verification
- `get-shit-done/workflows/execute-phase.md`: Wave-based orchestration
- `get-shit-done/workflows/plan-phase.md`: Planning orchestration

**Testing:**

- `get-shit-done/tests/`: System verification tests
- Test execution: No test runner (markdown-based specs)

## Naming Conventions

**Files:**
- Commands: `kebab-case.md` → `new-project.md`, `execute-phase.md`
- Agents: `gsd-kebab-case.md` → `gsd-executor.md`, `gsd-planner.md`
- Workflows: `kebab-case.md` → `execute-phase.md`, `plan-phase.md`
- Templates: `kebab-case.md` → `project.md`, `plan.md`
- References: `kebab-case.md` → `ui-brand.md`, `principles.md`
- Documentation: `UPPERCASE.md` → `README.md`, `CHANGELOG.md`, `GSD-STYLE.md`

**Directories:**
- Commands: `commands/gsd/`
- Agents: `agents/`
- Workflows: `get-shit-done/workflows/`
- Templates: `get-shit-done/templates/`
- References: `get-shit-done/references/`
- Utilities: `get-shit-done/lib/`
- Tests: `get-shit-done/tests/`

**Phase Directories (per-project):**
- Pattern: `.planning/phases/{PHASE}-{SLUG}/`
- Examples: `.planning/phases/01-api-setup/`, `.planning/phases/02.1-auth-tokens/`
- Contents: `{plan}-PLAN.md`, `{plan}-SUMMARY.md`, `{plan}-RESEARCH.md`

**YAML Frontmatter Keys (Commands):**
- `name`: Command name (gsd:kebab-case)
- `description`: One-line description
- `argument-hint`: Required or optional arguments
- `allowed-tools`: Permitted tools (Read, Write, Bash, Grep, Glob, Task, AskUserQuestion, etc.)
- `agent`: Which agent to spawn (optional, if subagent needed)

**YAML Frontmatter Keys (Plans):**
- `phase`: Phase number (01, 02, 02.1)
- `plan`: Plan identifier (001, 002)
- `type`: Plan type (standard, tdd, research, etc.)
- `autonomous`: Boolean (true = no checkpoints)
- `wave`: Wave number for parallelization
- `depends_on`: List of phase-plan dependencies

## Where to Add New Code

**New Command:**
1. Create `commands/gsd/new-command-name.md`
2. Add YAML frontmatter (name, description, argument-hint, allowed-tools)
3. Add `<objective>`, `<execution_context>`, `<context>`, `<process>` sections
4. If complex: Reference workflow from execution_context
5. If spawning subagent: Use Task tool with agent name from `agents/`

**New Workflow:**
1. Create `get-shit-done/workflows/workflow-name.md`
2. Add `<purpose>`, `<core_principle>`, `<required_reading>` sections
3. Add `<process>` with `<step name="step_name">` elements
4. Reference templates/references with @-notation
5. If orchestrating multiple agents: Define spawn timing and context expectations

**New Agent:**
1. Create `agents/gsd-new-agent.md`
2. Add YAML frontmatter (name, description, tools, color)
3. Add `<role>` explaining responsibility
4. Add `<execution_flow>` with `<step>` elements (with priority="first" for essential steps)
5. Add error handling and state management patterns

**New Reference:**
1. Create `get-shit-done/references/reference-name.md`
2. Use semantic XML containers (not generic `<section>`)
3. Include practical examples
4. Indicate where it's @-referenced

**New Template:**
1. Create `get-shit-done/templates/template-name.md`
2. Use square bracket placeholders: `[Project Name]`
3. Use curly brace placeholders: `{phase}-{plan}`
4. Include example output if complex
5. Update command that uses it to @-reference

**New Test:**
1. Create `get-shit-done/tests/verify-new-feature.md`
2. List verification steps as numbered list or code blocks
3. Include expected output

**New Codebase Analysis (after `/gsd:map-codebase`):**
1. Documents write directly to `.planning/codebase/`
2. Files: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, STACK.md, INTEGRATIONS.md, CONCERNS.md
3. Pattern: Use templates from agent spec, always include file paths with backticks

## Special Directories

**`.planning/` (per-project):**
- Purpose: Project metadata and execution tracking
- Generated by: `/gsd:new-project`
- Contents:
  - `PROJECT.md` — Project vision and context
  - `REQUIREMENTS.md` — Scoped feature list
  - `ROADMAP.md` — Phase structure
  - `STATE.md` — Current progress and decisions
  - `config.json` — Workflow preferences (model_profile, verification enabled, retry settings)
  - `phases/` — Phase directories with plans
  - `quick/` — Quick task directories
  - `codebase/` — Architecture analysis (ARCHITECTURE.md, STRUCTURE.md, etc.)
  - `research/` — Research files per phase
- Committed: Yes (git-tracked for history)

**`hooks/dist/`:**
- Purpose: Built JavaScript hooks
- Generated by: `npm run build:hooks`
- Committed: Yes (pre-built for distribution)

**`.claude/` or `.opencode/` (local installation):**
- Purpose: Local GSD installation
- Contents: Copy of `commands/`, `agents/`, `get-shit-done/`
- Generated by: `bin/install.js --local`
- Committed: No (gitignored)

**`~/.claude/` (global installation):**
- Purpose: Global GSD installation
- Contents: Copy of `commands/`, `agents/`, `get-shit-done/`, plus `rules/` auto-loading
- Generated by: `bin/install.js --global`
- Committed: No (home directory)

**`node_modules/`:**
- Purpose: NPM dependencies
- Generated by: `npm install`
- Committed: No (gitignored)

---

*Structure analysis: 2026-01-24*
