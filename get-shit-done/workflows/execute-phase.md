<purpose>
Execute all plans in a phase using wave-based parallel execution. Orchestrator stays lean by delegating plan execution to subagents.
</purpose>

<core_principle>
The orchestrator's job is coordination, not execution. Each subagent loads the full execute-plan context itself. Orchestrator discovers plans, analyzes dependencies, groups into waves, spawns agents, handles checkpoints, collects results.
</core_principle>

<required_reading>
Read STATE.md before any operation to load project context.
Read config.json for planning behavior settings.
</required_reading>

<executive_summary>
## Quick Reference

Execute all plans in a phase via wave-based parallelization with optional agentic features (failure prediction, adaptive retry, regression checking).

## Steps Overview

| # | Step | One-line Description |
|---|------|---------------------|
| 1 | resolve_model_profile | Load model profile from config for agent spawning |
| 2 | load_project_state | Read STATE.md and config.json for context and settings |
| 3 | validate_phase | Confirm phase exists and has plans |
| 4 | capture_baseline | Run tests/build to establish regression baseline (if verification enabled) |
| 5 | load_predictive_hints | Import patterns from knowledge base for proactive guidance (if adaptive enabled) |
| 6 | predict_failures | Analyze phase for potential failure points before execution (if adaptive enabled) |
| 7 | discover_plans | List all plans and extract metadata (wave, autonomous, gap_closure) |
| 8 | group_by_wave | Group plans by pre-computed wave numbers |
| 9 | execute_waves | Execute each wave in sequence with parallel autonomous plans |
| 10 | checkpoint_handling | Handle plans with checkpoints requiring user interaction |
| 11 | aggregate_results | Compile results from all waves |
| 12 | regression_check | Verify tests still pass after all changes (if verification enabled) |
| 13 | verify_phase_goal | Spawn verifier to check phase achieved its GOAL |
| 14 | update_roadmap | Mark phase complete in ROADMAP.md |
| 15 | offer_next | Present next steps (next phase or milestone complete) |

## Configuration Impact

```
config.json field          → Affected steps
─────────────────────────────────────────────
mode: "yolo"               → execute_waves, checkpoint_handling (auto-approve)
mode: "interactive"        → execute_waves, checkpoint_handling (pause for confirmation)
verification.enabled       → 4, 12 (baseline capture, regression checks)
adaptive.enabled           → 5, 6 (pattern hints, failure prediction)
retry.enabled              → 9 (use retry-orchestration on task failure)
retry.max_attempts         → 9 (how many retries before escalation)
```

## When to Use

This workflow is invoked by `/gsd:execute-phase [phase]` to execute all plans in a phase. Use `/gsd:execute-plan` for single plan execution.

## Related Files

- `@~/.claude/get-shit-done/workflows/execute-plan.md` — Single plan executor
- `@~/.claude/get-shit-done/workflows/predict-failures.md` — Failure prediction (if adaptive enabled)
- `@~/.claude/get-shit-done/workflows/import-patterns.md` — Pattern hints (if adaptive enabled)
- `@~/.claude/get-shit-done/workflows/retry-orchestration.md` — Retry logic (if retry enabled)
- `@~/.claude/get-shit-done/lib/regression-detection.md` — Regression detection (if verification enabled)
- `@~/.claude/get-shit-done/templates/summary.md` — SUMMARY.md template structure
- `@~/.claude/get-shit-done/references/checkpoints.md` — Checkpoint execution protocol
</executive_summary>

<process>

<step name="resolve_model_profile" priority="first">
Read model profile for agent spawning:

```bash
MODEL_PROFILE=$(cat .planning/config.json 2>/dev/null | grep -o '"model_profile"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "balanced")
```

Default to "balanced" if not set.

**Model lookup table:**

| Agent | quality | balanced | budget |
|-------|---------|----------|--------|
| gsd-executor | opus | sonnet | sonnet |
| gsd-verifier | sonnet | sonnet | haiku |
| general-purpose | — | — | — |

