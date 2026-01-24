# Project Health Status

**Last Updated:** [YYYY-MM-DD HH:MM:SS]
**Overall Status:** ✓ HEALTHY | ⚠ WARNING | ✗ FAILING

---

## Health Checks

<!-- Checkboxes indicate current status of core project health indicators -->

- [ ] **Build Status** - Project builds without errors
- [ ] **Test Suite** - All tests passing
- [ ] **Git Status** - Working directory clean, no conflicts
- [ ] **Dependencies** - All packages installed and compatible
- [ ] **Configuration** - Config files valid and complete

---

## Metrics

<!-- Key health metrics tracked over time -->

### Test Coverage
- **Total Tests:** [number]
- **Passing:** [number]
- **Failing:** [number]
- **Skipped:** [number]
- **Coverage:** [percentage]%

### Build Performance
- **Last Build Time:** [duration in ms or seconds]
- **Build Status:** [success | failure]
- **Warnings:** [count]
- **Errors:** [count]

### Code Health
- **Lint Issues:** [count]
- **Type Errors:** [count]
- **Security Vulnerabilities:** [count]

### Verification History
- **Last Successful Verification:** [YYYY-MM-DD HH:MM:SS]
- **Last Failure Detected:** [YYYY-MM-DD HH:MM:SS] or "None"
- **Regressions Detected (Last 24h):** [count]
- **Flaky Tests Identified:** [count]

---

## Recent Activity

<!-- Last 5 verification events showing health check history -->

### 2026-01-06 14:30:00 - Health Check
- **Trigger:** Manual health check command
- **Outcome:** ✓ All checks passed
- **Tests:** 42 passing, 0 failing
- **Build:** Success in 3.2s
- **Notes:** Clean baseline established

### 2026-01-06 12:15:00 - Post-Task Verification
- **Trigger:** After task "Refactor authentication module"
- **Outcome:** ⚠ Warning - 1 flaky test detected
- **Tests:** 41 passing, 1 flaky (auth/timeout)
- **Build:** Success in 3.1s
- **Notes:** Flaky test requires timeout adjustment

### 2026-01-06 10:45:00 - Regression Check
- **Trigger:** After task "Update API endpoints"
- **Outcome:** ✗ Regression detected
- **Tests:** 40 passing, 2 regressions
- **Build:** Success in 3.0s
- **Notes:** API change broke 2 tests - reverted and fixed

### 2026-01-06 09:30:00 - Baseline Capture
- **Trigger:** Enabling verification for first time
- **Outcome:** ✓ Baseline captured
- **Tests:** 42 passing, 0 failing
- **Build:** Success in 3.4s
- **Notes:** Initial baseline established

### 2026-01-06 08:00:00 - Health Check
- **Trigger:** Manual health check command
- **Outcome:** ✓ All checks passed
- **Tests:** 42 passing, 0 failing
- **Build:** Success in 3.3s
- **Notes:** Pre-work verification complete

---

## Configuration

<!-- Current verification settings from config.json -->

**Verification Enabled:** [true | false]
**Regression Check:** [enabled | disabled]
**Flaky Test Retries:** [number]
**Auto-Update Baseline:** [enabled | disabled]
**Baseline Location:** [path to baseline files]

---

## Guidelines

<guidelines>
This health status dashboard is automatically maintained by GSD's continuous verification system. It provides real-time visibility into project health and enables early detection of regressions.

### When to Update

This file is updated automatically by:

1. **Health Check Workflow** (`/gsd:health-check` command)
   - Runs full verification suite (tests, build, git status)
   - Updates all metrics and health checks
   - Captures baselines if requested
   - Frequency: On-demand when user runs command

2. **Post-Task Verification** (during plan execution)
   - Runs after each task that modifies code
   - Checks for regressions against baseline
   - Updates recent activity log
   - Frequency: After each task when `verification.enabled = true`

3. **Regression Detection** (automatic)
   - Runs when code changes detected
   - Compares current state to baseline
   - Logs regressions and flaky tests
   - Frequency: After code modifications

### How to Read This Dashboard

**Health Checks:**
- ✓ (checked) = Component healthy
- ☐ (unchecked) = Component failing or unknown
- Review failing checks first

**Metrics:**
- Compare current vs historical values
- Sudden changes indicate issues
- Track trends over time

