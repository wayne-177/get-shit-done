# Health Check Workflow

This workflow provides on-demand project health verification with comprehensive status reporting and optional baseline updates.

## Purpose

<purpose>
The health check workflow performs a full verification of project health by:
1. Running the test suite and capturing results
2. Executing builds and recording metrics
3. Checking git repository status
4. Calculating health metrics (coverage, test counts, build times)
5. Updating the HEALTH_STATUS.md dashboard
6. Reporting consolidated results to the user

This workflow can be invoked manually (via future `/gsd:health-check` command) or embedded in other workflows like execute-phase.md for pre-plan verification checks.
</purpose>

## When to Use

<use_cases>
  <case>Before starting work on a new phase to establish clean baseline</case>
  <case>After completing tasks to verify no regressions introduced</case>
  <case>When suspecting project health issues (flaky tests, build problems)</case>
  <case>Periodically to maintain visibility into project status</case>
  <case>Before capturing a new baseline to ensure current state is clean</case>
</use_cases>

## Prerequisites

<prerequisites>
  <required>
    - `.planning/config.json` exists with verification settings
    - Project has test script defined in package.json (`npm test`)
    - Project has build script defined in package.json (`npm run build`) OR health check runs with --skip-build flag
  </required>

  <optional>
    - Existing HEALTH_STATUS.md at `.planning/HEALTH_STATUS.md` (will be created if missing)
    - Existing baselines at `.planning/.baseline-tests.json` and `.planning/.baseline-build.json` (for regression detection)
  </optional>
</prerequisites>

## Process

<step name="load_configuration" number="1">
**Load verification configuration**

Read settings from `.planning/config.json`:

**Process:**
1. Check if `.planning/config.json` exists
2. Parse JSON and extract verification settings
3. Validate required fields

**Expected structure:**
```json
{
  "verification": {
    "enabled": true,
    "regression_check": true,
    "flaky_test_retries": 1,
    "baseline_tests": ".planning/.baseline-tests.json",
    "baseline_build": ".planning/.baseline-build.json",
    "auto_update_baseline": false
  }
}
```

**Configuration variables:**
```xml
<config>
  <enabled>true | false</enabled>
  <regression_check>true | false</regression_check>
  <flaky_test_retries>number (default: 1)</flaky_test_retries>
  <baseline_tests>path to test baseline</baseline_tests>
  <baseline_build>path to build baseline</baseline_build>
  <auto_update_baseline>true | false</auto_update_baseline>
</config>
```

**Error handling:**
- If config.json missing: Use default values, warn user
- If config.verification missing: Assume enabled=false, skip verification
- If config.verification.enabled = false: Display warning "Verification disabled - enable in config.json to run health checks"

**Decision:**
- If enabled = false: Exit workflow with message
- If enabled = true: Continue to step 2

</step>

<step name="initialize_status_file" number="2">
**Ensure HEALTH_STATUS.md exists**

Check if health status dashboard exists, create if missing:

**Process:**
1. Check if `.planning/HEALTH_STATUS.md` exists
2. If missing:
   - Copy template from `@get-shit-done/templates/health-status.md`
   - Update initial timestamp
   - Set overall status to "UNKNOWN" (pending first check)
3. If exists:
   - Load current content for updating

**Output:**
```xml
<status_file_state>
  <exists>true | false</exists>
  <location>.planning/HEALTH_STATUS.md</location>
  <action>created | loaded</action>
</status_file_state>
```

**Error handling:**
- If template missing: Create minimal dashboard with basic structure
- If permission denied: Error and exit with message
- If .planning/ directory missing: Create it first

</step>

<step name="run_tests" number="3">
**Execute test suite and capture results**

Run project tests and parse output:

**Process:**
1. Execute test command: `npm test -- --json 2>&1 | tee .planning/.current-tests.log`
2. Capture exit code, stdout, stderr
3. Parse test results:
   - Total test count
   - Passing tests
   - Failing tests
   - Skipped tests
   - Test duration
   - Individual test results (if JSON format available)

**Test execution:**
```bash
# Try JSON output first (Jest, Mocha with json-reporter)
npm test -- --json > .planning/.current-tests.json 2>&1

# If --json not supported, capture plain output
npm test 2>&1 | tee .planning/.current-tests.log
```

**Parse output:**
```xml
<test_results>
  <timestamp>2026-01-06T14:30:00Z</timestamp>
  <exit_code>0 | 1</exit_code>
  <total_tests>42</total_tests>
  <passing>40</passing>
  <failing>2</failing>
  <skipped>0</skipped>
  <duration_ms>4523</duration_ms>
  <status>success | failure</status>
  <details>
    [Array of individual test results if JSON available]
  </details>
</test_results>
```

