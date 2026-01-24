# Architecture

**Analysis Date:** 2026-01-24

## Pattern Overview

**Overall:** Multi-agent orchestration system with context-managed subagent spawning

**Key Characteristics:**
- Single orchestrator coordinates specialized subagents via Task tool
- Each agent writes output directly to disk (not back to orchestrator)
- Agents have fresh context, preventing token contamination
- Markdown-based workflows drive all execution flow
- State management via `.planning/` directory with frontmatter-driven documents
- Zero external dependencies (Node.js native only)

## Layers

**Presentation Layer - CLI Commands:**
- Purpose: Entry points for user interaction via `/gsd:command` syntax
- Location: `commands/gsd/`
- Contains: 40+ command definitions (markdown files with YAML frontmatter)
- Examples: `map-codebase.md`, `new-project.md`, `plan-phase.md`, `execute-phase.md`
- Depends on: Orchestrator workflows, agents
- Used by: Claude Code interface, user prompts

**Orchestration Layer - Workflow Controllers:**
- Purpose: Coordinate multiagent execution, manage state transitions, handle retries
- Location: `get-shit-done/workflows/`
- Contains: 40+ workflow markdown files that implement command logic
- Key workflows: `map-codebase.md`, `execute-phase.md`, `plan-phase.md`, `verify-phase.md`
- Pattern: Step-based execution with bash commands, Task spawning, state management
- Depends on: Agents, reference docs, validation libraries
- Used by: Commands (called via execution_context)

**Agent Layer - Specialized Executors:**
- Purpose: Deep-focus analysis/implementation with autonomous decision making
- Location: `agents/`
- Contains: 11 agent definitions (markdown files with YAML frontmatter)
- Key agents: `gsd-planner.md`, `gsd-executor.md`, `gsd-codebase-mapper.md`, `gsd-verifier.md`, `gsd-debugger.md`
- Spawning: Via Task tool with `subagent_type` parameter
- Pattern: Agents write documents directly to `.planning/` or `.planning/codebase/`
- Depends on: Reference docs, codebase files, project state
- Used by: Orchestrator workflows

**Knowledge Base Layer - References:**
- Purpose: Prescriptive guidance, patterns, schemas for agents and workflows
- Location: `get-shit-done/references/`
- Contains: 21 markdown files with patterns, schemas, guidelines
- Examples: `verification-patterns.md`, `tdd.md`, `checkpoints.md`, `monorepo-patterns.md`
- Pattern: Loaded via `@file` references in agent roles and workflow contexts
- Used by: Agents (via frontmatter context), workflows

**Template Layer - Document Blueprints:**
- Purpose: Pre-structured documents that agents and workflows populate
- Location: `get-shit-done/templates/`
- Contains: 37 markdown templates with YAML frontmatter
- Sub-folders: `codebase/` (7 structure templates), `research-project/` (6 templates)
- Usage: Copied and filled during workflow execution
- Used by: Workflows (populate and write to `.planning/`)

**Library Layer - Utilities:**
- Purpose: Reusable analysis, validation, detection logic
- Location: `get-shit-done/lib/`
- Contains: 7 markdown files with domain-specific logic
- Examples: `validate-config.md`, `failure-taxonomy.md`, `path-selection.md`
- Pattern: Hand-rolled validation/analysis without external dependencies
- Used by: Workflows, agents

**Installation Layer:**
- Purpose: Package distribution and runtime setup
- Location: `bin/install.js`, `scripts/`, `hooks/`
- Contains: Installation logic, statusline hooks, update checking
- Pattern: Cross-platform path handling, Claude Code + OpenCode support
- Used by: npm install process

## Data Flow

**Project Initialization Flow:**

1. User runs `/gsd:new-project`
2. `commands/gsd/new-project.md` (command) → `get-shit-done/workflows/new-project.md` (orchestrator)
3. Orchestrator questions user, spawns `gsd-project-researcher` agent (if domain research needed)
4. Outputs written: `.planning/PROJECT.md`, `.planning/REQUIREMENTS.md`, `.planning/ROADMAP.md`, `.planning/STATE.md`, `.planning/config.json`

