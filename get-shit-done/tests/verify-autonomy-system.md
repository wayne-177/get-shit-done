# Autonomy System Verification Tests

This guide provides manual test procedures to validate that the v4.0 Autonomy features work correctly. These tests ensure the knowledge base, pattern extraction, cross-project sync, pattern analysis, failure prediction, and proactive suggestions integrate properly with the existing v2.0/v3.0 foundation.

## Purpose

Validate v4.0 Autonomy enhancements:
1. Knowledge base initialization at `~/.claude/gsd-knowledge/`
2. Pattern extraction from EXECUTION_HISTORY.md
3. Cross-project pattern sync on milestone completion
4. Pattern import and hints display for new projects
5. Pattern analysis and effectiveness scoring
6. Pattern decay and lifecycle management
7. Failure prediction during planning
8. Proactive alternative suggestions
9. End-to-end learning pipeline
10. Complete backward compatibility with v2.0/v3.0

## Prerequisites

### Required Configuration

```bash
# Create or update .planning/config.json with full v4.0 features
cat > .planning/config.json << 'EOF'
{
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "verification": {
    "enabled": true
  },
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true,
    "max_api_calls_per_task": 5,
    "history_tracking": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
EOF
```

### Required Files

```bash
# Verify all v4.0 workflow dependencies exist
ls get-shit-done/workflows/init-knowledge-base.md
ls get-shit-done/workflows/extract-patterns.md
ls get-shit-done/workflows/import-patterns.md
ls get-shit-done/workflows/analyze-patterns.md
ls get-shit-done/workflows/predict-failures.md
ls get-shit-done/workflows/complete-milestone.md

# Verify reference documentation exists
ls get-shit-done/references/pattern-schema.md
ls get-shit-done/references/knowledge-base-api.md
ls get-shit-done/references/config-schema.md
```

### Test Workspace Setup

```bash
# Create test directory
mkdir -p .planning/test-workspace-autonomy
cd .planning/test-workspace-autonomy
```

---

## Test Scenario 1: Knowledge Base Initialization

**Objective:** Verify init-knowledge-base.md creates `~/.claude/gsd-knowledge/` structure correctly and is idempotent

### Setup

Remove knowledge base if exists (for clean test):

```bash
# CAUTION: Only run on test machine - this deletes global knowledge base
# mv ~/.claude/gsd-knowledge ~/.claude/gsd-knowledge-backup

# Or test on a fresh user account / container
```

### Test Execution

**Test 1a: Fresh initialization**

```bash
# Check knowledge base doesn't exist
ls ~/.claude/gsd-knowledge
# Expected: No such file or directory

# Run initialization workflow (manually invoke or trigger via execute-phase)
# The workflow should create all directories and files
```

### Verification Steps

1. **Verify directory structure created:**
   ```bash
   ls -la ~/.claude/gsd-knowledge/
   # Expected: patterns/ directory, index.json, config.json

   ls -la ~/.claude/gsd-knowledge/patterns/
   # Expected: strategy-effectiveness/, failure-pattern/, task-insight/
   ```

2. **Verify index.json structure:**
   ```bash
   cat ~/.claude/gsd-knowledge/index.json
   # Expected: JSON with version, last_updated, total_patterns=0, by_type arrays
   ```

3. **Verify config.json structure:**
   ```bash
   cat ~/.claude/gsd-knowledge/config.json
   # Expected: JSON with version, settings (auto_extract, min_confidence, etc.), statistics
   ```

4. **Verify JSON files are valid:**
   ```bash
   python3 -c "import json; json.load(open('$HOME/.claude/gsd-knowledge/index.json'))"
   python3 -c "import json; json.load(open('$HOME/.claude/gsd-knowledge/config.json'))"
   # Expected: No errors (valid JSON)
   ```

**Test 1b: Idempotent re-initialization**

```bash
# Run initialization again
# (Trigger init-knowledge-base workflow again)

# Check files not corrupted
cat ~/.claude/gsd-knowledge/index.json
# Expected: Same content as before, not overwritten

# Check total_patterns preserved if patterns existed
grep "total_patterns" ~/.claude/gsd-knowledge/index.json
```

