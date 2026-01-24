# Adaptive System Verification Tests

This guide provides manual test procedures to validate that the AI-powered adaptive execution system works correctly. These tests ensure adaptive failure analysis, intelligent strategy selection, execution history tracking, and learning signals integrate properly with the Phase 1 retry foundation.

## Purpose

Validate Phase 4 enhancements:
1. AI-powered failure analysis (vs static taxonomy lookup)
2. Adaptive strategy selection (vs hardcoded algorithm)
3. Execution history tracking across attempts
4. Learning signal generation for future improvements
5. Graceful fallback to static logic on AI errors
6. Complete backward compatibility when adaptive disabled

## Prerequisites

### Required Configuration

```bash
# Create or update .planning/config.json with adaptive enabled
cat > .planning/config.json << 'EOF'
{
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true,
    "max_api_calls_per_task": 5,
    "history_tracking": true
  }
}
EOF
```

### Required Files

```bash
# Verify all dependencies exist
ls get-shit-done/lib/failure-analysis.md           # Static failure taxonomy
ls get-shit-done/lib/path-selection.md             # Static path selection algorithm
ls get-shit-done/lib/adaptive-failure-analyzer.md  # AI failure analyzer
ls get-shit-done/lib/adaptive-strategy-selector.md # AI strategy selector
ls get-shit-done/lib/execution-history.md          # History tracker
ls get-shit-done/templates/attempts-log.md         # Attempts log template
ls get-shit-done/templates/execution-history.md    # History template
ls .planning/ALTERNATIVE_PATHS.md                  # Strategy library
```

### Test Workspace Setup

```bash
# Create test directory
mkdir -p .planning/test-workspace-adaptive
cd .planning/test-workspace-adaptive
```

---

## Test Scenario 1: Adaptive vs Static Comparison

**Objective:** Verify adaptive mode selects different/better strategies than static algorithm

### Setup

Create a test plan that will fail with a Tool category error:

```bash
cat > TEST-ADAPTIVE-01.md << 'EOF'
---
phase: test-adaptive
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Edit file with non-existent match</name>
  <files>sample.txt</files>
  <action>
    Create sample.txt with "Hello"
    Then use Edit tool to replace "Goodbye" with "World"
    (This will fail - "Goodbye" doesn't exist)
  </action>
  <verify>File sample.txt contains "World"</verify>
  <done>File updated successfully</done>
</task>

</tasks>
EOF
```

### Test Execution

**Test 1a: Run with adaptive disabled (static path selection)**

```bash
# Temporarily disable adaptive
cat > .planning/config.json << 'EOF'
{
  "retry": { "enabled": true, "log_attempts": true },
  "adaptive": { "enabled": false }
}
EOF

# Execute plan (manually invoke execute-phase.md workflow or use GSD agent)
# Expected: Static algorithm selects strategy based on failure-analysis.md taxonomy
```

**Test 1b: Run with adaptive enabled (AI-powered selection)**

```bash
# Enable adaptive
cat > .planning/config.json << 'EOF'
{
  "retry": { "enabled": true, "log_attempts": true },
  "adaptive": { "enabled": true, "fallback_to_static": true }
}
EOF

# Execute same plan again (reset test environment first)
# Expected: AI analyzes failure and selects strategy based on execution history
```

### Verification Steps

1. **Compare ATTEMPTS_LOG.md from both runs:**
   ```bash
   # Static run
   grep "<strategy>" .planning/ATTEMPTS_LOG-static.md
   # Example: <strategy>read-then-edit</strategy> (from static algorithm)

   # Adaptive run
   grep "<strategy>" .planning/ATTEMPTS_LOG.md
   # Example: <strategy>write-new-content</strategy> (from AI selection)
   ```

2. **Check AI reasoning in adaptive run:**
   ```bash
   # Look for AI analysis in ATTEMPTS_LOG.md
   grep -A 10 "AI Analysis" .planning/ATTEMPTS_LOG.md
   # Expected: Contains failure diagnosis and strategy rationale
   ```

