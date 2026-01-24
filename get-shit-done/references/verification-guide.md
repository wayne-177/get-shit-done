<overview>

Comprehensive guide to the GSD Continuous Verification and Self-Correction system (Phase 2 Enhancement).

This system provides automatic regression detection, safe rollback, and intelligent auto-correction during plan execution. It catches breaking changes immediately after each task, rolls back problematic commits, and retries with alternative approaches.

**Key benefits:**
- Catch regressions within seconds of introduction
- Automatic rollback via git revert
- Self-healing execution via Phase 1 retry integration
- Zero-intervention recovery for most issues
- Full audit trail in HEALTH_STATUS.md and REGRESSION_LOG.md

</overview>

<enabling_verification>

## Setting Up Verification

Verification is opt-in. Enable it by configuring `.planning/config.json`:

### Step 1: Open config.json

```bash
# Located at:
.planning/config.json
```

### Step 2: Add verification configuration

Add or update the `verification` object:

```json
{
  "workflow": {
    "mode": "yolo",
    "depth": "standard"
  },
  "verification": {
    "enabled": true,
    "regression_check": true,
    "flaky_test_retries": 1,
    "treat_flaky_as_regression": false,
    "baseline_tests": ".planning/.baseline-tests.json",
    "baseline_build": ".planning/.baseline-build.json",
    "auto_update_baseline": false,
    "auto_rollback": true,
    "max_rollback_attempts": 3,
    "skip_build": false,
    "skip_tests": false
  },
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "strategy_file": ".planning/ALTERNATIVE_PATHS.md"
  }
}
```

### Step 3: Customize settings (optional)

**Core settings:**
- `enabled`: Set to `true` to activate verification system
- `regression_check`: Enable regression detection (recommended: `true`)
- `auto_rollback`: Automatically revert commits that introduce regressions (recommended: `true`)

**Flaky test handling:**
- `flaky_test_retries`: Number of retries for failing tests (default: `1`)
- `treat_flaky_as_regression`: Treat flaky tests as regressions (default: `false`)

**Baseline files:**
- `baseline_tests`: Path to test baseline JSON (default: `.planning/.baseline-tests.json`)
- `baseline_build`: Path to build baseline JSON (default: `.planning/.baseline-build.json`)
- `auto_update_baseline`: Auto-update baseline after successful plans (default: `false`)

**Safety limits:**
- `max_rollback_attempts`: Max rollback retries before escalating (default: `3`)

**Performance optimization:**
- `skip_build`: Skip build checks (faster, tests only) (default: `false`)
- `skip_tests`: Skip test checks (build only) (default: `false`)

### Step 4: Capture initial baseline

Before starting work, establish a clean baseline:

```bash
# Run health check to capture initial baseline
/gsd:health-check

# This creates:
# - .planning/HEALTH_STATUS.md (dashboard)
# - .planning/.baseline-tests.json (test baseline)
# - .planning/.baseline-build.json (build baseline)
```

### Step 5: Verify setup

Confirm verification is active:

```bash
# Check config
cat .planning/config.json | grep -A 5 "verification"

# Verify baselines exist
ls -la .planning/.baseline-*.json

# Check health status dashboard
cat .planning/HEALTH_STATUS.md
```

You're ready! Verification will now run automatically during plan execution.

</enabling_verification>

<how_it_works>

## Verification Flow

The verification system operates at three points during plan execution:

### 1. Pre-Plan: Baseline Capture

**When:** Before starting any task in the plan
**Purpose:** Establish known-good state for regression detection

**Process:**
```
1. Check if verification.enabled = true
2. Load health-check.md workflow
3. Run full test suite → capture results
4. Run build → capture metrics
5. Save to .planning/.baseline-tests.json and .baseline-build.json
6. Update HEALTH_STATUS.md with clean baseline
```

**If baseline fails:**
- Display warning: "Current state has failures - cannot establish clean baseline"
- User must fix existing issues before starting plan
- Execution blocked until health check passes

### 2. Post-Task: Regression Check

**When:** Immediately after each task commit
**Purpose:** Detect regressions introduced by the task

**Process:**
```
1. Task completes → files modified → commit created
2. Load regression-detection.md algorithm
3. Run tests → compare against baseline
4. Run build → compare against baseline
5. Analyze results:
   - New failures? → Check if flaky (retry)
   - Still failing? → Mark as regression
   - Baseline tests now fail? → Regression detected
6. If regression detected → trigger rollback workflow
7. Update HEALTH_STATUS.md and REGRESSION_LOG.md
```

