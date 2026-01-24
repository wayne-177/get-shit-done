# Codebase Structure

**Analysis Date:** 2026-01-24

## Directory Layout

```
get-shit-done/
├── bin/                          # Installation and setup
├── commands/                     # User-facing CLI commands
│   └── gsd/                      # 40+ command markdown files
├── agents/                       # Specialized subagent definitions
├── get-shit-done/                # Core system (workflows, templates, lib)
│   ├── workflows/                # 40+ orchestrator workflows
│   ├── templates/                # Document blueprints
│   │   ├── codebase/            # 7 codebase analysis templates
│   │   └── research-project/    # 6 research templates
│   ├── references/               # 21 prescriptive guides and schemas
│   ├── lib/                      # Utility libraries (validation, patterns)
│   ├── tests/                    # Test data and references
│   └── workflow-templates/       # Generated workflow templates
├── hooks/                        # Git hooks and statusline scripts
├── scripts/                      # Build scripts (esbuild for hooks)
├── docs/                         # Public documentation
├── .claude/                      # Installed GSD copy (local projects only)
├── .planning/                    # Project state (created by /gsd:new-project)
│   ├── codebase/                # 7 codebase analysis docs (created by /gsd:map-codebase)
│   ├── phases/                  # Phase-specific state
│   │   └── phase-N/             # Plans, summaries, verification per phase
│   ├── PROJECT.md               # Project vision and scope
│   ├── REQUIREMENTS.md          # Feature specifications
│   ├── ROADMAP.md               # Phase sequencing
│   ├── STATE.md                 # Current execution state
│   └── config.json              # Workflow preferences
├── package.json                 # NPM metadata (no dependencies)
└── README.md                    # User documentation
```

## Directory Purposes

**`bin/`:**
- Purpose: Installation and bootstrap
- Contains: `install.js` - main CLI installer
- Key files: `install.js` (~1300 lines, handles interactive/non-interactive install, path resolution, settings configuration)

**`commands/gsd/`:**
- Purpose: User-facing command definitions
- Contains: 40+ command markdown files
- Key files:
  - `new-project.md` - Initialize project with questioning flow
  - `map-codebase.md` - Analyze existing codebase
  - `plan-phase.md` - Create executable phase plans
  - `execute-phase.md` - Run phase implementation
  - `verify-work.md` - Test completed work
  - `debug.md` - Troubleshoot failures
- Pattern: YAML frontmatter + markdown body with execution_context reference

**`agents/`:**
- Purpose: Specialized subagent implementations
- Contains: 11 agent markdown files
- Key agents:
  - `gsd-planner.md` - Creates PLAN.md files with task breakdown
  - `gsd-executor.md` - Executes plans atomically with per-task commits
  - `gsd-verifier.md` - Tests completed work against requirements
  - `gsd-codebase-mapper.md` - Analyzes codebase into STACK.md, INTEGRATIONS.md, etc.
  - `gsd-debugger.md` - Investigates failures and suggests fixes
  - `gsd-project-researcher.md` - Deep research on project domain
  - `gsd-phase-researcher.md` - Phase-specific discovery
  - `gsd-plan-checker.md` - Reviews plans before execution
  - `gsd-integration-checker.md` - Validates external service setup
  - `gsd-roadmapper.md` - Creates ROADMAP.md
  - `gsd-research-synthesizer.md` - Aggregates research findings

**`get-shit-done/workflows/`:**
- Purpose: Orchestrate agent execution and state management
- Contains: 40+ workflow markdown files
- Key workflows:
  - `new-project.md` - Project initialization orchestration
  - `map-codebase.md` - Coordinate 4 mapper agents in parallel
  - `plan-phase.md` - Call planner, optionally route to checker
  - `execute-phase.md` - Call executor, handle checkpoints, produce summary
  - `verify-phase.md` - Call verifier, detect failures
  - `auto-correction.md` - Automatic remediation of common failures
  - `retry-orchestration.md` - Intelligent retry path selection
  - `coordinated-execution.md` - Multi-agent coordination patterns
- Pattern: Step-based execution with bash commands, Task spawning, state management

