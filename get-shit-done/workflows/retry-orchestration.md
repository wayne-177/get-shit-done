# Retry Orchestration Workflow

This workflow implements automatic retry logic with alternative approaches for task failure recovery. It integrates with the failure analysis and alternative path systems to systematically attempt backup strategies before escalating to the user.

## Purpose

When a task fails during plan execution, this workflow:
1. Analyzes the failure to determine the category and retry appropriateness
2. Selects an alternative strategy from the library
3. Executes the alternative approach
4. Repeats up to 3 attempts before escalating

## Context Requirements

This workflow requires these files to be available:
- `@get-shit-done/lib/failure-analysis.md` - Failure classification and analysis
- `@get-shit-done/lib/path-selection.md` - Alternative strategy selection algorithm
- `@.planning/ALTERNATIVE_PATHS.md` - Library of backup strategies
- `.planning/config.json` - Configuration with retry settings

<executive_summary>
## Quick Reference

Automatic retry with alternative approaches: analyze failures, select backup strategies, retry up to 3 times, then escalate.

## Retry Flow

```
┌─────────────┐
│ Task Fails  │
└─────┬───────┘
      ▼
┌─────────────────────────┐
│ Step 3: Analyze Failure │──────────────────────┐
│ • Category (Tool/Valid/ │                      │
│   Dependency/Logic/Res) │                      │
│ • Retry appropriate?    │                      │
└─────────┬───────────────┘                      │
          │ Yes                                  │ No
          ▼                                      ▼
┌─────────────────────────┐            ┌─────────────────┐
│ Step 4: Select Strategy │            │ Step 7: Escalate│
│ • Filter by task type   │            │ • Create        │
│ • Filter by fail type   │            │   checkpoint    │
│ • Exclude tried ones    │            │ • User decides  │
└─────────┬───────────────┘            └─────────────────┘
          │                                      ▲
          ▼                                      │
┌─────────────────────────┐                      │
│ Step 5: Execute Alt     │                      │
│ • Run backup approach   │                      │
│ • Log attempt           │                      │
└─────────┬───────────────┘                      │
          ▼                                      │
┌─────────────────────────┐                      │
│ Step 6: Loop Check      │                      │
│ • Success? → Done       │                      │
│ • Max attempts? ────────┼──────────────────────┘
│ • More strategies? ─────┼───→ Back to Step 4
└─────────────────────────┘
```

## Steps Overview

| # | Step | One-line Description |
|---|------|---------------------|
| 0 | initialize_logging | Set up ATTEMPTS_LOG.md if logging enabled |
| 1 | execute_task | Run original task action from PLAN.md |
| 1.5 | track_execution_start | Initialize EXECUTION_HISTORY.md (if adaptive enabled) |
| 2 | check_success | Verify task completion against `<verify>` criteria |
| 3 | analyze_failure | Classify failure (Tool/Validation/Dependency/Logic/Resource), determine if retryable |
| 4 | select_alternative | Choose backup strategy from ALTERNATIVE_PATHS.md using selection algorithm |
| 5 | execute_alternative | Run selected backup approach, log attempt |
| 6 | loop_check | Decide: success → done, max attempts → escalate, more strategies → retry |
| 7 | escalate | Create checkpoint:decision with full context for user |

## Configuration Impact

```
config.json field          → Behavior
────────────────────────────────────────────
retry.enabled              → Whether retry logic activates (default: false)
retry.max_attempts         → Retries before escalation (default: 3, range: 1-5)
retry.escalation_threshold → Synonym for max_attempts (default: 3)
retry.log_attempts         → Log to ATTEMPTS_LOG.md (default: true)
adaptive.enabled           → Use AI-powered failure analysis (step 3) and strategy selection (step 4)
adaptive.fallback_to_static→ Fall back to static analysis on AI failure (default: true)
```

## Integration Points

- **Called by:** `execute-phase.md` step 11 (execute) when `config.retry.enabled = true`
- **Uses:** `failure-analysis.md` (step 3), `path-selection.md` (step 4), `ALTERNATIVE_PATHS.md` (strategies)
- **Outputs:** `ATTEMPTS_LOG.md` (attempt history), `EXECUTION_HISTORY.md` (if adaptive)
- **Escalates to:** `checkpoint:decision` with manual intervention, different approach, skip, or abort options
</executive_summary>