**Error handling:**
- If npm test fails: Still capture results, mark as failing
- If no test script: Skip test check, mark as "N/A"
- If timeout: Capture partial results, note timeout in status
- If parsing fails: Store raw output, mark parsing as failed

**Note:** Do not fail workflow if tests fail - we want to report test failures in health status

</step>

<step name="run_build" number="4">
**Execute build and capture metrics**

Run project build and record results:

**Process:**
1. Execute build command: `npm run build 2>&1 | tee .planning/.current-build.log`
2. Capture exit code, stdout, stderr
3. Measure build duration
4. Parse build output for warnings and errors

**Build execution:**
```bash
# Capture build output with timing
START_TIME=$(date +%s%3N)
npm run build 2>&1 | tee .planning/.current-build.log
BUILD_EXIT_CODE=$?
END_TIME=$(date +%s%3N)
BUILD_DURATION=$((END_TIME - START_TIME))
```

**Parse output:**
```xml
<build_results>
  <timestamp>2026-01-06T14:30:15Z</timestamp>
  <exit_code>0 | 1</exit_code>
  <duration_ms>3420</duration_ms>
  <status>success | failure</status>
  <warnings>2</warnings>
  <errors>0</errors>
  <summary>
    Compiled successfully. Bundle size: 245KB
  </summary>
</build_results>
```

**Parsing strategy:**
- Count lines with "warning:" or "warn:" for warning count
- Count lines with "error:" or "ERROR" for error count
- Extract bundle size if present
- Capture first/last 5 lines for summary

**Error handling:**
- If npm run build fails: Capture error, mark as failing
- If no build script: Skip build check, mark as "N/A"
- If timeout: Note timeout in status
- If parsing fails: Store raw output

**Note:** Do not fail workflow if build fails - report build failures in health status

</step>

<step name="check_git_status" number="5">
**Check git repository status**

Verify working directory state:

**Process:**
1. Run: `git status --porcelain`
2. Count modified, staged, untracked files
3. Check for merge conflicts
4. Verify branch state

**Git checks:**
```bash
# Get porcelain output for parsing
git status --porcelain > .planning/.git-status.txt

# Check for uncommitted changes
MODIFIED=$(git status --porcelain | wc -l)

# Check for conflicts
CONFLICTS=$(git diff --name-only --diff-filter=U | wc -l)

# Get current branch
BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

**Parse results:**
```xml
<git_status>
  <timestamp>2026-01-06T14:30:20Z</timestamp>
  <branch>main</branch>
  <clean>true | false</clean>
  <modified_files>5</modified_files>
  <staged_files>2</staged_files>
  <untracked_files>1</untracked_files>
  <conflicts>0</conflicts>
  <status>clean | dirty | conflicts</status>
</git_status>
```

**Status determination:**
- clean: No modified, staged, or untracked files
- dirty: Has changes but no conflicts
- conflicts: Merge conflicts present

**Error handling:**
- If not a git repo: Skip git checks, mark as "N/A"
- If git command fails: Log error, continue workflow

</step>

<step name="detect_regressions" number="6">
**Run regression detection if baselines exist**

Compare current state against baselines:

**Process:**
1. Check if baselines exist (`.planning/.baseline-tests.json`, `.planning/.baseline-build.json`)
2. If baselines exist AND config.regression_check = true:
   - Load regression-detection.md algorithm
   - Compare current test results against baseline
   - Compare current build results against baseline
   - Identify regressions, flaky tests, new failures
3. If no baselines:
   - Skip regression check
   - Note "No baseline - this will become the baseline"

**Regression detection:**
```xml
<regression_check>
  <baseline_exists>true | false</baseline_exists>
  <enabled>true | false</enabled>
  <regressions_found>3</regressions_found>
  <flaky_tests_found>1</flaky_tests_found>
  <new_failures>0</new_failures>
  <details>
    <!-- Use regression-detection.md algorithm output -->
    @get-shit-done/lib/regression-detection.md
  </details>