### Success Criteria

- ✅ All five directories created (`patterns/` and three subdirs)
- ✅ `index.json` created with valid JSON structure
- ✅ `config.json` created with valid JSON structure
- ✅ Re-initialization doesn't corrupt existing data
- ✅ Re-initialization doesn't reset total_patterns counter
- ✅ Workflow outputs "Knowledge base initialized successfully"

---

## Test Scenario 2: Pattern Extraction

**Objective:** Verify extract-patterns workflow extracts patterns from EXECUTION_HISTORY.md correctly

### Setup

Create mock EXECUTION_HISTORY.md with test data:

```bash
cat > .planning/EXECUTION_HISTORY.md << 'EOF'
# Execution History

## Strategy Effectiveness

| Strategy Name | Times Used | Success Rate | Avg Attempt # | Task Types | Notes |
|--------------|------------|--------------|---------------|------------|-------|
| retry-with-reruns | 5 | 80% (4/5) | 1.5 | test-run | Effective for flaky tests |
| clean-install | 4 | 100% (4/4) | 2.0 | bash-command | Always works for dep issues |
| read-then-edit | 3 | 67% (2/3) | 1.8 | file-edit | Good for missing files |

## Common Failure Patterns

| Failure Category | Task Type | Frequency | Typical Recovery | Last Seen | Notes |
|-----------------|-----------|-----------|------------------|-----------|-------|
| Dependency | bash-command | 8 | clean-install | 2026-01-07 | npm install failures |
| Validation | test-run | 5 | retry-with-reruns | 2026-01-07 | Test timeout issues |
| Tool | file-edit | 4 | read-then-edit | 2026-01-06 | Edit match failures |

## Task Type Analysis

### bash-command
**Success Rate:** 75% (12/16)
**Common Failures:**
- Dependency: npm install failures (8)
**Effective Strategies:**
- clean-install: 100% (always reinstall)
**Insights:**
- Always check node_modules exists before running npm scripts

### test-run
**Success Rate:** 80% (16/20)
**Common Failures:**
- Validation: test timeouts (5)
**Effective Strategies:**
- retry-with-reruns: 80% (rerun flaky tests)
**Insights:**
- Running tests in isolation often fixes timing issues
EOF
```

### Test Execution

```bash
# Run extract-patterns workflow
# (Manually invoke or use /gsd:extract-patterns command)
```

### Verification Steps

1. **Verify patterns written to knowledge base:**
   ```bash
   ls ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/
   # Expected: se-*.json files for each strategy

   ls ~/.claude/gsd-knowledge/patterns/failure-pattern/
   # Expected: fp-*.json files for each failure pattern

   ls ~/.claude/gsd-knowledge/patterns/task-insight/
   # Expected: ti-*.json files for insights
   ```

2. **Verify pattern content:**
   ```bash
   # Check a strategy-effectiveness pattern
   cat ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json | head -50
   # Expected: JSON with id, type, confidence, evidence, content fields
   ```

3. **Verify index.json updated:**
   ```bash
   grep "total_patterns" ~/.claude/gsd-knowledge/index.json
   # Expected: total_patterns > 0

   cat ~/.claude/gsd-knowledge/index.json | grep -A 5 "strategy-effectiveness"
   # Expected: Array with pattern entries
   ```

4. **Verify config.json stats updated:**
   ```bash
   grep "total_extractions" ~/.claude/gsd-knowledge/config.json
   # Expected: total_extractions >= 1
   ```

**Test 2b: Deduplication**

```bash
# Run extraction again on same EXECUTION_HISTORY.md
# Expected: Patterns merged, not duplicated

# Check pattern count didn't double
BEFORE=$(ls ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json 2>/dev/null | wc -l)
# Run extraction again
AFTER=$(ls ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json 2>/dev/null | wc -l)

echo "Before: $BEFORE, After: $AFTER"
# Expected: BEFORE == AFTER (no new files)

# Check occurrences merged
grep "occurrences" ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json
# Expected: Higher occurrence counts (merged)
```