## Process

<step name="initialize_logging" number="0">
**Initialize attempt tracking (if enabled)**

Before executing any task, set up attempt logging if configured:

**Process:**
1. Read `.planning/config.json`
2. Check if `config.retry.log_attempts` is `true`
3. If enabled:
   - Check if `.planning/ATTEMPTS_LOG.md` exists
   - If missing, copy from `@get-shit-done/templates/attempts-log.md`
   - Initialize frontmatter with task metadata:
     ```yaml
     ---
     phase: [from PLAN.md frontmatter]
     plan: [from PLAN.md frontmatter]
     task: [current task name]
     execution-date: [current timestamp]
     total-attempts: 0
     final-outcome: pending
     ---
     ```

**Error handling:**
- If logging initialization fails (permission denied, disk full, etc.):
  - Log warning to stderr: "Warning: Could not initialize ATTEMPTS_LOG.md - continuing without logging"
  - Set internal flag `logging_enabled = false`
  - **Continue task execution** (do not break retry flow)

**Notes:**
- Logging is optional and should never block task execution
- If config.retry.log_attempts is false or missing, skip all logging steps
- Each task should have its own log file or clearly delimited section

</step>

<step name="execute_task" number="1">
**Execute the original task action**

**Log attempt start (if logging enabled):**
```xml
<attempt>
  <attempt-number>1</attempt-number>
  <timestamp>[current timestamp YYYY-MM-DD HH:MM:SS]</timestamp>
  <strategy-used>original-plan</strategy-used>
  <approach-description>[task action description from PLAN.md]</approach-description>
```

Run the task as specified in the plan:
- Read task `<action>` from PLAN.md
- Execute the specified operations (file edits, bash commands, etc.)
- Capture all tool results and outputs
- Track execution time and resources used

**Output:**
- Task results (success/failure)
- Tool outputs and error messages
- Files created/modified
- Exit codes and status

**Variables to track:**
```xml
<execution_result>
  <task_name>Original task name from plan</task_name>
  <attempt_number>1</attempt_number>
  <approach>Original approach from task action</approach>
  <status>success | failure</status>
  <output>Tool results</output>
  <error_message>Error details if failed</error_message>
</execution_result>
```
</step>

<step name="track_execution_start" number="1.5">
**Record execution attempt (if adaptive enabled)**

Initialize execution history tracking when adaptive mode is active.

**Process:**

1. Read `.planning/config.json`
2. Check if `config.adaptive.enabled === true`
3. If adaptive enabled:
   - Check if `.planning/EXECUTION_HISTORY.md` exists
   - If missing, copy from `@get-shit-done/templates/execution-history.md`
   - Initialize frontmatter with current timestamp if this is first execution
   - Increment `total-executions` counter in frontmatter
   - Note: Final outcome (success/failure/recovery) will be recorded in step 6 or step 7

**Error handling:**
- If history initialization fails (permission denied, disk full, template missing):
  - Log warning to stderr: "Warning: Could not initialize EXECUTION_HISTORY.md - continuing without history tracking"
  - Set internal flag `history_tracking_enabled = false`
  - **Continue task execution** (do not break retry flow)

**Notes:**
- History tracking is optional and should never block task execution
- If config.adaptive.enabled is false or missing, skip all history tracking steps
- History updates are atomic - read, modify, write back
- Use Edit tool carefully to preserve existing history data

</step>

<step name="check_success" number="2">
**Verify task completion**

Evaluate whether the task succeeded:
- Read task `<verify>` criteria from PLAN.md
- Check all verification conditions
- If ALL verifications pass → Task complete, proceed to next task
- If ANY verification fails → Continue to step 3 (failure handling)

**Success criteria examples:**
- File exists at expected path
- Tests pass
- Output contains expected content
- No error messages in tool results
- Build completes without errors

**Decision:**
```xml
<verification_result>
  <passed>true | false</passed>
  <reason>Why verification passed or failed</reason>
</verification_result>
```

**Log attempt outcome (if logging enabled):**

