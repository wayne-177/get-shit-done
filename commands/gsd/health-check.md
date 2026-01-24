---
description: Run health check and update dashboard
allowed-tools:
  - Read
  - Bash
  - Write
  - Edit
---

<objective>

Run on-demand health verification to check project health and update HEALTH_STATUS.md dashboard.

This command provides manual health monitoring for GSD projects with verification enabled. It runs the complete health-check workflow, updates the health dashboard with current status, and reports results to the user.

Use this to verify project health before starting work, after completing tasks, or anytime you want to check for regressions.

</objective>

<execution_context>

@~/.claude/get-shit-done/workflows/health-check.md

</execution_context>

<context>

**Load verification configuration:**
@.planning/config.json

**Load current health status:**
@.planning/HEALTH_STATUS.md

**Load project context:**
@.planning/PROJECT.md

</context>

<process>

<step name="check_prerequisites">

**Verify project setup and verification enabled**

1. Check `.planning/` directory exists:
   ```bash
   [ -d .planning ] || (echo "ERROR: No .planning/ directory. Run /gsd:new-project first" && exit 1)
   ```

2. Load config.json and check if verification is enabled:
   ```bash
   # Read config.json
   # Parse verification.enabled field
   # If enabled = false, display message and exit
   ```

**If verification disabled:**
Display message to user:
```
Verification is disabled in config.json

To enable health checks:
1. Open .planning/config.json
2. Set "verification.enabled": true
3. Run /gsd:health-check again

See get-shit-done/references/verification-guide.md for setup instructions.
```
Exit command.

**If verification enabled:**
Continue to step 2.

</step>

<step name="run_health_check">

**Invoke health-check workflow**

Execute the health-check.md workflow which performs:

1. Load configuration from config.json
2. Initialize HEALTH_STATUS.md (create if missing)
3. Run test suite and capture results
4. Run build and capture metrics
5. Check git repository status
6. Detect regressions (if baselines exist)
7. Calculate health metrics
8. Update HEALTH_STATUS.md dashboard
9. Optionally update baseline (if requested)
10. Format consolidated report

**Workflow execution:**
Follow all steps in `@~/.claude/get-shit-done/workflows/health-check.md` exactly as specified.

**Capture results:**
Store health check results for reporting in step 4.

</step>

<step name="update_dashboard">

**Update HEALTH_STATUS.md with timestamp and results**

The health-check workflow already updates HEALTH_STATUS.md in step 8.

Verify the update:
1. Confirm HEALTH_STATUS.md exists at `.planning/HEALTH_STATUS.md`
2. Verify "Last Updated" timestamp is current
3. Verify "Overall Status" reflects health check results
4. Verify "Recent Activity" includes new entry

**Dashboard location:** `.planning/HEALTH_STATUS.md`

</step>

<step name="report_results">

**Display health report to user**

Present consolidated health check results formatted clearly:

**Format:**
```
═══════════════════════════════════════════════════
HEALTH CHECK RESULTS
═══════════════════════════════════════════════════

Overall Status: [✓ HEALTHY | ⚠ WARNING | ✗ FAILING]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHECKS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[✓/✗] Build Status       - [status and details]
[✓/✗] Test Suite         - [count passing/failing]
[✓/✗] Git Status         - [clean/dirty/conflicts]
[✓/✗] Dependencies       - [status]
[✓/✗] Configuration      - [status]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METRICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tests:           [N passing, N failing, N skipped]
Build:           [duration, N warnings, N errors]
Regressions:     [N detected]
Flaky Tests:     [N detected]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Overall summary message based on status]

Dashboard updated: .planning/HEALTH_STATUS.md
[If baseline updated:] Baseline updated: .planning/.baseline-tests.json

[Status-specific messages and recommendations]
```

**For WARNING status:**
Include specific warnings and recommended actions.

**For FAILING status:**
Highlight critical issues, list regressions, and suggest next steps.

**For HEALTHY status:**
Confirm all checks passed and project is ready.

</step>

<step name="handle_failures">

**Handle test or build failures gracefully**

If tests or build fail during health check:

1. **Do not exit command** - Health check should complete even if tests fail
2. **Capture error details:**
   - Which tests failed
   - Build error messages
   - Exit codes and logs
3. **Display in report:**
   - List failing tests
   - Show error messages
   - Suggest fixes based on error patterns
4. **Update dashboard:**
   - Mark failing checks with `[ ]` (unchecked)
   - Set overall status to FAILING
   - Log failure details in Recent Activity

**Common failure patterns and suggestions:**

**Test failures:**
```
✗ 3 tests failing

Failed tests:
  1. User authentication > login with valid credentials
     Error: Cannot read property 'validate' of undefined
     Fix: Check User model and validate function

  2. API > GET /users
     Error: Expected 200, got 404
     Fix: Verify API route exists and returns correct status

Action: Review test failures and fix issues before continuing work
```

**Build failures:**
```
✗ Build failed

Build errors:
  - TypeScript error in src/api/users.ts:42
    Property 'email' does not exist on type 'User'
  - TypeScript error in src/lib/auth.ts:18
    Cannot find module 'jose'

Action: Fix TypeScript errors and ensure dependencies installed
```

**Git conflicts:**
```
⚠ Git conflicts detected

Conflicts in:
  - src/api/users.ts
  - src/lib/auth.ts

Action: Resolve merge conflicts before running health check
```

**Display actionable guidance:**
- Clear description of what's wrong
- Specific files or tests affected
- Suggested fixes or next steps
- Command to re-run after fixes

</step>

</process>

<success_criteria>

- Command completes successfully
- HEALTH_STATUS.md updated with current timestamp
- User sees clear health report with status and metrics
- Failures handled gracefully with helpful error messages
- Dashboard reflects current project state

</success_criteria>
