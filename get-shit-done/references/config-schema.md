# GSD Configuration Schema

This document defines the complete configuration schema for the Get Shit Done (GSD) system. All settings are optional with safe defaults that maintain backward compatibility.

## Configuration File Location

`.planning/config.json` at project root

## Schema Structure

```json
{
  "mode": "interactive",
  "depth": "standard",
  "gates": { ... },
  "safety": { ... },
  "retry": { ... },
  "verification": { ... },
  "adaptive": { ... },
  "suggestions": { ... }
}
```

---

## Mode (config.mode)

Controls execution automation level.

**Type:** string
**Default:** "interactive"
**Options:**
- `"interactive"` - Confirm before executing plans and transitions
- `"yolo"` - Auto-execute all plans without confirmation (fast, risky)
- `"custom"` - Use granular gates configuration

---

## Depth (config.depth)

Controls planning detail level.

**Type:** string
**Default:** "standard"
**Options:**
- `"quick"` - Minimal planning, fewer tasks per plan
- `"standard"` - Balanced planning with 2-4 tasks per plan
- `"thorough"` - Detailed planning with comprehensive research

---

## Gates (config.gates)

Fine-grained control over confirmation prompts when mode = "custom".

**Schema:**
```json
{
  "gates": {
    "confirm_project": true,
    "confirm_phases": true,
    "confirm_roadmap": true,
    "confirm_breakdown": true,
    "confirm_plan": true,
    "execute_next_plan": true,
    "issues_review": true,
    "confirm_transition": true
  }
}
```

**Fields:**
- `confirm_project` (boolean, default: true): Confirm PROJECT.md before writing
- `confirm_phases` (boolean, default: true): Confirm phase breakdown before roadmap creation
- `confirm_roadmap` (boolean, default: true): Confirm ROADMAP.md before writing
- `confirm_breakdown` (boolean, default: true): Confirm plan breakdown before creating PLAN.md files
- `confirm_plan` (boolean, default: true): Confirm each PLAN.md before writing
- `execute_next_plan` (boolean, default: true): Confirm before executing next plan in sequence
- `issues_review` (boolean, default: true): Confirm before reviewing deferred issues
- `confirm_transition` (boolean, default: true): Confirm before transitioning to next phase

**Notes:**
- Only applies when mode = "custom"
- When mode = "interactive", all gates effectively true
- When mode = "yolo", all gates effectively false

---

## Safety (config.safety)

Safety overrides that apply regardless of mode.

**Schema:**
```json
{
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  }
}
```

**Fields:**
- `always_confirm_destructive` (boolean, default: true): Always confirm destructive operations (delete, force push, etc.) even in yolo mode
- `always_confirm_external_services` (boolean, default: true): Always confirm external service operations (deployments, API calls) even in yolo mode

---

## Retry (config.retry)

Automatic retry with alternative approaches when tasks fail.

**Schema:**
```json
{
  "retry": {
    "enabled": false,
    "max_attempts": 3,
    "escalation_threshold": 3,
    "log_attempts": true
  }
}
```

**Fields:**
- `enabled` (boolean, default: false): Enable automatic retry with alternative strategies
- `max_attempts` (integer, default: 3, range: 1-5): Maximum retry attempts per task before escalating to user
- `escalation_threshold` (integer, default: 3): Number of consecutive failures before escalating
- `log_attempts` (boolean, default: true): Log all retry attempts to .planning/ATTEMPTS_LOG.md for debugging

**Notes:**
- When enabled = false, original execute-phase.md flow used (backward compatible)
- Retry strategies sourced from .planning/ALTERNATIVE_PATHS.md
- Failure analysis uses get-shit-done/lib/failure-analysis.md taxonomy
- See Phase 1 documentation for retry workflow details

---

## Verification (config.verification)

Continuous verification and self-correction to prevent regressions.

**Schema:**
```json
{
  "verification": {
    "enabled": false,
    "regression_check": true,
    "auto_rollback": true,
    "baseline_tests": "npm test",
    "baseline_build": "npm run build"
  }
}
```

**Fields:**
- `enabled` (boolean, default: false): Master toggle for continuous verification features
- `regression_check` (boolean, default: true): Enable regression detection by comparing current tests against baseline (requires enabled = true)
- `auto_rollback` (boolean, default: true): Automatically rollback commits that introduce regressions (requires enabled = true)
- `baseline_tests` (string, default: "npm test"): Command to run for capturing and comparing test baselines (customize per project)
- `baseline_build` (string, default: "npm run build"): Command to run for capturing and comparing build baselines (customize per project)