If verification **passed** (success):
```xml
  <tool-results>
    [First 200 chars of stdout/stderr from step 1 execution]
  </tool-results>
  <outcome>success</outcome>
  <failure-category></failure-category>
  <learnings>[What worked and why - brief summary]</learnings>
</attempt>
```

If verification **failed**:
```xml
  <tool-results>
    [First 200 chars of stdout/stderr from step 1 execution]
  </tool-results>
  <outcome>failure</outcome>
  <failure-category>[Will be determined in step 3]</failure-category>
  <learnings>[Will be added after analysis]</learnings>
</attempt>
```

**Error handling:** If logging fails, continue to next step

**If passed:**
- Update ATTEMPTS_LOG.md frontmatter: `total-attempts: 1`, `final-outcome: success`
- Append summary section with what worked and learnings
- Exit retry workflow, mark task complete, continue to next task in plan

**If failed:** Continue to step 3
</step>

<step name="analyze_failure" number="3">
**Analyze failure and determine retry appropriateness**

**Check adaptive mode:**

1. Read `.planning/config.json`
2. If `config.adaptive.enabled === true`:
   - Load `@get-shit-done/workflows/adaptive-failure-analysis.md`
   - Execute adaptive analysis workflow with failure context:
     - tool, command, exit_code, stdout, stderr, task_context
     - attempt_number, prior_attempts (if available)
   - If adaptive analysis succeeds: Use its output (category, root cause, recommendation, confidence)
   - If adaptive analysis fails or returns malformed output: Fall through to static analysis below
3. Otherwise (adaptive disabled or config missing), use static failure-analysis.md workflow (existing logic below)

**Static analysis (fallback when adaptive disabled or fails):**

Use failure-analysis.md to classify the failure:

**Process:**
1. Read `@get-shit-done/lib/failure-analysis.md`
2. Apply the 5-phase diagnostic workflow:
   - Extract indicators (error messages, exit codes, outputs)
   - Map to failure patterns (using failure-detection.md)
   - Classify into category (Tool, Validation, Dependency, Logic, Resource)
   - Evaluate retry appropriateness
   - Generate recovery recommendation

3. Parse the analysis output:

```xml
<failure_analysis>
  <category>Tool | Validation | Dependency | Logic | Resource</category>
  <retry_appropriate>true | false</retry_appropriate>
  <reason>Explanation of classification</reason>
  <escalation_reason>If retry_appropriate=false, why escalate</escalation_reason>
</failure_analysis>
```

**Update attempt log with failure details (if logging enabled):**

After analysis completes, update the most recent attempt entry:
- Set `<failure-category>` to the classified category from analysis
- Set `<learnings>` with key insights from failure analysis
- Use Edit or Read-then-Write to update the incomplete XML entry from step 2

**Decision branches:**

**If retry_appropriate = false:**
- Update log with escalation reason
- Immediate escalation (go to step 7)
- Common reasons: Logic errors, ambiguous requirements, missing information

**If retry_appropriate = true:**
- Continue to step 4 (select alternative)
- Common reasons: Tool failures, transient resource issues, incorrect approach

**Special cases:**
- If attempt_number >= 3: Escalate regardless (max retries reached)
- If high-stakes task (deploy, data-migration): Apply stricter escalation criteria
- If identical failure repeated: Escalate (wrong approach)
</step>

<step name="select_alternative" number="4">
**Choose backup strategy from library**

**Check adaptive mode:**

1. Read `.planning/config.json`
2. If `config.adaptive.enabled === true`:
   - Load `@get-shit-done/workflows/adaptive-path-selection.md`
   - Execute adaptive selection workflow with failure context:
     - failure_category, task_type, attempt_number (from step 3 and current context)
     - prior_attempts (list of strategy names already tried)
     - failure_details (root cause and analysis from step 3)
     - task_context, recommended_strategy (if available from adaptive failure analysis)
   - If adaptive selection succeeds: Use selected strategy (proceed to parse output below)
   - If adaptive selection fails or returns invalid strategy: Fall through to static selection below
3. Otherwise (adaptive disabled or config missing), use static path-selection.md algorithm (existing logic below)