**Fast path optimization:**
If `skip_build = true`, only runs tests (faster iteration).

### 3. Post-Plan: Final Verification

**When:** After all tasks complete, before metadata commit
**Purpose:** Final safety check before marking plan complete

**Process:**
```
1. All tasks committed
2. Run full health check (tests + build + git status)
3. Verify overall status = HEALTHY
4. Update HEALTH_STATUS.md with final metrics
5. If status = FAILING → warn user, allow manual intervention
6. Create SUMMARY.md with verification metrics
7. Metadata commit (SUMMARY + STATE + ROADMAP)
```

**Three verification points ensure:**
- Clean starting state (pre-plan baseline)
- Immediate regression detection (post-task)
- Final safety gate (post-plan)

</how_it_works>

<verification_points>

## Three-Tier Verification Architecture

### Tier 1: Pre-Plan Baseline (Safety Gate)

**Goal:** Ensure clean starting state

**Checks:**
- All tests passing
- Build succeeds
- Git working directory clean
- No existing regressions

**If fails:** Block execution, require user to fix

**Output:** `.planning/.baseline-tests.json`, `.planning/.baseline-build.json`

### Tier 2: Post-Task Regression Check (Fast Feedback)

**Goal:** Catch regressions immediately after introduction

**Checks:**
- Compare current tests vs baseline
- Identify new test failures
- Detect flaky tests (retry logic)
- Check build still succeeds

**If fails:** Trigger auto-rollback and retry

**Output:** Entry in `.planning/REGRESSION_LOG.md`, update to `HEALTH_STATUS.md`

### Tier 3: Post-Plan Final Verification (Comprehensive)

**Goal:** Comprehensive health check before plan completion

**Checks:**
- Full test suite
- Full build
- Git status
- Regression count
- Flaky test count

**If fails:** Warn user, include metrics in SUMMARY.md

**Output:** Updated `HEALTH_STATUS.md`, metrics in `SUMMARY.md`

**Why three tiers?**
- Tier 1: Prevent building on broken foundation
- Tier 2: Fast feedback loop (seconds after task)
- Tier 3: Comprehensive safety net before completion

</verification_points>

<regression_detection>

## Regression Detection Examples

### Example 1: Regression Auto-Corrected

**Scenario:** Task modifies authentication logic, breaks existing test

**Execution:**
```
Task 2: Refactor user login endpoint
- Modified: src/api/auth/login.ts
- Commit: abc123f

POST-TASK VERIFICATION:
Running tests... 41/42 passing (1 new failure)

REGRESSION DETECTED:
  Test: "User authentication > login with valid credentials"
  Error: Cannot read property 'validate' of undefined
  File: tests/auth.test.js

  Baseline: PASSING
  Current:  FAILING

  → Confirmed regression (not flaky - failed 2/2 retries)

ROLLBACK INITIATED:
Strategy: git revert abc123f
Reverted commit: abc123f
Tests after revert: 42/42 passing ✓

RETRY WITH ALTERNATIVE:
Loading ALTERNATIVE_PATHS.md...
Alternative approach: "Use different validation method"
Retrying task 2 with alternative approach...

RETRY RESULT:
Task 2 completed (attempt 2/3)
Commit: def456g
Tests: 42/42 passing ✓
No regressions detected ✓

AUTO-CORRECTION: SUCCESS
```

**Logged to REGRESSION_LOG.md:**
```markdown
### 2026-01-06 15:30:00 - Regression Auto-Corrected

**Task:** Task 2: Refactor user login endpoint
**Commit:** abc123f (reverted)
**Regression:** "User authentication > login with valid credentials" - Cannot read property 'validate' of undefined
**Rollback:** git revert abc123f (successful)
**Retry:** Attempt 2/3 with alternative approach
**Resolution:** def456g - Tests passing, regression fixed
**Duration:** 45 seconds (detection → rollback → retry → success)
```

### Example 2: Regression Escalated After 3 Retries

**Scenario:** Task consistently breaks tests, no working alternative found

