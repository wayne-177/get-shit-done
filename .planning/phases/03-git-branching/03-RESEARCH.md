# Phase 3: Git Branching - Research

**Researched:** 2026-02-02
**Domain:** Git branching automation, workflow integration, config-driven behavior
**Confidence:** MEDIUM-HIGH

## Summary

Phase 3 implements git branching strategy configuration that allows users to choose how GSD creates and manages branches during execution. The domain is automated git branching within a bash-orchestrated CI/CD workflow that uses config.json for behavior settings.

The standard approach established in GSD (from Phase 2) is:
1. Store enum values in `.planning/config.json`
2. Parse config with bash `grep -o` regex patterns
3. Check config values in orchestrator commands before branch operations
4. Execute git branching logic at the start of phase/milestone execution
5. Display options in `/gsd:settings` with AskUserQuestion

Phase 3 extends this pattern from boolean toggles to enum-based strategy selection:
- **Git branching config** (`git.branching_strategy`) - controls where/when branches are created (none/phase/milestone)
- **Branch naming** - gsd/phase-{N}-{slug} for phase branches, gsd/{version}-{slug} for milestone branches
- **Squash merge** - optional feature for milestone completion (upstream status unclear)

**Primary recommendation:** Add git.branching_strategy enum to config, update settings UI with 3-option question, insert branch creation logic at start of execute-phase (after validation, before wave execution) and optionally at start of new-milestone. Generate slugs from phase/milestone names using established bash pattern. Research squash merge implementation separately as it may not exist upstream.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| JSON | Native | Config storage format | Already used throughout GSD (config.json, package.json) |
| Bash grep | Native | Config parsing | Established pattern for extracting config values |
| Git | 2.x+ | Branch operations | `git checkout -b` or `git switch -c` for branch creation |
| Bash tr/sed | Native | Slug generation | Established pattern in add-phase, insert-phase, quick commands |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| AskUserQuestion | GSD builtin | Interactive config prompts | Used by /gsd:settings |
| Git switch | Git 2.23+ | Modern branch creation | Preferred over checkout -b for clarity |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| git checkout -b | git switch -c | git switch is modern (2.23+) but checkout -b has wider compatibility |
| Slug at execution | Slug at planning | Execution-time allows phase name changes, planning-time is more predictable |
| Branch per plan | Branch per phase/milestone | Per-plan would create excessive branches, phase/milestone grouping is cleaner |

**Installation:**
No new dependencies required. All capabilities exist in current GSD installation.

## Architecture Patterns

### Recommended Project Structure
```
.planning/
├── config.json              # User preferences (ADD: git.branching_strategy)
get-shit-done/
├── templates/
│   └── config.json          # Default config template (UPDATE: add git.branching_strategy)
├── lib/
│   └── validate-config.md   # Config validation schema (UPDATE: add branching_strategy enum)
commands/gsd/
├── settings.md              # UPDATE: Add branching strategy question
├── execute-phase.md         # UPDATE: Create branch before wave execution (step 0.5)
└── new-milestone.md         # CONSIDER: Create milestone branch when starting new milestone
```

### Pattern 1: Enum Config Parsing
**What:** Parse enum string value from config (not just boolean)
**When to use:** Config value has multiple discrete options (none/phase/milestone)
**Example:**
```bash
# From Phase 2 pattern (mode parsing):
MODE=$(cat .planning/config.json 2>/dev/null | grep -o '"mode"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "interactive")

# Applied to Phase 3:
BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "none")
```

### Pattern 2: Slug Generation from Name
**What:** Convert human-readable name to git-safe branch slug
**When to use:** Creating branches from phase names or milestone versions
**Example:**
```bash
# From commands/gsd/add-phase.md:
slug=$(echo "$description" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')

# Applied to Phase 3 - Phase branch:
PHASE_NAME=$(grep "Phase ${PHASE}:" .planning/ROADMAP.md | sed 's/.*Phase [0-9]*: //' | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
BRANCH_NAME="gsd/phase-${PHASE}-${PHASE_NAME}"
git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"

# Applied to Phase 3 - Milestone branch:
MILESTONE_VERSION="v1.1"  # From PROJECT.md "Current Milestone: v1.1 ..."
MILESTONE_SLUG=$(echo "$MILESTONE_VERSION" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
BRANCH_NAME="gsd/${MILESTONE_SLUG}-upstream-port"  # Append milestone goal slug
git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"
```