**Notes:**
- Requires retry.enabled = true (verification integrates with retry workflow)
- Baselines captured before task execution, compared after
- Rollback uses `git revert` to create inverse commit
- See Phase 2 documentation for verification workflow details

---

## Adaptive Execution (config.adaptive)

AI-powered decision making during retry execution and cross-project learning.

**Schema:**
```json
{
  "adaptive": {
    "enabled": false,
    "fallback_to_static": true,
    "max_api_calls_per_task": 5,
    "history_tracking": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
```

**Fields:**
- `enabled` (boolean, default: false): Enable AI-powered failure analysis, strategy selection, and cross-project learning
- `fallback_to_static` (boolean, default: true): Fall back to static algorithms if AI analysis fails or returns invalid output
- `max_api_calls_per_task` (integer, default: 5): Max Task tool invocations per task execution (prevents runaway API usage)
- `history_tracking` (boolean, default: true): Track execution patterns in EXECUTION_HISTORY.md (requires enabled = true)
- `prediction_enabled` (boolean, default: true when adaptive.enabled=true): Enable failure prediction during planning. When true, plan-phase runs predict-failures workflow to assess risk before execution.
- `show_pattern_hints` (boolean, default: true when adaptive.enabled=true): Display pattern hints during execute-phase. Shows strategy hints, failure warnings, and task insights from the global knowledge base.

**Notes:**
- Requires retry.enabled = true (adaptive enhances retry workflow)
- When adaptive.enabled = false, all static Phase 1 logic used (backward compatible)
- Setting fallback_to_static = false risks retry failures if AI errors occur (not recommended)
- AI analysis uses Task tool with general-purpose subagent
- History tracking accumulates strategy effectiveness data for learning
- Prediction and hints use the global knowledge base at `~/.claude/gsd-knowledge/`
- See Phase 4 documentation for adaptive execution details
- See Phases 5-10 documentation for cross-project learning features

---

## Suggestions (config.suggestions)

Autonomous goal setting with codebase analysis and proactive suggestions.

**Schema:**
```json
{
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true,
    "refresh_suggestions_on_milestone": false
  }
}
```

**Fields:**
- `auto_suggest` (boolean, default: true): Automatically generate goal suggestions after running /gsd:analyze-codebase. When true, analysis triggers generate-suggestions workflow to create GOAL_PROPOSALS.md.
- `auto_update_claude_md` (boolean, default: true): Automatically update CLAUDE.md with suggestions after running /gsd:suggest. When true, the Suggested Goals section in CLAUDE.md is refreshed with current proposals.
- `refresh_suggestions_on_milestone` (boolean, default: false): Optionally refresh suggestions during /gsd:complete-milestone. When true, runs analyze-codebase and generate-suggestions before archiving milestone.

**Notes:**
- No dependencies on other config sections (suggestions.* options work independently)
- When auto_suggest = false, must manually run /gsd:suggest to generate proposals
- When auto_update_claude_md = false, GOAL_PROPOSALS.md is created but CLAUDE.md not updated
- When refresh_suggestions_on_milestone = true, adds processing time to milestone completion
- All suggestion features use graceful degradation (errors never block parent workflow)
- See Phases 11-14 documentation for autonomous goal setting details

---

## Suggestion Outputs

Files created by v4.5 autonomous goal setting features:

**`.planning/ANALYSIS_REPORT.md`**
- Created by: /gsd:analyze-codebase command
- Contains: Codebase analysis findings (dependencies, security, tech debt)
- Format: Markdown with Executive Summary, per-analyzer sections, Recommendations
- Used by: generate-suggestions workflow to create goal proposals

**`.planning/GOAL_PROPOSALS.md`**
- Created by: /gsd:suggest command (or auto-generated after analyze-codebase)
- Contains: Prioritized goal suggestions with scores and effort estimates
- Format: Markdown with numbered suggestions, scores, commands
- Used by: update-claude-md workflow to populate CLAUDE.md

