# Architecture

**Analysis Date:** 2026-01-24

## Pattern Overview

**Overall:** Command-Agent-Workflow meta-prompting system with orchestration, subagent delegation, and state management.

**Key Characteristics:**
- Orchestrator-subagent separation: Thin orchestrators coordinate, full-context subagents execute
- Wave-based parallelization: Plans group by dependencies and execute in parallel waves
- Prompt-as-code: PLAN.md, RESEARCH.md, and SUMMARY.md are executable prompts, not documents
- Context preservation: STATE.md maintains project memory across sessions
- Progressive disclosure: Commands → Workflows → Templates → References

## Layers

**User Interface Layer (Commands):**
- Purpose: Entry points for all user interactions
- Location: `commands/gsd/*.md`
- Contains: 35 command definitions with YAML frontmatter (name, description, allowed-tools)
- Depends on: Workflows, templates, references
- Used by: Claude Code / OpenCode interfaces
- Pattern: Each command is thin wrapper delegating to workflow

**Orchestration Layer (Workflows):**
- Purpose: Coordinate complex multi-step processes
- Location: `get-shit-done/workflows/*.md`
- Contains: 20+ workflow definitions (execute-phase, plan-phase, research-phase, etc.)
- Depends on: Agent specifications, references, lib files, templates
- Used by: Commands (via execution_context @-references)
- Pattern: Orchestrators parse input, validate state, spawn subagents, collect/aggregate results

**Execution Layer (Agents):**
- Purpose: Autonomous implementation of plans
- Location: `agents/*.md` (11 agent specs)
- Contains: gsd-executor, gsd-planner, gsd-verifier, gsd-debugger, and 7 others
- Depends on: Project state, plans, research, references
- Used by: Orchestrators via Task spawning
- Pattern: Each agent loads full context fresh, handles single responsibility

**Knowledge Base Layer (References & Templates):**
- Purpose: Reusable patterns and guidance documents
- Location: `get-shit-done/references/*.md` (UI, principles, patterns, TDD, git, etc.)
- Contains: UI-brand.md, verification-guide.md, model-profiles.md, checkpoints.md, etc.
- Depends on: Nothing
- Used by: Commands, workflows, agents via @-references
- Pattern: @-references are lazy-loading signals, not pre-embedded content

**Project State Layer:**
- Purpose: Persist project context across sessions
- Location: `.planning/` (created per-project)
- Contains: PROJECT.md, ROADMAP.md, STATE.md, config.json, phases/
- Depends on: Nothing
- Used by: Orchestrators and executors
- Pattern: Git-tracked memory system

## Data Flow

**New Project Flow:**

1. User runs `/gsd:new-project`
2. Orchestrator questions user (AskUserQuestion tools)
3. Orchestrator spawns gsd-project-researcher for domain research (if needed)
4. Orchestrator synthesizes to REQUIREMENTS.md
5. Orchestrator spawns gsd-roadmapper to create ROADMAP.md
6. Creates `.planning/` with PROJECT.md, ROADMAP.md, config.json, STATE.md

**Plan Phase Flow:**

1. User runs `/gsd:plan-phase [phase]`
2. Orchestrator validates phase exists in ROADMAP.md
3. If research needed: Spawns gsd-phase-researcher → produces RESEARCH.md
4. Spawns gsd-planner with RESEARCH, PROJECT, REQUIREMENTS → produces PLAN.md files
5. Spawns gsd-plan-checker with PLAN.md → verification loop (iterate until passes)
6. Returns to user with plans ready to execute

**Execute Phase Flow:**

1. User runs `/gsd:execute-phase [phase]`
2. Orchestrator loads STATE.md (project memory)
3. Orchestrator discovers all PLAN.md files in phase directory
4. Orchestrator groups plans by `wave` frontmatter (dependency analysis)
5. For each wave (sequentially):
   - Spawn gsd-executor for each plan in wave (parallel Task calls)
   - Wait for completion (Task blocks)
   - Collect SUMMARY.md files
6. Orchestrator spawns gsd-verifier for phase goal verification
7. Orchestrator updates ROADMAP.md, STATE.md
8. Returns completion report

**State Management:**

- `STATE.md`: Project memory, current phase/plan, accumulated decisions
- `config.json`: Workflow preferences (model_profile, verification enabled, retry settings)
- `ROADMAP.md`: Phase structure, completion status
- Git commits: Atomic record of each task completion (hash stored in SUMMARY.md)

## Key Abstractions

**Plan (PLAN.md):**
- Purpose: Executable prompt for single feature/task
- Examples: `phase/01-core-architecture/001-api-setup-PLAN.md`
- Pattern: Frontmatter (phase, plan, type, wave, depends_on) + Objective + Context (@-refs) + Tasks + Verification
- Characteristics: 2-3 tasks max (aggressive atomicity), contains verification criteria