3. **Verify both approaches recovered:**
   ```bash
   # Both should succeed, possibly via different strategies
   cat sample.txt  # Should contain "World" in both cases
   ```

### Success Criteria

- ✅ Both static and adaptive modes recover from failure
- ✅ Adaptive mode may select different strategy than static algorithm
- ✅ ATTEMPTS_LOG.md shows strategy selection source (static vs AI)
- ✅ AI reasoning documented in adaptive run
- ✅ Final outcome is "success" in both cases

---

## Test Scenario 2: API Failure Graceful Fallback

**Objective:** Verify system falls back to static analysis when AI fails

### Setup

Simulate Task tool failure (AI unavailable or returns invalid output):

```bash
cat > TEST-ADAPTIVE-02.md << 'EOF'
---
phase: test-adaptive-fallback
plan: 02
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Trigger AI failure scenario</name>
  <files>test-fallback.txt</files>
  <action>
    Attempt operation that will fail
    (AI analysis will be attempted but should fail/timeout)
  </action>
  <verify>Task recovers via static fallback</verify>
  <done>Fallback successful</done>
</task>

</tasks>
EOF
```

**Simulate AI failure by:**
- Setting very short timeout (causes Task tool to fail)
- Using invalid prompt format (causes parse error)
- Temporarily making Task tool unavailable

### Expected Behavior

**Attempt 1 (Original):**
- Task fails with some error
- Adaptive failure analyzer invoked (AI)
- **AI analysis fails** (timeout, invalid response, etc.)
- System detects AI failure
- **Falls back to static failure-analysis.md**
- Static analysis categorizes failure
- Retry continues with static algorithm

**Attempt 2 (Alternative - selected by static fallback):**
- Strategy selected using Phase 1 static algorithm
- Task recovers successfully
- ATTEMPTS_LOG.md documents fallback occurred

### Verification Steps

1. **Check fallback was triggered:**
   ```bash
   grep -i "fallback" .planning/ATTEMPTS_LOG.md
   # Expected: Message indicating AI failed, falling back to static
   ```

2. **Verify static analysis used:**
   ```bash
   grep "failure-category" .planning/ATTEMPTS_LOG.md
   # Expected: Category from failure-analysis.md taxonomy (not AI)
   ```

3. **Confirm task still recovered:**
   ```bash
   grep "final-outcome" .planning/ATTEMPTS_LOG.md
   # Expected: success (fallback worked)
   ```

4. **Check no execution breakage:**
   ```bash
   # Verify retry workflow didn't abort due to AI failure
   # Task should complete successfully via static path
   ```

### Success Criteria

- ✅ AI analysis fails (simulated failure condition)
- ✅ System detects AI failure and logs fallback
- ✅ Static analysis takes over automatically
- ✅ Retry continues normally with static algorithm
- ✅ Task recovers successfully despite AI unavailability
- ✅ No errors or workflow interruption

---

## Test Scenario 3: History Tracking Accumulation

**Objective:** Verify EXECUTION_HISTORY.md populates and updates correctly

### Setup

Execute multiple tasks with retry scenarios to accumulate history data:

```bash
# Test 1: Tool failure → success
cat > TEST-HISTORY-01.md << 'EOF'
<task type="auto">
  <name>Edit with wrong match</name>
  <action>Edit non-existent string (fails), retry with read-then-edit (succeeds)</action>
</task>
EOF

# Test 2: Dependency failure → success
cat > TEST-HISTORY-02.md << 'EOF'
<task type="auto">
  <name>Missing dependency</name>
  <action>Import missing module (fails), retry after install (succeeds)</action>
</task>
EOF

# Test 3: Resource failure → success
cat > TEST-HISTORY-03.md << 'EOF'
<task type="auto">
  <name>Permission denied</name>
  <action>Write to protected path (fails), retry with user-accessible path (succeeds)</action>
</task>
EOF
```