**`CLAUDE.md` (## Suggested Goals section)**
- Updated by: /gsd:suggest command (when auto_update_claude_md = true)
- Contains: Compact suggestions for session-start awareness
- Format: Quick Wins (numbered) + Requires Planning (bullets)
- Preserves: All other CLAUDE.md sections unchanged

---

## Knowledge Base (Global)

The knowledge base stores learned patterns across all projects at `~/.claude/gsd-knowledge/`.

**Location:** `~/.claude/gsd-knowledge/` (not configurable, global to user)

**Structure:**
```
~/.claude/gsd-knowledge/
├── index.json                    # Pattern index for fast lookups
├── config.json                   # Knowledge base settings
└── patterns/
    ├── strategy-effectiveness/   # What strategies work for what failures
    ├── failure-pattern/          # Common failure categories by task type
    └── task-insight/             # Best practices and pitfalls by task type
```

**Initialization:**
- Created automatically on first adaptive execution (init-knowledge-base.md workflow)
- Safe to call multiple times (idempotent)
- If creation fails (permissions, disk), logs warning and continues without blocking

**Pattern Types:**

| Type | Purpose | Example |
|------|---------|---------|
| `strategy-effectiveness` | Track which strategies work for which failures | "retry-with-reruns works 85% for test-run failures" |
| `failure-pattern` | Track common failure categories by task type | "bash-command tasks often fail with Dependency errors" |
| `task-insight` | Best practices and pitfalls learned from execution | "Always verify file exists before editing" |

**Pattern Lifecycle:**
- Patterns accumulate confidence as they're observed across projects
- Patterns decay over time if not seen (configurable half-life)
- Cross-project validated patterns (3+ projects) are more reliable
- Patterns can be manually deleted with `--decay` flag

**Not Configurable:**
- Knowledge base location is fixed at `~/.claude/gsd-knowledge/`
- Pattern schema version is determined by GSD version
- Decay parameters are in `~/.claude/gsd-knowledge/config.json` (not project config)

---

## Example Configurations

### Default (Backward Compatible)
```json
{
  "mode": "interactive",
  "depth": "standard",
  "retry": {
    "enabled": false
  },
  "verification": {
    "enabled": false
  },
  "adaptive": {
    "enabled": false
  }
}
```

### Full v3.0 Features Enabled
```json
{
  "mode": "interactive",
  "depth": "standard",
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "verification": {
    "enabled": true,
    "regression_check": true,
    "auto_rollback": true,
    "baseline_tests": "npm test",
    "baseline_build": "npm run build"
  },
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true,
    "max_api_calls_per_task": 5,
    "history_tracking": true
  }
}
```

### Full v4.0 Features Enabled
```json
{
  "mode": "interactive",
  "depth": "standard",
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "verification": {
    "enabled": true,
    "regression_check": true,
    "auto_rollback": true
  },
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true,
    "max_api_calls_per_task": 5,
    "history_tracking": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
```

### Full v4.5 Features Enabled
```json
{
  "mode": "interactive",
  "depth": "standard",
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "verification": {
    "enabled": true,
    "regression_check": true,
    "auto_rollback": true
  },
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true,
    "max_api_calls_per_task": 5,
    "history_tracking": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  },
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true,
    "refresh_suggestions_on_milestone": true
  }
}
```

### YOLO Mode (Maximum Automation)
```json
{
  "mode": "yolo",
  "depth": "quick",
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  },
  "retry": {
    "enabled": true,
    "max_attempts": 3
  },
  "verification": {
    "enabled": true,
    "auto_rollback": true
  },
  "adaptive": {
    "enabled": true
  }
}
```

---

## Dependency Graph

```
verification.enabled → requires retry.enabled = true
adaptive.enabled → requires retry.enabled = true
adaptive.history_tracking → requires adaptive.enabled = true
adaptive.prediction_enabled → requires adaptive.enabled = true
adaptive.show_pattern_hints → requires adaptive.enabled = true
suggestions.* → no prerequisites (independent feature set)
```

**Validation rules:**
- If verification.enabled = true but retry.enabled = false → Error
- If adaptive.enabled = true but retry.enabled = false → Error
- If adaptive.history_tracking = true but adaptive.enabled = false → Warning (silently disabled)
- If adaptive.prediction_enabled = true but adaptive.enabled = false → Warning (silently disabled)
- If adaptive.show_pattern_hints = true but adaptive.enabled = false → Warning (silently disabled)
- suggestions.* options have no prerequisites and work independently

---

## Configuration Combinations

Understanding how adaptive options interact:

| adaptive.enabled | prediction_enabled | show_pattern_hints | Behavior |
|-----------------|-------------------|-------------------|----------|
| `false` | N/A | N/A | v2.0/v3.0 behavior, no AI features, no cross-project learning |
| `true` | `true` | `true` | Full v4.0 autonomy: hints during execution + predictions during planning |
| `true` | `false` | `true` | Hints during execution but no risk predictions during planning |
| `true` | `true` | `false` | Risk predictions during planning but no hints during execution |
| `true` | `false` | `false` | AI failure analysis only (v3.0 adaptive), no cross-project features |

**Common use cases:**

1. **Legacy mode (v2.0):** `adaptive.enabled = false` - Static algorithms only
2. **Intelligence mode (v3.0):** `adaptive.enabled = true, prediction_enabled = false, show_pattern_hints = false` - AI for current execution only
3. **Full autonomy (v4.0):** `adaptive.enabled = true, prediction_enabled = true, show_pattern_hints = true` - Cross-project learning and prediction
4. **Full goal setting (v4.5):** All v4.0 options + `suggestions.auto_suggest = true` - Proactive goal proposals

---

## Suggestion Configuration Combinations

Understanding how suggestion options interact:

| auto_suggest | auto_update_claude_md | refresh_on_milestone | Behavior |
|--------------|----------------------|---------------------|----------|
| `true` | `true` | `false` | **Default:** Full pipeline on analyze, manual refresh on milestone |
| `true` | `true` | `true` | **Full automation:** Including milestone refresh |
| `true` | `false` | `false` | Generate proposals only, CLAUDE.md not auto-updated |
| `false` | `true` | `false` | Manual suggestion generation, auto CLAUDE.md update |
| `false` | `false` | `false` | Fully manual: explicit /gsd:suggest required |
| `false` | N/A | `true` | Milestone refresh without ongoing suggestions |

**Common use cases:**

1. **Default (recommended):** `auto_suggest: true, auto_update_claude_md: true, refresh_on_milestone: false`
   - Suggestions generated after analysis
   - CLAUDE.md auto-updated with suggestions
   - Manual refresh for milestones

2. **Full automation:** All options `true`
   - Everything automatic including milestone refresh
   - More processing time at milestone completion

3. **Minimal suggestions:** `auto_suggest: false, auto_update_claude_md: false`
   - All features opt-in via manual commands
   - No automatic file generation

---

## Migration Guide

### Existing Projects (No Config)

Projects without `.planning/config.json` get default behavior:
- Interactive mode
- No retry
- No verification
- No adaptive

**No changes required.** Add config.json only when opting into enhancements.

### Upgrading from v2.0 to v3.0

v2.0 config.json (retry + verification):
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true }
}
```

Add adaptive section to enable v3.0 features:
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": { "enabled": true }
}
```