Store resolved models for use in Task calls below.
</step>

<step name="load_project_state">
Before any operation, read project state:

```bash
cat .planning/STATE.md 2>/dev/null
```

**If file exists:** Parse and internalize:
- Current position (phase, plan, status)
- Accumulated decisions (constraints on this execution)
- Blockers/concerns (things to watch for)

**If file missing but .planning/ exists:**
```
STATE.md missing but planning artifacts exist.
Options:
1. Reconstruct from existing artifacts
2. Continue without project state (may lose accumulated context)
```

**If .planning/ doesn't exist:** Error - project not initialized.

**Load planning config:**

```bash
# Check if planning docs should be committed (default: true)
COMMIT_PLANNING_DOCS=$(cat .planning/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")
# Auto-detect gitignored (overrides config)
git check-ignore -q .planning 2>/dev/null && COMMIT_PLANNING_DOCS=false
```

Store `COMMIT_PLANNING_DOCS` for use in git operations.

**Parse agentic feature flags:**

```bash
# Read config.json for agentic features
CONFIG=$(cat .planning/config.json 2>/dev/null)

# Mode (yolo/interactive)
MODE=$(echo "$CONFIG" | grep -o '"mode"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "interactive")

# Verification enabled (default: false for backward compatibility)
VERIFICATION_ENABLED=$(echo "$CONFIG" | grep -A 4 '"verification"' | grep -o '"enabled"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "false")

# Adaptive enabled (default: false)
ADAPTIVE_ENABLED=$(echo "$CONFIG" | grep -A 4 '"adaptive"' | grep -o '"enabled"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "false")

# Retry enabled (default: false)
RETRY_ENABLED=$(echo "$CONFIG" | grep -A 4 '"retry"' | grep -o '"enabled"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "false")
RETRY_MAX_ATTEMPTS=$(echo "$CONFIG" | grep -A 4 '"retry"' | grep -o '"max_attempts"[[:space:]]*:[[:space:]]*[0-9]*' | grep -o '[0-9]*' || echo "3")
```

Store these flags for conditional steps below.
</step>

<step name="validate_phase">
Confirm phase exists and has plans:

```bash
# Match both zero-padded (05-*) and unpadded (5-*) folders
PADDED_PHASE=$(printf "%02d" ${PHASE_ARG} 2>/dev/null || echo "${PHASE_ARG}")
PHASE_DIR=$(ls -d .planning/phases/${PADDED_PHASE}-* .planning/phases/${PHASE_ARG}-* 2>/dev/null | head -1)
if [ -z "$PHASE_DIR" ]; then
  echo "ERROR: No phase directory matching '${PHASE_ARG}'"
  exit 1
fi

PLAN_COUNT=$(ls -1 "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l | tr -d ' ')
if [ "$PLAN_COUNT" -eq 0 ]; then
  echo "ERROR: No plans found in $PHASE_DIR"
  exit 1
fi
```

Report: "Found {N} plans in {phase_dir}"
</step>

<step name="capture_baseline">
**Capture baseline state for regression detection (if verification enabled)**

**Skip if:** `VERIFICATION_ENABLED = false` (default)

**If verification enabled:**

Load verification context:
- `@~/.claude/get-shit-done/lib/regression-detection.md` - Regression detection logic
- `@~/.claude/get-shit-done/lib/rollback-strategy.md` - Rollback patterns

**Capture test baseline:**

```bash
# Run tests and capture baseline state
BASELINE_TESTS_CMD=$(cat .planning/config.json | grep -o '"baseline_tests": *"[^"]*"' | cut -d'"' -f4)
if [ -z "$BASELINE_TESTS_CMD" ]; then
  BASELINE_TESTS_CMD="npm test"
fi

# Capture baseline (suppress if already exists and recent)
if [ ! -f .planning/.baseline-tests.json ] || [ $(find .planning/.baseline-tests.json -mmin +60 2>/dev/null | wc -l) -gt 0 ]; then
  echo "Capturing test baseline..."
  $BASELINE_TESTS_CMD --json > .planning/.baseline-tests.json 2>/dev/null || \
  $BASELINE_TESTS_CMD > .planning/.baseline-tests-stdout.log 2>&1
fi
```