**`get-shit-done/templates/`:**
- Purpose: Pre-structured document templates
- Contains: 37 markdown files
- Sub-directories:
  - `codebase/` - 7 analysis templates (architecture.md, structure.md, stack.md, integrations.md, conventions.md, testing.md, concerns.md)
  - `research-project/` - 6 research templates
- Pattern: YAML frontmatter + placeholder sections replaced during execution
- Used by: Workflows when creating `.planning/` documents

**`get-shit-done/references/`:**
- Purpose: Prescriptive guidance, patterns, schemas
- Contains: 21 markdown files
- Key references:
  - `verification-patterns.md` - How to verify work
  - `tdd.md` - Test-driven development patterns
  - `checkpoints.md` - Checkpoint design and handling
  - `monorepo-patterns.md` - Monorepo detection and handling
  - `git-integration.md` - Git commit strategies
  - `continuation-format.md` - Format for state serialization
  - `pattern-schema.md` - Pattern validation schema
- Pattern: Loaded via `@file` references in agent/workflow contexts
- Used by: Agents and workflows for guidance

**`get-shit-done/lib/`:**
- Purpose: Reusable analysis and validation logic
- Contains: 7 markdown files
- Files:
  - `validate-config.md` - Config.json schema validation
  - `failure-taxonomy.md` - Classification of failure types
  - `failure-detection.md` - Patterns to identify failures
  - `failure-analysis.md` - Root cause analysis patterns
  - `path-selection.md` - Failure recovery path selection
  - `regression-detection.md` - Identifying regressions
  - `rollback-strategy.md` - Rollback and recovery strategies
- Pattern: Hand-rolled validation without external dependencies

**`get-shit-done/tests/`:**
- Purpose: Test data and validation cases
- Contains: Private test files and fixtures

**`get-shit-done/workflow-templates/generated/`:**
- Purpose: Dynamically generated workflow configurations
- Generated by: Build process
- Contains: Compiled workflow schemas

**`hooks/`:**
- Purpose: Git hooks and Claude Code statusline
- Contains:
  - `gsd-statusline.js` - Real-time statusline showing model, task, context usage
  - `gsd-check-update.js` - Version checking hook
  - Source files in `hooks/` directory
  - Compiled dist in `hooks/dist/`

**`scripts/`:**
- Purpose: Build and development utilities
- Contains: `build-hooks.js` - esbuild configuration for bundling hooks

**`.planning/`:**
- Purpose: Project state and execution artifacts
- Created by: `/gsd:new-project` command
- Structure:
  ```
  .planning/
  ├── codebase/                    # Created by /gsd:map-codebase
  │   ├── STACK.md
  │   ├── INTEGRATIONS.md
  │   ├── ARCHITECTURE.md
  │   ├── STRUCTURE.md
  │   ├── CONVENTIONS.md
  │   ├── TESTING.md
  │   └── CONCERNS.md
  ├── phases/
  │   ├── phase-1/
  │   │   ├── PLAN.md
  │   │   ├── SUMMARY.md
  │   │   └── VERIFICATION.md
  │   ├── phase-2/
  │   └── ...
  ├── PROJECT.md                   # Project vision
  ├── REQUIREMENTS.md              # Feature specs
  ├── ROADMAP.md                   # Phase sequencing
  ├── STATE.md                     # Execution state
  ├── config.json                  # Workflow preferences
  └── [discovery/]                 # Optional research
  ```

## Key File Locations

**Entry Points:**
- `bin/install.js` - CLI installation script
- `commands/gsd/new-project.md` - Project initialization command
- `commands/gsd/map-codebase.md` - Codebase analysis command
- `commands/gsd/plan-phase.md` - Phase planning command
- `commands/gsd/execute-phase.md` - Phase execution command

**Core Logic:**
- `agents/gsd-planner.md` - Planning logic and task decomposition
- `agents/gsd-executor.md` - Execution logic and state management
- `agents/gsd-verifier.md` - Verification and testing patterns
- `agents/gsd-codebase-mapper.md` - Codebase analysis framework
- `get-shit-done/workflows/execute-phase.md` - Main execution orchestrator