**Backward compatible.** Omitting adaptive section preserves v2.0 behavior.

### Upgrading from v3.0 to v4.0

v3.0 config.json (retry + verification + adaptive):
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": { "enabled": true }
}
```

v4.0 features are enabled automatically when adaptive.enabled = true:
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
```

**No changes required for v4.0.** When adaptive.enabled = true:
- `prediction_enabled` defaults to true (failure prediction during planning)
- `show_pattern_hints` defaults to true (pattern hints during execution)
- Knowledge base initialized automatically at `~/.claude/gsd-knowledge/`

**To disable specific v4.0 features while keeping v3.0:**
```json
{
  "adaptive": {
    "enabled": true,
    "prediction_enabled": false,
    "show_pattern_hints": false
  }
}
```

This gives v3.0 behavior (AI failure analysis, adaptive strategy selection) without v4.0 cross-project learning features.

### Upgrading from v4.0 to v4.5

v4.0 config.json (full adaptive + prediction):
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
```

v4.5 adds suggestion features (enabled by default):
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  },
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true,
    "refresh_suggestions_on_milestone": false
  }
}
```

**No changes required for v4.5.** Suggestion defaults are:
- `auto_suggest` defaults to true (suggestions generated after analysis)
- `auto_update_claude_md` defaults to true (CLAUDE.md updated after suggestions)
- `refresh_suggestions_on_milestone` defaults to false (opt-in for milestone refresh)

**To disable v4.5 features while keeping v4.0:**
```json
{
  "suggestions": {
    "auto_suggest": false,
    "auto_update_claude_md": false
  }
}
```

This gives v4.0 behavior (cross-project learning, prediction) without automatic goal setting features.