### Test Execution

```bash
# Enable history tracking
cat > .planning/config.json << 'EOF'
{
  "retry": { "enabled": true, "log_attempts": true },
  "adaptive": { "enabled": true, "history_tracking": true }
}
EOF

# Execute all three test plans in sequence
# Each should fail → retry → succeed
```

### Verification Steps

1. **Check EXECUTION_HISTORY.md exists:**
   ```bash
   ls -la .planning/EXECUTION_HISTORY.md
   # Expected: File created after first retry execution
   ```

2. **Verify strategy effectiveness table populated:**
   ```bash
   grep -A 10 "## Strategy Effectiveness" .planning/EXECUTION_HISTORY.md
   # Expected: Table showing strategies used and success rates
   ```

3. **Check failure patterns recorded:**
   ```bash
   grep -A 10 "## Failure Patterns" .planning/EXECUTION_HISTORY.md
   # Expected: List of failure categories and frequencies
   ```

4. **Verify data accumulates across executions:**
   ```bash
   # After first test
   grep -c "Tool" .planning/EXECUTION_HISTORY.md  # Expected: 1

   # After second test
   grep -c "Dependency" .planning/EXECUTION_HISTORY.md  # Expected: 1

   # After third test
   grep -c "Resource" .planning/EXECUTION_HISTORY.md  # Expected: 1
   ```

5. **Check success rate calculations:**
   ```bash
   # Look for success percentages in effectiveness table
   grep "%" .planning/EXECUTION_HISTORY.md
   # Expected: Success rates for each strategy (e.g., "read-then-edit: 100%")
   ```

### Success Criteria

- ✅ EXECUTION_HISTORY.md created on first retry execution
- ✅ Strategy effectiveness table shows all used strategies
- ✅ Success rates calculated correctly (successes / total attempts)
- ✅ Failure patterns section lists failure categories
- ✅ Data accumulates across multiple test executions
- ✅ History file remains well-formed markdown (no corruption)

---

## Test Scenario 4: Learning Signal Generation

**Objective:** Verify ATTEMPTS_LOG.md contains actionable learning signals

### Setup

Execute a task that succeeds on retry attempt 2:

```bash
cat > TEST-LEARNING-01.md << 'EOF'
---
phase: test-learning
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Recoverable failure</name>
  <files>learning-test.txt</files>
  <action>
    Step 1: Try to edit non-existent string (will fail)
    Step 2: Retry with alternative strategy (will succeed)
  </action>
  <verify>File contains expected content</verify>
  <done>Task completed via retry</done>
</task>

</tasks>
EOF
```

### Test Execution

```bash
# Enable adaptive with history tracking
cat > .planning/config.json << 'EOF'
{
  "retry": { "enabled": true, "log_attempts": true },
  "adaptive": { "enabled": true, "history_tracking": true }
}
EOF

# Execute plan
# Expected: Attempt 1 fails, Attempt 2 succeeds
```

### Verification Steps

1. **Check learning signals section exists:**
   ```bash
   grep -A 20 "## Learning Signals" .planning/ATTEMPTS_LOG.md
   # Expected: Section with strategy effectiveness, failure pattern, recommendations
   ```

2. **Verify strategy effectiveness documented:**
   ```bash
   grep "Strategy effectiveness" .planning/ATTEMPTS_LOG.md
   # Expected: Which strategy worked and why
   ```

3. **Check failure pattern identified:**
   ```bash
   grep "Failure pattern" .planning/ATTEMPTS_LOG.md
   # Expected: Category and root cause identified
   ```

4. **Verify recommendations provided:**
   ```bash
   grep "Recommended next time" .planning/ATTEMPTS_LOG.md
   # Expected: Actionable guidance for similar failures
   ```

5. **Validate learning data structure:**
   ```bash
   # Check all learning signal components present:
   # - What worked (strategy that succeeded)
   # - Why it worked (root cause analysis)
   # - Pattern detected (failure category + context)
   # - Recommended approach (for future similar scenarios)
   ```