### Pattern 3: Conditional Branch Creation
**What:** Check branching_strategy before creating branch
**When to use:** At start of phase execution or milestone initialization
**Example:**
```bash
# In commands/gsd/execute-phase.md after Step 1 (validate phase)
BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "none")

if [ "$BRANCHING_STRATEGY" = "phase" ]; then
  # Extract phase name from ROADMAP.md
  PHASE_NAME=$(grep "Phase ${PHASE}:" .planning/ROADMAP.md | sed 's/.*Phase [0-9]*: //' | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  BRANCH_NAME="gsd/phase-${PHASE}-${PHASE_NAME}"

  # Check if branch already exists
  if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
    echo "Branch $BRANCH_NAME already exists, switching to it"
    git switch "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
  else
    echo "Creating branch: $BRANCH_NAME"
    git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"
  fi
elif [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  # Milestone branch should be created at milestone start, not per phase
  # Check if we're on a gsd/ branch already, if not warn
  CURRENT_BRANCH=$(git branch --show-current)
  if [[ ! "$CURRENT_BRANCH" =~ ^gsd/ ]]; then
    echo "⚠️  Branching strategy is 'milestone' but not on a gsd/ branch"
    echo "    Run /gsd:new-milestone to create milestone branch"
  fi
else
  # branching_strategy = "none", stay on current branch
  echo "Branching strategy: none (committing to current branch)"
fi
```

### Pattern 4: Settings UI with Enum Options
**What:** Add multi-option question to AskUserQuestion
**When to use:** Config field has >2 discrete options (not just Yes/No)
**Example:**
```markdown
# From Phase 2 (Model Profile question):
{
  question: "Which model profile for agents?",
  header: "Model",
  multiSelect: false,
  options: [
    { label: "Quality", description: "Opus everywhere except verification (highest cost)" },
    { label: "Balanced (Recommended)", description: "Opus for planning, Sonnet for execution/verification" },
    { label: "Budget", description: "Sonnet for writing, Haiku for research/verification (lowest cost)" }
  ]
}

# Applied to Phase 3:
{
  question: "Git branching strategy?",
  header: "Git Branching",
  multiSelect: false,
  options: [
    { label: "None (Recommended)", description: "Commit directly to current branch" },
    { label: "Per Phase", description: "Create branch for each phase (gsd/phase-{N}-{name})" },
    { label: "Per Milestone", description: "Create branch for entire milestone (gsd/{version}-{name})" }
  ]
}
```

### Pattern 5: Branch Cleanup at Milestone Completion
**What:** Optionally merge branch back to main and delete after milestone complete
**When to use:** User has branching_strategy set and wants to clean up after milestone
**Example:**
```bash
# In commands/gsd/complete-milestone.md (after archiving milestone)
BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "none")

if [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  CURRENT_BRANCH=$(git branch --show-current)

  # Offer to merge milestone branch back to main
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "Milestone complete. Currently on: $CURRENT_BRANCH"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""
  echo "Options:"
  echo "1. Merge to main and delete branch (clean history)"
  echo "2. Keep branch for reference (manual cleanup later)"
  echo ""
  # Use AskUserQuestion or simple read
fi
```