### Upgrading from v4.5 to v5.0

v4.5 config.json (full suggestions):
```json
{
  "retry": { "enabled": true },
  "verification": { "enabled": true },
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  },
  "suggestions": {
    "auto_suggest": true,
    "auto_update_claude_md": true
  }
}
```

**No changes required for v5.0.** v5.0 features are:
- **Zero-configuration** - No config.json changes needed
- **Accessed via commands** - `/gsd:discover-monorepo`, `/gsd:execute-monorepo`
- **Natural language routing** - Available via `route-intent.md` workflow
- **AI workflow generation** - Available for novel requests

**New commands available:**
- `/gsd:discover-monorepo` - Detect monorepo type and build dependency graph
- `/gsd:execute-monorepo` - Run tasks across packages in dependency order

**All v4.5 features continue working unchanged:**
- Codebase analysis and suggestions
- Cross-project learning
- Failure prediction
- Retry and verification

---

## v5.0 Transcendence Features

v5.0 introduces four major feature areas that require **no configuration** - they are accessed via explicit commands or natural language.

### Workflow Factory

**Zero-configuration feature** - Routes natural language to workflows via intent matching and AI generation.

**Key Files:**
- `~/.claude/get-shit-done/workflow-templates/template-index.json` - Built-in template registry (18 workflows)
- `~/.claude/get-shit-done/workflow-templates/generated/generated-index.json` - User-approved generated workflows

**Confidence Thresholds (informational, not configurable):**
| Level | Distance | Behavior |
|-------|----------|----------|
| HIGH | ≤0.3 | Direct execution without confirmation |
| MEDIUM | 0.3-0.5 | User confirmation required before execution |
| LOW | >0.5 | Falls back to AI workflow generation |

**How it works:**
1. Natural language input received (e.g., "plan phase 3")
2. Intent matcher compares against 18 built-in templates
3. If HIGH confidence: Variables extracted and workflow executes
4. If MEDIUM confidence: User confirms before execution
5. If LOW confidence: AI generates workflow, safety validated, user approves

**AI Generation Safety:**
- Blocklist patterns rejected: `rm -rf`, `eval`, `sudo`, etc.
- Allowlist commands pass: `npm test`, `git status`, `cat`, etc.
- Human confirmation **always required** before executing generated workflows

### Monorepo Support

**Zero-configuration feature** - Auto-detects monorepo type and coordinates execution.

**Commands:**
- `/gsd:discover-monorepo` - Detect monorepo type and build dependency graph
- `/gsd:execute-monorepo` - Run tasks across packages in dependency order

**Output Files:**
- `.planning/MONOREPO_INFO.md` - Discovery results with Mermaid dependency graph
- `.planning/EXECUTION_DASHBOARD.md` - Real-time execution status with package statuses

**Supported Monorepo Types (detection priority order):**
1. **pnpm** - Detected via `pnpm-workspace.yaml`
2. **turborepo** - Detected via `turbo.json`
3. **nx** - Detected via `nx.json`
4. **lerna** - Detected via `lerna.json`
5. **rush** - Detected via `rush.json`
6. **npm-workspaces** - Detected via `package.json` workspaces array
7. **single-package** - Fallback for non-monorepo projects

**Command Flags:**
| Command | Flags | Purpose |
|---------|-------|---------|
| `/gsd:discover-monorepo` | `--all`, `--impact`, `--refresh` | Full discovery, impact analysis, bypass cache |
| `/gsd:execute-monorepo` | `--all`, `--affected`, `--package`, `--status`, `--tasks` | Scope selection and task customization |

**Execution Strategies:**
- **continue-on-failure** (default): Continue execution for unrelated packages when one fails
- **stop-on-failure**: Stop execution for entire dependency chain when one fails

### Feature Summary

| Version | Features | Config Required | Key Capabilities |
|---------|----------|-----------------|------------------|
| v1.0 | Core workflow | mode, depth, gates | Planning and execution |
| v2.0 | Reliability | retry, verification | Automatic retry, regression detection |
| v3.0 | Intelligence | adaptive | AI-powered failure analysis |
| v4.0 | Learning | adaptive.prediction | Cross-project patterns, failure prediction |
| v4.5 | Goals | suggestions | Codebase analysis, goal suggestions |
| v5.0 | Transcendence | **None** | Workflow factory, monorepo coordination |

### Why v5.0 Requires No Configuration

