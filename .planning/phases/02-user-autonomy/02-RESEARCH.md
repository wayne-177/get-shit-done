# Phase 2: User Autonomy - Research

**Researched:** 2026-02-02
**Domain:** Configuration systems, user preference management, bash config parsing
**Confidence:** HIGH

## Summary

Phase 2 implements user control over GSD framework behaviors that currently override user direction. The domain is configuration management within a bash-based orchestration system that uses JSON config files and markdown agent specifications.

The standard approach established in GSD is:
1. Store boolean toggles in `.planning/config.json`
2. Parse config with bash `grep -o` regex patterns
3. Check config values in orchestrator commands before spawning agents
4. Pass toggle state to agents via environment or prompt context
5. Display toggles in `/gsd:settings` with AskUserQuestion

This phase extends the existing pattern from workflow agents (research, plan_check, verifier) to two new domains:
- **Git behavior** (`git.auto_commit`) - controls whether executor auto-commits or stages-and-prompts
- **Safety guardrails** (`safety.verify_interfaces`, `safety.verify_requirements`) - controls NON-NEGOTIABLE executor steps

**Primary recommendation:** Follow established config parsing patterns. Add new config sections (`git`, `safety`), update validation schema, modify executor to check toggles before NON-NEGOTIABLE steps, add questions to settings command.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| JSON | Native | Config storage format | Already used throughout GSD (config.json, package.json) |
| Bash grep | Native | Config parsing | Established pattern for extracting config values |
| Markdown | N/A | Agent specification format | GSD agent files use markdown with XML-like tags |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| AskUserQuestion | GSD builtin | Interactive config prompts | Used by /gsd:settings and /gsd:new-project |
| Git bash commands | 2.x+ | Stage/commit operations | Modified to respect auto_commit toggle |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| JSON config | YAML/TOML | JSON already established, changing format would break compatibility |
| grep parsing | jq parsing | jq not guaranteed installed, grep pattern works and is established |
| Markdown agents | Structured prompts | Markdown is GSD's core format, changing would require platform rewrite |

**Installation:**
No new dependencies required. All capabilities exist in current GSD installation.

## Architecture Patterns

### Recommended Project Structure
```
.planning/
├── config.json              # User preferences (NEW: git, safety sections)
get-shit-done/
├── templates/
│   └── config.json          # Default config template (UPDATE)
├── lib/
│   └── validate-config.md   # Config validation schema (UPDATE)
└── references/
    └── user-autonomy.md     # NEW: Toggle documentation (AUTO-06)
agents/
└── gsd-executor.md          # UPDATE: Check toggles before NON-NEGOTIABLE steps
commands/gsd/
└── settings.md              # UPDATE: Add auto-commit question
```

### Pattern 1: Config Toggle Checking
**What:** Check boolean config value before executing behavior
**When to use:** Any behavior that should be user-controllable
**Example:**
```bash
# Established pattern from commands/gsd/plan-phase.md:134
WORKFLOW_RESEARCH=$(cat .planning/config.json 2>/dev/null | grep -o '"research"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")

# If toggle is false, skip behavior
if [ "$WORKFLOW_RESEARCH" = "false" ]; then
  echo "Skipping research (workflow.research: false)"
  # Continue to next step
fi
```

**Applied to Phase 2:**
```bash
# In agents/gsd-executor.md, before verify_interfaces step
AUTO_VERIFY_INTERFACES=$(cat .planning/config.json 2>/dev/null | grep -o '"verify_interfaces"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")

if [ "$AUTO_VERIFY_INTERFACES" = "false" ]; then
  echo "⚠️  Interface verification DISABLED (safety.verify_interfaces: false)"
  echo "    Proceeding without verification. Ensure interfaces are correct."
else
  # Execute NON-NEGOTIABLE verification
fi
```

### Pattern 2: Settings Command Integration
**What:** Add new questions to /gsd:settings AskUserQuestion flow
**When to use:** Any config toggle that users should control via settings UI
**Example:**
```markdown
# From commands/gsd/settings.md:44
{
  question: "Spawn Plan Researcher?",
  header: "Research",
  multiSelect: false,
  options: [
    { label: "Yes", description: "Research phase goals before planning" },
    { label: "No", description: "Skip research, plan directly" }
  ]
}
```