**Configuration:**
- `get-shit-done/references/config-schema.md` - Config.json schema
- `get-shit-done/templates/config.json` - Default configuration
- `.planning/config.json` - Project-specific configuration

**Testing:**
- `get-shit-done/references/verification-patterns.md` - Testing patterns
- `get-shit-done/references/tdd.md` - TDD methodology

## Naming Conventions

**Files:**
- Commands: kebab-case (e.g., `new-project.md`, `map-codebase.md`)
- Agents: `gsd-{role}.md` (e.g., `gsd-planner.md`, `gsd-executor.md`)
- Workflows: kebab-case (e.g., `execute-phase.md`, `auto-correction.md`)
- State docs: UPPERCASE.md (e.g., `PROJECT.md`, `PLAN.md`, `SUMMARY.md`)
- Code files: camelCase with .js extension (e.g., `install.js`)

**Directories:**
- Lowercase, kebab-case for public directories: `commands/gsd/`, `agents/`, `bin/`
- Lowercase for private/generated: `workflows/`, `templates/`, `lib/`, `references/`
- Special: `.planning/` (dot prefix for invisible/configuration), `.claude/` (Claude Code config)

**Frontmatter Fields:**
- Commands: name, description, allowed-tools, argument-hint
- Agents: name, description, tools, color
- Workflows: purpose (no formal frontmatter, uses XML tags)
- Templates: phase, type, autonomous, waves, depends_on, user_setup

## Where to Add New Code

**New Command:**
1. Create `commands/gsd/{command-name}.md` with frontmatter + objective + execution_context
2. Reference workflow: `@/Users/macuser/.claude/get-shit-done/workflows/{workflow}.md`
3. Update `agents/` reference if new agent type needed
4. Document in `commands/gsd/help.md`

**New Agent:**
1. Create `agents/gsd-{role}.md` with YAML frontmatter + role + philosophy + process steps
2. Document tools needed (Read, Write, Bash, Grep, Glob, etc.)
3. Reference in workflow that calls it via `subagent_type="gsd-{role}"`
4. Add to agents list in `/gsd:help`

**New Workflow:**
1. Create `get-shit-done/workflows/{workflow-name}.md` with purpose section
2. Structure as step-based execution with `<step name>` XML tags
3. Include bash commands for state checking
4. Add Task spawning for agent orchestration
5. Handle error cases and escalation
6. Reference from command via `@path` in execution_context

**New Reference Guide:**
1. Create `get-shit-done/references/{topic}.md`
2. Include prescriptive examples and code patterns
3. Link via `@file` references in agent roles or workflow context
4. Document use case (agents, workflows that reference it)

**New Template:**
1. Create `get-shit-done/templates/{template-name}.md` or in subdirectory
2. Include YAML frontmatter with metadata
3. Use placeholder sections (marked with `[Placeholder]`)
4. Reference from workflow that populates it
5. Store generated version in `.planning/` during execution

**Utilities/Libraries:**
1. Create `get-shit-done/lib/{utility}.md` for new library functions
2. Use hand-rolled logic without external dependencies
3. Document validation/analysis approach
4. Reference from workflows/agents that use it

## Special Directories

**`.planning/`:**
- Purpose: Project-specific state and execution artifacts
- Generated: Yes (created by /gsd:new-project)
- Committed: Yes (git-tracked, .gitignore patterns can exclude it)
- Lifetime: Persists for entire project
- Contains: All phase plans, summaries, requirements, roadmap, configuration

**`.claude/` (local installs only):**
- Purpose: Local GSD installation for single project
- Generated: Yes (created by install.js --local)
- Committed: Typically no (added to .gitignore automatically)
- Lifetime: For single project only
- Contains: Copy of entire GSD system (bin/, commands/, agents/, get-shit-done/)

**`get-shit-done/workflow-templates/generated/`:**
- Purpose: Cached/compiled workflow templates
- Generated: Yes (by build process)
- Committed: Yes (committed to repo)
- Lifetime: Rebuilt during development
- Contains: Pre-processed workflow schemas

---

*Structure analysis: 2026-01-24*