**Execution:**
```
Task 3: Add pagination to user list
- Modified: src/api/users.ts
- Commit: ghi789j

POST-TASK VERIFICATION:
Regression detected: 2 tests failing

ATTEMPT 1: Rollback + retry
Result: Same regression (commit klm012n reverted)

ATTEMPT 2: Rollback + alternative approach
Result: Different error (commit opq345r reverted)

ATTEMPT 3: Rollback + simplified approach
Result: Still failing (commit stu678v reverted)

ESCALATION:
Max retry attempts (3) reached
Cannot auto-correct regression

USER INTERVENTION REQUIRED:
Task 3 failed after 3 attempts
Regressions detected:
  1. "API > GET /users should return 200"
  2. "API > GET /users should paginate results"

Recommendations:
  - Review pagination implementation approach
  - Check API route configuration
  - Test manually with curl
  - Consider breaking into smaller tasks

PLAN EXECUTION: PAUSED
Waiting for user to fix issues or provide guidance
```

**Logged to REGRESSION_LOG.md:**
```markdown
### 2026-01-06 16:15:00 - Regression Escalated

**Task:** Task 3: Add pagination to user list
**Attempts:** 3 (all failed)
**Regressions:**
  - "API > GET /users should return 200"
  - "API > GET /users should paginate results"
**Rollbacks:** 3 (ghi789j, klm012n, opq345r, stu678v all reverted)
**Status:** ESCALATED - requires user intervention
**Next Steps:** User must manually fix or provide alternative approach
```

### Example 3: False Positive (Flaky Test)

**Scenario:** Test fails intermittently due to timing issue, not actual regression

**Execution:**
```
Task 4: Update WebSocket connection handling
- Modified: src/lib/websocket.ts
- Commit: wxy901z

POST-TASK VERIFICATION:
Tests: 41/42 passing (1 failure)

POTENTIAL REGRESSION:
  Test: "WebSocket > should reconnect after disconnect"
  Error: Timeout waiting for reconnection

FLAKY TEST DETECTION:
Retrying test (attempt 1/1)...
Result: PASSING ✓

ANALYSIS:
  Attempt 1: FAIL
  Attempt 2: PASS
  → Flaky test detected (timing issue)
  → NOT a regression

LOGGED AS FLAKY:
  Test: "WebSocket > should reconnect after disconnect"
  Reason: Passed on retry
  Recommendation: Add proper wait conditions or increase timeout

VERIFICATION: PASSED
Tests: 42/42 passing (after retry)
No regressions confirmed
Flaky test logged to HEALTH_STATUS.md
```

**Logged to HEALTH_STATUS.md:**
```markdown
### 2026-01-06 17:00:00 - Post-Task Verification

**Trigger:** After Task 4: Update WebSocket connection handling
**Outcome:** ⚠ Warning - 1 flaky test detected
**Tests:** 42 passing (1 flaky on first run)
**Flaky Test:** "WebSocket > should reconnect after disconnect"
**Recommendation:** Fix flaky test - add proper wait conditions
**Regression:** None (flaky test passed on retry)
```

**Why this matters:**
- Flaky tests are warnings, not blockers
- No rollback triggered (not a real regression)
- User can continue work
- Issue logged for future fix

</regression_detection>

<health_monitoring>

## Health Status Dashboard

### HEALTH_STATUS.md Structure

Located at `.planning/HEALTH_STATUS.md`, automatically maintained:

```markdown
# Project Health Status

**Last Updated:** 2026-01-06 17:30:00
**Overall Status:** ✓ HEALTHY

## Health Checks

- [x] **Build Status** - Project builds without errors
- [x] **Test Suite** - All tests passing (42/42)
- [x] **Git Status** - Working directory clean
- [x] **Dependencies** - All packages installed
- [x] **Configuration** - Config files valid

## Metrics

### Test Coverage
- **Total Tests:** 42
- **Passing:** 42
- **Failing:** 0
- **Pass Rate:** 100%

### Build Performance
- **Last Build Time:** 3.2s
- **Build Status:** success
- **Warnings:** 2
- **Errors:** 0

### Verification History
- **Last Successful Verification:** 2026-01-06 17:30:00
- **Regressions Detected (Last 24h):** 1 (auto-corrected)
- **Flaky Tests Identified:** 1

## Recent Activity

### 2026-01-06 17:30:00 - Post-Plan Verification
- **Trigger:** Plan 02-04 completion
- **Outcome:** ✓ All checks passed
- **Tests:** 42 passing, 0 failing
- **Build:** Success in 3.2s
- **Notes:** Plan completed without regressions

[... last 5 activity entries ...]
```

### Using /gsd:health-check Command

**On-demand health verification:**