**Applied to Phase 2:**
```markdown
{
  question: "Auto-commit after each task?",
  header: "Git Behavior",
  multiSelect: false,
  options: [
    { label: "Yes (Recommended)", description: "Atomic commits for each completed task" },
    { label: "No", description: "Stage changes and prompt — you commit manually" }
  ]
}
```

### Pattern 3: Config Validation Extension
**What:** Add new fields to validate-config.md schema
**When to use:** Any new config section or field
**Example:**
```javascript
// From get-shit-done/lib/validate-config.md:142
if (config.safety !== undefined) {
  if (checkType(config.safety, 'object', 'safety')) {
    checkType(config.safety.always_confirm_destructive, 'boolean', 'safety.always_confirm_destructive');
  }
}
```

**Applied to Phase 2:**
```javascript
// Add to validation function
if (config.git !== undefined) {
  if (checkType(config.git, 'object', 'git')) {
    checkType(config.git.auto_commit, 'boolean', 'git.auto_commit');
  }
}

if (config.safety !== undefined) {
  if (checkType(config.safety, 'object', 'safety')) {
    // Existing validations...
    checkType(config.safety.verify_interfaces, 'boolean', 'safety.verify_interfaces');
    checkType(config.safety.verify_requirements, 'boolean', 'safety.verify_requirements');
  }
}
```

### Pattern 4: Staged Changes Instead of Commit
**What:** When auto_commit is false, stage files but return to user
**When to use:** git.auto_commit: false
**Example:**
```bash
# From agents/gsd-executor.md task_commit_protocol (MODIFIED)
AUTO_COMMIT=$(cat .planning/config.json 2>/dev/null | grep -o '"auto_commit"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")

if [ "$AUTO_COMMIT" = "true" ]; then
  # Standard commit
  git commit -m "feat(02-01): implement feature X"
  COMMIT_HASH=$(git rev-parse HEAD)
  echo "✓ Task complete: ${COMMIT_HASH:0:7}"
else
  # Stage without committing
  git add src/feature.ts src/feature.test.ts
  echo "✓ Task complete: Changes staged (not committed)"
  echo "  Run 'git commit' when ready"
  COMMIT_HASH="staged"
fi
```

### Anti-Patterns to Avoid
- **Inconsistent default behavior:** Default values must maintain backward compatibility (all toggles default to `true` for current behavior)
- **Silent toggle changes:** Always echo when a toggle changes expected behavior (e.g., "⚠️ Interface verification DISABLED")
- **Missing validation:** All new config fields must be added to validate-config.md schema
- **Bypassing existing safeguards:** Even when toggles are false, echo warnings so users know what was skipped

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JSON parsing in bash | Custom jq-based solution | Existing grep pattern | GSD established pattern, works without dependencies |
| Config merge logic | Deep merge utility | Simple key replacement | Config is flat enough, existing pattern sufficient |
| Interactive prompts | Custom readline | AskUserQuestion | Built into GSD/Claude Code |
| Config validation | Runtime checks in agents | validate-config.md library | Centralized validation already exists |

**Key insight:** GSD already has a working config system with validation, parsing, and display patterns. Extension is simpler than replacement.

## Common Pitfalls

### Pitfall 1: Breaking Backward Compatibility
**What goes wrong:** Changing default values or behavior causes existing projects to break
**Why it happens:** Developer assumes new behavior is "better" and makes it default
**How to avoid:**
- All new toggles MUST default to `true` (current behavior)
- Existing config.json files without new keys must work (use `|| echo "true"` fallback)
- Test with empty config, partial config, and full config
**Warning signs:** git bisect shows tests failing after config changes, users report "GSD stopped working"

### Pitfall 2: NON-NEGOTIABLE Language Without Escape Hatch
**What goes wrong:** Agent prompt says "NON-NEGOTIABLE" but toggle exists - conflicting instructions confuse Claude
**Why it happens:** Prompt updated to check toggle, but language still says "must always run"
**How to avoid:**
- Change "NON-NEGOTIABLE" to "STRONGLY RECOMMENDED"
- Add context: "This step is critical for production code. It can be disabled via `safety.verify_interfaces: false` for rapid prototyping."
- Make it clear the toggle EXISTS and is user-controlled
**Warning signs:** Claude ignores toggle and runs step anyway, or Claude skips step when toggle is true