**Task:**
- Purpose: Atomic unit of work
- Pattern: `<task type="auto|checkpoint:human-verify|checkpoint:decision">`
- Contains: files, action, verify, done criteria
- Characteristics: Type determines autonomy (auto = fully autonomous, checkpoint = pause for user)

**Summary (SUMMARY.md):**
- Purpose: Proof of execution
- Pattern: Frontmatter (phase, plan, completed_tasks with hashes, duration)
- Contains: Task results, deviations, verification proof
- Characteristics: Machine-readable frontmatter enables dependency resolution

**Checkpoint:**
- Purpose: User interaction point (after automation completes)
- Types: checkpoint:human-verify (verify work), checkpoint:decision (choose path), checkpoint:human-action (rare)
- Pattern: Executor STOPS at checkpoint, returns structured message, fresh continuation agent resumes
- Characteristics: Automation-first (only checkpoint what couldn't be automated)

**Wave:**
- Purpose: Parallelization group
- Pattern: Plan frontmatter `wave: N`
- Characteristics: Plans in same wave execute in parallel, next wave waits for completion
- Usage: Orchestrator groups plans, spawns parallel Tasks per wave

**Research (RESEARCH.md):**
- Purpose: Domain knowledge before planning
- Pattern: Produced by gsd-phase-researcher
- Characteristics: Answers "what should we build", not "how should we build"
- Used by: gsd-planner to derive task lists

## Entry Points

**`bin/install.js`:**
- Location: `bin/install.js`
- Triggers: `npx get-shit-done-cc` (npm install)
- Responsibilities: Parse install flags (--claude, --opencode, --global, --local), copy commands/agents to appropriate directories, create .claude/rules/ for auto-loading

**`commands/gsd/help.md`:**
- Location: `commands/gsd/help.md`
- Triggers: `/gsd:help`
- Responsibilities: List all 35 available commands, route to specific help

**`commands/gsd/new-project.md`:**
- Location: `commands/gsd/new-project.md`
- Triggers: `/gsd:new-project`
- Responsibilities: Initialize .planning/, run questioning, create PROJECT.md, ROADMAP.md

**`commands/gsd/plan-phase.md`:**
- Location: `commands/gsd/plan-phase.md`
- Triggers: `/gsd:plan-phase [phase]`
- Responsibilities: Spawn gsd-phase-researcher (if needed), spawn gsd-planner, iterate with gsd-plan-checker

**`commands/gsd/execute-phase.md`:**
- Location: `commands/gsd/execute-phase.md`
- Triggers: `/gsd:execute-phase [phase]`
- Responsibilities: Orchestrate wave-based execution, spawn gsd-executor per plan, handle checkpoints, verify phase goal

## Error Handling

**Strategy:** Automatic deviation handling, checkpoint gates, retry orchestration, state recovery

**Patterns:**

- **Auto-fixes:** gsd-executor automatically handles authentication errors, missing files, test failures (RULEs 1-9 in agent spec)
- **Checkpoints:** Plans with `checkpoint:*` type pause execution for user verification or decision
- **Retry:** If retry enabled in config.json, gsd-executor escalates failures to retry-orchestration.md workflow
- **State recovery:** STATE.md loaded first; if execution interrupted, fresh executor continues from completed_tasks
- **Verification loop:** gsd-plan-checker iterates with gsd-planner until plans pass or max iterations (prevents bad plans from executing)

## Cross-Cutting Concerns

**Logging:**
- Framework: console output (no structured logging framework)
- Pattern: Orchestrators report "about to spawn", "awaiting completion"; executors report "executing task N", completion, commits
- Visibility: User sees orchestrator context only, subagent context hidden

**Validation:**
- Phase existence: Orchestrator checks ROADMAP.md matches requested phase
- Plan validity: Orchestrator checks PLAN.md frontmatter, verifies wave dependencies
- Task structure: gsd-executor validates <task> XML structure before execution
- State consistency: gsd-executor loads STATE.md, verifies accumulated decisions apply

**Authentication:**
- Claude Code: Installed to `~/.claude/` or project `.claude/` via bin/install.js
- OpenCode: Installed to `~/.config/opencode/` (XDG Base Directory spec)
- Environment variables: CLAUDE_CONFIG_DIR, OPENCODE_CONFIG_DIR, OPENCODE_CONFIG (checked in order)

**Git Integration:**
- Atomic commits: One per task (hash stored in SUMMARY.md)
- Commit format: `{type}({phase}-{plan}): {description}` (feat, fix, test, refactor, docs, chore)
- State commits: Orchestrator commits ROADMAP.md, STATE.md changes with "docs(state): update"
- Co-Author: All commits include `Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>`

**Context Management:**
- Quality degradation: Claude degrades at 70%+ context usage
- Plan atomicity: 2-3 tasks maximum per plan to stay within 50% budget
- Wave parallelization: Multiple plans execute in parallel to reduce total session time
- Fresh subagents: Each agent spawned with full context, orchestrator stays lean

---

*Architecture analysis: 2026-01-24*