**Capture build baseline:**

```bash
# Capture build baseline if build command configured
BASELINE_BUILD_CMD=$(cat .planning/config.json | grep -o '"baseline_build": *"[^"]*"' | cut -d'"' -f4)
if [ -n "$BASELINE_BUILD_CMD" ]; then
  if [ ! -f .planning/.baseline-build.json ] || [ $(find .planning/.baseline-build.json -mmin +60 2>/dev/null | wc -l) -gt 0 ]; then
    echo "Capturing build baseline..."
    $BASELINE_BUILD_CMD 2>&1 | tee .planning/.baseline-build.log
    echo "{\"timestamp\":\"$(date -u +"%Y-%m-%dT%H:%M:%SZ")\",\"status\":\"success\"}" > .planning/.baseline-build.json
  fi
fi
```

**Initialize health status:**

```bash
# Create or update HEALTH_STATUS.md with baseline metrics
if [ ! -f .planning/HEALTH_STATUS.md ]; then
  cat > .planning/HEALTH_STATUS.md <<'EOF'
# Project Health Status

## Current Status
**Overall:** HEALTHY
**Last Verified:** $(date -u +"%Y-%m-%dT%H:%M:%SZ")

## Baseline Metrics
- Test baseline captured: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
- Build baseline captured: $(date -u +"%Y-%m-%dT%H:%M:%SZ")

## Recent Checks
No checks yet.
EOF
fi
```

Report: "Baseline captured for regression detection"
</step>

<step name="load_predictive_hints">
**Load predictive hints from knowledge base (if adaptive enabled)**

**Skip if:** `ADAPTIVE_ENABLED = false` (default)

**If adaptive enabled:**

Query the knowledge base for patterns relevant to this phase's task types.

**Reference:** `@~/.claude/get-shit-done/workflows/import-patterns.md`

```bash
# Check if knowledge base exists
if [ -d ~/.claude/gsd-knowledge ] && [ -f ~/.claude/gsd-knowledge/index.json ]; then
  echo "Knowledge base found - loading predictive hints..."

  # Parse phase plans to identify task types
  TASK_TYPES=""
  for plan in "$PHASE_DIR"/*-PLAN.md; do
    if grep -qiE "(bash|npm|yarn|git|command|run|install|execute)" "$plan"; then
      TASK_TYPES="$TASK_TYPES bash-command"
    fi
    if grep -qiE "(test|verify|check|validate|spec)" "$plan"; then
      TASK_TYPES="$TASK_TYPES test-run"
    fi
    if grep -qiE "(build|compile|bundle|transpile)" "$plan"; then
      TASK_TYPES="$TASK_TYPES build"
    fi
    if grep -qiE "(deploy|publish|release)" "$plan"; then
      TASK_TYPES="$TASK_TYPES deploy"
    fi
  done

  # Query matching patterns (import-patterns workflow)
  # Patterns stored in: ~/.claude/gsd-knowledge/patterns/{type}/
else
  echo "Knowledge base not found - skipping pattern hints"
fi
```

**Display hints if patterns found:**

```
===============================================
PREDICTIVE HINTS (from cross-project learning)
===============================================

**For test-run tasks:**
- Strategy "retry-with-reruns" has 85% success rate (12 uses across 4 projects)
- Common failure: validation errors - typically recovers with test isolation

**For bash-command tasks:**
- Watch for: dependency failures (reinstall usually works)
- Best practice: Always verify command exists before complex pipelines

===============================================
```

**Non-blocking guarantee:**
Pattern hint retrieval is best-effort. Failures in this step never block execution - log warning and continue.
</step>

<step name="predict_failures">
**Predict potential failure points before execution (if adaptive enabled)**