**Static selection (fallback when adaptive disabled or fails):**

Use path-selection.md to find appropriate alternative:

**Process:**
1. Read `@get-shit-done/lib/path-selection.md`
2. Prepare selection input:

```xml
<selection_input>
  <failure_category>From step 3 analysis</failure_category>
  <task_type>From PLAN.md task type (bash-command, file-edit, etc.)</task_type>
  <attempt_number>Current attempt (1, 2, or 3)</attempt_number>
  <prior_attempts>
    <attempt>Strategy name from previous retry</attempt>
  </prior_attempts>
  <task_context>What task is trying to accomplish</task_context>
</selection_input>
```

3. Apply 8-step selection algorithm:
   - Load ALTERNATIVE_PATHS.md
   - Filter by task type (exact match or "any")
   - Filter by failure category (exact match or "any")
   - Exclude already-tried strategies
   - Sort by difficulty (simple → moderate → complex) and risk (safe → risky)
   - Filter by attempt number (attempt 1: simple+safe, attempt 2: allow moderate risk, attempt 3: allow all)
   - Select top strategy
   - Check if strategies available

4. Parse selection output:

```xml
<selected_strategy>
  <available>true | false</available>
  <strategy_name>Name from ALTERNATIVE_PATHS.md</strategy_name>
  <approach>Specific action to execute</approach>
  <difficulty>simple | moderate | complex</difficulty>
  <risk>safe | moderate | risky</risk>
  <when_to_use>Context where this works</when_to_use>
  <success_indicators>How to verify this approach worked</success_indicators>
</selected_strategy>
```

**If available = false:**
- No applicable strategies remain
- Proceed to step 7 (escalate)

**If available = true:**
- Continue to step 5 (execute alternative)
</step>

<step name="execute_alternative" number="5">
**Retry task with selected alternative approach**

Execute the backup strategy:

**Process:**
1. Load the alternative `<approach>` from selected strategy
2. Replace or modify the original task action with alternative approach

**Log attempt start (if logging enabled):**

Append new attempt entry to ATTEMPTS_LOG.md:
```xml
<attempt>
  <attempt-number>[current attempt number: 2, 3, etc.]</attempt-number>
  <timestamp>[current timestamp YYYY-MM-DD HH:MM:SS]</timestamp>
  <strategy-used>[strategy name from selected_strategy]</strategy-used>
  <approach-description>[approach details from selected strategy]</approach-description>
```

3. Execute the alternative approach
4. Capture results (same as step 1)
5. Increment attempt counter

**Log attempt outcome (if logging enabled):**

After execution, complete the attempt entry:
```xml
  <tool-results>
    [First 200 chars of stdout/stderr from execution]
  </tool-results>
  <outcome>[success | failure]</outcome>
  <failure-category>[If failed, from failure analysis]</failure-category>
  <learnings>[What was learned from this attempt]</learnings>
</attempt>
```

**Execution notes:**
- Use `<success_indicators>` from strategy to guide verification
- Track which strategy was used for exclusion in next retry
- If logging fails at any point, log error but continue execution

**After execution:**
- Update execution_result with new attempt number and approach
- Continue to step 6 (loop check)
</step>

<step name="loop_check" number="6">
**Determine whether to retry again or proceed**

Evaluate retry loop continuation:

**Check conditions:**
1. Did alternative approach succeed?
   - Yes → Log final summary (if logging enabled), exit retry workflow, mark task complete
   - No → Continue evaluation

**Log success summary (if alternative succeeded and logging enabled):**

Update ATTEMPTS_LOG.md:
1. Update frontmatter:
   - `total-attempts: [final attempt count]`
   - `final-outcome: success`

2. Append summary section:
   ```markdown
   ## Summary

   **What worked:**
   [Description of the successful strategy and approach]

   **Why it worked:**
   [Analysis of why this approach succeeded where others failed]

   **Patterns discovered:**
   [Reusable insights - e.g., "Always read file before editing", "Retry with --maxWorkers=1 for flaky tests"]

   ## Adaptive Learning Signals

   **Strategy Effectiveness:**
   [List each strategy attempted with outcome - e.g., "original-plan → failure (reason), retry-with-reruns → success (eliminated race condition)"]

   **Failure Pattern:**
   [Classified pattern - e.g., "Flaky test timeout - common validation failure in test-run tasks"]

   **Recommended Next Time:**
   [Strategy to try first for similar failures - e.g., "For flaky tests, immediately try retry-with-reruns (--maxWorkers=1)"]

   **Confidence:**
   [High/Medium/Low - based on clarity of failure classification and strategy effectiveness]
   ```

