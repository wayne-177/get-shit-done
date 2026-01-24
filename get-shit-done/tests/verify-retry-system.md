# Retry System Verification Tests

This guide provides manual test procedures to validate that the failure recovery and retry system works correctly. These tests ensure the retry orchestration, failure analysis, alternative path selection, and attempt logging all integrate properly.

## Test Setup

Before running any tests:

1. **Enable retry in config:**
   ```bash
   # Create or update .planning/config.json
   cat > .planning/config.json << 'EOF'
   {
     "retry": {
       "enabled": true,
       "max_attempts": 3,
       "escalation_threshold": 3,
       "log_attempts": true
     }
   }
   EOF
   ```

2. **Ensure required files exist:**
   ```bash
   # Verify all dependencies are present
   ls get-shit-done/lib/failure-analysis.md
   ls get-shit-done/lib/path-selection.md
   ls get-shit-done/templates/attempts-log.md
   ```

3. **Seed ALTERNATIVE_PATHS.md with test strategies:**
   ```bash
   # If .planning/ALTERNATIVE_PATHS.md doesn't exist, copy template
   mkdir -p .planning
   cp get-shit-done/templates/alternative-paths.md .planning/ALTERNATIVE_PATHS.md
   ```

4. **Create test directory:**
   ```bash
   mkdir -p .planning/test-workspace
   cd .planning/test-workspace
   ```

---

## Test Scenario 1: Simulated Tool Failure (Success on Retry)

**Objective:** Verify retry detects failure, selects alternative strategy, and succeeds on attempt 2

### Setup

Create a test plan that will fail on first attempt but has a recoverable alternative:

```bash
cat > TEST-PLAN-01.md << 'EOF'
---
phase: test-retry
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Edit non-existent file (will fail, then retry)</name>
  <files>test-file.txt</files>
  <action>
    Use Edit tool to add "Hello World" to test-file.txt
    old_string: "placeholder"
    new_string: "Hello World"
  </action>
  <verify>File test-file.txt contains "Hello World"</verify>
  <done>Test file updated with new content</done>
</task>

</tasks>
EOF
```

### Expected Behavior

**Attempt 1 (Original Plan):**
- Edit tool tries to find "placeholder" in test-file.txt
- **FAILS:** File doesn't exist or doesn't contain "placeholder"
- Failure analysis: Category = "Tool" (edit-match-failed)
- Retry appropriate: true

**Attempt 2 (Alternative Strategy):**
- Alternative selected: "read-then-edit" or "write-new-file"
- Strategy: Read file first to get content, OR write new file
- **SUCCEEDS:** File created/updated with "Hello World"

### Verification Steps

1. **Run the plan execution** (manually invoke execute-phase.md workflow or use GSD agent)

2. **Check ATTEMPTS_LOG.md was created:**
   ```bash
   ls -la .planning/ATTEMPTS_LOG.md
   ```

3. **Verify log contains 2 attempts:**
   ```bash
   grep -c "<attempt>" .planning/ATTEMPTS_LOG.md
   # Expected: 2
   ```

4. **Check frontmatter shows success:**
   ```bash
   head -10 .planning/ATTEMPTS_LOG.md
   # Expected:
   # total-attempts: 2
   # final-outcome: success
   ```

5. **Verify summary section exists:**
   ```bash
   grep -A 5 "## Summary" .planning/ATTEMPTS_LOG.md
   # Expected: "What worked", "Why it worked", "Patterns discovered"
   ```

6. **Check file was actually created:**
   ```bash
   cat test-file.txt
   # Expected: Contains "Hello World"
   ```

### Success Criteria

- ✅ Task completes successfully after 2 attempts
- ✅ ATTEMPTS_LOG.md created with 2 attempt entries
- ✅ Attempt 1 shows failure with failure-category
- ✅ Attempt 2 shows success
- ✅ Summary section explains what worked and why
- ✅ Final outcome is "success"

---

## Test Scenario 2: Exhausted Alternatives (Escalation)

**Objective:** Verify retry escalates to user after 3 failed attempts

### Setup

Create a test plan that will fail all 3 attempts (simulate permission denied):

