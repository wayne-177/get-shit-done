# Auto-Correction Workflow

This workflow bridges Phase 2 verification with Phase 1 retry system, enabling automatic rollback and retry when regressions are detected. It orchestrates the complete recovery cycle: detect → rollback → select alternative → retry.

## Purpose

<purpose>
Auto-correction workflow triggers when regression-detection.md identifies that a task introduced failures to previously passing tests. It:

1. Determines appropriate rollback strategy (immediate vs post-commit)
2. Executes safe git rollback using rollback-strategy.md patterns
3. Logs regression details to REGRESSION_LOG.md
4. Analyzes failure using failure-analysis.md (treats as Validation category)
5. Selects alternative strategy using path-selection.md
6. Invokes retry-orchestration.md to re-execute task with different approach
7. Verifies resolution or escalates after max attempts

This creates a seamless bridge: verification detects problems, auto-correction fixes them, retry ensures success.
</purpose>

## Prerequisites

<prerequisites>
  <config_requirements>
    Both retry and verification must be enabled in .planning/config.json:

    ```json
    {
      "retry": {
        "enabled": true,
        "max_attempts": 3
      },
      "verification": {
        "enabled": true,
        "regression_check": true,
        "auto_rollback": true
      }
    }
    ```

    If either disabled, auto-correction will not activate.
    Fallback: Manual regression handling (escalate immediately).
  </config_requirements>

  <required_files>
    Must be available in context:
    - @get-shit-done/lib/regression-detection.md (detects regressions)
    - @get-shit-done/lib/rollback-strategy.md (rollback patterns)
    - @get-shit-done/lib/failure-analysis.md (classifies failures)
    - @get-shit-done/lib/path-selection.md (selects alternatives)
    - @get-shit-done/workflows/retry-orchestration.md (retry execution)
    - @.planning/ALTERNATIVE_PATHS.md (alternative strategies)
    - @.planning/REGRESSION_LOG.md (regression tracking)
  </required_files>

  <state_requirements>
    Clean execution state:
    - Git working tree should be clean or stashable
    - No ongoing merge or rebase operations
    - Baseline tests captured in .planning/.baseline-tests.json
    - Task verification completed (task itself passed)
  </state_requirements>
</prerequisites>

## Process

<step name="detect_regression" number="1">
**Detect regression using regression-detection.md**

**Context:** Task completed successfully, now checking for regressions.

**Process:**
1. Load regression-detection.md logic
2. Run comparison:
   - Current tests: npm test --json > .planning/.current-tests.json
   - Baseline tests: cat .planning/.baseline-tests.json
   - Compare: tests in baseline.passing AND current.failing = regression

3. Parse regression detection output:
```xml
<regression_detected>
  <found>true | false</found>
  <failing_tests>
    <test>test-file: test-name</test>
    <test>test-file: test-name</test>
  </failing_tests>
  <regression_count>Number of tests that regressed</regression_count>
  <commit_sha>Current HEAD (commit that likely caused regression)</commit_sha>
</regression_detected>
```

**Decision branches:**
- If found = false: No regression, continue to next task
- If found = true: Continue to step 2 (determine rollback type)

**Error handling:**
- If regression detection fails (can't run tests, can't parse output):
  - Log warning to REGRESSION_LOG.md
  - Escalate: "Regression detection failed - manual verification needed"
  - Do not proceed with auto-correction (unsafe)
</step>

<step name="determine_rollback_type" number="2">
**Determine rollback type from rollback-strategy.md**

**Context:** Regression confirmed, need to decide how to rollback.

**Process:**
1. Load rollback-strategy.md
2. Check current git state:
   - git log -1 --format=%H → Get current commit SHA
   - git status → Check working tree state

3. Determine scenario:
```xml
<rollback_scenario>
  <type>immediate | post_commit</type>
  <rationale>Why this type selected</rationale>
</rollback_scenario>
```

**Decision logic:**
- **Immediate rollback** if:
  - Task not yet committed (working tree dirty, no commit created)
  - Changes only in working tree
  - Use: git reset --hard HEAD