**Codebase Analysis Flow:**

1. User runs `/gsd:map-codebase [focus]` (focus: tech|arch|quality|concerns)
2. `commands/gsd/map-codebase.md` → `get-shit-done/workflows/map-codebase.md`
3. Orchestrator spawns 4 parallel `gsd-codebase-mapper` agents (one per focus area)
4. Each mapper writes documents directly: `STACK.md`, `INTEGRATIONS.md`, `ARCHITECTURE.md`, `STRUCTURE.md`, `CONVENTIONS.md`, `TESTING.md`, `CONCERNS.md`
5. Orchestrator collects confirmations (not content), verifies files exist

**Phase Planning Flow:**

1. User runs `/gsd:plan-phase [n]`
2. `commands/gsd/plan-phase.md` → `get-shit-done/workflows/plan-phase.md`
3. Orchestrator loads `.planning/ROADMAP.md`, `.planning/codebase/` docs, relevant discovery docs
4. Orchestrator spawns `gsd-planner` agent with loaded context
5. Planner reads codebase docs, produces `PLAN.md` file to `.planning/phases/phase-N/`
6. Orchestrator verifies plan, optionally routes to `gsd-plan-checker` for review

**Phase Execution Flow:**

1. User runs `/gsd:execute-phase [n]`
2. `commands/gsd/execute-phase.md` → `get-shit-done/workflows/execute-phase.md`
3. Orchestrator loads `PLAN.md`, project state, codebase docs
4. Orchestrator spawns `gsd-executor` agent
5. Executor reads PLAN, implements tasks atomically:
   - Create file(s) using Write tool
   - Run bash commands using Bash tool
   - Commit via git with per-task commits
   - Handle checkpoints by returning structured pause message
6. Executor writes `SUMMARY.md`, updates `STATE.md`
7. Orchestrator collects summary, offers next steps

**Verification Flow:**

1. After execute completes, user runs `/gsd:verify-work`
2. `commands/gsd/verify-work.md` → `get-shit-done/workflows/verify-phase.md`
3. Orchestrator spawns `gsd-verifier` agent
4. Verifier reads PLAN, tests against requirements, produces `VERIFICATION.md`
5. If failures: Orchestrator routes to `gsd-debugger` or suggests `/gsd:plan-phase --gaps`

## State Management

**State Files in `.planning/`:**
- `PROJECT.md` - Project vision, problem statement, scope
- `REQUIREMENTS.md` - Feature requirements, acceptance criteria
- `ROADMAP.md` - Phase breakdown and sequencing
- `STATE.md` - Current position, accumulated decisions, blockers
- `config.json` - Workflow preferences (mode, gates, safety, retry)

**Phase-Specific State in `.planning/phases/phase-N/`:**
- `PLAN.md` - Executable plan (frontmatter + objective + tasks)
- `SUMMARY.md` - Execution summary (what was completed, commits made)
- `VERIFICATION.md` - Test results, failures, acceptance status
- `DISCOVERY.md` - Research findings (when research phase runs)

**Codebase Knowledge in `.planning/codebase/`:**
- `STACK.md` - Technology stack, versions, package manager
- `INTEGRATIONS.md` - External services, APIs, authentication
- `ARCHITECTURE.md` - System design patterns, layers, data flow
- `STRUCTURE.md` - Directory layout, naming conventions, file placement
- `CONVENTIONS.md` - Code style, naming, patterns
- `TESTING.md` - Test framework, patterns, coverage
- `CONCERNS.md` - Technical debt, bugs, performance issues

## Key Abstractions

**Agent Pattern:**
- Purpose: Encapsulates specialized expertise (planning, execution, verification)
- Examples: `gsd-planner.md`, `gsd-executor.md`, `gsd-verifier.md`
- Pattern: Markdown file with YAML frontmatter defining role, tools, philosophy; body contains multi-step execution logic
- Key property: Agents write output directly to prevent context bloat in orchestrator