```bash
# Run manual health check
/gsd:health-check

# Output:
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: ✓ HEALTHY

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHECKS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Build Status       - Success in 3.2s
✓ Test Suite         - 42/42 passing (100%)
✓ Git Status         - Working directory clean
✓ Dependencies       - All packages installed
✓ Configuration      - Valid

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METRICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tests:           42 passing, 0 failing, 0 skipped
Build:           3.2s, 2 warnings, 0 errors
Regressions:     0 detected
Flaky Tests:     0 detected

Dashboard updated: .planning/HEALTH_STATUS.md
```

**When to run health checks:**
- Before starting work on a new phase
- After completing tasks to verify no regressions
- When suspecting project health issues
- Before capturing a new baseline
- Periodically to maintain visibility

</health_monitoring>

<rollback_safety>

## Rollback Strategy

### Git Revert Approach

Verification uses `git revert` (not `git reset`) for safety:

**Why git revert?**
- Creates new commit that undoes changes (non-destructive)
- Preserves full history (audit trail)
- Safe for shared branches
- Can be reverted again if needed

**Rollback process:**
```bash
# After regression detected on commit abc123f

# 1. Revert the problematic commit
git revert abc123f --no-edit

# 2. Verify rollback succeeded
npm test  # Should match baseline

# 3. If tests pass after revert
# → Regression confirmed (commit was the cause)
# → Proceed to retry with alternative

# 4. If tests still fail after revert
# → Regression existed before this commit
# → Rollback failed, escalate to user
```

### When Rollback Occurs

**Automatic rollback triggers:**
1. **Post-task regression detected:** Tests fail that were passing in baseline
2. **Build failure:** Build succeeds in baseline, fails after task
3. **New test failures:** Tests pass in baseline, fail after task

**Rollback does NOT occur for:**
- Flaky tests (pass on retry)
- Pre-existing failures (already failing in baseline)
- New tests failing (not in baseline)
- Build warnings (not errors)

### Rollback Safety Checks

**Before rollback:**
- Verify commit is the most recent
- Ensure working directory is clean
- Confirm baseline exists

**During rollback:**
- Capture rollback commit hash
- Log to REGRESSION_LOG.md
- Update HEALTH_STATUS.md

**After rollback:**
- Re-run verification
- Confirm baseline state restored
- If not restored → escalate (rollback failed)

**Safety limits:**
- Max rollback attempts: 3 (configurable)
- After 3 failed rollbacks → escalate to user
- Each rollback attempt logged with details

</rollback_safety>

<integration_with_retry>

## Phase 2 ↔ Phase 1 Integration

Verification (Phase 2) triggers Retry (Phase 1) for automatic recovery:

### Flow: Regression → Rollback → Retry

```
1. TASK EXECUTES
   Task 2: Refactor authentication
   Commit: abc123f

2. REGRESSION DETECTED (Phase 2)
   Verification finds test failures

3. ROLLBACK INITIATED (Phase 2)
   git revert abc123f
   Tests restored to baseline ✓

4. RETRY TRIGGERED (Phase 1)
   Load retry-orchestration.md
   Load ALTERNATIVE_PATHS.md

5. ALTERNATIVE SELECTED (Phase 1)
   Strategy: Use different auth library

6. RETRY EXECUTION (Phase 1)
   Re-attempt Task 2 with alternative
   Commit: def456g

7. REGRESSION CHECK (Phase 2)
   Verify new commit doesn't regress
   Tests passing ✓

8. SUCCESS
   Task completed on attempt 2/3
   Logged to REGRESSION_LOG.md
```

### Unified Configuration

Both systems use `.planning/config.json`:

```json
{
  "verification": {
    "enabled": true,
    "auto_rollback": true,
    "max_rollback_attempts": 3
  },
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "strategy_file": ".planning/ALTERNATIVE_PATHS.md"
  }
}
```

**How they work together:**

**Phase 2 (Verification):**
- Detects regression
- Rolls back problematic commit
- Passes control to retry orchestration

**Phase 1 (Retry):**
- Loads alternative approaches from ALTERNATIVE_PATHS.md
- Selects next strategy
- Re-attempts task with alternative
- Returns control to verification

**Phase 2 (Verification again):**
- Checks if retry succeeded
- If still regresses → trigger another retry
- If max retries exceeded → escalate to user

### Alternative Paths Example

`.planning/ALTERNATIVE_PATHS.md`:

```markdown
## Task: Refactor authentication

### Attempt 1: Inline validation
Status: FAILED (regression detected)
Issue: Cannot read property 'validate' of undefined

### Attempt 2: Use auth helper library
Status: TRYING
Approach: Import validation from @/lib/auth-helpers
Rationale: Centralized validation logic, already tested

### Attempt 3: Simplified validation
Approach: Basic email/password check without external dependencies
Rationale: Minimal change, lower regression risk
```

**Retry orchestration uses this to:**
1. Understand what failed (Attempt 1)
2. Select next approach (Attempt 2)
3. Execute with alternative strategy
4. Fall back further if needed (Attempt 3)

</integration_with_retry>

<troubleshooting>

## Common Issues and Solutions

### Issue 1: False Positives from Flaky Tests

**Symptom:** Tests fail intermittently, rollback triggered unnecessarily

**Cause:** Timing issues, race conditions, network-dependent tests

**Solution:**
```json
// Increase flaky test retries in config.json
{
  "verification": {
    "flaky_test_retries": 2,  // Retry up to 2 times
    "treat_flaky_as_regression": false  // Don't block on flaky tests
  }
}
```

**Better solution:** Fix flaky tests
- Add proper wait conditions (waitFor, waitUntil)
- Mock network requests
- Use deterministic test data
- Increase timeouts for slow operations

### Issue 2: Slow Tests Slowing Down Verification

**Symptom:** Verification takes too long after each task

**Cause:** Large test suite, slow integration tests

**Solution 1 - Skip build checks:**
```json
{
  "verification": {
    "skip_build": true  // Only run tests, skip build
  }
}
```

**Solution 2 - Optimize test suite:**
- Run unit tests only for quick feedback
- Move slow tests to separate suite
- Use test.only during development
- Run full suite only in pre-plan and post-plan

**Solution 3 - Selective verification:**
```json
{
  "verification": {
    "regression_check": true,  // Only check for regressions
    "skip_build": true  // Skip build during post-task checks
  }
}
```

### Issue 3: Git Conflicts During Rollback

**Symptom:** `git revert` fails with merge conflicts

**Cause:** Subsequent tasks modified same files as reverted commit

**Solution:** Manual intervention required
```bash
# Rollback failed, resolve conflicts manually:

1. Check conflict status:
   git status

2. Resolve conflicts in affected files

3. Complete the revert:
   git add <resolved-files>
   git revert --continue

4. Re-run verification:
   npm test

5. If baseline restored → continue with retry
   If still failing → investigate further
```

**Prevention:**
- Keep tasks small and focused (fewer conflicts)
- Atomic commits per task (easier to revert)
- Avoid modifying same files in consecutive tasks

### Issue 4: Baseline Becomes Outdated

**Symptom:** Verification detects "regressions" that are actually intentional changes

**Cause:** Baseline captured long ago, project evolved

**Solution:** Update baseline periodically
```bash
# After confirming tests/build are correct:

1. Run health check:
   /gsd:health-check

2. Verify all checks pass (HEALTHY status)

3. Update baseline:
   # Manually in config.json:
   "auto_update_baseline": true

   # Or copy current results:
   cp .planning/.current-tests.json .planning/.baseline-tests.json
   cp .planning/.current-build.log .planning/.baseline-build.json

4. Verify new baseline:
   /gsd:health-check
   # Should show: "Baseline updated: .planning/.baseline-tests.json"
```

**Recommendation:** Update baseline after:
- Completing major milestones
- Intentionally changing test suite
- Upgrading dependencies that affect tests
- Refactoring test infrastructure

### Issue 5: Verification Disabled But Still Referenced

**Symptom:** Error messages about missing verification files

**Cause:** `verification.enabled = false` but execute-phase still tries to load verification resources

**Solution:** Check conditional loading
```bash
# Verify config.json:
cat .planning/config.json | grep -A 3 "verification"

# Should show:
{
  "verification": {
    "enabled": false
  }
}

# If enabled = false, verification should not load resources
# If error persists, check execute-phase.md for proper conditional:
# "If config.verification.enabled = true: load verification resources"
```

**Expected behavior:**
- `enabled: false` → No verification, no resource loading, no errors
- `enabled: true` → Full verification with all resources loaded

</troubleshooting>

<anti_patterns>

## What NOT to Do

### Anti-Pattern 1: Disable Auto-Rollback on Critical Projects

**Bad:**
```json
{
  "verification": {
    "enabled": true,
    "auto_rollback": false  // ❌ Defeats the purpose
  }
}
```

**Why bad:** Rollback is the safety mechanism. Without it, you detect regressions but don't recover automatically.