### Success Criteria

- ✅ Learning Signals section present in ATTEMPTS_LOG.md
- ✅ Strategy effectiveness explains what worked
- ✅ Failure pattern identifies root cause
- ✅ Recommendations are actionable and specific
- ✅ Learning data formatted consistently
- ✅ Signals suitable for EXECUTION_HISTORY.md aggregation

---

## Test Scenario 5: Max API Calls Limit

**Objective:** Verify max_api_calls_per_task prevents runaway API usage

### Setup

Configure low API call limit and create task requiring multiple retries:

```bash
# Set low limit
cat > .planning/config.json << 'EOF'
{
  "retry": { "enabled": true, "max_attempts": 5, "log_attempts": true },
  "adaptive": {
    "enabled": true,
    "max_api_calls_per_task": 2,
    "fallback_to_static": true
  }
}
EOF

cat > TEST-API-LIMIT-01.md << 'EOF'
---
phase: test-api-limit
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Multiple failures requiring retries</name>
  <files>api-limit-test.txt</files>
  <action>
    Attempt operation that will fail 3+ times
    (Tests that API calls are limited to 2, then fallback kicks in)
  </action>
  <verify>Task eventually succeeds</verify>
  <done>Completed with API limit respected</done>
</task>

</tasks>
EOF
```

### Expected Behavior