**Workflow Pattern:**
- Purpose: Orchestrates steps, agents, state transitions
- Examples: `execute-phase.md`, `plan-phase.md`
- Pattern: Markdown with `<step name>` tags, bash commands, Task spawning
- Key property: Workflows are controllers (call agents, manage state), not executors

**Command Pattern:**
- Purpose: User-facing entrypoints
- Examples: `map-codebase.md`, `new-project.md`
- Pattern: Markdown with frontmatter (name, description, allowed-tools), contains objective + execution_context
- Key property: Commands reference workflows (via @path in execution_context)

**Task Spawn Pattern:**
- Purpose: Create subagent with fresh context
- Pattern: Task tool with `subagent_type`, `model`, prompt context
- Key property: Subagent output goes to filesystem, orchestrator gets confirmation only

**Frontmatter Pattern:**
- Purpose: Metadata for execution control and planning
- Used in: Commands, agents, workflows, templates
- Fields: name, description, tools, color, phase, type, waves, depends_on, user_setup
- Pattern: YAML between `---` delimiters

## Entry Points

**CLI Entry Point:**
- Location: `bin/install.js`
- Triggers: `npx get-shit-done-cc` or direct file invocation
- Responsibilities: Interactive/non-interactive installation, path resolution, settings.json configuration

**Command Entry Points:**
- Location: `commands/gsd/` (Claude Code) or flattened as `command/gsd-*.md` (OpenCode)
- Triggers: User types `/gsd:command` in Claude Code interface
- Pattern: Command file references workflow file via `execution_context`
- Examples: `/gsd:new-project`, `/gsd:plan-phase`, `/gsd:execute-phase`

**Workflow Entry Point:**
- Location: `get-shit-done/workflows/`
- Triggers: Referenced by command via `execution_context: @path`
- Pattern: Multi-step orchestration with bash commands and Task spawning
- Responsibilities: Step coordination, state management, agent orchestration

**Agent Entry Point:**
- Location: `agents/`
- Triggers: Spawned by orchestrator via Task tool with `subagent_type`
- Pattern: Autonomous execution with exploration loop → analysis → document output
- Responsibilities: Deep work (planning, executing, verifying, debugging)

## Error Handling

**Strategy:** Multi-layer with escalation

**Layer 1 - CLI Installation:**
- Errors in `bin/install.js`: Early termination with error message
- Validation: Path existence, JSON syntax, file permissions
- Recovery: User re-runs with corrected arguments

**Layer 2 - Workflow Validation:**
- Errors in step execution: Bash command failures caught, logged to stderr
- Validation: State file existence, config schema correctness
- Recovery: Retry step, suggest debug command, escalate to debugger agent

**Layer 3 - Agent Execution:**
- Errors during agent work: Try/catch patterns in agent prompts
- Validation: File write success, bash exit codes, verification criteria
- Recovery: Handle deviations in executor, log to attempts-log, escalate to debugger

**Layer 4 - Retry Orchestration:**
- Workflow: `retry-orchestration.md` handles plan failures
- Strategy: Identify failure class (verification, execution, integration), select retry path
- Options: Retry same plan, invoke debugger, create gap-closure plan

## Cross-Cutting Concerns

**Logging:** Markdown-based event logs stored in `.planning/` (attempts-log.md, regression-log.md)

**Validation:** Hand-rolled schema validation in `lib/validate-config.md` without external dependencies

**Authentication:** None internally; external service auth via environment variables (Stripe, Supabase, etc. via INTEGRATIONS.md)

**Authorization:** None; single-user system (solo developer + Claude)

**Context Management:** Documents written directly to `.planning/` to prevent context bloat; agents receive only necessary references via `@file` syntax

**State Persistence:** All state in git-committed `.planning/` directory; no external databases

**Idempotency:** Tasks designed for re-execution; file writes include safeguards (read before write, diff checking)

---

*Architecture analysis: 2026-01-24*