**Skip if:** `ADAPTIVE_ENABLED = false` (default)

**If adaptive enabled:**

Analyze the phase plans to predict potential failures proactively.

**Reference:** `@~/.claude/get-shit-done/workflows/predict-failures.md`

**Analysis inputs:**
- All PLAN.md files in this phase
- Task types and dependencies
- Historical failure patterns from knowledge base
- Project-specific context from STATE.md

**Output:**

```
===============================================
FAILURE PREDICTION ANALYSIS
===============================================

**High Risk Tasks:**
- Plan 01-02, Task 3: External API integration
  Risk: Rate limiting, auth failures
  Mitigation: Add retry logic, cache responses

- Plan 01-03, Task 1: Database migration
  Risk: Schema conflicts, data loss
  Mitigation: Run on test data first, backup before migration

**Medium Risk Tasks:**
- Plan 01-01, Task 2: NPM install
  Risk: Dependency resolution
  Mitigation: Clear node_modules if needed

**Recommendations:**
1. Execute Plan 01-02 with extra verification pauses
2. Consider manual checkpoint before migration tasks

===============================================
```

**Non-blocking guarantee:**
Failure prediction is advisory only. Even if prediction finds high-risk items, execution continues. User can abort manually if concerned.
</step>

<step name="discover_plans">
List all plans and extract metadata:

```bash
# Get all plans
ls -1 "$PHASE_DIR"/*-PLAN.md 2>/dev/null | sort

# Get completed plans (have SUMMARY.md)
ls -1 "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null | sort
```

For each plan, read frontmatter to extract:
- `wave: N` - Execution wave (pre-computed)
- `autonomous: true/false` - Whether plan has checkpoints
- `gap_closure: true/false` - Whether plan closes gaps from verification/UAT

Build plan inventory:
- Plan path
- Plan ID (e.g., "03-01")
- Wave number
- Autonomous flag
- Gap closure flag
- Completion status (SUMMARY exists = complete)

**Filtering:**
- Skip completed plans (have SUMMARY.md)
- If `--gaps-only` flag: also skip plans where `gap_closure` is not `true`

If all plans filtered out, report "No matching incomplete plans" and exit.
</step>

<step name="group_by_wave">
Read `wave` from each plan's frontmatter and group by wave number:

```bash
# For each plan, extract wave from frontmatter
for plan in $PHASE_DIR/*-PLAN.md; do
  wave=$(grep "^wave:" "$plan" | cut -d: -f2 | tr -d ' ')
  autonomous=$(grep "^autonomous:" "$plan" | cut -d: -f2 | tr -d ' ')
  echo "$plan:$wave:$autonomous"
done
```

**Group plans:**
```
waves = {
  1: [plan-01, plan-02],
  2: [plan-03, plan-04],
  3: [plan-05]
}
```

**No dependency analysis needed.** Wave numbers are pre-computed during `/gsd:plan-phase`.

Report wave structure with context:
```
## Execution Plan

**Phase {X}: {Name}** — {total_plans} plans across {wave_count} waves

| Wave | Plans | What it builds |
|------|-------|----------------|
| 1 | 01-01, 01-02 | {from plan objectives} |
| 2 | 01-03 | {from plan objectives} |
| 3 | 01-04 [checkpoint] | {from plan objectives} |

```

The "What it builds" column comes from skimming plan names/objectives. Keep it brief (3-8 words).
</step>

<step name="execute_waves">
Execute each wave in sequence. Autonomous plans within a wave run in parallel.

**For each wave:**

1. **Describe what's being built (BEFORE spawning):**

   Read each plan's `<objective>` section. Extract what's being built and why it matters.

   **Output:**
   ```
   ---

   ## Wave {N}

   **{Plan ID}: {Plan Name}**
   {2-3 sentences: what this builds, key technical approach, why it matters in context}

   **{Plan ID}: {Plan Name}** (if parallel)
   {same format}

   Spawning {count} agent(s)...

   ---
   ```

   **Examples:**
   - Bad: "Executing terrain generation plan"
   - Good: "Procedural terrain generator using Perlin noise — creates height maps, biome zones, and collision meshes. Required before vehicle physics can interact with ground."