### Success Criteria

- ✅ strategy-effectiveness patterns extracted from Strategy Effectiveness table
- ✅ failure-pattern patterns extracted from Common Failure Patterns table
- ✅ task-insight patterns extracted from Task Type Analysis sections
- ✅ Pattern JSON follows schema (id, type, confidence, evidence, content)
- ✅ index.json updated with pattern references
- ✅ config.json statistics updated
- ✅ Re-extraction merges patterns (doesn't duplicate)
- ✅ Occurrence counts accumulate on merge

---

## Test Scenario 3: Cross-Project Pattern Export (Milestone Integration)

**Objective:** Verify complete-milestone.md triggers pattern extraction when adaptive.enabled=true

### Setup

```bash
# Ensure adaptive is enabled
grep '"adaptive"' .planning/config.json
grep '"enabled": true' .planning/config.json

# Create EXECUTION_HISTORY.md with patterns (reuse from Scenario 2)
# Ensure ROADMAP.md indicates milestone complete
```

### Test Execution

```bash
# Run /gsd:complete-milestone command
# (This triggers the milestone completion workflow)
```

### Verification Steps

1. **Verify pattern extraction triggered:**
   ```bash
   # Check for extraction output during milestone completion
   # Look for "PATTERN EXTRACTION COMPLETE" message
   ```

2. **Verify patterns synced to knowledge base:**
   ```bash
   ls ~/.claude/gsd-knowledge/patterns/*/*.json | wc -l
   # Expected: > 0 patterns
   ```

3. **Verify PATTERNS_EXPORTED placeholder populated:**
   ```bash
   # Check milestone archive for pattern count
   grep -r "PATTERNS_EXPORTED" .planning/ || echo "Check milestone archive"
   ```

### Success Criteria

- ✅ complete-milestone checks adaptive.enabled
- ✅ Pattern extraction triggered when enabled
- ✅ Patterns written to global knowledge base
- ✅ Milestone archive documents patterns exported
- ✅ Extraction failure doesn't block milestone completion (best-effort)

---

## Test Scenario 4: Cross-Project Pattern Import (New Project)

**Objective:** Verify import-patterns.md queries knowledge base and displays hints during execute-phase

### Setup

Ensure knowledge base has patterns (from previous scenarios):

```bash
ls ~/.claude/gsd-knowledge/patterns/*/*.json | wc -l
# Expected: > 0 patterns
```

Create a new project or plan:

```bash
cat > TEST-IMPORT-PLAN.md << 'EOF'
---
phase: test-import
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Run npm test</name>
  <files>package.json</files>
  <action>
    Run npm test to verify all tests pass
  </action>
  <verify>npm test exit code 0</verify>
  <done>Tests pass</done>
</task>

<task type="auto">
  <name>Task 2: Edit config file</name>
  <files>src/config.ts</files>
  <action>
    Update configuration settings
  </action>
  <verify>Config file updated</verify>
  <done>Config updated</done>
</task>

</tasks>
EOF
```

### Test Execution

```bash
# Execute the plan (triggers import-patterns during execute-phase)
```

### Verification Steps

1. **Verify pattern hints displayed:**
   ```bash
   # Look for "PREDICTIVE HINTS" block in execute-phase output
   # Example output:
   # ===============================================
   # PREDICTIVE HINTS (from cross-project learning)
   # ===============================================
   # **For test-run tasks:**
   # - Strategy "retry-with-reruns" has 80% success rate
   ```

2. **Verify confidence threshold filtering:**
   ```bash
   # Only patterns with confidence >= 0.6 should be shown as hints
   # Lower confidence patterns should be filtered out
   ```

3. **Verify task type matching:**
   ```bash
   # test-run hints shown for npm test task
   # file-edit hints shown for config editing task
   ```

### Success Criteria

- ✅ import-patterns queries knowledge base
- ✅ Pattern hints displayed during execute-phase
- ✅ Only patterns with confidence >= 0.6 shown
- ✅ Task types correctly matched to patterns
- ✅ Strategy hints include success rate
- ✅ Failure warnings show typical recovery

---

## Test Scenario 5: Pattern Analysis (Effectiveness Scoring)

**Objective:** Verify analyze-patterns workflow calculates effectiveness scores correctly

### Setup

Ensure knowledge base has patterns with evidence:

```bash
# Patterns should have evidence.occurrences and evidence.project_count
cat ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json | grep -A 5 '"evidence"'
```

### Test Execution

```bash
# Run analyze-patterns workflow
# (Manually invoke or use /gsd:analyze-patterns if command exists)
```

### Verification Steps

1. **Verify effectiveness_score calculated:**
   ```bash
   # Check patterns now have analysis.effectiveness_score
   grep "effectiveness_score" ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json
   # Expected: effectiveness_score values between 0 and 1
   ```

2. **Verify score formula:**
   ```bash
   # effectiveness_score = success_rate * confidence * log10(occurrences + 1)
   # Example: success_rate=0.8, confidence=0.75, occurrences=5
   # score = 0.8 * 0.75 * log10(6) = 0.8 * 0.75 * 0.778 = 0.467

   # Check a pattern's score is reasonable
   cat ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json | head -1
   ```

3. **Verify cross-project validation:**
   ```bash
   # Patterns with project_count >= 3 should have validated=true
   grep -l '"project_count": [3-9]' ~/.claude/gsd-knowledge/patterns/*/*.json | while read f; do
     grep "validated" "$f"
   done
   # Expected: validated: true for patterns with 3+ projects
   ```

4. **Verify score interpretation:**
   ```bash
   # Score > 0.8: high-value
   # Score 0.5-0.8: useful
   # Score < 0.5: decay candidate

   # Check high-value patterns
   for f in ~/.claude/gsd-knowledge/patterns/*/*.json; do
     SCORE=$(grep -o '"effectiveness_score": *[0-9.]*' "$f" | grep -o '[0-9.]*')
     [ -n "$SCORE" ] && (( $(echo "$SCORE > 0.8" | bc -l) )) && echo "High-value: $f ($SCORE)"
   done
   ```

### Success Criteria

- ✅ effectiveness_score calculated for all patterns
- ✅ Score formula: success_rate * confidence * log10(occurrences + 1)
- ✅ Cross-project validation at 3+ projects
- ✅ Score interpretation: >0.8 high-value, 0.5-0.8 useful, <0.5 decay candidate
- ✅ Analysis doesn't corrupt existing pattern data

---

## Test Scenario 6: Pattern Decay System

**Objective:** Verify decay system correctly ages patterns and respects validation status

### Setup

Create or modify patterns with different ages:

```bash
# Manually set last_seen dates for testing
# (Or wait sufficient time between tests)

# Create an old pattern for decay testing
cat > /tmp/old-pattern.json << 'EOF'
{
  "id": "se-test0001",
  "type": "strategy-effectiveness",
  "version": "1.0.0",
  "created": "2025-06-01T00:00:00Z",
  "last_updated": "2025-06-01T00:00:00Z",
  "confidence": 0.8,
  "evidence": {
    "occurrences": 5,
    "project_count": 1,
    "first_seen": "2025-06-01T00:00:00Z",
    "last_seen": "2025-06-01T00:00:00Z"
  },
  "content": {
    "strategy_name": "old-test-strategy",
    "task_types": ["test"],
    "success_rate": 0.8,
    "total_uses": 5,
    "successful_uses": 4
  },
  "analysis": {
    "effectiveness_score": 0.5,
    "validated": false
  }
}
EOF
cp /tmp/old-pattern.json ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/se-test0001.json
```

### Test Execution

```bash
# Run decay analysis
# (Check current state before any decay operations)

# Formula: decay_factor = 0.5^(days_since_last_seen / half_life_days)
# Default half_life_days = 365
```

### Verification Steps

1. **Verify decay formula:**
   ```bash
   # For a pattern 180 days old with half_life=365:
   # decay_factor = 0.5^(180/365) = 0.5^0.493 = 0.71

   # Decayed confidence = original_confidence * decay_factor
   # 0.8 * 0.71 = 0.57
   ```

2. **Verify lifecycle states:**
   ```bash
   # Active: confidence >= 0.5
   # Warning: confidence 0.3-0.5
   # Expired: confidence < 0.3

   for f in ~/.claude/gsd-knowledge/patterns/*/*.json; do
     CONF=$(grep -o '"confidence": *[0-9.]*' "$f" | grep -o '[0-9.]*')
     if [ -n "$CONF" ]; then
       if (( $(echo "$CONF >= 0.5" | bc -l) )); then
         STATE="Active"
       elif (( $(echo "$CONF >= 0.3" | bc -l) )); then
         STATE="Warning"
       else
         STATE="Expired"
       fi
       echo "$f: confidence=$CONF, state=$STATE"
     fi
   done
   ```

3. **Verify validated patterns decay at half rate:**
   ```bash
   # Validated patterns (project_count >= 3) decay slower
   # Half rate: use half_life_days * 2 for validated patterns
   ```

4. **Verify --decay flag required for deletion:**
   ```bash
   # Without --decay flag, expired patterns are not deleted
   # Only marked as Expired state

   # Check for deletion protection
   ls ~/.claude/gsd-knowledge/patterns/*/*.json | wc -l
   # Patterns should still exist even if expired
   ```

### Success Criteria

- ✅ Decay formula: 0.5^(days_since_last_seen/half_life_days)
- ✅ Lifecycle states: Active (>=0.5), Warning (0.3-0.5), Expired (<0.3)
- ✅ Validated patterns decay at half rate
- ✅ --decay flag required for pattern deletion
- ✅ Audit trail for deleted patterns (if implemented)

---

## Test Scenario 7: Failure Prediction

**Objective:** Verify predict-failures workflow produces accurate risk assessments

### Setup

Ensure knowledge base has failure patterns:

```bash
ls ~/.claude/gsd-knowledge/patterns/failure-pattern/*.json
# Expected: At least one failure pattern
```

Create test plan with various task types:

```bash
cat > TEST-PREDICT-PLAN.md << 'EOF'
---
phase: test-predict
plan: 01
type: execute
---

<tasks>

<task type="auto">
  <name>Task 1: Install dependencies</name>
  <files>package.json</files>
  <action>
    Run npm install to install all dependencies
  </action>
  <verify>node_modules exists</verify>
  <done>Dependencies installed</done>
</task>

<task type="auto">
  <name>Task 2: Run tests</name>
  <files>tests/</files>
  <action>
    Run npm test with jest
  </action>
  <verify>All tests pass</verify>
  <done>Tests passed</done>
</task>

<task type="auto">
  <name>Task 3: Deploy to production</name>
  <files>vercel.json</files>
  <action>
    Deploy using vercel --prod
  </action>
  <verify>Deployment successful</verify>
  <done>Deployed</done>
</task>

</tasks>
EOF
```

### Test Execution

```bash
# Run predict-failures workflow on the test plan
# (Should be triggered during plan-phase display)
```

### Verification Steps

1. **Verify risk assessment output:**
   ```bash
   # Look for "PREDICTIVE RISK ASSESSMENT" block
   # Example:
   # ================================================
   # PREDICTIVE RISK ASSESSMENT
   # ================================================
   # [!] **bash-command tasks:** HIGH risk (65% failure probability)
   ```

2. **Verify probability calculation:**
   ```bash
   # Formula: base * confidence * effectiveness_factor
   # base = min(frequency/20, 0.8)
   # Cap at 0.95

   # Check probabilities are reasonable (0-95%)
   ```

3. **Verify risk levels:**
   ```bash
   # LOW: < 0.3 (proceed normally)
   # MEDIUM: 0.3-0.6 (be prepared for retries)
   # HIGH: > 0.6 (consider alternatives)
   ```

4. **Verify evidence strength indicators:**
   ```bash
   # Output should include "(X occurrences, Y projects)"
   # Cross-project validated patterns should be marked
   ```

### Success Criteria

- ✅ Risk assessment block produced
- ✅ Per-task type risk levels (LOW/MEDIUM/HIGH)
- ✅ Probability percentages displayed
- ✅ Evidence strength shown (occurrences, projects)
- ✅ Cross-project validation noted
- ✅ Overall plan risk summarized

---

## Test Scenario 8: Proactive Suggestions Integration

**Objective:** Verify HIGH risk tasks get alternative suggestions during planning

### Setup

Ensure knowledge base has:
- Failure patterns with high frequency (for HIGH risk prediction)
- Strategy patterns with good success rates (for alternatives)

```bash
# Check for high-frequency failure patterns
grep -l '"frequency": [5-9]' ~/.claude/gsd-knowledge/patterns/failure-pattern/*.json

# Check for effective strategies
grep -l '"success_rate": 0.[7-9]' ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json
```

### Test Execution

```bash
# Run plan-phase workflow with prediction enabled
# (Should integrate predict-failures and suggest alternatives for HIGH risk)
```

### Verification Steps

1. **Verify HIGH risk tasks get alternatives:**
   ```bash
   # Look for "Suggested alternatives" section in output
   # Only appears for tasks with > 0.6 failure probability

   # Example:
   # [!] **bash-command tasks:** HIGH risk (65% failure probability)
   #    **Suggested alternatives (consider before execution):**
   #    1. clean-install-strategy - 85% success rate
   ```

2. **Verify strategy thresholds:**
   ```bash
   # Only strategies with:
   # - confidence >= 0.6
   # - success_rate >= 0.7
   # Should be suggested
   ```

3. **Verify contraindication filtering:**
   ```bash
   # Strategies with contraindications matching task context
   # should be filtered out

   # Example: If plan mentions "performance benchmarks",
   # strategies with contraindication "performance" should not appear
   ```

4. **Verify top 3 alternatives limit:**
   ```bash
   # Maximum 3 alternatives per task type
   # Sorted by effectiveness score
   ```

5. **Verify "view alternatives" option:**
   ```bash
   # User should be able to re-display alternatives
   # without re-running prediction
   ```

### Success Criteria

- ✅ Alternative suggestions only for HIGH risk (>0.6)
- ✅ Strategy threshold: confidence >= 0.6, success_rate >= 0.7
- ✅ Contraindication filtering works
- ✅ Top 3 alternatives per task type
- ✅ Sorted by effectiveness score
- ✅ View alternatives option available

---

## Test Scenario 9: End-to-End Pipeline

**Objective:** Verify complete learning → prediction pipeline works across executions

### Setup

Start with clean knowledge base:

```bash
# Backup and reset
mv ~/.claude/gsd-knowledge ~/.claude/gsd-knowledge-backup-e2e
```

### Test Execution (Multi-Step)

**Step 1: Execute plan with failures/retries**

```bash
# Create plan that will trigger retries
cat > TEST-E2E-PLAN-01.md << 'EOF'
---
phase: test-e2e
plan: 01
type: execute
---

<tasks>
<task type="auto">
  <name>Task 1: Operation that fails then succeeds</name>
  <action>Edit non-existent file (will fail, retry with write)</action>
  <verify>File exists</verify>
</task>
</tasks>
EOF

# Execute plan (triggers retry, populates EXECUTION_HISTORY.md)
```

**Step 2: Verify EXECUTION_HISTORY.md populated**

```bash
cat .planning/EXECUTION_HISTORY.md
# Expected: Strategy Effectiveness and Failure Patterns tables populated
```

**Step 3: Extract patterns**

```bash
# Run pattern extraction
# Verify patterns written to knowledge base
ls ~/.claude/gsd-knowledge/patterns/*/*.json
```

**Step 4: Analyze patterns**

```bash
# Run pattern analysis
# Verify effectiveness scores calculated
grep "effectiveness_score" ~/.claude/gsd-knowledge/patterns/*/*.json
```

**Step 5: Create new plan and verify prediction**

```bash
# Create similar plan
cat > TEST-E2E-PLAN-02.md << 'EOF'
---
phase: test-e2e
plan: 02
type: execute
---

<tasks>
<task type="auto">
  <name>Task 1: Similar file operation</name>
  <action>Edit another file</action>
  <verify>File updated</verify>
</task>
</tasks>
EOF

# Run plan-phase (should show predictions based on learned patterns)
# Expected: Risk assessment mentions file-edit failure pattern
```

### Verification Steps

1. **Verify learning improves predictions:**
   ```bash
   # First execution: No predictions (empty knowledge base)
   # After extraction: Predictions based on observed patterns
   ```

2. **Verify pattern confidence grows with use:**
   ```bash
   # Execute similar tasks multiple times
   # Extract patterns after each
   # Check confidence increases (more evidence)
   grep "confidence" ~/.claude/gsd-knowledge/patterns/*/*.json
   ```

### Success Criteria

- ✅ Execute plan → EXECUTION_HISTORY.md populated
- ✅ Extract patterns → Knowledge base populated
- ✅ Analyze patterns → Effectiveness scores calculated
- ✅ Predict on new plan → Uses learned patterns
- ✅ Confidence grows with more evidence
- ✅ Predictions improve accuracy over time

---

## Test Scenario 10: Backward Compatibility

**Objective:** Verify existing v2.0/v3.0 projects work without v4.0 features

### Test 10a: adaptive.enabled = false

```bash
# Create config without adaptive features
cat > .planning/config.json << 'EOF'
{
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "log_attempts": true
  },
  "verification": {
    "enabled": true
  },
  "adaptive": {
    "enabled": false
  }
}
EOF
```

### Expected Behavior

- No knowledge base initialization
- No pattern extraction on milestone completion
- No pattern hints during execute-phase
- No failure predictions during plan-phase
- Standard v3.0 behavior (static failure analysis, static path selection)

### Verification Steps

```bash
# Execute a plan
# Check for NO autonomy features:

# 1. No "PREDICTIVE HINTS" block
# 2. No "PREDICTIVE RISK ASSESSMENT" block
# 3. No knowledge base queries
# 4. No pattern extraction messages
```

### Test 10b: adaptive.enabled = true but no knowledge base

```bash
# Enable adaptive but ensure knowledge base doesn't exist
rm -rf ~/.claude/gsd-knowledge

cat > .planning/config.json << 'EOF'
{
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "show_pattern_hints": true
  }
}
EOF
```

### Expected Behavior

- Knowledge base auto-initialized
- No patterns to import (empty knowledge base)
- No predictions (no failure patterns)
- System proceeds normally without blocking

### Verification Steps

```bash
# Execute a plan
# 1. No errors about missing knowledge base
# 2. Knowledge base directory created (graceful init)
# 3. No pattern hints (none available)
# 4. No predictions (no data)
# 5. Plan executes normally
```

### Test 10c: prediction_enabled = false

```bash
cat > .planning/config.json << 'EOF'
{
  "adaptive": {
    "enabled": true,
    "prediction_enabled": false,
    "show_pattern_hints": true
  }
}
EOF
```

### Expected Behavior

- Pattern hints shown during execute-phase (show_pattern_hints = true)
- No risk predictions during plan-phase (prediction_enabled = false)
- Hybrid mode: hints but no prediction

### Verification Steps

```bash
# During execute-phase: Check for hints
# During plan-phase: No PREDICTIVE RISK ASSESSMENT block
```

### Success Criteria

- ✅ adaptive.enabled=false: No autonomy features, v3.0 behavior
- ✅ No knowledge base: Graceful skip, no errors
- ✅ prediction_enabled=false: Hints work, no prediction
- ✅ No warnings or errors in any backward-compat scenario
- ✅ Primary workflow never blocked by autonomy features

---

## Test Results Checklist

After running all scenarios, verify:

- [ ] **Scenario 1:** Knowledge base initializes correctly and is idempotent
- [ ] **Scenario 2:** Pattern extraction works with deduplication
- [ ] **Scenario 3:** Milestone completion triggers pattern export
- [ ] **Scenario 4:** Pattern import shows hints during execution
- [ ] **Scenario 5:** Pattern analysis calculates effectiveness scores
- [ ] **Scenario 6:** Decay system manages pattern lifecycle
- [ ] **Scenario 7:** Failure prediction produces risk assessments
- [ ] **Scenario 8:** HIGH risk tasks get alternative suggestions
- [ ] **Scenario 9:** End-to-end pipeline improves predictions over time
- [ ] **Scenario 10:** Backward compatibility with v2.0/v3.0

**Overall Autonomy System Health:**
- [ ] Knowledge base structure valid at `~/.claude/gsd-knowledge/`
- [ ] All pattern types extracted correctly (strategy, failure, insight)
- [ ] Pattern JSON follows schema (id, type, confidence, evidence, content)
- [ ] Effectiveness scores in valid range (0-1)
- [ ] Predictions match known patterns
- [ ] Alternatives only suggested for HIGH risk
- [ ] Non-blocking guarantee honored (errors never block workflow)
- [ ] Configuration options all functional
- [ ] Backward compatibility 100% maintained

---

## Cleanup After Tests

```bash
# Remove test files
rm -f .planning/test-workspace-autonomy/*.md
rm -f .planning/EXECUTION_HISTORY.md
rm -f TEST-*.md

# Reset config
git checkout .planning/config.json

# Optionally restore knowledge base backup
# mv ~/.claude/gsd-knowledge-backup ~/.claude/gsd-knowledge

# Remove test workspace
rmdir .planning/test-workspace-autonomy
```

---

## Troubleshooting

### Knowledge Base Not Initializing

**Check:**
```bash
# Verify ~/.claude directory exists and is writable
ls -la ~/.claude/
touch ~/.claude/test-write && rm ~/.claude/test-write

# Check for disk space
df -h ~/.claude/
```

### Patterns Not Extracting

**Check:**
```bash
# Verify EXECUTION_HISTORY.md exists and has data
ls -la .planning/EXECUTION_HISTORY.md
head -50 .planning/EXECUTION_HISTORY.md

# Verify adaptive.enabled = true
grep -A 5 '"adaptive"' .planning/config.json

# Check extraction thresholds (min 3 occurrences)
grep "Times Used\|Frequency" .planning/EXECUTION_HISTORY.md
```

### Predictions Not Showing

**Check:**
```bash
# Verify prediction_enabled = true (or not explicitly false)
grep "prediction_enabled" .planning/config.json

# Verify knowledge base has failure patterns
ls ~/.claude/gsd-knowledge/patterns/failure-pattern/

# Check pattern confidence >= 0.5
grep "confidence" ~/.claude/gsd-knowledge/patterns/failure-pattern/*.json

# Check task types match pattern task_types
grep "task_types" ~/.claude/gsd-knowledge/patterns/failure-pattern/*.json
```

### Alternatives Not Suggested

**Check:**
```bash
# Verify risk is HIGH (> 0.6)
# MEDIUM risk doesn't get alternatives

# Verify strategy patterns exist with success_rate >= 0.7
grep "success_rate" ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json

# Check contraindications aren't filtering all strategies
grep "contraindications" ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json
```

---

## Summary

These tests validate that v4.0 Autonomy:

1. **Learns from execution** - Extracts patterns from EXECUTION_HISTORY.md
2. **Shares across projects** - Global knowledge base at ~/.claude/gsd-knowledge/
3. **Predicts failures** - Risk assessment before plan execution
4. **Suggests alternatives** - Proactive strategies for HIGH risk tasks
5. **Improves over time** - Confidence grows with evidence
6. **Decays gracefully** - Outdated patterns lose confidence
7. **Never blocks** - Best-effort, non-blocking guarantee
8. **Stays compatible** - v2.0/v3.0 projects unaffected when disabled

**Test coverage:** 10 scenarios covering all v4.0 autonomy features
**Execution:** Manual testing with bash simulation
**Outcome:** Complete validation of v4.0 Autonomy milestone