**Attempt 1 (Original):**
- Task fails
- AI analysis invoked (API call #1)
- Strategy selected via AI
- Retry with AI-selected strategy

**Attempt 2 (First retry):**
- Task fails again
- AI analysis invoked (API call #2)
- Strategy selected via AI
- Retry with second AI-selected strategy

**Attempt 3 (Second retry):**
- Task fails again
- **Max API calls reached (2)**
- **Falls back to static analysis** (no API call)
- Strategy selected via static algorithm
- Retry continues

**Attempts 4-5 (if needed):**
- Continue with static fallback (no more AI calls)
- Task eventually succeeds or escalates

### Verification Steps

1. **Count AI invocations in ATTEMPTS_LOG.md:**
   ```bash
   grep -c "AI Analysis" .planning/ATTEMPTS_LOG.md
   # Expected: Exactly 2 (not more)
   ```

2. **Verify fallback triggered at attempt 3:**
   ```bash
   grep -A 5 "<attempt>3</attempt>" .planning/ATTEMPTS_LOG.md
   # Expected: Shows static analysis used (not AI)
   ```

3. **Check fallback message logged:**
   ```bash
   grep "max_api_calls" .planning/ATTEMPTS_LOG.md
   # Expected: Message indicating limit reached, falling back
   ```

4. **Confirm task didn't fail due to limit:**
   ```bash
   grep "final-outcome" .planning/ATTEMPTS_LOG.md
   # Expected: success (limit didn't break execution)
   ```

5. **Verify no runaway API usage:**
   ```bash
   # Check that total Task tool calls <= max_api_calls_per_task
   # System should respect limit and fall back gracefully
   ```

### Success Criteria

- ✅ AI analysis invoked exactly max_api_calls_per_task times (2)
- ✅ After limit reached, falls back to static analysis
- ✅ Fallback message logged in ATTEMPTS_LOG.md
- ✅ Task execution continues normally after fallback
- ✅ Task eventually succeeds or escalates normally
- ✅ No API call limit exceeded errors

---

## Test Scenario 6: Backward Compatibility

**Objective:** Verify existing projects work without adaptive features

### Setup

Simulate existing project (no adaptive configuration):

```bash
# Create config WITHOUT adaptive section (v2.0 style)
cat > .planning/config.json << 'EOF'
{
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  }
}
EOF

# Or completely remove config.json (default behavior)
rm .planning/config.json
```

### Test Execution

```bash
# Use simple test plan from Scenario 1
cat > TEST-BACKWARD-COMPAT.md << 'EOF'
---
phase: test-backward-compat
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Basic retry scenario</name>
  <files>backward-compat.txt</files>
  <action>
    Attempt operation that will fail then succeed on retry
  </action>
  <verify>Task completes successfully</verify>
  <done>Backward compatibility confirmed</done>
</task>

</tasks>
EOF

# Execute plan
```

### Expected Behavior

- Retry workflow executes using **Phase 1 static logic only**
- Failure analysis uses failure-analysis.md taxonomy (not AI)
- Path selection uses path-selection.md algorithm (not AI)
- ATTEMPTS_LOG.md created with standard format (no learning signals)
- No EXECUTION_HISTORY.md created (history tracking disabled)
- No AI analysis attempted
- No errors about missing adaptive configuration
- **Identical behavior to v2.0 retry system**

### Verification Steps

1. **Verify no adaptive features invoked:**
   ```bash
   grep -i "AI" .planning/ATTEMPTS_LOG.md
   # Expected: No matches (no AI analysis)
   ```

2. **Check static failure analysis used:**
   ```bash
   grep "failure-category" .planning/ATTEMPTS_LOG.md
   # Expected: Category from failure-analysis.md (static taxonomy)
   ```

3. **Verify no history file created:**
   ```bash
   ls .planning/EXECUTION_HISTORY.md
   # Expected: File not found (history tracking off)
   ```

4. **Check no learning signals section:**
   ```bash
   grep "Learning Signals" .planning/ATTEMPTS_LOG.md
   # Expected: No matches (signals not generated without adaptive)
   ```

5. **Confirm task completes normally:**
   ```bash
   cat backward-compat.txt
   # Expected: File contains expected content (retry worked)

   grep "final-outcome" .planning/ATTEMPTS_LOG.md
   # Expected: success
   ```

6. **Verify no configuration errors:**
   ```bash
   # Check execution output for any warnings about missing adaptive config
   # Expected: No warnings (backward compatible defaults used)
   ```

### Success Criteria

- ✅ Retry works identically to Phase 1 static behavior
- ✅ No AI analysis attempted (pure static)
- ✅ No EXECUTION_HISTORY.md created
- ✅ No learning signals generated
- ✅ ATTEMPTS_LOG.md uses standard format
- ✅ No errors or warnings about missing adaptive config
- ✅ Perfect backward compatibility with v2.0

---

## Test Results Checklist

After running all scenarios, verify:

- [ ] **Scenario 1:** Adaptive selects different/better strategies than static
- [ ] **Scenario 2:** AI failure gracefully falls back to static analysis
- [ ] **Scenario 3:** EXECUTION_HISTORY.md accumulates data across executions
- [ ] **Scenario 4:** Learning signals generated with actionable recommendations
- [ ] **Scenario 5:** max_api_calls_per_task limit respected, fallback works
- [ ] **Scenario 6:** Backward compatibility perfect (v2.0 behavior without adaptive)

**Overall Adaptive System Health:**
- [ ] AI analysis produces valid failure diagnoses
- [ ] Strategy selection uses execution history effectively
- [ ] Learning signals are actionable and specific
- [ ] History tracking aggregates patterns correctly
- [ ] API call limits prevent runaway usage
- [ ] Fallback to static is seamless
- [ ] No execution breakage from adaptive features
- [ ] Backward compatibility 100% maintained

---

## Cleanup After Tests

```bash
# Remove test files
rm -f .planning/test-workspace-adaptive/*.txt
rm -f .planning/test-workspace-adaptive/TEST-*.md
rm -f .planning/ATTEMPTS_LOG.md
rm -f .planning/EXECUTION_HISTORY.md

# Reset config (or delete if created just for testing)
git checkout .planning/config.json
# Or restore original
cp .planning/config.json.backup .planning/config.json

# Remove test workspace
rmdir .planning/test-workspace-adaptive
```

---

## Advanced Testing

### Performance Comparison

Compare execution time and strategy effectiveness:

```bash
# Run same failing task 10 times with static
time ./run-test-static.sh  # Measure average recovery time

# Run same failing task 10 times with adaptive
time ./run-test-adaptive.sh  # Measure average recovery time

# Compare results
# Expected: Adaptive may recover faster due to learning from history
```

### History Learning Validation

Test that history actually improves future decisions:

```bash
# First run: Task fails with Strategy A, succeeds with Strategy B
# EXECUTION_HISTORY.md records: Strategy B effective for this failure pattern

# Second run: Same failure pattern occurs
# Expected: Adaptive selector chooses Strategy B immediately (learned from history)
# Result: Faster recovery, fewer attempts needed
```

### API Cost Monitoring

Track actual API usage:

```bash
# Enable detailed logging
export GSD_DEBUG=1

# Run test plan with multiple retries
# Count actual Task tool invocations

# Verify:
# - Respects max_api_calls_per_task limit
# - Falls back before exceeding limit
# - No surprise API costs
```

---

## Troubleshooting

### Adaptive Not Activating

**Check:**
```bash
# Verify retry enabled (dependency)
grep "retry.*enabled.*true" .planning/config.json

# Verify adaptive enabled
grep "adaptive.*enabled.*true" .planning/config.json

# Check Task tool available
# Ensure not in sandboxed environment where Task tool blocked
```

### AI Analysis Returning Invalid Output

**Expected:**
- Fallback to static should trigger automatically
- Check fallback_to_static = true in config
- Verify fallback message in ATTEMPTS_LOG.md

**If fallback fails:**
- Check static failure-analysis.md and path-selection.md exist
- Ensure static algorithms still functional

### History Not Tracking

**Check:**
```bash
# Verify adaptive enabled (dependency)
grep "adaptive.*enabled.*true" .planning/config.json

# Verify history_tracking enabled
grep "history_tracking.*true" .planning/config.json

# Check .planning/ directory writable
touch .planning/test-write && rm .planning/test-write

# Ensure at least one retry occurred
# History only updates after retry completion
```

### Learning Signals Missing

**Check:**
- Requires adaptive.enabled = true
- Only generated after retry succeeds or escalates
- Not generated if retry disabled or task succeeds on first attempt

---

## Automated Test Runner (Optional)

```bash
#!/bin/bash
# run-adaptive-tests.sh - Automated adaptive system verification

set -e

echo "=== Adaptive System Verification Tests ==="
echo ""

# Setup
echo "Setting up test environment..."
mkdir -p .planning/test-workspace-adaptive
cd .planning/test-workspace-adaptive

# Enable adaptive
cat > ../config.json << 'EOF'
{
  "retry": { "enabled": true, "log_attempts": true },
  "adaptive": { "enabled": true, "fallback_to_static": true }
}
EOF

echo "✓ Test environment ready"
echo ""

# Tests 1-6 require manual execution
for i in {1..6}; do
  echo "Test Scenario $i: [Manual execution required]"
  echo "  → See verify-adaptive-system.md for test plan"
  echo ""
done

echo "=== All test scenarios defined ==="
echo "Execute each manually and verify results"
echo "See verify-adaptive-system.md for expected outcomes"
```

---

## Summary

These tests validate that Phase 4's adaptive execution system:

1. **Enhances retry with AI** - Smarter failure analysis and strategy selection
2. **Learns from history** - Tracks patterns and improves over time
3. **Falls back gracefully** - Never breaks on AI errors
4. **Respects API limits** - Prevents runaway costs
5. **Maintains compatibility** - Works identically to v2.0 when disabled
6. **Generates learning signals** - Provides actionable insights for improvement

**Test coverage:** 6 scenarios covering all adaptive features and edge cases
**Execution:** Manual testing with bash simulation (follows Phase 1 pattern)
**Outcome:** Complete validation of v3.0 Intelligence milestone
