# Regression Detection

This document defines the algorithm for detecting regressions in test suites and builds. Used by the health check workflow and execute-phase verification to catch when previously-passing tests break due to recent changes.

## Purpose

Enable GSD to automatically detect when code changes introduce regressions by:
1. Capturing baselines of passing tests and successful builds
2. Comparing current results against baselines
3. Distinguishing regressions from expected failures and new tests
4. Handling flaky tests appropriately

## Detection Strategy

<detection_strategy>
Regression detection works by comparing two states:
- **Baseline state**: Known-good snapshot of tests/builds (stored in .planning/.baseline-tests.json and .planning/.baseline-build.json)
- **Current state**: Latest test/build results from current execution

A **regression** occurs when:
- A test was PASSING in baseline
- AND the same test is FAILING in current run
- AND the failure persists across retries (not flaky)

This is different from:
- **New test failure**: Test doesn't exist in baseline (new test added)
- **Expected failure**: Test was already failing in baseline
- **Flaky test**: Test fails inconsistently across runs
</detection_strategy>

## Baseline Capture

### Test Baseline

<test_baseline_capture>
  <when>Capture baseline after successful task completion or on-demand via health check</when>
  <command>npm test -- --json > .planning/.baseline-tests.json</command>
  <fallback>If --json not supported, capture stdout and parse test framework output</fallback>

  <data_structure>
    {
      "timestamp": "2026-01-06T10:30:00Z",
      "total_tests": 42,
      "passing": 40,
      "failing": 2,
      "skipped": 0,
      "test_results": [
        {
          "name": "User authentication > should login with valid credentials",
          "status": "passed",
          "duration_ms": 45,
          "file": "tests/auth.test.js"
        },
        {
          "name": "User authentication > should reject invalid password",
          "status": "failed",
          "error": "Expected 401, got 500",
          "file": "tests/auth.test.js"
        }
      ]
    }
  </data_structure>

  <notes>
    - Store in .planning/ to avoid cluttering project root
    - File is gitignored by default (project-specific baseline)
    - Only capture when config.verification.enabled = true
    - Update baseline only when explicitly requested or after verified success
  </notes>
</test_baseline_capture>

### Build Baseline

<build_baseline_capture>
  <when>Capture baseline after successful build or on-demand via health check</when>
  <command>npm run build 2>&1 | tee .planning/.baseline-build.log</command>

  <data_structure>
    {
      "timestamp": "2026-01-06T10:30:00Z",
      "status": "success",
      "duration_ms": 3420,
      "warnings": 2,
      "errors": 0,
      "output_summary": "Compiled successfully. Bundle size: 245KB"
    }
  </data_structure>

  <notes>
    - Capture both exit code and key metrics (warnings, errors, bundle size)
    - Build warnings are not regressions unless count increases significantly
    - Store minimal data - full logs are too large
  </notes>
</build_baseline_capture>

## Comparison Logic

### Test Regression Detection

<test_regression_algorithm>
  <step number="1">Load baseline from .planning/.baseline-tests.json</step>
  <step number="2">Run current tests: npm test --json > .planning/.current-tests.json</step>
  <step number="3">Parse both JSON structures</step>
  <step number="4">
    For each test in current results:
      - Find matching test in baseline by name + file path
      - If not in baseline: Mark as "new test" (not regression)
      - If in baseline and baseline.status = "passed" and current.status = "failed":
        → **Potential regression detected**
      - If in baseline and baseline.status = "failed" and current.status = "failed":
        → Expected failure (not regression)
      - If in baseline and baseline.status = "passed" and current.status = "passed":
        → Still passing (good)
  </step>
  <step number="5">
    For potential regressions, apply flaky test filter (see below)
  </step>
  <step number="6">
    Report confirmed regressions
  </step>
</test_regression_algorithm>

### Build Regression Detection