</regression_check>
```

**Algorithm integration:**
Load and apply `@get-shit-done/lib/regression-detection.md`:
- Use test comparison logic (step 3: compare baseline vs current)
- Use flaky test detection (re-run failing tests)
- Use build comparison logic
- Generate regression report

**Output:**
- List of confirmed regressions
- List of flaky tests
- Summary of changes since baseline

**Error handling:**
- If baseline file corrupted: Invalidate and note need for re-capture
- If parsing fails: Skip regression check, log error
- If regression detection fails: Continue workflow, note error

</step>

<step name="calculate_metrics" number="7">
**Calculate health metrics**

Aggregate metrics from all checks:

**Process:**
1. Combine results from steps 3-6
2. Calculate derived metrics:
   - Test pass rate: (passing / total) * 100
   - Build success: true/false
   - Overall health score
3. Determine overall health status

**Metrics calculation:**
```xml
<health_metrics>
  <!-- Test metrics -->
  <tests>
    <total>42</total>
    <passing>40</passing>
    <failing>2</failing>
    <pass_rate>95.2%</pass_rate>
  </tests>

  <!-- Build metrics -->
  <build>
    <status>success</status>
    <duration_ms>3420</duration_ms>
    <warnings>2</warnings>
    <errors>0</errors>
  </build>

  <!-- Git metrics -->
  <git>
    <status>clean</status>
    <branch>main</branch>
  </git>

  <!-- Regression metrics -->
  <regressions>
    <count>0</count>
    <flaky_tests>0</flaky_tests>
  </regressions>

  <!-- Overall health -->
  <overall_status>HEALTHY | WARNING | FAILING</overall_status>
</health_metrics>
```

**Overall status determination:**
```
HEALTHY:
  - All tests passing (or pass rate >= 95%)
  - Build succeeds
  - No regressions detected
  - No conflicts

WARNING:
  - Some tests failing (pass rate 80-95%)
  - OR build has warnings (no errors)
  - OR flaky tests detected
  - OR git dirty (no conflicts)

FAILING:
  - Many tests failing (pass rate < 80%)
  - OR build fails
  - OR regressions detected
  - OR git conflicts present
```

</step>

<step name="update_health_status" number="8">
**Update HEALTH_STATUS.md dashboard**

Write consolidated results to health dashboard:

**Process:**
1. Load current `.planning/HEALTH_STATUS.md`
2. Update sections with new data:
   - Header: Last Updated timestamp, Overall Status
   - Health Checks: Update checkboxes based on results
   - Metrics: Update all metric values
   - Recent Activity: Prepend new activity entry, keep last 5
   - Configuration: Reflect current config.json settings
3. Write updated content back to file

**Update strategy:**

**Header update:**
```markdown
**Last Updated:** 2026-01-06 14:30:25
**Overall Status:** ✓ HEALTHY
```

**Health Checks update:**
```markdown
- [x] **Build Status** - Project builds without errors
- [x] **Test Suite** - All tests passing (40/40)
- [x] **Git Status** - Working directory clean
- [x] **Dependencies** - All packages installed
- [x] **Configuration** - Config files valid
```

Use `[x]` for passing checks, `[ ]` for failing checks.

**Metrics update:**
Replace placeholder values with actual metrics from step 7.

**Recent Activity prepend:**
```markdown
### 2026-01-06 14:30:25 - Health Check
- **Trigger:** Manual health check command
- **Outcome:** ✓ All checks passed
- **Tests:** 40 passing, 0 failing
- **Build:** Success in 3.4s
- **Notes:** Clean baseline verified
```

Keep only the 5 most recent entries.

**Implementation:**
Use Edit tool to update specific sections, or Read-then-Write entire file.

**Error handling:**
- If file update fails: Warn user, continue to step 9
- If file locked: Retry once, then warn

</step>

<step name="update_baseline" number="9">
**Optionally update baseline (if requested)**

Capture current state as new baseline:

**Process:**
1. Check if baseline update requested:
   - Via command flag: `--update-baseline`
   - Via config: `config.verification.auto_update_baseline = true`
   - Via interactive prompt (if workflow invoked manually)
2. Verify current state is clean:
   - All tests passing
   - Build succeeds
   - No regressions (can't regress against self)
3. If clean AND update requested:
   - Write `.planning/.baseline-tests.json` with current test results
   - Write `.planning/.baseline-build.json` with current build results
   - Update HEALTH_STATUS.md "Last Successful Verification" timestamp
4. If not clean:
   - Warn: "Cannot update baseline - current state has failures"
   - Skip baseline update

**Baseline capture:**
```bash
# Tests
cp .planning/.current-tests.json .planning/.baseline-tests.json