All v5.0 features are:
- **Zero-configuration** - Just use the commands
- **Backward compatible** - Existing projects unchanged
- **Opt-in** - Only activated when explicitly used
- **Graceful** - Non-monorepo projects handled elegantly

---

## Best Practices

1. **Start conservative:** Enable features incrementally (retry → verification → adaptive)
2. **Keep fallback_to_static = true:** Ensures reliability even if AI fails
3. **Monitor history:** Review EXECUTION_HISTORY.md periodically to validate learning
4. **Respect max_api_calls:** Prevents runaway costs from Task tool invocations
5. **Test in interactive mode first:** Validate config before switching to yolo
6. **Customize baseline commands:** Match your project's test/build commands

---

## Troubleshooting

### Retry Not Working
- Check retry.enabled = true
- Verify .planning/ALTERNATIVE_PATHS.md exists
- Check failure category is retry-appropriate (see failure-analysis.md)

### Verification Failing
- Ensure retry.enabled = true (dependency)
- Verify baseline commands work: run manually first
- Check baseline data in .planning/baselines/

### Adaptive Not Activating
- Ensure retry.enabled = true (dependency)
- Verify adaptive.enabled = true
- Check Task tool is available (not in restricted environment)
- Review fallback_to_static setting

### History Not Tracking
- Ensure adaptive.enabled = true (dependency)
- Verify history_tracking = true
- Check .planning/EXECUTION_HISTORY.md is writable
- Confirm at least one retry attempt occurred

### Suggestions Not Generating
- Verify /gsd:analyze-codebase was run first (creates ANALYSIS_REPORT.md)
- Check auto_suggest config (default: true)
- Verify get-shit-done/workflows/generate-suggestions.md exists
- Check .planning/GOAL_PROPOSALS.md is writable

### CLAUDE.md Not Updating
- Verify GOAL_PROPOSALS.md exists (suggestions must be generated first)
- Check auto_update_claude_md config (default: true)
- Verify get-shit-done/workflows/update-claude-md.md exists
- Ensure CLAUDE.md file exists (workflow doesn't create it)

### Analysis Missing Data
- Verify package.json exists for dependency analysis
- Check npm is installed and available
- Optional tools (jq, Knip, lockfile-lint) add detail but aren't required
- Check .planning/analysis/ directory for individual analyzer outputs

### Intent Matching Not Working (v5.0)
- Verify template-index.json exists at `~/.claude/get-shit-done/workflow-templates/`
- Check input is being normalized (lowercase, no punctuation)
- Verify intent-matcher.md workflow exists
- Try explicit workflow name if matching fails

### Monorepo Not Detected (v5.0)
- Verify monorepo config file exists (pnpm-workspace.yaml, turbo.json, etc.)
- Check detection priority order (pnpm > turborepo > nx > lerna > rush > npm-workspaces)
- For npm workspaces, ensure `workspaces` array exists in package.json
- Single-package fallback is normal for non-monorepo projects

### Coordinated Execution Fails (v5.0)
- Run `/gsd:discover-monorepo` first if MONOREPO_INFO.md missing
- Check build order in MONOREPO_INFO.md matches expected dependencies
- Verify packages have valid package.json files
- Check for circular dependencies (cycle detection should warn)

### Generated Workflows Not Saving (v5.0)
- Ensure workflow was approved (not just generated)
- Check `~/.claude/get-shit-done/workflow-templates/generated/` directory exists
- Verify generated-index.json is writable
- Approval required before workflow is saved for re-use

---

## Schema Version

**Current:** v5.0 (2026-01-08)

**Changelog:**
- v5.0 (2026-01-08): Added v5.0 Transcendence Features section (workflow factory, monorepo support), Feature Summary table, v4.5→v5.0 migration guide, v5.0 troubleshooting sections. Note: v5.0 features require no configuration.
- v4.5 (2026-01-08): Added suggestions configuration (auto_suggest, auto_update_claude_md, refresh_suggestions_on_milestone), Suggestion Outputs section, Suggestion Configuration Combinations table, v4.0→v4.5 migration guide
- v4.0 (2026-01-08): Added cross-project learning configuration (prediction_enabled, show_pattern_hints), Knowledge Base section, Configuration Combinations table
- v3.0 (2026-01-07): Added adaptive execution configuration
- v2.0 (2026-01-06): Added retry and verification configuration
- v1.0 (2025-12-XX): Initial schema with mode, depth, gates, safety