### Pitfall 3: Config Check in Wrong Location
**What goes wrong:** Checking toggle in executor agent instead of orchestrator command
**Why it happens:** Misunderstanding of GSD architecture (orchestrators spawn agents, agents execute)
**How to avoid:**
- Workflow-level toggles checked in commands/gsd/ (orchestrators)
- Task-level toggles checked in agents/ (executors)
- git.auto_commit is task-level (checked in executor)
- safety.verify_* are task-level (checked in executor)
**Warning signs:** Toggle doesn't work, config value never read, echo statements don't appear

### Pitfall 4: Forgetting Settings UI
**What goes wrong:** Toggle works but users don't know it exists or how to change it
**Why it happens:** Developer adds config key but forgets to add question to /gsd:settings
**How to avoid:**
- Every new config toggle needs AskUserQuestion entry in commands/gsd/settings.md
- Every new config toggle needs documentation in reference doc (AUTO-06)
- Test: Can user discover and change this via /gsd:settings?
**Warning signs:** User asks "how do I disable auto-commit?" and must edit JSON manually

### Pitfall 5: git add -u When Staging Selectively
**What goes wrong:** `git add -u` stages ALL tracked modified files, not just task changes
**Why it happens:** Developer copies orchestrator correction pattern to task commit protocol
**How to avoid:**
- Task commits: Stage files explicitly by path (already correct in executor)
- Orchestrator corrections: Use `git add -u` cautiously, explain what it stages
- When auto_commit is false, stage same files as would be committed
**Warning signs:** Unrelated files committed together, git blame loses atomicity

## Code Examples

Verified patterns from GSD codebase:

### Config Toggle Check (Executor Level)
```bash
# Source: agents/gsd-executor.md:46-52 (commit_docs check)
COMMIT_PLANNING_DOCS=$(cat .planning/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")
# Auto-detect gitignored (overrides config)
git check-ignore -q .planning 2>/dev/null && COMMIT_PLANNING_DOCS=false
```

### Config Toggle Check (Orchestrator Level)
```bash
# Source: commands/gsd/plan-phase.md:131-135
WORKFLOW_RESEARCH=$(cat .planning/config.json 2>/dev/null | grep -o '"research"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")

if [ "$WORKFLOW_RESEARCH" = "false" ] && [ "$RESEARCH_FLAG" != "true" ]; then
  echo "Skipping research (workflow.research: false)"
  # Skip to step 6
fi
```

### AskUserQuestion Pattern
```markdown
<!-- Source: commands/gsd/settings.md:54-62 -->
{
  question: "Spawn Plan Researcher? (researches domain before planning)",
  header: "Research",
  multiSelect: false,
  options: [
    { label: "Yes", description: "Research phase goals before planning" },
    { label: "No", description: "Skip research, plan directly" }
  ]
}
```

### Config Update in Settings
```javascript
// Source: commands/gsd/settings.md:90-99 (implied from structure)
{
  ...existing_config,
  "model_profile": "quality" | "balanced" | "budget",
  "workflow": {
    "research": true/false,
    "plan_check": true/false,
    "verifier": true/false
  }
}
```

### Validation Schema Extension
```javascript
// Source: get-shit-done/lib/validate-config.md:140-145
if (config.safety !== undefined) {
  if (checkType(config.safety, 'object', 'safety')) {
    checkType(config.safety.always_confirm_destructive, 'boolean', 'safety.always_confirm_destructive');
    checkType(config.safety.always_confirm_external_services, 'boolean', 'safety.always_confirm_external_services');
  }
}
```