### Anti-Patterns to Avoid
- **Creating branches after work starts:** Branching must happen BEFORE any plan execution to avoid orphaned commits
- **Hardcoding branch names:** Always derive from phase/milestone names so branches are self-documenting
- **Forgetting idempotency:** Check if branch exists before creating (support resume after interruption)
- **Phase branches in milestone mode:** When strategy is "milestone", don't create phase branches (already on milestone branch)
- **Slugifying too aggressively:** Keep enough of the name to be recognizable (don't truncate to meaninglessness)

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Slug generation | Custom sanitization | Established tr/sed pattern | GSD already has this in 3 commands (add-phase, insert-phase, quick) |
| Config parsing | jq-based solution | Existing grep pattern | No jq dependency, works across all systems |
| Branch name validation | Custom regex | Git's own validation | git checkout -b fails gracefully on invalid names |
| Merge strategies | Custom conflict resolution | git merge --squash OR standard merge | Let git handle merge semantics |

**Key insight:** Git branching automation is mostly orchestration logic (when/where to create branches), not complex branching algorithms. Use git's native commands and established GSD patterns.

## Common Pitfalls

### Pitfall 1: Creating Branch Too Late in Execution
**What goes wrong:** Branch created after some commits already made on main, leaving orphaned commits
**Why it happens:** Developer adds branching logic inside wave execution instead of before it
**How to avoid:**
- Branch creation must be in execute-phase Step 0.5 (after validation, BEFORE wave execution)
- Test: Run phase execution, check git log --all --graph, verify all phase commits on branch
**Warning signs:** Phase commits appear on main branch when branching_strategy is "phase"

### Pitfall 2: Milestone Branch Not Created at Milestone Start
**What goes wrong:** User sets branching_strategy to "milestone" but starts working on main, then phases create individual branches
**Why it happens:** Milestone branch creation logic missing from new-milestone command
**How to avoid:**
- Add branching logic to new-milestone command (or document that user must create branch manually)
- Check at phase execution: if strategy is "milestone" and not on gsd/ branch, warn user
**Warning signs:** Phases execute on main despite milestone strategy setting

### Pitfall 3: Non-Idempotent Branch Creation
**What goes wrong:** Script tries to create branch that already exists, fails with error
**Why it happens:** Resume after interruption, or user manually created branch
**How to avoid:**
- Always check if branch exists before creating: `git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"`
- If exists: switch to it; if not: create it
**Warning signs:** "fatal: A branch named 'gsd/phase-1-...' already exists"

### Pitfall 4: Forgetting Slug Truncation
**What goes wrong:** Very long phase names create unwieldy branch names (>100 chars)
**Why it happens:** Phase name is entire goal description, not just title
**How to avoid:**
- Extract just the phase title from ROADMAP.md (e.g., "Context Flow & Display" not full goal)
- Optionally truncate slug: `cut -c1-40` or `cut -c1-60`
- Test: Create phase with long name, verify branch name is reasonable
**Warning signs:** Branch names like "gsd/phase-3-users-can-choose-a-branching-strategy-and-have-gsd-create-manage-branches-automatically-during-execution"

### Pitfall 5: Squash Merge Without Understanding Tradeoffs
**What goes wrong:** All atomic per-task commits lost when milestone is squash-merged
**Why it happens:** User thinks "clean history" means "one commit per milestone"
**How to avoid:**
- Document squash merge tradeoffs in reference doc
- Default to standard merge (preserves all commits)
- Offer squash merge as OPTION with warning about losing granular history
**Warning signs:** Git bisect becomes useless, blame points to single massive commit

### Pitfall 6: Config Not Updated After Settings Question Added
**What goes wrong:** Settings UI shows branching question but config.json doesn't get updated
**Why it happens:** Forgot to add git.branching_strategy to config merge logic
**How to avoid:**
- Follow 3-step pattern from Phase 2: template → validation → settings UI
- Test: Run /gsd:settings, select "Per Phase", verify config.json has "branching_strategy": "phase"
**Warning signs:** Settings displays strategy but execute-phase still says "none"

## Code Examples

Verified patterns from GSD codebase and research:

### Slug Generation Pattern (Established)
```bash
# Source: commands/gsd/add-phase.md, insert-phase.md, quick.md
slug=$(echo "$description" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')

# Applied to phase name extraction:
PHASE_NAME=$(grep "Phase ${PHASE}:" .planning/ROADMAP.md | sed 's/.*Phase [0-9]*: //' | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
```

### Enum Config Parsing (Extension of Phase 2 Pattern)
```bash
# Source: Phase 2 mode parsing (execute-phase workflow)
BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "none")
```

### Git Branch Creation with Fallback
```bash
# Recommended pattern (git switch preferred, checkout fallback)
git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"

# With idempotency:
if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
  git switch "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
else
  git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"
fi
```

### Phase Branch Creation (Proposed)
```bash
# NEW PATTERN (proposed for execute-phase.md Step 0.5)
# After: Step 1 (validate phase)
# Before: Step 2 (discover plans)

BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "none")

if [ "$BRANCHING_STRATEGY" = "phase" ]; then
  # Extract clean phase name from ROADMAP.md
  PHASE_TITLE=$(grep "^### Phase ${PHASE}:" .planning/ROADMAP.md | sed 's/.*Phase [0-9]*: //' | head -1)

  if [ -z "$PHASE_TITLE" ]; then
    echo "⚠️  Could not extract phase name from ROADMAP.md"
    echo "    Using default slug"
    PHASE_SLUG="phase-${PHASE}"
  else
    # Generate slug from title
    PHASE_SLUG=$(echo "$PHASE_TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//' | cut -c1-50)
  fi

  BRANCH_NAME="gsd/phase-${PHASE}-${PHASE_SLUG}"

  # Check if branch exists (idempotency)
  if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Switching to existing branch: $BRANCH_NAME"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    git switch "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
  else
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Creating phase branch: $BRANCH_NAME"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"
  fi
elif [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  # Milestone branching: verify we're on the right branch
  CURRENT_BRANCH=$(git branch --show-current)
  if [[ ! "$CURRENT_BRANCH" =~ ^gsd/ ]]; then
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "⚠️  Warning: Branching strategy is 'milestone'"
    echo "    Expected: On a gsd/{version}-{name} branch"
    echo "    Actual: On branch '$CURRENT_BRANCH'"
    echo ""
    echo "    Create milestone branch with /gsd:new-milestone"
    echo "    or change strategy with /gsd:settings"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    # Continue anyway (don't block execution)
  fi
else
  # branching_strategy = "none"
  CURRENT_BRANCH=$(git branch --show-current)
  echo "Branching strategy: none (using current branch: $CURRENT_BRANCH)"
fi
```

### Milestone Branch Creation (Proposed)
```bash
# NEW PATTERN (proposed for new-milestone.md or discuss-milestone.md)
# After: Milestone initialized in PROJECT.md
# Before: User starts planning phases

BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "none")

if [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  # Extract milestone version from PROJECT.md
  MILESTONE_VERSION=$(grep "^## Current Milestone:" .planning/PROJECT.md | sed 's/.*: v/v/' | sed 's/ .*//')

  # Extract milestone goal/name
  MILESTONE_GOAL=$(grep "^## Current Milestone:" .planning/PROJECT.md | sed 's/.*v[0-9.]* //' | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//' | cut -c1-40)

  BRANCH_NAME="gsd/${MILESTONE_VERSION}-${MILESTONE_GOAL}"

  # Check if branch exists
  if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Switching to existing milestone branch: $BRANCH_NAME"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    git switch "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
  else
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Creating milestone branch: $BRANCH_NAME"
    echo "All phases in this milestone will be committed to this branch"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    git switch -c "$BRANCH_NAME" 2>/dev/null || git checkout -b "$BRANCH_NAME"
  fi
fi
```

### Settings UI Question (Proposed)
```markdown
# NEW PATTERN (proposed for commands/gsd/settings.md)
# Add as question 5 (after Model Profile, before workflow toggles)

{
  question: "Git branching strategy?",
  header: "Git Branching",
  multiSelect: false,
  options: [
    { label: "None (Recommended)", description: "Commit directly to current branch" },
    { label: "Per Phase", description: "Create branch for each phase (gsd/phase-{N}-{name})" },
    { label: "Per Milestone", description: "Create branch for entire milestone (gsd/{version}-{name})" }
  ]
}
```

### Response Mapping (Proposed)
```javascript
// NEW PATTERN (proposed for settings.md response parsing)
// Map user responses to config values

// Branching strategy mapping
let branchingStrategy = "none";
if (responses[1].includes("Per Phase")) {
  branchingStrategy = "phase";
} else if (responses[1].includes("Per Milestone")) {
  branchingStrategy = "milestone";
}

// Config update
{
  ...existing_config,
  "git": {
    ...existing_git_config,
    "branching_strategy": branchingStrategy  // "none" | "phase" | "milestone"
  }
}
```

### Validation Schema Extension (Proposed)
```javascript
// NEW PATTERN (proposed for lib/validate-config.md)
// Add to Git Section table

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| auto_commit | boolean | true | Auto-commit after each task (if false, stages changes only) |
| branching_strategy | string | "none" | Branching approach: "none" (current branch), "phase" (gsd/phase-N-name), "milestone" (gsd/vX.Y-name) |

// Validation function
if (config.git !== undefined) {
  if (checkType(config.git, 'object', 'git')) {
    checkType(config.git.auto_commit, 'boolean', 'git.auto_commit');

    // Validate branching_strategy enum
    if (config.git.branching_strategy !== undefined) {
      if (checkType(config.git.branching_strategy, 'string', 'git.branching_strategy')) {
        const validStrategies = ['none', 'phase', 'milestone'];
        if (!validStrategies.includes(config.git.branching_strategy)) {
          errors.push(`git.branching_strategy must be one of: ${validStrategies.join(', ')} (got: ${config.git.branching_strategy})`);
        }
      }
    }
  }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual branch creation | Automated via config | Phase 3 (2026-02-02) | Users specify strategy once, GSD handles branching |
| All work on main | Per-phase or per-milestone branches | Phase 3 (2026-02-02) | Better isolation, easier PR creation |
| Boolean config only | Enum config values | Phase 3 (2026-02-02) | Richer configuration options (3+ choices) |
| git checkout -b | git switch -c (with fallback) | Git 2.23+ | Clearer intent, safer operations |

**Current best practices (2026):**
- **Trunk-based development** dominates for teams <10 with CI/CD (source: WebSearch - Java Code Geeks)
- **Feature branch workflow** still common for phase-like work isolation (source: WebSearch - Atlassian)
- **Branch naming conventions:** Use prefixes (feature/, hotfix/, release/) + lowercase + hyphens (source: WebSearch - Graphite)
- **git switch over git checkout:** Modern Git (2.23+) prefers switch for branch ops (source: WebSearch - Refine)

**Deprecated/outdated:**
- Gitflow - "legacy strategy" falling out of favor for trunk-based (source: WebSearch - Atlassian)
- git checkout -b - Still works but git switch -c is clearer (source: WebSearch - Git docs)

## Open Questions

Things that couldn't be fully resolved:

1. **Where does squash merge implementation exist upstream?**
   - What we know: Changelog v1.11.1 mentions "squash merge option at milestone completion"
   - What's unclear: complete-milestone.md files are identical local/upstream, feature may not be implemented
   - Recommendation: Create requirement GITB-05 with caveat "if upstream implementation found" - investigate during planning, document if not found

2. **Should milestone branch be created in new-milestone or first execute-phase?**
   - What we know: Phase branch creation happens at execute-phase start
   - What's unclear: Milestone branches span multiple phases, when is the right time to create?
   - Recommendation: Create at new-milestone time (user explicitly starting milestone), warn at execute-phase if strategy is "milestone" but not on gsd/ branch

3. **What happens to branch when user changes strategy mid-milestone?**
   - What we know: Config can be changed anytime via /gsd:settings
   - What's unclear: If user is on gsd/phase-2-foo and changes to "none", should we switch branches?
   - Recommendation: Don't auto-switch branches. Display warning: "Strategy changed, but still on branch X. Switch manually if needed."

4. **Should phase branches be deleted after phase completion?**
   - What we know: GitHub/GitLab auto-delete merged branches, but this is local workflow
   - What's unclear: Do we want accumulation of phase branches, or clean up after merge?
   - Recommendation: Keep branches (user can manually delete). Auto-deletion risky without upstream tracking.

5. **How to handle git.branching_strategy with decimal phases (2.1, 2.2)?**
   - What we know: Roadmap supports decimal phases for urgent insertions
   - What's unclear: Branch name for phase 2.1? gsd/phase-2-1-name or gsd/phase-2.1-name?
   - Recommendation: Use gsd/phase-2-1-name (replace . with -) for consistency with slug rules

## Sources

### Primary (HIGH confidence)
- GSD Codebase: `commands/gsd/add-phase.md`, `insert-phase.md`, `quick.md` - Slug generation pattern (tr/sed)
- GSD Codebase: `commands/gsd/execute-phase.md` - Step structure, config parsing pattern
- GSD Codebase: `commands/gsd/settings.md` - AskUserQuestion pattern, enum question examples
- GSD Codebase: `get-shit-done/lib/validate-config.md` - Schema validation, enum validation approach
- Phase 2 Research: `.planning/phases/02-user-autonomy/02-RESEARCH.md` - Config toggle patterns
- Phase 2 Summary: `.planning/phases/02-user-autonomy/02-01-SUMMARY.md` - 3-step config extension pattern
- Upstream Diff: `.planning/research/02-file-diff-comparison.md` - Git branching config exists in settings.md upstream
- Portability Assessment: `.planning/research/03-portability-assessment.md` - Unknown implementation location finding

### Secondary (MEDIUM confidence)
- Git Documentation: Official git-checkout, git-switch docs (via WebSearch)
- Branching Conventions: Multiple sources (Graphite, Medium, Phoenix NAP, DEV Community) on branch naming best practices 2026
- Git Workflow Strategies: Atlassian, DataCamp, Business Compass sources on feature branches, trunk-based, GitFlow
- Slug Generation: Yoast, DEV Community, Codecademy on URL slug best practices

### Tertiary (LOW confidence - needs verification)
- Squash merge implementation upstream - Changelog claims it exists but files are identical, may be false positive
- Milestone branch auto-cleanup - No evidence of implementation, user may need manual branch management

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All bash/git/config patterns verified in existing codebase
- Architecture: MEDIUM-HIGH - Patterns established but insertion points (execute-phase step 0.5, new-milestone) need validation
- Pitfalls: MEDIUM-HIGH - Derived from git best practices and GSD patterns, but branch workflow pitfalls are from general knowledge
- Squash merge: LOW - Uncertain if feature exists upstream

**Research date:** 2026-02-02
**Valid until:** 2026-03-02 (30 days - stable domain, git and GSD patterns unlikely to change rapidly)

**Key gaps requiring planning investigation:**
1. Confirm execute-phase.md insertion point (step 0.5) is correct location
2. Decide whether new-milestone should create branch or just document requirement
3. Investigate squash merge implementation (may not exist despite changelog claim)
4. Test slug generation with actual phase names to verify length/readability
5. Verify milestone version extraction from PROJECT.md works correctly

**Sources requiring citation in final output:**
- [Agile Git Branching Strategies 2026](https://www.javacodegeeks.com/2025/11/agile-git-branching-strategies-in-2026.html)
- [Git Branch Naming Conventions Best Practices](https://graphite.com/guides/git-branch-naming-conventions)
- [Git Workflow Automation](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow)
- [Slug Generation Best Practices](https://dev.to/bybydev/how-to-slugify-a-string-in-javascript-4o9n)
- [Git Merge Squash Best Practices](https://www.lloydatkinson.net/posts/2022/should-you-squash-merge-or-merge-commit/)
- [Git Branch Cleanup Strategies](https://graphite.com/guides/git-delete-local-branch-been-merged)