**Recent Activity:**
- Most recent events at top
- Shows verification triggers and outcomes
- Use to diagnose when issues started

**Status Indicators:**
- ✓ HEALTHY = All checks passing, no regressions
- ⚠ WARNING = Some checks failing or flaky tests detected (not blocking)
- ✗ FAILING = Critical checks failing or regressions detected

### Maintenance

**Updating Health Checks:**
To manually update a health check:
1. Run the check (e.g., `npm test`, `npm run build`)
2. Update the checkbox: `- [x]` for passing, `- [ ]` for failing
3. Update the "Last Updated" timestamp

**Updating Metrics:**
Metrics are auto-populated from:
- Test output (npm test --json)
- Build output (npm run build)
- Git status (git status --porcelain)
- Package.json and lock files

**Adding Activity Events:**
Each verification run adds an entry to Recent Activity:
- Keep last 5-10 events (remove oldest)
- Include timestamp, trigger, outcome, and notes
- Use emoji indicators: ✓ success, ⚠ warning, ✗ failure

**Baseline Management:**
- Baselines stored in `.planning/.baseline-tests.json` and `.planning/.baseline-build.json`
- Update baseline after confirmed clean state: Run health check with update flag
- Never update baseline when regressions present
- Baseline age shown in "Last Successful Verification"

### Integration with Workflows

**execute-phase.md:**
After each auto task completion:
```xml
<step name="verify_health">
  If config.verification.enabled:
    1. Run regression check against baseline
    2. Update HEALTH_STATUS.md with results
    3. If regressions: Log and optionally escalate
    4. If clean: Update "Last Successful Verification"
</step>
```

**health-check.md:**
On-demand full verification:
```xml
<step name="full_health_check">
  1. Run tests: npm test
  2. Run build: npm run build
  3. Check git status: git status --porcelain
  4. Calculate all metrics
  5. Update all sections of HEALTH_STATUS.md
  6. Report summary to user
</step>
```

**regression-detection.md:**
When regressions detected:
```xml
<step name="log_regression">
  1. Parse regression details
  2. Update health checks (uncheck "Test Suite")
  3. Update metrics (increment regression count)
  4. Add activity event with regression details
  5. Set overall status to ✗ FAILING
</step>
```

### Example Update Flow

**Scenario:** Task completes, verification runs

1. **Before Task:**
   ```
   Overall Status: ✓ HEALTHY
   Test Suite: [x] Passing (42/42)
   Last Verification: 2026-01-06 10:00:00
   ```

2. **Task Execution:**
   - Task: "Add user profile endpoint"
   - Modifies: src/api/users.js, tests/api.test.js

3. **Post-Task Verification:**
   ```bash
   npm test --json > .planning/.current-tests.json
   # Compare against .planning/.baseline-tests.json
   # Result: 1 regression detected (broke existing /users endpoint)
   ```

4. **After Update:**
   ```
   Overall Status: ✗ FAILING
   Test Suite: [ ] Failing (41/42, 1 regression)
   Last Verification: 2026-01-06 10:30:00
   Recent Activity:
     2026-01-06 10:30:00 - Post-Task Verification
     Trigger: After task "Add user profile endpoint"
     Outcome: ✗ Regression detected
     Tests: 41 passing, 1 regression (API > GET /users)
     Notes: New endpoint broke existing /users route
   ```

5. **Resolution:**
   - Fix the regression
   - Re-run tests: All pass
   - Update status back to ✓ HEALTHY

### Tips

- **Check this file before starting work** to ensure clean baseline
- **Review after each task** to catch issues early
- **Update baselines deliberately** after verified clean states
- **Don't ignore warnings** - flaky tests and warnings compound over time
- **Use Recent Activity** to correlate failures with specific tasks
- **Git commit this file** to track health over time (optional)

</guidelines>

---

## Notes

<!-- Additional context, known issues, or planned maintenance -->

- This file is created from `get-shit-done/templates/health-status.md` when verification is enabled
- Located at `.planning/HEALTH_STATUS.md` in your project
- Gitignored by default (project-specific state)
- Can be committed to track health history across team
- Update placeholders above with actual values during health checks
- Remove this Notes section after first update

---

*Automated health monitoring for GSD projects*