<build_regression_algorithm>
  <step number="1">Load baseline from .planning/.baseline-build.json</step>
  <step number="2">Run current build: npm run build</step>
  <step number="3">Compare states:
    - If baseline.status = "success" and current.status = "failure":
      → **Build regression detected**
    - If baseline.errors = 0 and current.errors > 0:
      → **New build errors (regression)**
    - If baseline.warnings significantly less than current.warnings (increase > 50%):
      → **Warning regression** (note but don't block)
  </step>
  <step number="4">Report regressions with error details</step>
</build_regression_algorithm>

## Flaky Test Handling

<flaky_test_detection>
  <purpose>
    Avoid false positives from intermittent test failures. A test is flaky if it fails inconsistently.
  </purpose>

  <algorithm>
    When potential regression detected:
    1. Record the failing test name
    2. Re-run ONLY that specific test (not full suite)
       - npm test -- --testNamePattern="test name"
    3. If test passes on second run → Mark as **flaky** (not regression)
    4. If test fails on second run → Mark as **confirmed regression**
  </algorithm>

  <configuration>
    - config.verification.flaky_test_retries: Number of retries (default: 1)
    - config.verification.treat_flaky_as_regression: If true, flaky tests are regressions (default: false)
  </configuration>

  <output>
    When flaky tests detected, log to HEALTH_STATUS.md:
    - Test name
    - How many retries needed to pass
    - Recommendation to fix flakiness
  </output>
</flaky_test_detection>

## Edge Cases

### Case 1: New Tests Added

<edge_case type="new_tests">
  <scenario>
    Developer adds new test. Current run has test that doesn't exist in baseline.
  </scenario>

  <detection>
    Test name + file path not found in baseline.test_results[]
  </detection>

  <handling>
    - **Not a regression** (can't regress if it never existed)
    - If new test fails: Report as "new test failure" (different category)
    - Update HEALTH_STATUS.md to show "X new tests added"
    - Don't block execution for new test failures (task may be adding failing tests intentionally)
  </handling>
</edge_case>

### Case 2: Intentional Breaking Changes

<edge_case type="intentional_breaks">
  <scenario>
    Task deliberately breaks tests (e.g., refactoring API, changing behavior)
  </scenario>

  <detection>
    Large number of regressions (>10% of test suite) with related error messages
  </detection>

  <handling>
    - Report all regressions but flag as "possible intentional change"
    - Provide option to update baseline: "Accept current state as new baseline"
    - Don't auto-escalate if task description mentions "refactor", "breaking change", or "rewrite"
  </handling>
</edge_case>

### Case 3: Test Suite Restructuring

<edge_case type="test_restructure">
  <scenario>
    Tests renamed or moved to different files. Same test exists but path/name changed.
  </scenario>

  <detection>
    - Many "new tests" detected
    - AND many baseline tests not found in current run
    - AND total test count similar
  </detection>

  <handling>
    - Warn: "Test suite structure changed - baseline may be stale"
    - Suggest: "Update baseline after verifying tests pass"
    - Don't report as regressions (can't match tests reliably)
  </handling>
</edge_case>

### Case 4: No Baseline Exists

<edge_case type="missing_baseline">
  <scenario>
    First run after enabling verification, or baseline file deleted
  </scenario>

  <detection>
    .planning/.baseline-tests.json does not exist
  </detection>

  <handling>
    - Skip regression detection
    - Capture current state as initial baseline
    - Log: "No baseline found - capturing current state as baseline"
    - Next run will use this baseline for comparison
  </handling>
</edge_case>

### Case 5: Test Framework Changes

<edge_case type="framework_change">
  <scenario>
    Baseline captured with Jest, current run uses Mocha (or different version with format changes)
  </scenario>

  <detection>
    JSON structure mismatch or parsing errors
  </detection>

  <handling>
    - Attempt graceful parsing (support multiple formats)
    - If parsing fails: Invalidate baseline and re-capture
    - Log: "Baseline format incompatible - recapturing"
  </handling>
</edge_case>

## Integration Points

### Called By

<integration_point name="execute-phase.md">
  After each task completion:
  1. If config.verification.enabled = true
  2. AND task modified code files (*.js, *.ts, etc.)
  3. Run regression check:
     - npm test --json > .planning/.current-tests.json
     - Compare against baseline using algorithm above
     - If regressions detected → Log to HEALTH_STATUS.md, optionally escalate
     - If no regressions → Update "last verified clean" timestamp
</integration_point>

<integration_point name="health-check.md">
  On-demand health check workflow:
  1. Run full regression check (tests + build)
  2. Update HEALTH_STATUS.md with results
  3. Optionally update baseline if current state is verified good
</integration_point>

### Reads From

- `.planning/.baseline-tests.json` - Stored test baseline
- `.planning/.baseline-build.json` - Stored build baseline
- `.planning/config.json` - Verification settings

### Writes To

- `.planning/.current-tests.json` - Latest test results
- `.planning/HEALTH_STATUS.md` - Regression detection outcomes

## Configuration

Regression detection controlled by `.planning/config.json`:

```json
{
  "verification": {
    "enabled": false,
    "regression_check": true,
    "flaky_test_retries": 1,
    "treat_flaky_as_regression": false,
    "baseline_tests": ".planning/.baseline-tests.json",
    "baseline_build": ".planning/.baseline-build.json",
    "auto_update_baseline": false
  }
}
```

**Fields:**
- `enabled`: Master toggle for all verification features (default: false)
- `regression_check`: Enable regression detection specifically (default: true if enabled)
- `flaky_test_retries`: Number of retries before confirming regression (default: 1)
- `treat_flaky_as_regression`: Whether flaky tests count as regressions (default: false)
- `baseline_tests`: Path to test baseline file (default: .planning/.baseline-tests.json)
- `baseline_build`: Path to build baseline file (default: .planning/.baseline-build.json)
- `auto_update_baseline`: Auto-update baseline on successful runs (default: false, safer to require manual update)

## Output Format

<regression_report>
  When regressions detected, output this structure:

  ```xml
  <regression_detection>
    <timestamp>2026-01-06T11:45:00Z</timestamp>
    <baseline_date>2026-01-06T10:30:00Z</baseline_date>
    <regressions_found>3</regressions_found>
    <flaky_tests_found>1</flaky_tests_found>

    <regressions>
      <regression>
        <test_name>User authentication > should hash passwords</test_name>
        <test_file>tests/auth.test.js</test_file>
        <baseline_status>passed</baseline_status>
        <current_status>failed</current_status>
        <error_message>TypeError: Cannot read property 'hash' of undefined</error_message>
        <confirmed>true</confirmed>
      </regression>

      <regression>
        <test_name>API endpoints > GET /users should return 200</test_name>
        <test_file>tests/api.test.js</test_file>
        <baseline_status>passed</baseline_status>
        <current_status>failed</current_status>
        <error_message>Expected 200, got 404</error_message>
        <confirmed>true</confirmed>
      </regression>
    </regressions>

    <flaky_tests>
      <flaky>
        <test_name>WebSocket connection > should reconnect on disconnect</test_name>
        <test_file>tests/websocket.test.js</test_file>
        <retries_needed>1</retries_needed>
        <recommendation>Fix test flakiness - add proper wait conditions</recommendation>
      </flaky>
    </flaky_tests>

    <summary>
      3 confirmed regressions detected since baseline from 75 minutes ago.
      1 flaky test found (not blocking).
      Recommend: Review changes to auth.test.js and api.test.js.
    </summary>
  </regression_detection>
  ```
</regression_report>

## Examples

### Example 1: Clean Run (No Regressions)

**Baseline state (10:30 AM):**
- 42 tests total
- 40 passing, 2 failing (auth/rate-limiting tests known broken)

**Current state (11:45 AM after code change):**
- 42 tests total
- 40 passing, 2 failing (same auth/rate-limiting tests)

**Analysis:**
```xml
<comparison>
  <new_tests>0</new_tests>
  <tests_fixed>0</tests_fixed>
  <regressions>0</regressions>
  <still_failing>2</still_failing>
  <verdict>✓ No regressions detected</verdict>
</comparison>
```

**Outcome:** Clean status, update "last verified" timestamp in HEALTH_STATUS.md

---

### Example 2: Confirmed Regression

**Baseline state:**
- Test "User login > should accept valid credentials" = PASSING

**Current state (after refactoring auth module):**
- Test "User login > should accept valid credentials" = FAILING
- Error: "Cannot read property 'validate' of undefined"

**Flaky check:**
- Re-run test individually: FAILING (confirmed not flaky)

**Analysis:**
```xml
<regression>
  <test_name>User login > should accept valid credentials</test_name>
  <was>passed</was>
  <now>failed</now>
  <error>Cannot read property 'validate' of undefined</error>
  <flaky>false</flaky>
  <verdict>⚠ REGRESSION DETECTED</verdict>
  <likely_cause>Auth module refactoring broke login validation</likely_cause>
</regression>
```

**Outcome:**
- Log regression to HEALTH_STATUS.md
- Escalate to user: "Regression detected - previously passing test now fails"
- Suggest rollback or fix

---

### Example 3: Flaky Test (Not Regression)

**Baseline state:**
- Test "API > concurrent requests should not race" = PASSING

**Current state:**
- Test "API > concurrent requests should not race" = FAILING
- Error: "Timeout: async operation exceeded 5000ms"

**Flaky check:**
- Re-run test: PASSING

**Analysis:**
```xml
<flaky_test>
  <test_name>API > concurrent requests should not race</test_name>
  <first_run>failed</first_run>
  <retry_run>passed</retry_run>
  <verdict>⚠ Flaky test detected (not regression)</verdict>
  <recommendation>Increase timeout or fix race condition in test</recommendation>
</flaky_test>
```

**Outcome:**
- Log as flaky test in HEALTH_STATUS.md
- Don't escalate (not blocking)
- Continue execution

---

### Example 4: New Test Failure (Not Regression)

**Baseline state:**
- 42 tests (no test named "New feature > should validate input")

**Current state (after adding new feature):**
- 43 tests
- New test "New feature > should validate input" = FAILING

**Analysis:**
```xml
<new_test>
  <test_name>New feature > should validate input</test_name>
  <in_baseline>false</in_baseline>
  <status>failed</status>
  <verdict>❌ New test failing (not a regression)</verdict>
  <note>Cannot regress if test didn't exist before</note>
</new_test>
```

**Outcome:**
- Report as "new test failure" (different from regression)
- Don't block if task was adding new tests
- Update baseline to include new test count

---

### Example 5: Intentional Breaking Changes

**Baseline state:**
- 42 tests, 42 passing

**Current state (after API refactor task: "Change /users endpoint to /api/users"):**
- 42 tests, 25 failing
- All failures: "GET /users returned 404" (endpoint moved)

**Analysis:**
```xml
<large_scale_breakage>
  <regressions_detected>25</regressions_detected>
  <percentage>59.5%</percentage>
  <error_pattern>All errors mention "/users" endpoint (related)</error_pattern>
  <task_context>Task: "Refactor API endpoints - move to /api/* structure"</task_context>
  <verdict>⚠ Possible intentional breaking changes</verdict>
  <recommendation>Verify if test updates needed (expected) vs unintentional breakage</recommendation>
</large_scale_breakage>
```

**Outcome:**
- Report regressions but flag as "possibly intentional"
- Ask user: "Task involves API refactoring. Are these expected test failures?"
- Offer to update baseline after tests fixed

## Success Indicators

Regression detection working correctly when:
- True regressions caught reliably (high precision)
- Flaky tests distinguished from real regressions (low false positive rate)
- New tests don't trigger false alarms
- Intentional breakage flagged for review rather than auto-escalated
- Baseline stays current (updated after verified clean states)

---

*Document-driven regression detection for GSD continuous verification system*