# Build
echo '{
  "timestamp": "2026-01-06T14:30:25Z",
  "status": "success",
  "duration_ms": 3420,
  "warnings": 2,
  "errors": 0
}' > .planning/.baseline-build.json
```

**Baseline data structure:**
Use same format as regression-detection.md specifies.

**Error handling:**
- If baseline write fails: Warn user, don't fail workflow
- If current state not clean: Skip update, notify user

**Note:** Baseline updates should be deliberate - don't auto-update on every run unless explicitly configured

</step>

<step name="report_results" number="10">
**Display consolidated health report to user**

Present health check results:

**Process:**
1. Format summary report with all key findings
2. Include actionable recommendations
3. Display to user

**Report format:**
```
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: ✓ HEALTHY

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHECKS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Build Status       - Success in 3.4s
✓ Test Suite         - 40/40 passing (100%)
✓ Git Status         - Working directory clean
✓ Dependencies       - All packages installed
✓ Configuration      - Valid

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METRICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tests:           40 passing, 0 failing, 0 skipped
Build:           3.4s, 2 warnings, 0 errors
Regressions:     0 detected
Flaky Tests:     0 detected

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All health checks passed. Project is in good state.

Dashboard updated: .planning/HEALTH_STATUS.md
Baseline updated: .planning/.baseline-tests.json

Next: Ready to start work or continue with plan execution.
```

**For WARNING status:**
Include specific warnings and recommended actions:
```
Overall Status: ⚠ WARNING

Issues detected:
  - 2 flaky tests requiring attention
  - Build warnings increased from 1 to 3

Recommendations:
  - Review flaky tests: auth/timeout.test.js, websocket/reconnect.test.js
  - Investigate new build warnings in src/api/

Action required: Fix flaky tests before merging changes
```

**For FAILING status:**
Highlight critical issues and blocking problems:
```
Overall Status: ✗ FAILING

Critical issues:
  - 3 regressions detected (tests that were passing now fail)
  - Build fails with TypeScript errors

Regressions:
  1. User login > should accept valid credentials
  2. API > GET /users should return 200
  3. Database > should connect successfully

Action required: Fix regressions before continuing work
Review: .planning/HEALTH_STATUS.md for details
```

**Output destination:**
- Display to stdout (user sees immediately)
- Also logged to HEALTH_STATUS.md Recent Activity

</step>

## Configuration

Health check behavior controlled by `.planning/config.json`:

```json
{
  "verification": {
    "enabled": true,
    "regression_check": true,
    "flaky_test_retries": 1,
    "treat_flaky_as_regression": false,
    "baseline_tests": ".planning/.baseline-tests.json",
    "baseline_build": ".planning/.baseline-build.json",
    "auto_update_baseline": false,
    "skip_build": false,
    "skip_tests": false
  }
}
```

**Command-line flags** (for future `/gsd:health-check` command):
- `--update-baseline` - Update baseline after check completes
- `--skip-build` - Skip build verification (tests only)
- `--skip-tests` - Skip test verification (build only)
- `--verbose` - Show detailed output from tests and build

## Output

<output>
The health check workflow produces:

**Primary outputs:**
1. **HEALTH_STATUS.md** - Updated dashboard at `.planning/HEALTH_STATUS.md`
2. **Console report** - Formatted summary displayed to user
3. **Baseline files** - Updated `.planning/.baseline-tests.json` and `.planning/.baseline-build.json` (if requested)

**Temporary files:**
- `.planning/.current-tests.json` - Latest test results
- `.planning/.current-tests.log` - Raw test output
- `.planning/.current-build.log` - Raw build output
- `.planning/.git-status.txt` - Git status snapshot

**Status codes:**
- 0: All checks passed (HEALTHY)
- 1: Warnings detected (WARNING)
- 2: Critical failures (FAILING)
- 3: Health check itself failed (error in workflow)

</output>

## Integration Points

### Called By

<integration_point name="gsd:health-check">
  Future command implementation:
  - User runs `/gsd:health-check`
  - Loads this workflow
  - Executes all steps
  - Reports results interactively
</integration_point>

<integration_point name="execute-phase.md">
  Pre-plan verification:
  - Before starting phase execution
  - Run health check to verify clean baseline
  - Abort if FAILING status (user must fix first)
  - Warn if WARNING status (user can choose to continue)

  Post-task verification:
  - After each task that modifies code
  - Run streamlined health check (tests + regression detection only)
  - Log results to HEALTH_STATUS.md
  - Escalate if regressions detected
</integration_point>

<integration_point name="plan-phase.md">
  Pre-planning check:
  - Before creating plan, verify project health
  - Surface existing failures that might affect planning
  - Recommend fixing critical issues before planning new work
</integration_point>

### Uses

- `@get-shit-done/lib/regression-detection.md` - Regression detection algorithm
- `@get-shit-done/templates/health-status.md` - Dashboard template
- `.planning/config.json` - Verification configuration
- `.planning/.baseline-tests.json` - Test baseline for comparison
- `.planning/.baseline-build.json` - Build baseline for comparison

## Examples

### Example 1: Clean Health Check (All Passing)

**Scenario:** Developer runs health check before starting work

**Execution:**
```bash
# User invokes: /gsd:health-check