**Error handling:** If summary logging fails, continue to mark task complete

**Update execution history (if adaptive enabled):**

If `config.adaptive.enabled === true` and `history_tracking_enabled === true`:

1. Read `.planning/EXECUTION_HISTORY.md`
2. Update frontmatter:
   - Increment `total-successes`
   - If `attempt_number > 1`: increment `total-recoveries` (recovered after initial failure)
   - Update `last-updated` timestamp
3. Update Strategy Effectiveness table:
   - Find row for strategy used (or add new row if first use)
   - Increment "Times Used" counter
   - Recalculate "Success Rate" (successes / total uses)
   - Update "Avg Attempt #" (running average of attempt numbers for this strategy)
   - Add task type to "Task Types" column if not already listed
4. Update Task Type Analysis section for this task's type:
   - Increment success counter
   - Recalculate success rate
   - Add to "Effective Strategies" if not already listed
   - Add insights if patterns emerge
5. Write updated EXECUTION_HISTORY.md back

**Error handling:** If history update fails, log warning but continue to mark task complete (history tracking never blocks execution)

2. Have we reached max attempts?
   - If attempt_number >= max_attempts (default 3) → Escalate (step 7)
   - If attempt_number < max_attempts → Check more alternatives

3. Are more alternatives available?
   - Run step 4 selection again (with updated prior_attempts)
   - If strategies available → Return to step 5 (try next alternative)
   - If no strategies → Escalate (step 7)

**Loop flow:**
```
Attempt 1 fails → Analyze → Select alternative A → Execute → Fails
  → Loop check → More attempts? Yes → More strategies? Yes
  → Select alternative B (exclude A) → Execute → Fails
  → Loop check → More attempts? Yes (at max=3) → Escalate

OR

Attempt 1 fails → Analyze → Select alternative A → Execute → Succeeds
  → Loop check → Success! → Exit, mark complete
```

**Decision:**
- **Continue retry:** Return to step 4 with updated attempt count
- **Success:** Exit workflow, continue plan execution
- **Escalate:** Proceed to step 7
</step>

<step name="escalate" number="7">
**Create checkpoint:decision for user guidance**

When all retries exhausted or retry inappropriate:

**Log escalation summary (if logging enabled):**

Update ATTEMPTS_LOG.md:
1. Update frontmatter:
   - `total-attempts: [final attempt count]`
   - `final-outcome: escalated`

2. Append summary section:
   ```markdown
   ## Summary

   **What worked:**
   Nothing - all [N] attempts failed.

   **Why it failed:**
   [Root cause analysis - e.g., "Permission issues requiring user credentials", "Missing prerequisite file", "Logic error in plan assumptions"]

   **Patterns discovered:**
   [Insights for future - e.g., "Permission errors should trigger early escalation rather than burning retries", "File existence should be verified in planning phase"]

   **Escalation reason:**
   [Specific reason: "All available alternatives exhausted" OR "Retry inappropriate for this failure category" OR "Max attempts (3) reached"]

   ## Adaptive Learning Signals

   **Strategy Effectiveness:**
   [List all strategies attempted with outcomes - e.g., "reinstall-dependencies → failure (did not resolve), explicit-install-missing → failure (still failed)"]

   **Failure Pattern:**
   [Classified pattern - e.g., "Permission-denied - system configuration issue requiring elevated privileges"]

   **Recommended Next Time:**
   [Alternative approach or early escalation - e.g., "For permission errors, immediately escalate with fix instructions rather than burning retries"]

   **Confidence:**
   [High/Medium/Low - based on failure analysis certainty and pattern recognition]
   ```

**Error handling:** If logging fails, continue with escalation

**Update execution history (if adaptive enabled):**

If `config.adaptive.enabled === true` and `history_tracking_enabled === true`:

1. Read `.planning/EXECUTION_HISTORY.md`
2. Update frontmatter:
   - Increment `total-failures`
   - Update `last-updated` timestamp
3. Update Common Failure Patterns table:
   - Check if this failure-category + task-type combination already exists
   - If exists: increment "Frequency", update "Last Seen" timestamp
   - If new pattern: add new row with frequency=1, typical-recovery from attempted strategies
   - If frequency >= 3: this is a recurring pattern worth special attention
4. Update Strategy Effectiveness table for all strategies attempted:
   - For each strategy tried during retries:
     - Find row for strategy (or add new row if first use)
     - Increment "Times Used" counter
     - Recalculate "Success Rate" (note: this attempt failed, so rate may decrease)
     - Update "Avg Attempt #" if applicable
5. Update Task Type Analysis section for this task's type:
   - Increment failure counter (if tracked separately) or adjust success rate
   - Add to "Common Failures" if not already listed
   - Note ineffective strategies (ones that were tried but failed)
6. Write updated EXECUTION_HISTORY.md back

**Error handling:** If history update fails, log warning but continue with escalation (history tracking never blocks execution)

**Process:**
1. Aggregate all attempt information:
   - Original approach and why it failed
   - Each alternative tried and outcomes
   - Failure category and analysis
   - Current task state

2. Create dynamic checkpoint:decision in plan flow:

```xml
<checkpoint type="checkpoint:decision">
  <name>Retry Escalation: [Task Name]</name>
  <context>
    Task: [task name]
    Objective: [what task was trying to accomplish]

    Original approach failed:
    - Action: [original task action]
    - Error: [error message]
    - Reason: [failure analysis]

    Attempted alternatives:
    1. [Strategy A name]: [approach] → [result and why it failed]
    2. [Strategy B name]: [approach] → [result and why it failed]
    3. [Strategy C name]: [approach] → [result and why it failed]

    Failure category: [category from analysis]
    Analysis: [diagnostic summary]
  </context>

  <question>
    All automatic retry attempts have been exhausted. How would you like to proceed?
  </question>

  <options>
    <option>
      <label>Manual intervention</label>
      <description>I'll handle this task manually and let you know when ready to continue</description>
    </option>
    <option>
      <label>Try different approach</label>
      <description>Suggest a specific alternative approach to try</description>
    </option>
    <option>
      <label>Skip task</label>
      <description>Mark this task as deferred and continue with plan (log to issues)</description>
    </option>
    <option>
      <label>Abort plan</label>
      <description>Stop execution and review the plan</description>
    </option>
  </options>
</checkpoint>
```

3. Present checkpoint to user
4. Wait for user decision
5. Execute based on user's choice:
   - **Manual intervention:** Pause, wait for user to complete task, then verify and continue
   - **Try different approach:** User specifies new approach, execute it, then continue retry workflow
   - **Skip task:** Log to .planning/DEFERRED_ISSUES.md, continue to next task
   - **Abort plan:** Exit plan execution, create partial summary

**Escalation logging:**
Record escalation in plan execution log for summary:
```xml
<escalation>
  <task>Task name</task>
  <attempts>Number of attempts made</attempts>
  <strategies_tried>List of strategy names</strategies_tried>
  <user_decision>Choice made at checkpoint</user_decision>
  <outcome>How issue was resolved</outcome>
</escalation>
```
</step>

## Configuration

Retry behavior controlled by `.planning/config.json`:

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
- `enabled`: Whether to use retry logic (default: false for backward compatibility)
- `max_attempts`: Maximum retry attempts per task (default: 3, range: 1-5)
- `escalation_threshold`: Fail count before escalating (default: 3)
- `log_attempts`: Whether to log attempts to ATTEMPTS_LOG.md (default: true)

## Examples

### Example 1: Test Failure with 2 Retries (Success on Attempt 2)

**Scenario:** Test suite fails due to flaky test

**Attempt 1 (Original):**
```xml
<task type="auto">
  <action>Run tests: npm test</action>
  <verify>All tests pass</verify>
</task>

→ Execute: npm test
→ Result: FAILURE (1 flaky test timeout)
→ Verify: Failed (not all tests passed)
```