2. **Read files and spawn all autonomous agents in wave simultaneously:**

   Before spawning, read file contents. The `@` syntax does not work across Task() boundaries - content must be inlined.

   ```bash
   # Read each plan in the wave
   PLAN_CONTENT=$(cat "{plan_path}")
   STATE_CONTENT=$(cat .planning/STATE.md)
   CONFIG_CONTENT=$(cat .planning/config.json 2>/dev/null)
   ```

   Use Task tool with multiple parallel calls. Each agent gets prompt with inlined content:

   ```
   <objective>
   Execute plan {plan_number} of phase {phase_number}-{phase_name}.

   Commit each task atomically. Create SUMMARY.md. Update STATE.md.
   </objective>

   <execution_context>
   @/Users/macuser/.claude/get-shit-done/workflows/execute-plan.md
   @/Users/macuser/.claude/get-shit-done/templates/summary.md
   @/Users/macuser/.claude/get-shit-done/references/checkpoints.md
   @/Users/macuser/.claude/get-shit-done/references/tdd.md
   </execution_context>

   <agentic_context>
   <!-- Include if retry.enabled = true -->
   @/Users/macuser/.claude/get-shit-done/workflows/retry-orchestration.md

   <!-- Include if verification.enabled = true -->
   @/Users/macuser/.claude/get-shit-done/lib/regression-detection.md
   </agentic_context>

   <context>
   Plan:
   {plan_content}

   Project state:
   {state_content}

   Config (if exists):
   {config_content}
   </context>

   <success_criteria>
   - [ ] All tasks executed
   - [ ] Each task committed individually
   - [ ] SUMMARY.md created in plan directory
   - [ ] STATE.md updated with position and decisions
   </success_criteria>
   ```

2. **Wait for all agents in wave to complete:**

   Task tool blocks until each agent finishes. All parallel agents return together.

3. **Report completion and what was built:**

   For each completed agent:
   - Verify SUMMARY.md exists at expected path
   - Read SUMMARY.md to extract what was built
   - Note any issues or deviations

   **Output:**
   ```
   ---

   ## Wave {N} Complete

   **{Plan ID}: {Plan Name}**
   {What was built — from SUMMARY.md deliverables}
   {Notable deviations or discoveries, if any}

   **{Plan ID}: {Plan Name}** (if parallel)
   {same format}

   {If more waves: brief note on what this enables for next wave}

   ---
   ```

   **Examples:**
   - Bad: "Wave 2 complete. Proceeding to Wave 3."
   - Good: "Terrain system complete — 3 biome types, height-based texturing, physics collision meshes. Vehicle physics (Wave 3) can now reference ground surfaces."

4. **Handle failures:**

   If any agent in wave fails:

   **If retry.enabled = true:**
   - Load retry context: `@~/.claude/get-shit-done/workflows/retry-orchestration.md`
   - Analyze failure using failure-analysis logic
   - If retryable: Select alternative strategy, respawn agent with modified approach
   - Track attempts in `.planning/ATTEMPTS_LOG.md` if `retry.log_attempts = true`
   - Max attempts: `RETRY_MAX_ATTEMPTS` (default: 3)
   - If max attempts reached: Escalate to user

   **If retry.enabled = false (default):**
   - Report which plan failed and why
   - Ask user: "Continue with remaining waves?" or "Stop execution?"
   - If continue: proceed to next wave (dependent plans may also fail)
   - If stop: exit with partial completion report

   **Mode-specific behavior:**

   <if mode="yolo">
   Auto-continue with remaining waves on failure. Log failures but don't pause.
   ```
   ⚡ Plan {X} failed - auto-continuing (yolo mode)
   Reason: {failure_reason}
   Remaining waves will execute.
   ```
   </if>

   <if mode="interactive">
   Pause and ask user for decision on failure.
   </if>