**Do instead:**
```json
{
  "verification": {
    "enabled": true,
    "auto_rollback": true,  // ✓ Let system recover automatically
    "max_rollback_attempts": 3  // Safety limit prevents infinite loops
  }
}
```

### Anti-Pattern 2: Ignore REGRESSION_LOG.md

**Bad:** Never reviewing regression log, missing patterns

**Why bad:** Regression log reveals systemic issues:
- Same test failing repeatedly → Flaky test that needs fixing
- Same file causing regressions → Problematic code area
- Frequent rollbacks → Tasks too large or complex

**Do instead:**
- Review REGRESSION_LOG.md after each plan
- Identify patterns (which tests are flaky, which areas fragile)
- Fix root causes (stabilize flaky tests, refactor fragile code)
- Update ALTERNATIVE_PATHS.md with learnings

### Anti-Pattern 3: Auto-Update Baseline After Every Plan

**Bad:**
```json
{
  "verification": {
    "auto_update_baseline": true  // ❌ Too aggressive
  }
}
```

**Why bad:**
- Can lock in regressions if plan completed despite issues
- Loses reference to last known-good state
- Makes regression detection useless (always comparing to latest)

**Do instead:**
```json
{
  "verification": {
    "auto_update_baseline": false,  // ✓ Manual control
  }
}
```

Update baseline deliberately:
- After completing major milestones
- After verifying health check shows HEALTHY
- Never when warnings or failures present

### Anti-Pattern 4: Treat Flaky Tests as Regressions

**Bad:**
```json
{
  "verification": {
    "flaky_test_retries": 0,  // ❌ No retry tolerance
    "treat_flaky_as_regression": true  // ❌ Blocks on flaky tests
  }
}
```

**Why bad:**
- Flaky tests trigger unnecessary rollbacks
- Slows down development
- Doesn't fix the underlying flaky test issue

**Do instead:**
```json
{
  "verification": {
    "flaky_test_retries": 1,  // ✓ Retry once to detect flakiness
    "treat_flaky_as_regression": false  // ✓ Log but don't block
  }
}
```

Then fix flaky tests properly:
- Add to ISSUES.md for future fix
- Improve test stability (proper waits, mocks)
- Remove flaky tests from critical path if unfixable

### Anti-Pattern 5: Skip Verification on "Small" Changes

**Bad thinking:** "This task is tiny, I'll disable verification to save time"

**Why bad:**
- Small changes often have surprising side effects
- Regression detection is fastest on small changes (fewer tests affected)
- Skipping verification defeats the purpose of continuous verification

**Do instead:**
- Keep verification enabled always
- Optimize test suite if verification is slow
- Use `skip_build: true` for faster feedback if needed
- Trust the system - it's designed for continuous use

### Anti-Pattern 6: Massive Test Suites Without Optimization

**Bad:** 10,000 tests taking 10 minutes to run after each task

**Why bad:**
- Verification becomes bottleneck
- Slows down development significantly
- Tempts developers to disable verification

**Do instead:**
- Split test suites (unit vs integration vs e2e)
- Run fast unit tests for post-task verification
- Run full suite only pre-plan and post-plan
- Parallelize test execution
- Consider test sharding for very large suites

**Configuration for large projects:**
```json
{
  "verification": {
    "enabled": true,
    "skip_build": true,  // Save time on post-task checks
    "fast_test_command": "npm run test:unit",  // Fast subset
    "full_test_command": "npm test"  // Complete suite for pre/post-plan
  }
}
```

### Anti-Pattern 7: Not Using Alternative Paths

**Bad:** Let retry system fail 3 times with same approach

**Why bad:**
- Wastes time retrying same failing approach
- Doesn't leverage Phase 1 retry intelligence
- Leads to escalation when alternative approaches exist

**Do instead:**
- Maintain ALTERNATIVE_PATHS.md with strategies
- Document what failed and why
- Provide 2-3 alternative approaches for critical tasks
- Let retry orchestration use alternatives automatically

**Example ALTERNATIVE_PATHS.md:**
```markdown
## Authentication Tasks

### Approach 1: JWT with jose library
Status: DEFAULT
Works for: Standard auth flows
Issues: Edge runtime compatibility

### Approach 2: JWT with jsonwebtoken
Status: FALLBACK
Works for: Node.js runtime only
Issues: CommonJS import issues with ESM

### Approach 3: Session-based auth
Status: LAST_RESORT
Works for: Always (server-side sessions)
Issues: Requires database, not stateless
```

</anti_patterns>

