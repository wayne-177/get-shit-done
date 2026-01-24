# Regression Log

**Total regressions detected:** 0
**Last detected:** Never

---

This log tracks all regressions detected during plan execution and their resolution outcomes. A regression occurs when a task passes its own verification but breaks tests that were previously passing.

## Entry Template

Use this template for each regression detected:

```xml
<entry>
  <timestamp>[YYYY-MM-DD HH:MM:SS]</timestamp>
  <task-id>[phase-plan Task N, e.g., "02-03 Task 2"]</task-id>
  <commit-sha>[git commit SHA that introduced regression]</commit-sha>
  <failure-description>
    [Clear description of what broke and how it was detected]
    [Include specific test names, error messages, or behavior changes]
  </failure-description>
  <affected-tests>
    <test>[test-file: test-name]</test>
    <test>[test-file: test-name]</test>
  </affected-tests>
  <rollback-action>
    [What git operations were performed to undo the regression]
    [Include: git revert SHA, git reset, stash operations, conflict resolution]
  </rollback-action>
  <retry-strategy>
    [Which alternative strategy was selected from ALTERNATIVE_PATHS.md]
    [Brief description of the different approach taken]
  </retry-strategy>
  <resolution>[success | escalated | deferred]</resolution>
  <learnings>
    [What was learned from this regression - why it happened, how to prevent]
    [Reusable patterns for future work]
  </learnings>
</entry>
```

## Guidelines

<guidelines>
  <when_to_log>
    <trigger>Log an entry whenever regression-detection.md identifies a regression</trigger>

    <conditions>
      Must meet ALL of:
      1. Task completed successfully (passed its own verification)
      2. Tests that WERE passing are NOW failing
      3. Failure caused by changes in current or recent task
      4. Not a new test (must be in baseline)
    </conditions>

    <non_regressions>
      DO NOT log these as regressions:
      - New tests failing (expected during development)
      - Tests that were already failing in baseline
      - Known flaky tests (marked as skip or retry)
      - Test failures during task execution (before commit)
      - Environment issues (network, permissions) not caused by code changes
    </non_regressions>
  </when_to_log>

  <how_to_format>
    <xml_structure>
      All entries MUST use XML structure for machine parsing:
      - Enables future analysis and reporting
      - Allows automated pattern detection
      - Facilitates learning from historical regressions
      - Supports regression metrics and dashboards
    </xml_structure>

    <entry_completeness>
      Every field should be filled:
      - timestamp: Exact time regression detected
      - task-id: Phase-Plan Task number for traceability
      - commit-sha: Git commit that introduced regression (for rollback)
      - failure-description: Clear, specific description (not generic "tests failed")
      - affected-tests: List of specific test names that failed
      - rollback-action: Exact git commands used to revert
      - retry-strategy: Which alternative approach was tried
      - resolution: Outcome (success/escalated/deferred)
      - learnings: Key insights (most valuable for future prevention)
    </entry_completeness>

    <clarity_standards>
      failure-description should include:
      - What broke (specific functionality)
      - How it was detected (test suite, command)
      - Error messages or symptoms
      - Why this is a regression (what was working before)

      learnings should capture:
      - Root cause (why regression happened)
      - Prevention strategy (how to avoid in future)
      - Pattern recognition (similar to past regressions?)
      - Reusable insights (applies to other tasks)
    </clarity_standards>
  </how_to_format>

  <retention_policy>
    <keep_all>
      Never delete regression log entries:
      - Historical data valuable for pattern analysis
      - Shows project health trends over time
      - Helps identify recurring failure modes
      - Provides context for future debugging
    </keep_all>

    <archival>
      For long-running projects (>100 entries):
      - Consider archiving old entries to REGRESSION_LOG_ARCHIVE.md
      - Keep recent entries (last 50) in main log
      - Maintain searchability across archive
      - Reference archive in main log header
    </archival>

    <metrics>
      Update header counts after each entry:
      - Total regressions detected: Increment by 1
      - Last detected: Update to current timestamp
      - (Optional) Calculate success rate: successful auto-corrections / total
    </metrics>
  </retention_policy>

  <integration>
    <workflow_integration>
      This template is used by:
      - auto-correction.md workflow (appends entries when regressions detected)
      - health-check.md workflow (reads log to report regression history)
      - execute-phase.md (checks log before starting plans)
      - /gsd:health-check command (displays recent regressions)
    </workflow_integration>

    <initialization>
      When to create this file:
      1. User enables verification (verification.enabled = true in config.json)
      2. Copy from get-shit-done/templates/regression-log.md to .planning/REGRESSION_LOG.md
      3. Initialize with header (0 regressions, never detected)
      4. Auto-correction workflow appends entries as regressions occur
    </initialization>

    <location>
      Template: get-shit-done/templates/regression-log.md (this file)
      Active log: .planning/REGRESSION_LOG.md (copied to each project)
      Archive: .planning/REGRESSION_LOG_ARCHIVE.md (if needed)
    </location>
  </integration>