5. **Execute checkpoint plans between waves:**

   See `<checkpoint_handling>` for details.

6. **Proceed to next wave**

</step>

<step name="checkpoint_handling">
Plans with `autonomous: false` require user interaction.

**Detection:** Check `autonomous` field in frontmatter.

**Execution flow for checkpoint plans:**

1. **Spawn agent for checkpoint plan:**
   ```
   Task(prompt="{subagent-task-prompt}", subagent_type="gsd-executor", model="{executor_model}")
   ```

2. **Agent runs until checkpoint:**
   - Executes auto tasks normally
   - Reaches checkpoint task (e.g., `type="checkpoint:human-verify"`) or auth gate
   - Agent returns with structured checkpoint (see checkpoint-return.md template)

3. **Agent return includes (structured format):**
   - Completed Tasks table with commit hashes and files
   - Current task name and blocker
   - Checkpoint type and details for user
   - What's awaited from user

4. **Orchestrator presents checkpoint to user:**

   Extract and display the "Checkpoint Details" and "Awaiting" sections from agent return:
   ```
   ## Checkpoint: [Type]

   **Plan:** 03-03 Dashboard Layout
   **Progress:** 2/3 tasks complete

   [Checkpoint Details section from agent return]

   [Awaiting section from agent return]
   ```

   **Mode-specific behavior:**

   <if mode="yolo">
   Auto-approve checkpoint:human-verify types. Still pause for checkpoint:decision and checkpoint:human-action.
   ```
   ⚡ Auto-approved: {checkpoint description}
   [Continuing execution...]
   ```
   </if>

   <if mode="interactive">
   Always pause for user response on checkpoints.
   </if>

5. **User responds:**
   - "approved" / "done" → spawn continuation agent
   - Description of issues → spawn continuation agent with feedback
   - Decision selection → spawn continuation agent with choice

6. **Spawn continuation agent (NOT resume):**

   Use the continuation-prompt.md template:
   ```
   Task(
     prompt=filled_continuation_template,
     subagent_type="gsd-executor",
     model="{executor_model}"
   )
   ```

   Fill template with:
   - `{completed_tasks_table}`: From agent's checkpoint return
   - `{resume_task_number}`: Current task from checkpoint
   - `{resume_task_name}`: Current task name from checkpoint
   - `{user_response}`: What user provided
   - `{resume_instructions}`: Based on checkpoint type (see continuation-prompt.md)

7. **Continuation agent executes:**
   - Verifies previous commits exist
   - Continues from resume point
   - May hit another checkpoint (repeat from step 4)
   - Or completes plan

8. **Repeat until plan completes or user stops**

**Why fresh agent instead of resume:**
Resume relies on Claude Code's internal serialization which breaks with parallel tool calls.
Fresh agents with explicit state are more reliable and maintain full context.

**Checkpoint in parallel context:**
If a plan in a parallel wave has a checkpoint:
- Spawn as normal
- Agent pauses at checkpoint and returns with structured state
- Other parallel agents may complete while waiting
- Present checkpoint to user
- Spawn continuation agent with user response
- Wait for all agents to finish before next wave
</step>

<step name="aggregate_results">
After all waves complete, aggregate results:

```markdown
## Phase {X}: {Name} Execution Complete

**Waves executed:** {N}
**Plans completed:** {M} of {total}

### Wave Summary

| Wave | Plans | Status |
|------|-------|--------|
| 1 | plan-01, plan-02 | ✓ Complete |
| CP | plan-03 | ✓ Verified |
| 2 | plan-04 | ✓ Complete |
| 3 | plan-05 | ✓ Complete |

### Plan Details

1. **03-01**: [one-liner from SUMMARY.md]
2. **03-02**: [one-liner from SUMMARY.md]
...

### Issues Encountered
[Aggregate from all SUMMARYs, or "None"]
```
</step>

<step name="regression_check">
**Verify tests still pass after all changes (if verification enabled)**