```bash
cat > TEST-PLAN-02.md << 'EOF'
---
phase: test-retry
plan: 02
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Attempt impossible operation</name>
  <files>/root/protected-file.txt</files>
  <action>
    Write to /root/protected-file.txt (requires root permissions)
    Content: "This will fail"
  </action>
  <verify>File /root/protected-file.txt exists</verify>
  <done>Protected file created</done>
</task>

</tasks>
EOF
```

### Alternative: Simulate with Bash Command Failure

If you don't want to use permission-denied, use a command that always fails:

```bash
cat > TEST-PLAN-02-ALT.md << 'EOF'
---
phase: test-retry
plan: 02
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Run command that always fails</name>
  <files>n/a</files>
  <action>
    Execute bash command: exit 1
    (This command always returns failure)
  </action>
  <verify>Command succeeds (exit code 0)</verify>
  <done>Command executed successfully</done>
</task>

</tasks>
EOF
```

### Expected Behavior

**Attempt 1 (Original):**
- Try to write to /root/protected-file.txt
- **FAILS:** Permission denied (EACCES)
- Failure analysis: Category = "Resource" or "Dependency" (permission-denied)
- Retry appropriate: true

**Attempt 2 (Alternative 1):**
- Try alternative strategy (e.g., use sudo, change permissions, use different path)
- **FAILS:** Still can't access (sudo needs password, or alternative also blocked)
- Retry appropriate: true (not at max attempts yet)

**Attempt 3 (Alternative 2):**
- Try another alternative (e.g., write to temp location)
- **FAILS:** Still doesn't satisfy verification criteria
- Max attempts reached: Escalate

**Escalation:**
- checkpoint:decision presented to user
- Options: Manual intervention, Try different approach, Skip task, Abort plan

### Verification Steps

1. **Run the plan execution**

2. **Verify 3 attempts were made:**
   ```bash
   grep -c "<attempt>" .planning/ATTEMPTS_LOG.md
   # Expected: 3
   ```

3. **Check frontmatter shows escalation:**
   ```bash
   head -10 .planning/ATTEMPTS_LOG.md
   # Expected:
   # total-attempts: 3
   # final-outcome: escalated
   ```

4. **Verify all attempts show failure:**
   ```bash
   grep "<outcome>failure</outcome>" .planning/ATTEMPTS_LOG.md
   # Expected: 3 matches
   ```

5. **Check summary explains escalation:**
   ```bash
   grep -A 10 "## Summary" .planning/ATTEMPTS_LOG.md
   # Expected: "What worked: Nothing", "Escalation reason: All available alternatives exhausted"
   ```

6. **Verify checkpoint:decision was presented:**
   - Check that execution paused for user input
   - Verify context includes all 3 attempts and their outcomes

### Success Criteria

- ✅ System attempts exactly 3 retries before escalating
- ✅ ATTEMPTS_LOG.md contains 3 attempt entries, all failures
- ✅ Each attempt uses different strategy
- ✅ Final outcome is "escalated"
- ✅ Summary explains why escalation occurred
- ✅ checkpoint:decision presented with full context

---

## Test Scenario 3: Backward Compatibility (Retry Disabled)

**Objective:** Verify system works without retry when disabled in config

### Setup

1. **Disable retry in config:**
   ```bash
   cat > .planning/config.json << 'EOF'
   {
     "retry": {
       "enabled": false,
       "max_attempts": 3,
       "log_attempts": true
     }
   }
   EOF
   ```

2. **Create simple test plan:**
   ```bash
   cat > TEST-PLAN-03.md << 'EOF'
   ---
   phase: test-backward-compat
   plan: 03
   type: execute
   ---

   <tasks>

   <task type="auto">
     <name>Task 1: Simple successful operation</name>
     <files>backward-compat-test.txt</files>
     <action>
       Write "Backward compatibility test" to backward-compat-test.txt
     </action>
     <verify>File backward-compat-test.txt exists</verify>
     <done>File created</done>
   </task>

   </tasks>
   EOF
   ```

### Expected Behavior

- Plan executes using original execute-phase.md flow (without retry wrapper)
- Task executes once (no retry attempts)
- No ATTEMPTS_LOG.md created
- No failure analysis or alternative selection invoked
- If task fails, immediate escalation (no automatic retry)

### Verification Steps

1. **Run the plan execution**