- **Post-commit rollback** if:
  - Task already committed (git log shows new commit)
  - Regression detected in subsequent task
  - Use: git stash + git revert + git stash pop

**Most common:** Post-commit (regression typically detected after task completes and commits)

**Output:**
- rollback_type: immediate or post_commit
- commit_to_revert: SHA if post_commit, null if immediate
- safety_checks: What to verify after rollback
</step>

<step name="execute_rollback" number="3">
**Execute rollback using safe git operations**

**Context:** Rollback type determined, execute appropriate git commands.

**Process:**

### If rollback_type = immediate:
```bash
# Discard uncommitted changes
git reset --hard HEAD
```

Verify:
- git status shows "working tree clean"
- No uncommitted changes remain

### If rollback_type = post_commit:
```bash
# 1. Stash current work (verification detection code)
git stash push -m "Auto-correction before rollback of [commit_sha]"

# 2. Revert the problematic commit
git revert [commit_sha] --no-edit

# 3. Restore stashed work
git stash pop
```

Verify:
- git log shows new revert commit
- git stash list is empty (or original stash count)
- working tree contains verification code

**Conflict handling:**

If git revert or git stash pop fails with CONFLICT:
```xml
<conflict_detected>
  <operation>revert | stash_pop</operation>
  <conflicted_files>
    <file>path/to/file1</file>
    <file>path/to/file2</file>
  </conflicted_files>
</conflict_detected>
```

Actions:
1. Abort conflicted operation:
   - git revert --abort (if revert conflicted)
   - git stash pop already failed, stash still in list

2. Log conflict to REGRESSION_LOG.md:
   - resolution: escalated
   - escalation-reason: "Rollback conflict - manual resolution required"

3. Create checkpoint:decision:
   - Show conflict details
   - Options: resolve manually, skip rollback, abort plan

4. STOP auto-correction, wait for user

**Success output:**
```xml
<rollback_result>
  <success>true | false</success>
  <revert_commit_sha>SHA if post_commit successful</revert_commit_sha>
  <verification_work_restored>true | false</verification_work_restored>
  <conflicts>none | [list of conflicts]</conflicts>
</rollback_result>
```

**Error handling:**
- Any git command failure: log error, escalate immediately
- Do not continue to retry if rollback fails
</step>

<step name="log_regression" number="4">
**Log regression to REGRESSION_LOG.md**

**Context:** Rollback completed (or failed), record the event.

**Process:**
1. Read .planning/REGRESSION_LOG.md
2. Update header counts:
   - Increment "Total regressions detected"
   - Update "Last detected" timestamp

3. Append new entry using template from regression-log.md:
```xml
<entry>
  <timestamp>[current timestamp YYYY-MM-DD HH:MM:SS]</timestamp>
  <task-id>[phase-plan Task N from current execution]</task-id>
  <commit-sha>[commit that caused regression or "uncommitted"]</commit-sha>
  <failure-description>
    [Regression details from step 1: which tests failed, error messages]
  </failure-description>
  <affected-tests>
    [List of tests from regression detection]
    <test>test-file: test-name</test>
  </affected-tests>
  <rollback-action>
    [Git commands executed in step 3: revert SHA or reset, conflicts if any]
  </rollback-action>
  <retry-strategy>
    [Will be filled in step 6 after strategy selected]
    [For now: "Pending - selecting alternative"]
  </retry-strategy>
  <resolution>[pending | success | escalated]</resolution>
  <learnings>
    [Will be updated after retry completes or escalation]
    [For now: "TBD after retry attempt"]
  </learnings>
</entry>
```

4. Write updated REGRESSION_LOG.md

**Error handling:**
- If log write fails (permissions, disk full):
  - Log error to stderr
  - Continue with auto-correction (logging is not critical)
  - Store entry in memory for later retry

**Note:** Entry will be updated in step 7 with final resolution and learnings.
</step>

<step name="analyze_failure" number="5">
**Analyze failure using failure-analysis.md**

**Context:** Need to classify regression for strategy selection.