</guidelines>

## Example Entries

### Example 1: Successfully Auto-Corrected Regression

```xml
<entry>
  <timestamp>2026-01-06 14:23:15</timestamp>
  <task-id>02-03 Task 2</task-id>
  <commit-sha>a7b3c9d</commit-sha>
  <failure-description>
    Added authentication middleware to API routes, broke 3 existing integration tests.
    Tests failing: api.test.js - "GET /users returns user list", "POST /users creates user", "DELETE /users/:id removes user"
    Error: All endpoints returning 401 Unauthorized (expected 200 OK).
    Regression: Tests were passing in baseline, broke after auth middleware added.
  </failure-description>
  <affected-tests>
    <test>api.test.js: GET /users returns user list</test>
    <test>api.test.js: POST /users creates user</test>
    <test>api.test.js: DELETE /users/:id removes user</test>
  </affected-tests>
  <rollback-action>
    git stash push -m "Task 3 verification before rollback of a7b3c9d"
    git revert a7b3c9d --no-edit
    git stash pop
    Rollback successful, no conflicts. New revert commit: e9f2a1b
  </rollback-action>
  <retry-strategy>
    Alternative: "Add authentication with test fixtures"
    Approach: Implement auth middleware but provide test authentication tokens in integration test setup.
    Modified test setup to include valid JWT tokens in request headers.
  </retry-strategy>
  <resolution>success</resolution>
  <learnings>
    Root cause: Auth middleware required valid tokens but tests used unauthenticated requests.
    Prevention: When adding authentication, always update test fixtures with valid credentials.
    Pattern: Integration tests need to match production authentication flow.
    Reusable: Create test utility for generating valid auth tokens (added to test/utils/auth.js).
  </learnings>
</entry>
```

### Example 2: Escalated After 3 Retries

```xml
<entry>
  <timestamp>2026-01-06 15:47:32</timestamp>
  <task-id>03-01 Task 1</task-id>
  <commit-sha>d4e8f2a</commit-sha>
  <failure-description>
    Refactored database schema from SQL to NoSQL, broke 12 tests across multiple test files.
    Tests failing: database.test.js (5 tests), models.test.js (4 tests), queries.test.js (3 tests)
    Error: Model.find() is not a function (SQL models don't have .find() method from NoSQL)
    Regression: All database interaction tests failing after schema migration.
  </failure-description>
  <affected-tests>
    <test>database.test.js: connects to database</test>
    <test>database.test.js: creates tables</test>
    <test>database.test.js: inserts records</test>
    <test>database.test.js: queries records</test>
    <test>database.test.js: updates records</test>
    <test>models.test.js: User model CRUD operations (4 tests)</test>
    <test>queries.test.js: complex queries (3 tests)</test>
  </affected-tests>
  <rollback-action>
    git stash push -m "Task 2 verification before rollback of d4e8f2a"
    git revert d4e8f2a --no-edit
    git stash pop
    Rollback successful, revert commit: f7b9c3e
  </rollback-action>
  <retry-strategy>
    Attempt 1: "Gradual migration strategy"
    - Implemented dual adapter (supports both SQL and NoSQL)
    - Result: FAILURE - 8 tests still failing due to query syntax differences

    Attempt 2: "Update all tests to NoSQL syntax"
    - Rewrote tests using NoSQL query methods (.find(), .findOne(), etc.)
    - Result: FAILURE - 3 tests still failing, complex queries don't translate

    Attempt 3: "Schema compatibility layer"
    - Added translation layer between SQL and NoSQL queries
    - Result: FAILURE - Too complex, introduces new bugs

    All 3 retry attempts failed. Escalated to user.
  </retry-strategy>
  <resolution>escalated</resolution>
  <learnings>
    Root cause: Schema migration from SQL to NoSQL is architectural change, not simple refactor.
    Why escalated: Requires rewriting application logic, not just fixing implementation.
    User decision: Reverted to SQL (keep d4e8f2a reverted), defer NoSQL migration to separate phase.
    Pattern: Major architectural changes need separate planning phase, not inline refactoring.
    Reusable: Database schema changes are high-risk - should be planned carefully with migration strategy.
  </learnings>
</entry>
```