**Skip if:** `VERIFICATION_ENABLED = false` (default)

**If verification enabled:**

After all plans complete, run regression detection to ensure no tests were broken.

**Reference:** `@~/.claude/get-shit-done/lib/regression-detection.md`

```bash
# Run current tests
BASELINE_TESTS_CMD=$(cat .planning/config.json | grep -o '"baseline_tests": *"[^"]*"' | cut -d'"' -f4)
if [ -z "$BASELINE_TESTS_CMD" ]; then
  BASELINE_TESTS_CMD="npm test"
fi

$BASELINE_TESTS_CMD --json > .planning/.current-tests.json 2>/dev/null || \
$BASELINE_TESTS_CMD > .planning/.current-tests-stdout.log 2>&1
```

**Compare against baseline:**
- Load baseline: `.planning/.baseline-tests.json`
- Load current: `.planning/.current-tests.json`
- Identify tests that were PASSING in baseline but FAILING in current
- Filter flaky tests (retry failing tests individually)
- Confirm regressions

**If regression detected:**

```
⚠️ REGRESSION DETECTED

Tests that were passing before phase execution are now failing:
- test_user_login (auth.test.js)
- test_session_refresh (session.test.js)

**Options:**
1. Investigate - Review changes and fix regressions
2. Accept - Update baseline (tests intentionally changed)
3. Rollback - Revert phase changes
```

**If verification.auto_rollback = true AND retry.enabled = true:**
- Attempt auto-correction using `@~/.claude/get-shit-done/workflows/auto-correction.md`
- Rollback problematic commits using rollback-strategy.md
- Log to REGRESSION_LOG.md
- Analyze failure and retry with alternative approach

**If no regression detected:**
- Update HEALTH_STATUS.md: "Phase {X} verified clean: [timestamp]"
- Continue to verify_phase_goal
</step>

<step name="verify_phase_goal">
Verify phase achieved its GOAL, not just completed its TASKS.

**Spawn verifier:**

```
Task(
  prompt="Verify phase {phase_number} goal achievement.

Phase directory: {phase_dir}
Phase goal: {goal from ROADMAP.md}

Check must_haves against actual codebase. Create VERIFICATION.md.
Verify what actually exists in the code.",
  subagent_type="gsd-verifier",
  model="{verifier_model}"
)
```

**Read verification status:**

```bash
grep "^status:" "$PHASE_DIR"/*-VERIFICATION.md | cut -d: -f2 | tr -d ' '
```

**Route by status:**

| Status | Action |
|--------|--------|
| `passed` | Continue to update_roadmap |
| `human_needed` | Present items to user, get approval or feedback |
| `gaps_found` | Present gap summary, offer `/gsd:plan-phase {phase} --gaps` |

**If passed:**

Phase goal verified. Proceed to update_roadmap.

**If human_needed:**

```markdown
## ✓ Phase {X}: {Name} — Human Verification Required

All automated checks passed. {N} items need human testing:

### Human Verification Checklist

{Extract from VERIFICATION.md human_verification section}

---

**After testing:**
- "approved" → continue to update_roadmap
- Report issues → will route to gap closure planning
```

If user approves → continue to update_roadmap.
If user reports issues → treat as gaps_found.

**If gaps_found:**

Present gaps and offer next command:

```markdown
## ⚠ Phase {X}: {Name} — Gaps Found

**Score:** {N}/{M} must-haves verified
**Report:** {phase_dir}/{phase}-VERIFICATION.md

### What's Missing

{Extract gap summaries from VERIFICATION.md gaps section}

---

## ▶ Next Up

**Plan gap closure** — create additional plans to complete the phase

`/gsd:plan-phase {X} --gaps`

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- `cat {phase_dir}/{phase}-VERIFICATION.md` — see full report
- `/gsd:verify-work {X}` — manual testing before planning
```