**Step 3 - Analyze:**
```xml
<failure_analysis>
  <category>Validation</category>
  <retry_appropriate>true</retry_appropriate>
  <reason>Flaky test - common transient issue</reason>
</failure_analysis>
```

**Step 4 - Select Alternative:**
```xml
<selected_strategy>
  <strategy_name>retry-with-reruns</strategy_name>
  <approach>Run with jest --maxWorkers=1 --runInBand to isolate tests</approach>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
</selected_strategy>
```

**Attempt 2 (Alternative):**
```xml
→ Execute: npm test -- --maxWorkers=1 --runInBand
→ Result: SUCCESS (all tests pass)
→ Verify: Passed ✓
```

**Outcome:** Task complete after 2 attempts, continue to next task

---

### Example 2: Build Failure with 3 Retries (Escalation)

**Scenario:** Build fails due to missing dependency

**Attempt 1 (Original):**
```xml
<task type="auto">
  <action>Build project: npm run build</action>
  <verify>Build completes, dist/ directory created</verify>
</task>

→ Execute: npm run build
→ Result: FAILURE (Error: Cannot find module 'lodash')
→ Verify: Failed
```

**Step 3 - Analyze:**
```xml
<failure_analysis>
  <category>Dependency</category>
  <retry_appropriate>true</retry_appropriate>
  <reason>Missing dependency - auto-recoverable</reason>
</failure_analysis>
```

**Step 4 - Select Alternative (Attempt 1):**
```xml
<selected_strategy>
  <strategy_name>reinstall-dependencies</strategy_name>
  <approach>rm -rf node_modules && npm install</approach>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
</selected_strategy>
```

**Attempt 2:**
```xml
→ Execute: rm -rf node_modules && npm install
→ Result: SUCCESS (dependencies installed)
→ Execute: npm run build
→ Result: FAILURE (Error: Cannot find module 'lodash' - still missing)
→ Verify: Failed
```

**Step 4 - Select Alternative (Attempt 2):**
```xml
<selected_strategy>
  <strategy_name>explicit-install-missing</strategy_name>
  <approach>npm install lodash && npm run build</approach>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
</selected_strategy>
```

**Attempt 3:**
```xml
→ Execute: npm install lodash && npm run build
→ Result: FAILURE (Error: lodash installed but TypeScript errors)
→ Verify: Failed
```

**Step 7 - Escalate:**
```
Max attempts (3) reached. Creating checkpoint:decision...

═══════════════════════════════════════════
RETRY ESCALATION: Build Project
═══════════════════════════════════════════

Task failed after 3 automatic retry attempts.

Original approach:
  npm run build
  → Error: Cannot find module 'lodash'

Alternatives tried:
  1. reinstall-dependencies: rm -rf node_modules && npm install
     → Dependencies installed but build still failed

  2. explicit-install-missing: npm install lodash && npm run build
     → lodash installed but TypeScript compilation errors remain

Failure category: Dependency
Analysis: Missing module indicates dependency issue, but TypeScript errors suggest deeper configuration problem

How would you like to proceed?
  [1] Manual intervention - I'll fix the TypeScript config
  [2] Try different approach - Suggest specific fix
  [3] Skip task - Defer and continue
  [4] Abort plan - Stop and review

User choice: _
```

---

### Example 3: Logic Error (Immediate Escalation)

**Scenario:** Task depends on file that doesn't exist

**Attempt 1 (Original):**
```xml
<task type="auto">
  <action>Edit user authentication in src/auth/login.ts to add 2FA</action>
  <verify>File contains 2FA logic</verify>
</task>

→ Execute: Edit src/auth/login.ts (file doesn't exist)
→ Result: FAILURE (File not found)
→ Verify: Failed
```

**Step 3 - Analyze:**
```xml
<failure_analysis>
  <category>Logic</category>
  <retry_appropriate>false</retry_appropriate>
  <reason>Task assumes file exists but it doesn't - indicates plan assumption error or missing prerequisite</reason>
  <escalation_reason>Cannot auto-recover - need user to clarify file location or create prerequisite</escalation_reason>
</failure_analysis>
```