## Log Statistics

<statistics>
  <calculation>
    Calculate these metrics from log entries:

    - Total regressions: Count of all &lt;entry&gt; elements
    - Success rate: (entries with resolution=success) / total * 100%
    - Escalation rate: (entries with resolution=escalated) / total * 100%
    - Most affected areas: Group by test file names, find top 3
    - Common causes: Analyze failure-description and learnings for patterns
    - Average retries: Count retry attempts in retry-strategy field
  </calculation>

  <reporting>
    Include in health-status.md dashboard:
    - Recent regressions (last 5 entries)
    - Success rate trend (improving/declining)
    - Most common regression types
    - Learnings summary (top patterns discovered)
  </reporting>
</statistics>

## Usage Example

### Initialization (when user enables verification)

```markdown
# Regression Log

**Total regressions detected:** 0
**Last detected:** Never

---

This log tracks all regressions detected during plan execution and their resolution outcomes. A regression occurs when a task passes its own verification but breaks tests that were previously passing.

[Guidelines and template sections...]
```

### After First Regression Detected

```markdown
# Regression Log

**Total regressions detected:** 1
**Last detected:** 2026-01-06 14:23:15

---

This log tracks all regressions detected during plan execution and their resolution outcomes. A regression occurs when a task passes its own verification but breaks tests that were previously passing.

<entry>
  <timestamp>2026-01-06 14:23:15</timestamp>
  <task-id>02-03 Task 2</task-id>
  ...
</entry>

[Guidelines and template sections...]
```

### After Multiple Regressions

```markdown
# Regression Log

**Total regressions detected:** 5
**Last detected:** 2026-01-06 17:32:08

---

This log tracks all regressions detected during plan execution and their resolution outcomes. A regression occurs when a task passes its own verification but breaks tests that were previously passing.

<entry>
  <timestamp>2026-01-06 17:32:08</timestamp>
  <task-id>04-01 Task 3</task-id>
  ...
</entry>

<entry>
  <timestamp>2026-01-06 16:15:22</timestamp>
  <task-id>03-02 Task 1</task-id>
  ...
</entry>

<entry>
  <timestamp>2026-01-06 15:47:32</timestamp>
  <task-id>03-01 Task 1</task-id>
  ...
</entry>

<entry>
  <timestamp>2026-01-06 14:58:10</timestamp>
  <task-id>02-04 Task 2</task-id>
  ...
</entry>

<entry>
  <timestamp>2026-01-06 14:23:15</timestamp>
  <task-id>02-03 Task 2</task-id>
  ...
</entry>

[Guidelines and template sections...]
```

## Maintenance

<maintenance>
  <regular_review>
    Periodically review regression log:
    - Identify recurring patterns (same tests failing repeatedly)
    - Update ALTERNATIVE_PATHS.md with successful retry strategies
    - Add prevention guidance to relevant workflow documents
    - Share learnings with team in project documentation
  </regular_review>

  <cleanup>
    When log grows large (>100 entries):
    1. Create .planning/REGRESSION_LOG_ARCHIVE.md
    2. Move entries older than 6 months to archive
    3. Keep recent 50 entries in main log
    4. Update header to reference archive:
       "**Archived entries:** See REGRESSION_LOG_ARCHIVE.md (X older entries)"
  </cleanup>

  <backup>
    Regression log is valuable project data:
    - Committed to git (part of .planning/ directory)
    - Included in project backups
    - Consider exporting to structured format (JSON, CSV) for analysis
  </backup>
</maintenance>

---

*Regression tracking template for GSD continuous verification system*