Step 1: Load config ✓
Step 2: Initialize HEALTH_STATUS.md ✓ (loaded existing)
Step 3: Run tests ✓ (42/42 passing in 4.5s)
Step 4: Run build ✓ (success in 3.4s, 2 warnings)
Step 5: Check git ✓ (clean)
Step 6: Detect regressions ✓ (0 found)
Step 7: Calculate metrics ✓
Step 8: Update dashboard ✓
Step 9: Update baseline ✓ (user confirmed)
Step 10: Report results ✓
```

**Output:**
```
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: ✓ HEALTHY

All checks passed. Project ready for work.

Dashboard: .planning/HEALTH_STATUS.md
Baseline: .planning/.baseline-tests.json (updated)
```

---

### Example 2: Warning Status (Flaky Tests)

**Scenario:** Health check detects flaky tests

**Execution:**
```bash
Step 3: Run tests ⚠ (41/42 passing, 1 flaky)
  - Test "WebSocket > reconnect" failed first run
  - Re-run: passed (marked flaky)
Step 6: Detect regressions ⚠ (0 regressions, 1 flaky)
Step 7: Overall status: WARNING
```

**Output:**
```
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: ⚠ WARNING

Issues:
  - 1 flaky test detected

Flaky tests:
  • WebSocket > reconnect (tests/websocket.test.js)
    Failed initially, passed on retry
    Recommendation: Add proper wait conditions

Action: Fix flaky test before merging changes
```

---

### Example 3: Failing Status (Regressions Detected)

**Scenario:** Recent code change broke tests

**Execution:**
```bash
Step 3: Run tests ✗ (39/42 passing, 3 failing)
Step 6: Detect regressions ✗ (3 regressions found)
  - "User login > should accept valid credentials" (was passing)
  - "API > GET /users" (was passing)
  - "Database > connect" (was passing)
Step 7: Overall status: FAILING
```

**Output:**
```
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: ✗ FAILING

Critical issues:
  - 3 regressions detected

Regressions:
  1. User login > should accept valid credentials
     Error: Cannot read property 'validate' of undefined
     File: tests/auth.test.js

  2. API > GET /users should return 200
     Error: Expected 200, got 404
     File: tests/api.test.js

  3. Database > should connect successfully
     Error: Connection refused
     File: tests/db.test.js

Action required: Fix regressions before continuing
Baseline age: 2 hours (last clean state)

Recommendations:
  - Review recent changes to auth, API, and database modules
  - Consider reverting recent commits if cause unclear
  - Run individual test files to debug
```

---

### Example 4: No Baseline (First Run)

**Scenario:** First health check after enabling verification

**Execution:**
```bash
Step 6: Detect regressions ⚠ (no baseline exists)
  - Skip regression detection
  - Will capture current state as baseline
Step 9: Update baseline ✓ (initial capture)
```

**Output:**
```
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: ✓ HEALTHY

Note: No baseline existed - captured current state

Tests:     40/40 passing
Build:     Success in 3.4s
Baseline:  .planning/.baseline-tests.json (created)

Next: Future health checks will detect regressions against this baseline
```

---

### Example 5: Embedded in execute-phase (Streamlined)

**Scenario:** Post-task verification during plan execution

**Context:** Task just completed: "Refactor user authentication"

**Execution (streamlined - tests only, skip build):**
```bash
Step 3: Run tests ✓ (42/42 passing)
Step 6: Detect regressions ✓ (0 found)
Step 8: Update dashboard ✓ (append activity entry)
```

**Output (logged to HEALTH_STATUS.md, brief console message):**
```
Post-task verification: ✓ No regressions detected
  Tests: 42/42 passing
  Updated: .planning/HEALTH_STATUS.md
```

**HEALTH_STATUS.md Recent Activity:**
```markdown
### 2026-01-06 15:45:00 - Post-Task Verification
- **Trigger:** After task "Refactor user authentication"
- **Outcome:** ✓ All checks passed
- **Tests:** 42 passing, 0 regressions
- **Build:** Skipped (post-task quick check)
- **Notes:** Authentication refactor did not break tests
```

---

## Success Indicators

Health check workflow working correctly when:
- Runs in <10 seconds for typical projects
- Accurately detects regressions vs new failures
- Flaky tests identified and handled appropriately
- Clear, actionable reports for all status levels
- Dashboard stays current with every run
- Baselines updated safely (only when clean state)
- Integrates seamlessly with execute-phase and manual commands

---

*On-demand health verification for GSD continuous verification system*
