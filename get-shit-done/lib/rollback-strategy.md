# Rollback Strategy Library

This document defines safe rollback patterns for recovering from regressions detected during task execution. Integrates with regression-detection.md to automatically revert problematic commits and restore working state.

## Purpose

When regression-detection.md identifies that a task introduced failures, this library guides the rollback process to:
1. Safely undo problematic changes without data loss
2. Preserve work in progress and git history
3. Enable retry with alternative approach
4. Maintain bisectable commit history for debugging

## Rollback Scenarios

<rollback_scenarios>
  <scenario type="immediate">
    <name>Immediate Rollback (During Task Execution, Before Commit)</name>
    <timing>Regression detected DURING task execution, before git commit</timing>
    <state>Changes exist in working tree but not committed</state>

    <approach>
      Use git reset --hard HEAD to discard all uncommitted changes:

      1. Verify current HEAD is safe (last known good commit)
      2. Run: git reset --hard HEAD
      3. All working tree changes discarded
      4. Repository restored to last commit state
      5. Ready to retry task with alternative approach
    </approach>

    <when_to_use>
      - Task execution in progress (files modified, commands run)
      - Verification check fails before commit step
      - No commit has been created yet
      - Working tree is dirty (git status shows modifications)
    </when_to_use>

    <safety_level>Moderate risk - loses uncommitted work</safety_level>

    <prerequisites>
      - No valuable uncommitted changes to preserve
      - Last commit (HEAD) is known good state
      - Not in middle of git rebase or merge
    </prerequisites>
  </scenario>

  <scenario type="post_commit">
    <name>Post-Commit Rollback (Regression After Commit)</name>
    <timing>Regression detected AFTER task committed to git</timing>
    <state>Problematic changes are in git history as a commit</state>

    <approach>
      Use git stash + git revert + git stash pop for safe rollback:

      1. Stash current work (e.g., verification detection code):
         git stash push -m "Verification work before rollback of [commit-sha]"

      2. Revert the problematic commit (creates new commit):
         git revert [commit-sha] --no-edit

         This creates a NEW commit that undoes changes from [commit-sha]
         Original commit stays in history (non-destructive)

      3. Restore stashed work:
         git stash pop

         If conflicts: resolve them, then git stash drop

      4. Result:
         - Problematic commit still in history (for debugging)
         - New revert commit undoes the damage
         - Working tree contains detection/verification work
         - Ready to retry task with alternative approach
    </approach>

    <when_to_use>
      - Task completed and committed (git commit succeeded)
      - Next task detects regression in prior task
      - Need to undo changes while preserving history
      - Want to maintain bisectable commit history
    </when_to_use>

    <safety_level>Safe - preserves all history, non-destructive</safety_level>

    <prerequisites>
      - Know the specific commit SHA to revert
      - No ongoing merge or rebase
      - Working tree can be stashed (no conflicts)
    </prerequisites>
  </scenario>
</rollback_scenarios>

## Safety Protocol