**Process:**
1. Load failure-analysis.md
2. Prepare analysis input:
```xml
<analysis_input>
  <tool>Bash</tool>
  <command>npm test</command>
  <exit_code>1</exit_code>
  <stdout>[Test output showing failures]</stdout>
  <stderr>[Error messages if any]</stderr>
  <task_context>[What task was trying to accomplish]</task_context>
  <task_type>[Task type from PLAN.md]</task_type>
  <previous_attempts>0</previous_attempts>
</analysis_input>
```

3. Apply failure-analysis.md 5-phase workflow:
   - Phase 1: Detect failure (confirmed: regression = validation failure)
   - Phase 2: Classify (category: Validation)
   - Phase 3: Diagnose (which tests failed, why)
   - Phase 4: Strategize (select recovery approach)
   - Phase 5: Decide (RETRY - regressions are auto-retryable)

4. Parse analysis output:
```xml
<failure_analysis>
  <category>Validation</category>
  <confidence>high</confidence>
  <root_cause>[Specific reason tests failed]</root_cause>
  <retry_appropriate>true</retry_appropriate>
  <decision>RETRY</decision>
</failure_analysis>
```

**Regression-specific handling:**
- Always classify as "Validation" category (tests failing)
- Always retry_appropriate = true (unless max attempts reached)
- Root cause should explain WHY tests regressed (what changed)

**Decision override:**
- If attempt_number >= max_attempts (usually 3): decision = ESCALATE
- If rollback failed with conflicts: decision = ESCALATE
- If identical regression occurred 3+ times: decision = ESCALATE
</step>

<step name="select_alternative" number="6">
**Select alternative strategy using path-selection.md**

**Context:** Failure classified, need to choose different approach.

**Process:**
1. Load path-selection.md
2. Prepare selection input:
```xml
<selection_input>
  <failure_category>Validation</failure_category>
  <task_type>[From original task in PLAN.md]</task_type>
  <attempt_number>[Current attempt: 1, 2, or 3]</attempt_number>
  <prior_attempts>
    [List of approaches already tried, including original]
    <attempt>[Original approach that caused regression]</attempt>
  </prior_attempts>
  <failure_details>[From step 5 analysis]</failure_details>
</selection_input>
```

3. Apply path-selection.md 8-step algorithm:
   - Load ALTERNATIVE_PATHS.md
   - Filter by task_type
   - Filter by failure_category (Validation)
   - Exclude already-tried strategies
   - Sort by difficulty and risk
   - Filter by attempt_number
   - Select top strategy
   - Check escalation criteria

4. Parse selection output:
```xml
<selected_strategy>
  <available>true | false</available>
  <strategy_name>[Name from ALTERNATIVE_PATHS.md]</strategy_name>
  <approach>[Specific alternative implementation]</approach>
  <when_to_use>[Context where this applies]</when_to_use>
  <success_indicators>[How to verify success]</success_indicators>
</selected_strategy>
```

**Decision branches:**
- If available = true: Continue to step 7 (invoke retry)
- If available = false: No strategies left, escalate

**Update regression log:**
- Read current regression entry (added in step 4)
- Update retry-strategy field with selected strategy name and approach
- Write back to REGRESSION_LOG.md
</step>

<step name="invoke_retry" number="7">
**Delegate to retry-orchestration.md for execution**

**Context:** Alternative strategy selected, ready to retry task.

**Process:**
1. Load retry-orchestration.md
2. Prepare retry context:
```xml
<retry_context>
  <task>[Original task from PLAN.md]</task>
  <modified_action>[Replace original action with alternative approach]</modified_action>
  <failure_category>Validation</failure_category>
  <attempt_number>[Incremented from step 6]</attempt_number>
  <prior_strategies>[List of all strategies tried]</prior_strategies>
  <regression_context>
    <rollback_performed>true</rollback_performed>
    <revert_commit>[SHA from step 3]</revert_commit>
    <affected_tests>[From step 1]</affected_tests>
  </regression_context>
</retry_context>
```