2. **Verify no ATTEMPTS_LOG.md created:**
   ```bash
   ls .planning/ATTEMPTS_LOG.md
   # Expected: File not found (or file doesn't exist error)
   ```

3. **Check that task completed successfully:**
   ```bash
   cat backward-compat-test.txt
   # Expected: "Backward compatibility test"
   ```

4. **Create a failing test** to verify no retry happens:
   ```bash
   cat > TEST-PLAN-03-FAIL.md << 'EOF'
   ---
   phase: test-backward-compat
   plan: 03-fail
   type: execute
   ---

   <tasks>

   <task type="auto">
     <name>Task 1: Operation that will fail</name>
     <files>n/a</files>
     <action>
       Execute bash command: exit 1
     </action>
     <verify>Command succeeds</verify>
     <done>Task completed</done>
   </task>

   </tasks>
   EOF
   ```

5. **Run failing test:**
   - Expected: Task fails immediately
   - No retry attempts
   - No ATTEMPTS_LOG.md
   - Execution stops (or continues to next task, depending on execute-phase.md design)

### Success Criteria

- ✅ Retry logic completely bypassed when config.retry.enabled = false
- ✅ No ATTEMPTS_LOG.md created
- ✅ No alternative paths loaded or consulted
- ✅ No failure analysis performed
- ✅ Original execute-phase.md behavior preserved
- ✅ Failing tasks don't trigger retry loop

---

## Test Scenario 4: Logging Disabled (Retry Works, No Logs)

**Objective:** Verify retry works but doesn't create logs when log_attempts = false

### Setup

1. **Enable retry but disable logging:**
   ```bash
   cat > .planning/config.json << 'EOF'
   {
     "retry": {
       "enabled": true,
       "max_attempts": 3,
       "log_attempts": false
     }
   }
   EOF
   ```

2. **Use test plan from Scenario 1:**
   ```bash
   # Reuse TEST-PLAN-01.md (the one that fails then succeeds on retry)
   cat TEST-PLAN-01.md
   ```

### Expected Behavior

- Retry workflow executes normally
- Failure analysis runs
- Alternative strategy selected
- Task succeeds on attempt 2
- **No ATTEMPTS_LOG.md created** (logging disabled)
- No logging errors (gracefully skipped)

### Verification Steps

1. **Run the plan execution**

2. **Verify task completed successfully:**
   ```bash
   cat test-file.txt
   # Expected: Contains "Hello World"
   ```

3. **Verify NO ATTEMPTS_LOG.md created:**
   ```bash
   ls .planning/ATTEMPTS_LOG.md
   # Expected: File not found (or no such file error)
   ```

4. **Check no logging errors occurred:**
   - Review execution output
   - Should see no warnings about failed logging
   - Retry workflow should have silently skipped all logging steps

### Success Criteria

- ✅ Retry workflow executes successfully
- ✅ Task completes after retry (proving retry logic worked)
- ✅ No ATTEMPTS_LOG.md file created
- ✅ No error messages about logging failures
- ✅ Retry system works independently of logging feature

---

## Additional Test Cases

### Test 5: Logging Failure Doesn't Break Execution

**Objective:** Verify execution continues even if logging fails

**Setup:**
1. Enable retry and logging in config
2. Make .planning/ directory read-only to simulate logging failure:
   ```bash
   chmod 444 .planning/
   ```

**Expected:**
- Retry workflow executes
- Warning logged: "Could not initialize ATTEMPTS_LOG.md"
- Execution continues normally
- Task completes successfully

**Cleanup:**
```bash
chmod 755 .planning/
```

---

### Test 6: Logic Error (Immediate Escalation)

**Objective:** Verify certain failure types skip retry and escalate immediately

**Setup:**
```bash
cat > TEST-PLAN-06.md << 'EOF'
---
phase: test-logic-error
plan: 06
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Edit file that doesn't exist (logic error)</name>
  <files>non-existent-file-never-created.txt</files>
  <action>
    Edit non-existent-file-never-created.txt
    Add line: "This file doesn't exist"
  </action>
  <verify>File contains new line</verify>
  <done>File updated</done>
</task>

</tasks>
EOF
```