<safety_protocol>
  <principle name="prefer_revert_over_reset">
    <rationale>
      git revert is NON-DESTRUCTIVE:
      - Preserves complete git history
      - Creates audit trail of what was reverted and why
      - Allows "revert the revert" if mistake made
      - Maintains bisectable history for debugging
      - Safe for shared branches (already pushed)

      git reset --hard is DESTRUCTIVE:
      - Erases commits from history permanently
      - Cannot undo without reflog (temporary)
      - Dangerous for pushed commits (requires force push)
      - Loses ability to see what was tried
      - Breaks bisect and debugging workflows
    </rationale>

    <guideline>
      Use git revert for ALL post-commit rollbacks.
      Only use git reset --hard for uncommitted changes (working tree cleanup).
      Never use git reset --hard on pushed commits.
    </guideline>
  </principle>

  <principle name="stash_before_rollback">
    <rationale>
      Verification detection code is valuable:
      - Represents work done to identify regression
      - Contains test results and diagnostic information
      - May include partial fixes attempted during detection
      - Should be preserved across rollback operation

      git stash safely stores this work:
      - Temporary storage outside main history
      - Can be reapplied after rollback
      - Handles conflicts gracefully
      - Provides safety net if rollback goes wrong
    </rationale>

    <guideline>
      Always stash working tree before rollback operations.
      Use descriptive stash message with commit SHA being reverted.
      Pop stash immediately after revert to restore verification work.
      If conflicts: resolve carefully, commit stash separately if needed.
    </guideline>
  </principle>

  <principle name="verify_rollback_success">
    <rationale>
      Rollback can fail:
      - Git revert may have conflicts (file changed in later commits)
      - Stash pop may conflict (verification modified same files)
      - File system errors (permissions, disk full)
      - Working tree state unexpected (merge in progress)

      Must verify rollback completed:
      - Check git status shows expected state
      - Verify revert commit created (git log)
      - Confirm stash applied successfully
      - Run baseline tests to confirm regression resolved
    </rationale>

    <guideline>
      After each rollback operation, verify:
      1. git status output matches expectation
      2. git log shows revert commit (post-commit) or clean tree (immediate)
      3. git stash list is empty (stash was popped) or contains expected entry
      4. npm test (or baseline test command) passes
      5. No unexpected files in working tree

      If verification fails: escalate immediately, do not retry.
    </guideline>
  </principle>

  <principle name="log_rollback_action">
    <rationale>
      Rollback is critical event:
      - Indicates regression occurred
      - Shows which commit caused problem
      - Records what was reverted
      - Informs retry strategy selection

      Must be logged for:
      - Debugging future issues
      - Understanding project health trends
      - Learning which changes cause regressions
      - Audit trail for post-mortem analysis
    </rationale>

    <guideline>
      Every rollback must be logged to REGRESSION_LOG.md with:
      - Timestamp of rollback
      - Commit SHA that was reverted
      - Task ID and description
      - Failure description (what regression occurred)
      - Rollback action taken (revert vs reset)
      - Outcome (success/conflict/escalated)
      - Retry strategy selected

      Use auto-correction.md workflow to perform logging.
    </guideline>
  </principle>
</safety_protocol>

## Edge Cases

