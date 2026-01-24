# Execution History

---
created: [YYYY-MM-DD HH:MM:SS]
last-updated: [YYYY-MM-DD HH:MM:SS]
total-executions: 0
total-successes: 0
total-failures: 0
total-recoveries: 0
---

## Strategy Effectiveness

Tracks which strategies work best for which failure categories across all task executions.

| Strategy Name | Times Used | Success Rate | Avg Attempt # | Task Types | Notes |
|---------------|------------|--------------|---------------|------------|-------|
| <!-- Example: retry-with-reruns | 5 | 80% (4/5) | 2.0 | test-run | Works well for flaky tests --> |

**Usage:** When a strategy succeeds or fails, update its row. If strategy doesn't exist, add new row.

**Success Rate Calculation:** (successful uses) / (total uses)

**Avg Attempt #:** Track which attempt number this strategy typically succeeds on (1 = first retry, 2 = second retry, etc.)

---

## Common Failure Patterns

Recurring failures observed across task executions to identify systemic issues.

| Failure Category | Task Type | Frequency | Typical Recovery | Last Seen | Notes |
|------------------|-----------|-----------|------------------|-----------|-------|
| <!-- Example: Dependency | npm-install | 12 | reinstall-dependencies | 2026-01-06 | Often cache-related --> |

**Usage:** When a task fails, check if this is a recurring pattern. If yes, increment frequency. If new pattern, add row.

**Typical Recovery:** What strategy or approach usually works for this pattern?

**Pattern Detection:** If same failure-category + task-type combination occurs 3+ times, it's a pattern worth noting.

---

## Task Type Analysis

Performance and patterns grouped by task type to guide strategy selection.

### bash-command

**Success Rate:** [X%] ([successes]/[total])

**Common Failures:**
- [failure-category]: [description] ([frequency])

**Effective Strategies:**
- [strategy-name]: [success-rate] ([description of when it works])

**Insights:**
- [Lessons learned from bash-command executions]

---

### file-edit

**Success Rate:** [X%] ([successes]/[total])

**Common Failures:**
- [failure-category]: [description] ([frequency])

**Effective Strategies:**
- [strategy-name]: [success-rate] ([description of when it works])

**Insights:**
- [Lessons learned from file-edit executions]

---

### test-run

**Success Rate:** [X%] ([successes]/[total])

**Common Failures:**
- [failure-category]: [description] ([frequency])

**Effective Strategies:**
- [strategy-name]: [success-rate] ([description of when it works])

**Insights:**
- [Lessons learned from test-run executions]

---

### other

**Success Rate:** [X%] ([successes]/[total])

**Common Failures:**
- [failure-category]: [description] ([frequency])

**Effective Strategies:**
- [strategy-name]: [success-rate] ([description of when it works])

**Insights:**
- [Lessons learned from other task type executions]

---

## Learning Signals

**Reserved for Stage 3 learning capabilities**

This section will be populated in future versions with AI-powered pattern detection and predictive failure analysis. Currently, learning signals are captured at the attempt level in ATTEMPTS_LOG.md.

**Planned capabilities:**
- Automatic pattern detection from execution history
- Predictive failure analysis (anticipate likely failures before execution)
- Strategy recommendation engine (suggest best strategy based on historical effectiveness)
- Confidence scoring (how confident are we in classifications and recommendations)

**Current status:** Manual pattern tracking in sections above. Stage 3 will automate this.

---

## Examples

### Example 1: After executing 3 tasks with retries

---
created: 2026-01-06 10:00:00
last-updated: 2026-01-06 11:30:00
total-executions: 3
total-successes: 2
total-failures: 1
total-recoveries: 2
---

#### Strategy Effectiveness

| Strategy Name | Times Used | Success Rate | Avg Attempt # | Task Types | Notes |
|---------------|------------|--------------|---------------|------------|-------|
| retry-with-reruns | 2 | 100% (2/2) | 2.0 | test-run | Isolated test execution prevents flakiness |
| reinstall-dependencies | 1 | 0% (0/1) | - | bash-command | Did not resolve missing module issue |
| explicit-install-missing | 1 | 100% (1/1) | 3.0 | bash-command | Works when specific package known |

#### Common Failure Patterns

| Failure Category | Task Type | Frequency | Typical Recovery | Last Seen | Notes |
|------------------|-----------|-----------|------------------|-----------|-------|
| Validation | test-run | 2 | retry-with-reruns | 2026-01-06 11:15 | Flaky tests - isolation helps |
| Dependency | bash-command | 1 | explicit-install-missing | 2026-01-06 11:30 | Missing lodash package |

#### Task Type Analysis

##### test-run

**Success Rate:** 100% (2/2)

**Common Failures:**
- Validation: Flaky test timeouts (2 occurrences)

**Effective Strategies:**
- retry-with-reruns: 100% success rate (isolation prevents race conditions)

**Insights:**
- Flaky tests benefit from isolated execution (--maxWorkers=1 --runInBand)
- First retry usually sufficient if strategy correct

##### bash-command

**Success Rate:** 50% (1/2)

**Common Failures:**
- Dependency: Missing npm packages (1 occurrence)

**Effective Strategies:**
- explicit-install-missing: 100% success rate when package name known

**Insights:**
- Reinstall-dependencies alone insufficient for genuinely missing packages
- Need to identify specific missing package for targeted install