**Expected:**
- Attempt 1 fails (file not found)
- Failure analysis: Category = "Logic" (file assumed to exist but doesn't)
- retry_appropriate = false
- **Immediate escalation** (no retries attempted)
- checkpoint:decision presented

**Verification:**
```bash
# If ATTEMPTS_LOG.md created, should show only 1 attempt and immediate escalation
grep -c "<attempt>" .planning/ATTEMPTS_LOG.md
# Expected: 1

grep "final-outcome" .planning/ATTEMPTS_LOG.md
# Expected: escalated
```

---

## Test Results Checklist

After running all scenarios, verify:

- [ ] **Scenario 1:** Retry succeeds after 2 attempts, log shows success
- [ ] **Scenario 2:** System escalates after 3 attempts, log shows escalation
- [ ] **Scenario 3:** Retry disabled = original flow, no logs created
- [ ] **Scenario 4:** Retry works without logging, no ATTEMPTS_LOG.md
- [ ] **Test 5:** Logging failures don't break execution
- [ ] **Test 6:** Logic errors trigger immediate escalation

**Overall System Health:**
- [ ] ATTEMPTS_LOG.md structure matches template
- [ ] XML entries are well-formed and parseable
- [ ] Frontmatter updates correctly
- [ ] Summary sections provide useful insights
- [ ] Failure categories match taxonomy
- [ ] Alternative path selection works correctly
- [ ] Escalation context is comprehensive
- [ ] No execution breakage from any retry/logging operation

---

## Simulating Failures with Bash

Useful commands for creating test failures:

```bash
# Permission denied
touch /root/test.txt  # Fails unless running as root

# File not found
cat non-existent-file.txt  # Exit code 1

# Command failure
exit 1  # Always fails

# Network timeout (if testing network-dependent tasks)
curl --max-time 1 http://10.255.255.1  # Unreachable IP, times out

# Disk full (dangerous - don't actually fill disk!)
# Instead, write to tmpfs with limited space
dd if=/dev/zero of=/tmp/test-file bs=1M count=99999  # May fail if /tmp is small

# Invalid syntax
bash -c "invalid syntax here"  # Exit code 2
```

## Cleanup After Tests

```bash
# Remove test files
rm -f .planning/test-workspace/*.txt
rm -f .planning/test-workspace/TEST-PLAN-*.md
rm -f .planning/ATTEMPTS_LOG.md

# Reset config (or delete if you created it just for testing)
rm .planning/config.json
# Or restore original:
git checkout .planning/config.json

# Remove test workspace
rmdir .planning/test-workspace
```

---

## Automated Test Script (Optional)

For convenience, here's a bash script to run all tests:

```bash
#!/bin/bash
# run-retry-tests.sh - Automated retry system verification

set -e  # Exit on error

echo "=== Retry System Verification Tests ==="
echo ""

# Setup
echo "Setting up test environment..."
mkdir -p .planning/test-workspace
cd .planning/test-workspace

# Enable retry
cat > ../config.json << 'EOF'
{
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  }
}
EOF

echo "✓ Test environment ready"
echo ""

# Test 1: Success on retry
echo "Test 1: Simulated tool failure (success on retry)"
# [Manual execution required - execute TEST-PLAN-01.md]
echo "  → Manual execution required"
echo ""

# Test 2: Exhausted alternatives
echo "Test 2: Exhausted alternatives (escalation)"
# [Manual execution required - execute TEST-PLAN-02.md]
echo "  → Manual execution required"
echo ""

# Test 3: Backward compatibility
echo "Test 3: Backward compatibility (retry disabled)"
cat > ../config.json << 'EOF'
{
  "retry": {
    "enabled": false
  }
}
EOF
# [Manual execution required - execute TEST-PLAN-03.md]
echo "  → Manual execution required"
echo ""

# Test 4: Logging disabled
echo "Test 4: Logging disabled"
cat > ../config.json << 'EOF'
{
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": false
  }
}
EOF
# [Manual execution required - execute TEST-PLAN-01.md]
echo "  → Manual execution required"
echo ""

echo "=== All tests defined ==="
echo "Execute each test plan manually and verify results"
echo "See verify-retry-system.md for expected outcomes"
```

**Note:** Since retry-orchestration.md is a workflow guide (not executable code), these tests require manual execution by an agent following the workflow. The test plans and verification steps provide the structure for manual testing.