User runs `/gsd:plan-phase {X} --gaps` which:
1. Reads VERIFICATION.md gaps
2. Creates additional plans (04, 05, etc.) with `gap_closure: true` to close gaps
3. User then runs `/gsd:execute-phase {X} --gaps-only`
4. Execute-phase runs only gap closure plans (04-05)
5. Verifier runs again after new plans complete

User stays in control at each decision point.
</step>

<step name="update_roadmap">
Update ROADMAP.md to reflect phase completion:

```bash
# Mark phase complete
# Update completion date
# Update status
```

**Check planning config:**

If `COMMIT_PLANNING_DOCS=false` (set in load_project_state):
- Skip all git operations for .planning/ files
- Planning docs exist locally but are gitignored
- Log: "Skipping planning docs commit (commit_docs: false)"
- Proceed to offer_next step

If `COMMIT_PLANNING_DOCS=true` (default):
- Continue with git operations below

Commit phase completion (roadmap, state, verification):
```bash
git add .planning/ROADMAP.md .planning/STATE.md .planning/phases/{phase_dir}/*-VERIFICATION.md
git add .planning/REQUIREMENTS.md  # if updated
git commit -m "docs(phase-{X}): complete phase execution"
```
</step>

<step name="offer_next">
Present next steps based on milestone status:

**If more phases remain:**
```
## Next Up

**Phase {X+1}: {Name}** — {Goal}

`/gsd:plan-phase {X+1}`

<sub>`/clear` first for fresh context</sub>
```

**If milestone complete:**
```
MILESTONE COMPLETE!

All {N} phases executed.

`/gsd:complete-milestone`
```
</step>

</process>

<context_efficiency>
Orchestrator: ~10-15% context (frontmatter, spawning, results).
Subagents: Fresh 200k each (full workflow + execution).
No polling (Task blocks). No context bleed.
</context_efficiency>

<failure_handling>
**Subagent fails mid-plan:**
- SUMMARY.md won't exist
- Orchestrator detects missing SUMMARY
- Reports failure, asks user how to proceed
- If retry.enabled: Attempt retry with alternative strategy

**Dependency chain breaks:**
- Wave 1 plan fails
- Wave 2 plans depending on it will likely fail
- Orchestrator can still attempt them (user choice)
- Or skip dependent plans entirely

**All agents in wave fail:**
- Something systemic (git issues, permissions, etc.)
- Stop execution
- Report for manual investigation

**Checkpoint fails to resolve:**
- User can't approve or provides repeated issues
- Ask: "Skip this plan?" or "Abort phase execution?"
- Record partial progress in STATE.md

**Regression detected (if verification enabled):**
- Tests failing that previously passed
- If auto_rollback enabled: Attempt automatic fix
- Otherwise: Present options to user (investigate, accept, rollback)
</failure_handling>

<resumption>
**Resuming interrupted execution:**

If phase execution was interrupted (context limit, user exit, error):

1. Run `/gsd:execute-phase {phase}` again
2. discover_plans finds completed SUMMARYs
3. Skips completed plans
4. Resumes from first incomplete plan
5. Continues wave-based execution

**STATE.md tracks:**
- Last completed plan
- Current wave
- Any pending checkpoints
</resumption>

<config_reference>
## Configuration Options

All agentic features are opt-in via `.planning/config.json`:

```json
{
  "mode": "interactive",  // or "yolo" for auto-approve

  "verification": {
    "enabled": false,          // Enable regression detection
    "baseline_tests": "npm test",
    "baseline_build": "npm run build",
    "auto_rollback": false,    // Auto-rollback on regression
    "regression_check": true   // Check after each task
  },

  "adaptive": {
    "enabled": false,          // Enable knowledge base features
    "knowledge_base_enabled": true,
    "show_pattern_hints": true,
    "min_pattern_confidence": 0.6
  },

  "retry": {
    "enabled": false,          // Enable adaptive retry
    "max_attempts": 3,
    "log_attempts": true
  }
}
```

**Backward compatibility:** All features default to off. Existing projects work unchanged.
</config_reference>