3. Invoke retry-orchestration.md workflow:
   - Starts at step 5 "execute_alternative" (not step 1)
   - Uses modified action from alternative strategy
   - Executes alternative approach
   - Runs task verification
   - Runs regression check again (step 1 of this workflow)
   - Loops if still failing

4. Monitor retry outcome:
```xml
<retry_outcome>
  <status>success | failure | escalated</status>
  <final_attempt_number>2 | 3 | ...</final_attempt_number>
  <resolution>[What happened]</resolution>
</retry_outcome>
```

**Recursive regression detection:**
If retry creates ANOTHER regression:
- This workflow triggers again (nested)
- Rollback the retry commit
- Select next alternative strategy
- Continue until success or escalation

**Success criteria:**
- Alternative approach passes its own verification
- Regression check passes (baseline tests still pass)
- No new regressions introduced

**Final regression log update:**
After retry completes (success or escalation):
1. Read current regression entry
2. Update fields:
   - resolution: success | escalated
   - learnings: Why original failed, why alternative worked (or didn't)
   - If success: What pattern to reuse in future
   - If escalated: Why all alternatives failed, what user should consider
3. Write final version to REGRESSION_LOG.md
</step>

## Integration

<integration>
  <called_from>
    This workflow is invoked from execute-phase.md after task verification:

    ```xml
    <step name="execute_task">
      <!-- Execute task -->
      <!-- Run task verification -->

      <!-- NEW: Regression check (if verification.enabled) -->
      <if condition="config.verification.enabled AND config.verification.regression_check">
        <regression_check>
          Run regression-detection.md
          If regression detected:
            Call auto-correction.md workflow
            (This document executes)
            Returns: success (corrected) or escalated (user needed)
        </regression_check>
      </if>

      <!-- Continue to next task or handle escalation -->
    </step>
    ```
  </called_from>

  <calls_to>
    This workflow orchestrates several other components:
    - regression-detection.md (step 1): Detects regressions
    - rollback-strategy.md (steps 2-3): Safe rollback patterns
    - failure-analysis.md (step 5): Classifies regression as failure
    - path-selection.md (step 6): Selects alternative strategy
    - retry-orchestration.md (step 7): Executes retry with alternative
    - REGRESSION_LOG.md (step 4): Logs regression and resolution
  </calls_to>

  <data_flow>
    ```
    execute-phase.md
      → Task completes
      → Verification passes
      → auto-correction.md invoked
        → regression-detection.md: Is there a regression?
        → rollback-strategy.md: How to rollback safely?
        → Git operations: Rollback executed
        → REGRESSION_LOG.md: Log the regression
        → failure-analysis.md: Classify as Validation failure
        → path-selection.md: Select alternative strategy
        → retry-orchestration.md: Retry with alternative
          → (If still fails, retry-orchestration loops)
          → (If succeeds, return to auto-correction)
        → REGRESSION_LOG.md: Update with resolution
      → Return to execute-phase.md
      → Continue to next task
    ```
  </data_flow>

  <config_integration>
    Behavior controlled by .planning/config.json:

    ```json
    {
      "retry": {
        "enabled": true,        // Required for auto-correction
        "max_attempts": 3       // Max retry attempts per regression
      },
      "verification": {
        "enabled": true,              // Master toggle for verification
        "regression_check": true,     // Enable regression detection
        "auto_rollback": true,        // Enable automatic rollback
        "baseline_tests": "npm test", // Command to run baseline tests
        "auto_correction": true       // Enable this workflow (default true if verification enabled)
      }
    }
    ```

    Feature gates:
    - If retry.enabled = false: Escalate immediately (no retry)
    - If verification.regression_check = false: Skip regression detection
    - If verification.auto_rollback = false: Detect but don't rollback, escalate
    - If verification.auto_correction = false: Detect and rollback, but escalate instead of retry
  </config_integration>
</integration>

## Examples

### Example 1: Regression Auto-Corrected on Retry 2

**Scenario:** API change breaks integration tests, auto-corrected with compatibility layer

```xml
<example id="auto_corrected">
  <initial_state>
    Plan: 03-02 (API Enhancement)
    Task 2: "Update API response format to include metadata"
    Status: Task 2 complete, committed (SHA: abc123)
    Task 3: Starting verification
  </initial_state>

  <step1_detect>
    Task 3 runs regression check:
    → npm test
    → FAIL: 4 integration tests failing
    → Baseline: All 4 tests passing
    → REGRESSION DETECTED

    <regression_detected>
      <found>true</found>
      <failing_tests>
        <test>api.test.js: GET /users format check</test>
        <test>api.test.js: POST /users response validation</test>
        <test>integration.test.js: user creation flow</test>
        <test>integration.test.js: user update flow</test>
      </failing_tests>
      <regression_count>4</regression_count>
      <commit_sha>abc123</commit_sha>
    </regression_detected>
  </step1_detect>

  <step2_determine>
    Check git state:
    → git log -1: Shows abc123 (task committed)
    → Type: post_commit

    <rollback_scenario>
      <type>post_commit</type>
      <rationale>Task 2 already committed, Task 3 detected regression</rationale>
    </rollback_scenario>
  </step2_determine>

  <step3_rollback>
    Execute post-commit rollback:
    → git stash push -m "Task 3 verification before rollback of abc123"
    → git revert abc123 --no-edit
    → New commit: def456 "Revert 'Update API response format to include metadata'"
    → git stash pop
    → Success, no conflicts

    <rollback_result>
      <success>true</success>
      <revert_commit_sha>def456</revert_commit_sha>
      <verification_work_restored>true</verification_work_restored>
      <conflicts>none</conflicts>
    </rollback_result>
  </step3_rollback>

  <step4_log>
    Create regression log entry:
    → .planning/REGRESSION_LOG.md updated
    → Entry timestamp: 2026-01-06 16:22:15
    → Task: 03-02 Task 2
    → Affected tests: 4 integration tests
    → Rollback: git revert abc123 → def456
    → Resolution: pending (retry in progress)
  </step4_log>

  <step5_analyze>
    Classify failure:
    → Category: Validation (tests failing)
    → Root cause: API response format changed, tests expect old format
    → Retry appropriate: true

    <failure_analysis>
      <category>Validation</category>
      <root_cause>Tests expect old response format {data: ...}, new format is {data: ..., metadata: ...}</root_cause>
      <retry_appropriate>true</retry_appropriate>
    </failure_analysis>
  </step5_analyze>

  <step6_select>
    Select alternative:
    → Task type: api-modification
    → Failure category: Validation
    → Attempt: 1 (first retry)
    → Strategy selected: "backward-compatibility-layer"

    <selected_strategy>
      <available>true</available>
      <strategy_name>backward-compatibility-layer</strategy_name>
      <approach>
        Add compatibility layer that supports both old and new response formats.
        Use Accept header or version parameter to determine which format to return.
        Update new tests to use new format, keep existing tests on old format.
      </approach>
    </selected_strategy>

    Update regression log:
    → retry-strategy: "backward-compatibility-layer (version-based format selection)"
  </step6_select>

  <step7_retry>
    Invoke retry-orchestration.md:
    → Attempt 2: Implement backward compatibility layer

    Execution:
    → Edit API response handler to check version header
    → If version=2: return new format (with metadata)
    → If version=1 or missing: return old format (data only)
    → Update new tests to send version=2 header
    → Existing tests remain unchanged (version=1 default)

    Task verification:
    → API endpoints work: ✓
    → New metadata feature available: ✓

    Regression check:
    → npm test
    → All 4 integration tests pass: ✓
    → New tests also pass: ✓
    → NO REGRESSION

    <retry_outcome>
      <status>success</status>
      <final_attempt_number>2</final_attempt_number>
      <resolution>Backward compatibility layer resolved regression</resolution>
    </retry_outcome>

    Final regression log update:
    → resolution: success
    → learnings: "API format changes need backward compatibility. Version headers allow gradual migration. Keep existing tests on old format until clients upgrade."
  </step7_retry>

  <outcome>
    - Regression detected automatically
    - Rolled back safely (no data loss)
    - Alternative strategy auto-selected
    - Retry succeeded on attempt 2
    - Total time: ~3 minutes (automatic)
    - Commits in history: abc123 (original), def456 (revert), ghi789 (fix with compatibility)
  </outcome>
</example>
```

### Example 2: Regression Escalated After 3 Retries

**Scenario:** Schema change incompatible with existing data model, all retries fail

```xml
<example id="escalated">
  <initial_state>
    Plan: 02-01 (Database Refactoring)
    Task 1: "Change user ID from integer to UUID"
    Status: Task 1 complete, committed (SHA: xxx111)
    Task 2: Starting verification
  </initial_state>

  <step1_detect>
    Regression detected:
    → 15 tests failing across database, models, queries
    → All expect integer IDs, now getting UUIDs

    <regression_detected>
      <found>true</found>
      <regression_count>15</regression_count>
      <commit_sha>xxx111</commit_sha>
    </regression_detected>
  </step1_detect>

  <step2_to_step4>
    Rollback and logging:
    → git revert xxx111 → yyy111
    → Regression logged to REGRESSION_LOG.md
  </step2_to_step4>

  <step5_to_step7_attempt1>
    Retry attempt 1:
    → Strategy: "gradual-migration-dual-support"
    → Approach: Support both integer and UUID IDs during transition

    Execution:
    → Modified ID field to accept both types
    → Added conversion layer

    Result:
    → Task verification: ✓ (ID changes work)
    → Regression check: ✗ (12 tests still failing - conversion issues)

    <retry_outcome>
      <status>failure</status>
      <resolution>Conversion layer incomplete, type mismatches persist</resolution>
    </retry_outcome>
  </step5_to_step7_attempt1>

  <step3_rollback_again>
    Rollback attempt 1 commit:
    → git revert [attempt1-sha]
  </step3_rollback_again>

  <step5_to_step7_attempt2>
    Retry attempt 2:
    → Strategy: "update-all-tests-for-uuid"
    → Approach: Change all test fixtures and expectations to UUIDs

    Execution:
    → Updated test fixtures (50+ files)
    → Changed ID assertions in tests

    Result:
    → Task verification: ✓ (Tests updated)
    → Regression check: ✗ (8 tests still failing - production data incompatibility)

    <retry_outcome>
      <status>failure</status>
      <resolution>Tests updated but production data migration not handled</resolution>
    </retry_outcome>
  </step5_to_step7_attempt2>

  <step3_rollback_again>
    Rollback attempt 2 commit:
    → git revert [attempt2-sha]
  </step3_rollback_again>

  <step5_to_step7_attempt3>
    Retry attempt 3:
    → Strategy: "data-migration-with-id-change"
    → Approach: Include data migration script with ID type change

    Execution:
    → Created migration script
    → Attempted to convert existing IDs to UUIDs

    Result:
    → Task verification: ✗ (Migration script errors on complex references)
    → Foreign key constraints fail
    → Data integrity issues

    <retry_outcome>
      <status>failure</status>
      <resolution>Data migration too complex for automatic handling</resolution>
    </retry_outcome>
  </step5_to_step7_attempt3>

  <step3_rollback_final>
    Rollback attempt 3 commit:
    → git revert [attempt3-sha]

    Max attempts reached (3), escalate.
  </step3_rollback_final>

  <escalation>
    Update regression log:
    → resolution: escalated
    → learnings: "ID type change is architectural decision, requires:
       1. Data migration strategy
       2. Foreign key updates across all tables
       3. API compatibility layer
       4. Phased rollout plan
       Not suitable for single task, needs separate planning phase."

    Create checkpoint:decision:

    ═══════════════════════════════════════════
    AUTO-CORRECTION ESCALATION: User ID Type Change
    ═══════════════════════════════════════════

    Regression detected and auto-correction attempted but failed after 3 retries.

    Original issue:
      Task 1 changed user IDs from integer to UUID
      Broke 15 tests expecting integer IDs

    Retry attempts:
      1. Dual support (int + UUID): 12 tests still failing
      2. Update all tests to UUID: 8 tests still failing (prod data issue)
      3. Include data migration: Migration too complex, foreign key errors

    Root cause:
      ID type change is architectural change requiring:
      - Database schema migration
      - Foreign key updates
      - API compatibility
      - Data migration strategy

    This is beyond scope of automatic correction.

    Options:
      [1] Manual implementation - I'll design migration strategy
      [2] Defer to separate phase - Plan ID migration properly
      [3] Revert permanently - Keep integer IDs
      [4] Abort plan - Review approach

    Recommendation: Option 2 (defer to separate phase)
    ID type changes need careful planning with migration strategy.
  </escalation>

  <outcome>
    - Regression detected automatically
    - 3 rollbacks performed (all successful)
    - 3 retry attempts (all failed)
    - Escalated to user after exhausting options
    - User chooses: Defer to separate migration phase
    - Total time: ~8 minutes (automatic detection + retries + escalation)
    - Learnings: Architectural changes need separate planning
  </outcome>
</example>
```

## Success Indicators

<success_indicators>
  Auto-correction workflow working correctly when:

  - **Detection accuracy:** Identifies real regressions (not false positives)
  - **Rollback safety:** All rollbacks preserve history and working tree
  - **Strategy selection:** Chooses appropriate alternatives based on failure type
  - **Retry success rate:** >70% of regressions auto-corrected within 3 attempts
  - **Escalation appropriateness:** Escalates when auto-correction won't help (architectural issues, data problems)
  - **Logging completeness:** Every regression logged with full context and learnings
  - **Config compliance:** Respects enabled/disabled settings, fails gracefully when disabled
  - **No data loss:** No commits permanently lost, all history preserved
  - **Performance:** Full cycle (detect → rollback → retry) completes in <5 minutes for simple regressions
</success_indicators>

## Troubleshooting

<troubleshooting>
  <issue name="rollback_conflicts">
    <symptoms>git revert or git stash pop fails with CONFLICT</symptoms>
    <cause>Later commits modified same files as regressing commit</cause>
    <resolution>
      Automatic: Abort and escalate (conflicts unsafe to auto-resolve)
      Manual: User resolves conflicts, continues or aborts plan
    </resolution>
  </issue>

  <issue name="repeated_regressions">
    <symptoms>Same regression occurs across multiple retry attempts</symptoms>
    <cause>Alternative strategies not addressing root cause</cause>
    <resolution>
      After 3 identical regressions: Escalate with note "Strategies ineffective, need different approach"
      Review ALTERNATIVE_PATHS.md - may need new strategies for this scenario
    </resolution>
  </issue>

  <issue name="regression_detection_fails">
    <symptoms>Cannot run regression check (test command fails, JSON parsing error)</symptoms>
    <cause>Test suite broken, environment issue, baseline missing</cause>
    <resolution>
      Do NOT proceed with auto-correction (unsafe without detection)
      Escalate immediately: "Regression detection failed - verify test suite"
    </resolution>
  </issue>

  <issue name="config_disabled">
    <symptoms>Auto-correction doesn't trigger despite regression</symptoms>
    <cause>retry.enabled=false or verification.auto_correction=false</cause>
    <resolution>
      Expected behavior (feature disabled)
      If user wants auto-correction: Update config.json, enable both retry and verification
    </resolution>
  </issue>

  <issue name="infinite_retry_loop">
    <symptoms>Auto-correction triggers repeatedly without progress</symptoms>
    <cause>Bug in workflow, max_attempts not enforced, same strategy retried</cause>
    <resolution>
      Safety check: Hard limit 3 attempts per task (from config.retry.max_attempts)
      If exceeded: Force escalation, log error
      Fix: Review retry-orchestration.md loop check logic
    </resolution>
  </issue>
</troubleshooting>

---

*Auto-correction workflow bridging Phase 2 verification with Phase 1 retry system*