**Step 7 - Immediate Escalation:**
```
Retry not appropriate for this failure type. Escalating...

═══════════════════════════════════════════
RETRY ESCALATION: Add 2FA to Login
═══════════════════════════════════════════

Task cannot be auto-recovered.

Issue: File src/auth/login.ts does not exist

Analysis: Plan assumes this file exists, but it's missing. This indicates:
  - File is at different location
  - Prerequisite task was skipped
  - Plan needs correction

How would you like to proceed?
  [1] Manual intervention - I'll create the file or point to correct location
  [2] Try different approach - Specify correct file path
  [3] Skip task - Defer and continue
  [4] Abort plan - Review plan assumptions

User choice: _
```

**Outcome:** User selects option 2, provides correct path `src/services/auth/login.service.ts`, task retried with corrected path

---

## Integration Notes

**For execute-phase.md:**

This workflow should be invoked conditionally:

```xml
<step name="execute">
  <!-- Load config -->
  <config>cat .planning/config.json</config>

  <!-- For each task -->
  <if condition="config.retry.enabled == true">
    <!-- Wrap task execution in retry workflow -->
    <retry>
      @get-shit-done/workflows/retry-orchestration.md
      - Load failure-analysis.md, path-selection.md, ALTERNATIVE_PATHS.md
      - Execute task with retry orchestration
      - Handle escalations as checkpoint:decision
    </retry>
  </if>

  <if condition="config.retry.enabled == false OR missing">
    <!-- Execute task directly (backward compatible) -->
    <execute>Original task execution flow</execute>
  </if>
</step>
```

**Required context when retry enabled:**
- `@get-shit-done/lib/failure-analysis.md`
- `@get-shit-done/lib/path-selection.md`
- `@.planning/ALTERNATIVE_PATHS.md`

**Adaptive failure analysis (v3.0 Intelligence enhancement):**

If `config.adaptive.enabled === true`:
- Step 3 uses AI-powered failure analysis via `@get-shit-done/workflows/adaptive-failure-analysis.md`
- Provides richer diagnostics than static decision trees
- Falls back to static analysis (`failure-analysis.md`) if AI analysis fails or returns malformed output
- Backward compatible: disabled by default, requires explicit opt-in

**Fallback behavior:**
- Adaptive analysis failures never block retry workflow
- Static analysis always available as reliable fallback
- Users can disable fallback with `config.adaptive.fallback_to_static = false` (not recommended)

**Adaptive strategy selection (v3.0 Intelligence enhancement):**

If `config.adaptive.enabled === true`:
- Step 4 uses AI-powered strategy selection via `@get-shit-done/workflows/adaptive-path-selection.md`
- Selects strategies based on failure context and execution history, not just fixed priority rules
- Considers root cause analysis and prior attempt patterns
- Falls back to static algorithm (`path-selection.md`) if AI selection fails or returns invalid strategy
- Backward compatible: disabled by default, requires explicit opt-in

**Fallback behavior:**
- Adaptive selection failures never block retry workflow
- Static algorithm always available as reliable fallback
- Output format identical whether adaptive or static selection used

**Configuration:**
```json
{
  "adaptive": {
    "enabled": true,           // Use AI-powered failure analysis AND strategy selection
    "fallback_to_static": true // Fall back to static on AI failure (recommended)
  }
}
```

**Logging:**
If `config.retry.log_attempts = true`:
- Initialize `.planning/ATTEMPTS_LOG.md` from template `@get-shit-done/templates/attempts-log.md`
- Log each attempt with: attempt number, timestamp, strategy used, approach, tool results (200 char max), outcome, failure category, learnings
- Log final summary with: what worked/failed, why, patterns discovered
- Handle logging failures gracefully (never break execution flow)
- Use XML structure for attempts (enables future parsing/analysis)
- Update frontmatter with total attempts and final outcome (success/escalated)

## Success Indicators

Retry workflow working correctly when:
- Tasks succeed after 1-2 retries for common failures (flaky tests, transient network)
- Appropriate escalation for logic errors and ambiguous failures
- Attempt logs provide clear audit trail
- User escalations include full context of what was tried
- Backward compatible (works when retry disabled or config missing)