<edge_cases>
  <edge_case id="conflict_during_revert">
    <description>
      Git revert creates conflicts when:
      - Files modified in problematic commit were also modified in later commits
      - Cannot automatically apply inverse patch
      - Requires manual conflict resolution
    </description>

    <detection>
      git revert command exits with non-zero code
      Output contains "CONFLICT" or "conflict"
      git status shows "You are currently reverting commit"
      Conflicted files marked with &lt;&lt;&lt;&lt;&lt;&lt;&lt;
    </detection>

    <handling>
      Automatic conflict resolution is UNSAFE - escalate:

      1. Detect conflict during git revert
      2. DO NOT attempt automatic resolution
      3. Log conflict details to REGRESSION_LOG.md:
         - Which files conflicted
         - Commit being reverted
         - Commits that modified same files since
      4. Create checkpoint:decision for user:
         - Show conflict details
         - Options: resolve manually, skip revert (keep regression), abort
      5. Wait for user resolution

      Rationale: Conflicts indicate complex interaction between commits.
      Automatic resolution risks introducing new bugs or losing important changes.
    </handling>

    <prevention>
      - Keep tasks atomic (small, focused changes)
      - Detect regressions immediately after each task
      - Don't let multiple tasks accumulate before verification
    </prevention>
  </edge_case>

  <edge_case id="stash_pop_conflict">
    <description>
      git stash pop creates conflicts when:
      - Stashed changes modified same lines as revert commit
      - Working tree state incompatible with stashed changes
    </description>

    <detection>
      git stash pop command reports "CONFLICT"
      git status shows conflicted files
      Stash remains in stash list (not dropped)
    </detection>

    <handling>
      Stash conflicts are usually RESOLVABLE:

      1. Detect conflict during stash pop
      2. Read conflicted files to assess complexity:
         - Simple conflicts (single lines): attempt resolution
         - Complex conflicts (multiple regions): escalate

      3. If simple:
         a. Resolve conflicts (keep both changes where possible)
         b. git add resolved files
         c. git stash drop (manually drop since pop conflicted)
         d. Continue with retry

      4. If complex:
         a. Log conflict to REGRESSION_LOG.md
         b. Escalate with options:
            - User resolves conflicts manually
            - Discard stash (lose verification work)
            - Keep stash (continue without verification code)

      Rationale: Stash contains verification detection code (less critical than main codebase).
      Simple conflicts safe to auto-resolve. Complex conflicts need user judgment.
    </handling>

    <prevention>
      - Stash before ANY git operations (revert, reset)
      - Use specific stash paths if possible (git stash push -- path/to/files)
      - Keep verification code in separate files from main implementation
    </prevention>
  </edge_case>

  <edge_case id="dirty_working_tree">
    <description>
      Rollback attempted when working tree has uncommitted changes:
      - Files modified but not staged
      - Files staged but not committed
      - Untracked files present
      - Mix of all above
    </description>

    <detection>
      git status shows:
      - "Changes not staged for commit"
      - "Changes to be committed"
      - "Untracked files"

      git diff or git diff --cached produce output
    </detection>

    <handling>
      Determine rollback type and handle accordingly:

      Immediate rollback (before commit):
      1. This is EXPECTED state (task in progress)
      2. Verify changes are from current failing task
      3. Proceed with git reset --hard HEAD (intentionally discard)

      Post-commit rollback (after commit):
      1. This is UNEXPECTED (verification should run on clean tree)
      2. Stash all changes before proceeding:
         git stash push -m "Unexpected changes before rollback"
      3. Continue with git revert
      4. Pop stash after revert
      5. Log warning: "Rollback had dirty tree (unexpected)"

      Unknown state:
      1. Escalate - unclear what changes are and why
      2. User must review git status and decide
    </handling>

    <prevention>
      - Always commit after task completion
      - Run verification on committed state (not working tree)
      - Clean working tree between tasks
    </prevention>
  </edge_case>

  <edge_case id="multiple_commits_rollback">
    <description>
      Need to rollback multiple commits:
      - Single task created multiple commits (anti-pattern but possible)
      - Multiple tasks need rollback (cascading regression)
      - Want to undo entire phase or plan
    </description>

    <detection>
      Regression log shows:
      - Multiple commits in single task
      - Multiple tasks failed verification
      - User requests rollback of range
    </detection>

    <handling>
      Revert in REVERSE chronological order:

      1. Identify all commits to revert (newest to oldest):
         Example: commits C, D, E need rollback

      2. Revert in reverse order (newest first):
         git revert E --no-edit
         git revert D --no-edit
         git revert C --no-edit

      3. Rationale: Reverting newest first minimizes conflicts
         - Each revert undoes one layer
         - Earlier commits less likely to conflict
         - Matches reverse execution flow

      4. If any revert conflicts:
         - Stop immediately
         - Log which commits were successfully reverted
         - Escalate with partial rollback state
         - User decides: continue manually, reset to safe point, abort

      Alternative (RISKY - avoid if possible):
      - git revert --no-commit E D C (revert all at once)
      - Single revert commit undoes all three
      - Higher conflict risk, less granular control
      - Only use if individual reverts fail
    </handling>

    <prevention>
      - One commit per task (atomic changes)
      - Detect regressions immediately (don't accumulate)
      - If multiple commits needed, use sub-tasks with individual verification
    </prevention>
  </edge_case>

  <edge_case id="rollback_already_pushed">
    <description>
      Need to rollback commit that was already pushed to remote:
      - Task committed locally
      - Plan execution pushed commits (auto-push enabled)
      - Regression detected after push
    </description>

    <detection>
      git log shows commit exists
      git branch -vv shows "ahead 0" (up to date with remote)
      OR git log origin/main shows commit is on remote
    </detection>

    <handling>
      Use revert (NOT reset) - critical for shared branches:

      1. Verify commit was pushed:
         git log origin/[branch] --oneline | grep [commit-sha]

      2. Use standard post-commit rollback (git revert):
         - Creates new revert commit
         - Can be pushed normally (no force needed)
         - Preserves history for collaborators

      3. Push revert commit:
         git push origin [branch]

      4. Log rollback to REGRESSION_LOG.md with note:
         "Reverted pushed commit [sha] - revert also pushed"

      DO NOT use git reset --hard for pushed commits:
      - Would require force push (git push --force)
      - Destructive to collaborators
      - Loses history permanently
      - Against GSD safety principles
    </handling>

    <prevention>
      - Detect regressions before pushing (verification in execute-phase)
      - Disable auto-push until verification passes
      - Use verification gates before remote operations
    </prevention>
  </edge_case>
</edge_cases>

## Integration Points

<integration_points>
  <integration name="regression_detection">
    <trigger>regression-detection.md identifies regression</trigger>

    <flow>
      1. regression-detection.md compares baseline vs current tests
      2. Detects regression: tests that were passing now fail
      3. Calls rollback-strategy.md to determine rollback approach
      4. Passes context:
         - commit_sha: which commit to revert
         - timing: immediate (before commit) or post-commit
         - working_tree_state: clean, dirty, stashed
         - regression_details: which tests failed
    </flow>

    <output>
      rollback-strategy.md returns:
      - rollback_type: immediate or post_commit
      - git_commands: exact commands to run
      - safety_checks: what to verify after rollback
      - escalation_needed: true if conflicts or unsafe
    </output>
  </integration>

  <integration name="auto_correction">
    <trigger>auto-correction.md workflow executes rollback</trigger>

    <flow>
      1. auto-correction.md receives rollback strategy from this document
      2. Executes git commands via Bash tool
      3. Verifies rollback success (git status, baseline tests)
      4. Logs rollback to REGRESSION_LOG.md
      5. Calls retry-orchestration.md with alternative strategy
    </flow>

    <coordination>
      rollback-strategy.md provides GUIDANCE (what to do)
      auto-correction.md provides EXECUTION (how to do it)

      Separation allows:
      - Strategy reuse across workflows
      - Testing rollback logic independently
      - Documentation separate from automation
    </coordination>
  </integration>

  <integration name="retry_orchestration">
    <trigger>After rollback, need to retry with alternative</trigger>

    <flow>
      1. Rollback completes successfully
      2. Repository restored to clean state
      3. auto-correction.md classifies failure (using failure-analysis.md)
      4. Treats regression as "Validation" failure category
      5. Calls retry-orchestration.md with:
         - failure_category: Validation
         - task_type: from original task
         - attempt_number: incremented
         - prior_attempts: original approach that caused regression
      6. retry-orchestration.md selects alternative from ALTERNATIVE_PATHS.md
      7. Task re-executed with different approach
      8. Verification runs again to check for regression
    </flow>

    <retry_strategy>
      Regressions are treated as auto-retryable:
      - Max 3 attempts (same as other failures)
      - Each retry uses different implementation approach
      - If 3 attempts all cause regressions: escalate
      - User decides: fix manually, skip task, abort plan
    </retry_strategy>
  </integration>
</integration_points>

## Examples

### Example 1: Successful Post-Commit Rollback and Retry

**Scenario:** Task 2 adds new feature, Task 3 detects Task 2 broke existing tests

```xml
<example id="successful_rollback">
  <initial_state>
    - Phase 2, Plan 3 execution
    - Task 2 complete (commit abc123: "feat(02-03): add user profile API")
    - Task 3 starting: verify changes
  </initial_state>

  <detection>
    Task 3 runs verification:
    → npm test
    → FAIL: 3 tests failing in auth.test.js
    → Tests were passing in baseline (.planning/.baseline-tests.json)
    → REGRESSION DETECTED caused by commit abc123
  </detection>

  <rollback_execution>
    1. Determine rollback type:
       - Commit abc123 exists in git history
       - Type: post_commit rollback

    2. Stash current work (Task 3 verification code):
       → git stash push -m "Task 3 verification before rollback of abc123"
       → Saved ref: stash@{0}

    3. Revert problematic commit:
       → git revert abc123 --no-edit
       → New commit created: def456 "Revert 'feat(02-03): add user profile API'"
       → Output: 5 files changed, 120 insertions(+), 120 deletions(-)

    4. Restore verification work:
       → git stash pop
       → No conflicts, stash applied cleanly
       → Stash dropped

    5. Verify rollback success:
       → git status: On branch main, working tree clean (verification code restored)
       → git log -2: Shows def456 (revert) and abc123 (original)
       → npm test: All tests pass ✓ (regression resolved)
  </rollback_execution>

  <logging>
    auto-correction.md logs to REGRESSION_LOG.md:

    ```xml
    <entry>
      <timestamp>2026-01-06 15:32:10</timestamp>
      <task-id>02-03 Task 2</task-id>
      <commit-sha>abc123</commit-sha>
      <failure-description>
        Added user profile API broke 3 authentication tests.
        Tests failing: auth.test.js - "should validate token", "should refresh expired token", "should reject invalid token"
      </failure-description>
      <affected-tests>
        <test>auth.test.js: should validate token</test>
        <test>auth.test.js: should refresh expired token</test>
        <test>auth.test.js: should reject invalid token</test>
      </affected-tests>
      <rollback-action>
        git revert abc123 → commit def456 created
        Stashed and restored verification work successfully
      </rollback-action>
      <retry-strategy>
        Alternative approach: Add profile API with separate authentication context
        (isolated from existing auth module)
      </retry-strategy>
      <resolution>success</resolution>
      <learnings>
        Profile API shared auth state with existing auth module, causing interference.
        Solution: Use separate context/instance for profile operations.
      </learnings>
    </entry>
    ```
  </logging>

  <retry_execution>
    1. auto-correction.md classifies failure:
       - Category: Validation (tests failing)
       - Retry appropriate: Yes

    2. Calls retry-orchestration.md:
       - failure_category: Validation
       - task_type: feature-implementation
       - attempt_number: 2
       - prior_attempts: ["shared auth state approach"]

    3. retry-orchestration.md selects alternative from ALTERNATIVE_PATHS.md:
       - Strategy: "isolate-dependencies"
       - Approach: Create separate auth context for new feature

    4. Task 2 re-executed with isolation approach:
       → Implement profile API with separate authentication context
       → Commit: ghi789 "feat(02-03): add user profile API with isolated auth"

    5. Task 3 verification runs:
       → npm test: All tests pass ✓ (including new profile tests)
       → No regression detected
       → SUCCESS - continue to Task 4
  </retry_execution>

  <outcome>
    - Regression auto-corrected
    - 2 commits in history: abc123 (original), def456 (revert), ghi789 (fix)
    - Complete audit trail of what happened
    - Learned pattern: isolate new features from shared state
    - Total time: ~2 minutes (automatic)
  </outcome>
</example>
```

### Example 2: Rollback with Conflict (Escalation)

**Scenario:** Revert creates conflicts due to later commits modifying same files

```xml
<example id="conflict_escalation">
  <initial_state>
    - Phase 1, Plan 2 execution
    - Task 1 complete (commit aaa111: "refactor database schema")
    - Task 2 complete (commit bbb222: "add user migration")
    - Task 3 complete (commit ccc333: "update user queries")
    - Task 4 detects: Task 1 broke existing admin queries
  </initial_state>

  <detection>
    Task 4 runs verification:
    → npm test -- admin.test.js
    → FAIL: 5 admin tests failing
    → Tests were passing in baseline
    → REGRESSION DETECTED caused by commit aaa111 (Task 1)
  </detection>

  <rollback_execution>
    1. Determine rollback type:
       - Commit aaa111 in history
       - Type: post_commit rollback

    2. Stash current work:
       → git stash push -m "Task 4 verification before rollback of aaa111"
       → Saved

    3. Attempt revert:
       → git revert aaa111 --no-edit
       → ERROR: Conflict in files:
         - src/database/schema.sql (modified by bbb222 and ccc333)
         - src/models/User.js (modified by ccc333)
       → git status: "You are currently reverting commit aaa111"
       → Revert FAILED - conflicts present

    4. Detect conflict:
       → Exit code: 1
       → Output contains "CONFLICT"
       → Conflicted files in git status
  </rollback_execution>

  <conflict_handling>
    auto-correction.md detects conflict and escalates:

    1. Abort the conflicted revert:
       → git revert --abort
       → Repository returned to pre-revert state

    2. Restore stashed work:
       → git stash pop
       → Verification code restored

    3. Log conflict to REGRESSION_LOG.md:
       ```xml
       <entry>
         <timestamp>2026-01-06 16:15:30</timestamp>
         <task-id>01-02 Task 1</task-id>
         <commit-sha>aaa111</commit-sha>
         <failure-description>
           Database schema refactor broke 5 admin queries.
           Cannot auto-rollback due to conflicts with later commits.
         </failure-description>
         <rollback-action>
           git revert aaa111 → CONFLICT
           Conflicted files: src/database/schema.sql, src/models/User.js
           Modified by later commits: bbb222 (migration), ccc333 (queries)
         </rollback-action>
         <resolution>escalated</resolution>
         <escalation-reason>
           Revert conflicts with commits bbb222 and ccc333 that modified same files.
           Manual resolution required.
         </escalation-reason>
       </entry>
       ```

    4. Create checkpoint:decision:
       ═══════════════════════════════════════════
       ROLLBACK CONFLICT: Database Schema Refactor
       ═══════════════════════════════════════════

       Regression detected in Task 1 (commit aaa111) but cannot auto-rollback.

       Issue: Task 1 refactored database schema, breaking admin queries.
              Later tasks (2 and 3) modified same files.

       Conflict details:
         - src/database/schema.sql: Modified by Task 2 (migration) and Task 3 (queries)
         - src/models/User.js: Modified by Task 3 (query updates)

       Options:
         [1] Manual revert - I'll resolve conflicts and revert Task 1
         [2] Revert Tasks 3, 2, then 1 - Rollback all three in order
         [3] Fix forward - Keep all commits, fix admin queries in new commit
         [4] Abort plan - Stop and review approach

       Recommendation: Option 3 (fix forward) - Tasks 2 and 3 are correct,
       only Task 1 needs adjustment to preserve admin query compatibility.
  </conflict_handling>

  <outcome>
    - Auto-rollback FAILED due to conflicts
    - Escalated to user with clear context
    - No commits reverted (all history preserved)
    - User chooses option 3: fix forward
    - Creates Task 5: "Fix admin queries for new schema"
    - Regression resolved in ~10 minutes (5 min detect + escalate, 5 min user fix)
  </outcome>
</example>
```

### Example 3: Rollback of Multiple Commits

**Scenario:** Three sequential tasks need rollback

```xml
<example id="multiple_commits">
  <initial_state>
    - Phase 3, Plan 1 execution
    - Task 1 complete (commit xxx111: "add payment API")
    - Task 2 complete (commit xxx222: "add payment UI")
    - Task 3 complete (commit xxx333: "add payment tests")
    - Task 4 detects: ALL payment features broke existing checkout flow
  </initial_state>

  <detection>
    Task 4 runs verification:
    → npm test -- checkout.test.js
    → FAIL: Entire checkout flow broken
    → Tests were passing in baseline
    → REGRESSION DETECTED caused by payment feature (Tasks 1-3)
  </detection>

  <rollback_execution>
    Need to rollback all three commits: xxx333, xxx222, xxx111

    1. Stash verification work:
       → git stash push -m "Task 4 verification before multi-commit rollback"

    2. Revert in reverse order (newest first):

       Revert Task 3:
       → git revert xxx333 --no-edit
       → Success: commit yyy333 "Revert 'add payment tests'"

       Revert Task 2:
       → git revert xxx222 --no-edit
       → Success: commit yyy222 "Revert 'add payment UI'"

       Revert Task 1:
       → git revert xxx111 --no-edit
       → Success: commit yyy111 "Revert 'add payment API'"

    3. Restore verification work:
       → git stash pop
       → Success

    4. Verify rollback:
       → npm test -- checkout.test.js
       → All tests pass ✓
       → Regression resolved
  </rollback_execution>

  <logging>
    Log to REGRESSION_LOG.md:

    ```xml
    <entry>
      <timestamp>2026-01-06 17:20:45</timestamp>
      <task-id>03-01 Tasks 1-3 (payment feature)</task-id>
      <commit-sha>xxx111, xxx222, xxx333</commit-sha>
      <failure-description>
        Payment feature (API + UI + tests) broke checkout flow.
        All checkout tests failing after payment integration.
      </failure-description>
      <rollback-action>
        Multi-commit rollback (3 commits):
        - git revert xxx333 → yyy333
        - git revert xxx222 → yyy222
        - git revert xxx111 → yyy111
        All reverts successful, no conflicts.
      </rollback-action>
      <retry-strategy>
        Alternative approach: Implement payment as separate module
        (not integrated into checkout flow initially)
      </retry-strategy>
      <resolution>success</resolution>
      <learnings>
        Payment integration affected checkout state management.
        Should implement payment in isolation first, integrate after verification.
      </learnings>
    </entry>
    ```
  </logging>

  <retry_execution>
    1. All three tasks need retry with different approach
    2. auto-correction.md selects alternative strategy:
       - Implement payment module standalone
       - Create separate payment flow (not integrated)
       - Verify payment works independently
       - THEN integrate with checkout (separate task)

    3. Tasks 1-3 re-executed:
       → Task 1: Payment API as standalone module
       → Task 2: Payment UI as separate page
       → Task 3: Payment tests (isolated)
       → All verify successfully

    4. New integration task created:
       → Task 5: Integrate payment with checkout
       → Incremental integration with verification at each step
       → Checkout tests pass throughout
  </retry_execution>

  <outcome>
    - 3 commits reverted successfully
    - 6 new commits (3 reverts + 3 new implementations)
    - Complete history preserved
    - Learned: isolate features before integration
    - Auto-corrected in ~5 minutes
  </outcome>
</example>
```

## Quick Reference

<quick_reference>
  <scenario name="Before commit - task in progress">
    <command>git reset --hard HEAD</command>
    <safety>Moderate - loses uncommitted work</safety>
    <when>Verification fails during task, no commit yet</when>
  </scenario>

  <scenario name="After commit - regression detected">
    <commands>
      git stash push -m "Work before rollback"
      git revert [commit-sha] --no-edit
      git stash pop
    </commands>
    <safety>Safe - preserves all history</safety>
    <when>Task committed, later task detects regression</when>
  </scenario>

  <scenario name="Conflict during revert">
    <commands>
      git revert --abort
      git stash pop
    </commands>
    <safety>Safe - returns to pre-revert state</safety>
    <when>git revert fails with CONFLICT</when>
    <next>Escalate to user with conflict details</next>
  </scenario>

  <scenario name="Multiple commits rollback">
    <commands>
      git stash push -m "Work before rollback"
      git revert [newest-commit] --no-edit
      git revert [middle-commit] --no-edit
      git revert [oldest-commit] --no-edit
      git stash pop
    </commands>
    <safety>Safe if no conflicts</safety>
    <when>Multiple tasks need rollback</when>
    <order>Newest to oldest (reverse chronological)</order>
  </scenario>
</quick_reference>

---

*Safe rollback patterns for GSD automatic regression recovery*