### Commit vs Stage Decision
```bash
# NEW PATTERN (not in codebase, proposed for implementation)
AUTO_COMMIT=$(cat .planning/config.json 2>/dev/null | grep -o '"auto_commit"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")

if [ "$AUTO_COMMIT" = "true" ]; then
  git commit -m "feat(02-01): task description"
  COMMIT_HASH=$(git rev-parse HEAD)
  echo "✓ Task complete: ${COMMIT_HASH:0:7}"
else
  # Stage files (already done in task_commit_protocol step 3)
  # Don't commit, return to user
  echo "✓ Task complete: Changes staged"
  echo "  Review with: git diff --staged"
  echo "  Commit with: git commit -m 'your message'"
  COMMIT_HASH="staged"
fi
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hard-coded NON-NEGOTIABLE | STRONGLY RECOMMENDED with toggle | Phase 2 (2026-02-02) | Users can disable guardrails for prototyping |
| Forced auto-commit | Configurable staging | Phase 2 (2026-02-02) | Users can review before committing |
| No safety config section | safety.* toggles | Phase 2 (2026-02-02) | Granular control over verification steps |
| Settings UI: 4 questions | Settings UI: 6 questions | Phase 2 (2026-02-02) | Auto-commit and verifications configurable |

**Deprecated/outdated:**
- "NON-NEGOTIABLE" language in local fork - Being replaced with STRONGLY RECOMMENDED + config toggle (Finding 4.1 from autonomy audit)
- Assumption that all GSD executions want atomic per-task commits - Now user-configurable via git.auto_commit

## Open Questions

Things that couldn't be fully resolved:

1. **Should git.auto_commit affect planning doc commits?**
   - What we know: `planning.commit_docs` already controls .planning/ commits
   - What's unclear: Should git.auto_commit also gate planning commits, or only code commits?
   - Recommendation: Keep separate. planning.commit_docs for .planning/, git.auto_commit for src/

2. **What happens to SUMMARY.md when commits are staged but not created?**
   - What we know: SUMMARY.md tracks commit hashes for each task
   - What's unclear: How to represent "staged" state in SUMMARY.md (no hash exists)
   - Recommendation: Use placeholder like "staged" or "uncommitted", document that user must commit manually

3. **Should verify_interfaces have granularity levels (all/critical/none)?**
   - What we know: Autonomy audit Finding 3.1 suggests granularity for deviation rules
   - What's unclear: Does verify_interfaces need same treatment (all/db-only/none)?
   - Recommendation: Start with boolean toggle. Add granularity in future milestone if users request it

4. **Should AUTO-06 reference doc be in get-shit-done/references/ or .planning/?**
   - What we know: Other reference docs in get-shit-done/references/
   - What's unclear: User autonomy is fork-specific, might belong with .planning/ docs
   - Recommendation: get-shit-done/references/user-autonomy.md (consistent with other references)

## Sources

### Primary (HIGH confidence)
- User Autonomy Audit: `.planning/research/04b-user-autonomy-audit.md` - Finding 4.1 (NON-NEGOTIABLE), Finding 1.1 (auto-commit)
- Executor agent: `agents/gsd-executor.md` - verify_interfaces (line 72), verify_requirement_coverage (line 103), task_commit_protocol (line 628)
- Settings command: `commands/gsd/settings.md` - Question patterns, config update flow
- Plan-phase command: `commands/gsd/plan-phase.md` - workflow.research toggle check (line 131-135)
- Execute-phase command: `commands/gsd/execute-phase.md` - workflow.verifier toggle check (line 102)
- Config validation: `get-shit-done/lib/validate-config.md` - Schema and validation patterns
- Config template: `get-shit-done/templates/config.json` - Default values and structure
- Requirements: `.planning/REQUIREMENTS.md` - AUTO-01 through AUTO-06 specifications

### Secondary (MEDIUM confidence)
- Planning config reference: `get-shit-done/references/planning-config.md` - commit_docs behavior documentation
- Project context: `.planning/PROJECT.md` - Constraints and key decisions

### Tertiary (LOW confidence)
- None used. All research based on local codebase analysis.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All patterns verified in existing codebase
- Architecture: HIGH - Following established GSD config patterns exactly
- Pitfalls: HIGH - Derived from autonomy audit findings and codebase analysis

**Research date:** 2026-02-02
**Valid until:** 2026-03-02 (30 days - stable domain, GSD patterns unlikely to change)
