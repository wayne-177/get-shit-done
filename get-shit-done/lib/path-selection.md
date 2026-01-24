# Path Selection Logic

This document defines how to select alternative strategies from ALTERNATIVE_PATHS.md when tasks fail. Works in conjunction with failure-analysis.md to automatically choose backup approaches based on failure context.

## Purpose

When a task fails and failure-analysis.md determines RETRY is appropriate, this logic:
1. Filters available alternative strategies based on failure type
2. Excludes previously attempted approaches
3. Selects the most appropriate strategy for current attempt
4. Determines when to escalate vs continue retrying

## Input Specification

<selection_input>
  <required>
    <field name="failure_category">Category from failure-analysis.md: Tool | Validation | Dependency | Logic | Resource</field>
    <field name="task_type">Type of task that failed: bash-command | file-edit | test-run | build-task | etc.</field>
    <field name="attempt_number">Current attempt count (1, 2, or 3)</field>
  </required>

  <optional>
    <field name="prior_attempts">List of strategy names already tried (from previous retries)</field>
    <field name="failure_details">Specific error context from failure-analysis.md output</field>
    <field name="task_context">What the task was trying to accomplish</field>
  </optional>
</selection_input>

## Selection Algorithm

<algorithm>
  <step number="1">
    <name>Load Alternative Paths</name>
    <action>Read ALTERNATIVE_PATHS.md and parse all &lt;alternative-path&gt; entries</action>
  </step>

  <step number="2">
    <name>Filter by Task Type</name>
    <action>
      Keep only strategies where:
      - &lt;task-type&gt; matches input task_type, OR
      - &lt;task-type&gt; is "any"
    </action>
  </step>

  <step number="3">
    <name>Filter by Failure Category</name>
    <action>
      Keep only strategies where:
      - &lt;failure-category&gt; matches input failure_category, OR
      - &lt;failure-category&gt; is "any"
    </action>
  </step>

  <step number="4">
    <name>Exclude Already Tried</name>
    <action>
      Remove strategies where &lt;strategy-name&gt; appears in prior_attempts list
    </action>
  </step>

  <step number="5">
    <name>Sort by Priority</name>
    <action>
      Sort remaining strategies by:
      1. Difficulty (simple → moderate → complex)
      2. Risk (safe → moderate → risky)
      3. Order in file (earlier entries preferred)
    </action>
  </step>

  <step number="6">
    <name>Filter by Attempt Number</name>
    <action>
      Apply attempt-specific filtering:

      Attempt 1 (first retry):
      - Prefer difficulty=simple AND risk=safe
      - Allow difficulty=moderate if no simple options
      - Avoid risk=risky

      Attempt 2 (second retry):
      - Allow any difficulty
      - Prefer risk=safe or risk=moderate
      - Allow risk=risky if no other options

      Attempt 3 (final retry):
      - Allow any difficulty and risk
      - Prefer unexplored approaches over refinements
    </action>
  </step>

  <step number="7">
    <name>Select Top Strategy</name>
    <action>
      Choose first strategy from sorted, filtered list.
      If no strategies remain, proceed to escalation (Step 8).
    </action>
  </step>

  <step number="8">
    <name>Check Escalation Criteria</name>
    <action>
      Escalate if ANY of:
      - No strategies remain after filtering
      - Attempt number = 3 AND selected strategy is high-risk
      - Same failure occurred 3+ times with different strategies
      - Prerequisites for all remaining strategies cannot be met
    </action>
  </step>
</algorithm>

## Selection Decision Tree

<decision_tree>
  <question id="S1">
    <text>Are there strategies matching task-type AND failure-category?</text>
    <yes>Go to S2</yes>
    <no>
      Try fallback: match failure-category with task-type="any"
      If still none: ESCALATE (no applicable strategies)
    </no>
  </question>

  <question id="S2">
    <text>After removing already-tried strategies, any remaining?</text>
    <yes>Go to S3</yes>
    <no>ESCALATE (all strategies exhausted for this failure type)</no>
  </question>

  <question id="S3">
    <text>What is the attempt number?</text>
    <attempt_1>
      Filter for simple + safe strategies. If none, allow moderate difficulty.
      Select top strategy. Go to S4.
    </attempt_1>
    <attempt_2>
      Filter out only risky + complex combinations. Allow most strategies.
      Select top strategy. Go to S4.
    </attempt_2>
    <attempt_3>
      Allow all remaining strategies (final attempt).
      Select top strategy. Go to S4.
    </attempt_3>
  </question>

  <question id="S4">
    <text>Can prerequisites for selected strategy be met?</text>
    <yes>RETURN selected strategy</yes>
    <no>
      Mark strategy as ineligible, remove from list.
      Go back to S2 with updated list.
    </no>
  </question>
</decision_tree>

## Output Specification

<selection_output>
  <field name="selected_strategy">
    The chosen &lt;strategy-name&gt; from ALTERNATIVE_PATHS.md
  </field>
  <field name="approach">
    Full &lt;approach&gt; description from the selected strategy
  </field>
  <field name="when_to_use">
    The &lt;when-to-use&gt; guidance from selected strategy
  </field>
  <field name="success_indicators">
    Expected &lt;success-indicators&gt; for this strategy
  </field>
  <field name="risk_level">
    The &lt;risk&gt; level: safe | moderate | risky
  </field>
  <field name="prerequisites">
    Any &lt;prerequisites&gt; that must be verified before execution
  </field>
  <field name="decision">
    SELECT_STRATEGY | ESCALATE
  </field>
  <field name="escalation_reason">
    If ESCALATE: specific reason (no strategies left, prerequisites unmet, etc.)
  </field>
</selection_output>

## Escalation Criteria (Detailed)

<escalation_rules>
  <rule id="E1">
    <condition>No strategies exist for this task-type + failure-category combination</condition>
    <reason>Alternative paths library doesn't cover this scenario yet</reason>
    <action>ESCALATE - Recommend user intervention and suggest adding this scenario to library</action>
  </rule>

  <rule id="E2">
    <condition>All applicable strategies have been tried</condition>
    <reason>Exhausted all known alternatives for this failure type</reason>
    <action>ESCALATE - Likely requires user decision or novel approach</action>
  </rule>

  <rule id="E3">
    <condition>Attempt number = 3 (max retries reached)</condition>
    <reason>Hit maximum retry limit</reason>
    <action>ESCALATE - Three attempts without success suggests deeper issue</action>
  </rule>

  <rule id="E4">
    <condition>Same root cause failure occurred 3+ times with different strategies</condition>
    <reason>Strategies aren't addressing the actual problem</reason>
    <action>ESCALATE - Problem may be with requirements understanding or environment</action>
  </rule>

  <rule id="E5">
    <condition>Only high-risk strategies remain AND critical task (e.g., deployment, data migration)</condition>
    <reason>Risk of making problem worse exceeds benefit of auto-retry</reason>
    <action>ESCALATE - User should decide whether to proceed with risky approach</action>
  </rule>

  <rule id="E6">
    <condition>Prerequisites for all remaining strategies cannot be met</condition>
    <reason>Environmental constraints prevent any viable alternative</reason>
    <action>ESCALATE - Environment setup or user configuration needed</action>
  </rule>
</escalation_rules>

## Worked Examples

### Example 1: First Retry - File Not Found

<example id="1">
  <input>
    <failure_category>Tool</failure_category>
    <task_type>file-edit</task_type>
    <attempt_number>1</attempt_number>
    <prior_attempts>[]</prior_attempts>
    <failure_details>File /app/src/components/Button.tsx not found</failure_details>
  </input>

  <process>
    <step1>Load ALTERNATIVE_PATHS.md</step1>
    <step2>Filter task-type="file-edit" OR "any" → 8 strategies remain</step2>
    <step3>Filter failure-category="Tool" OR "any" → 5 strategies remain</step3>
    <step4>Exclude prior_attempts (empty) → 5 strategies remain</step4>
    <step5>Sort by difficulty+risk:
      1. "Use glob to find file" (simple, safe)
      2. "Search with grep for pattern" (simple, safe)
      3. "Check common naming variants" (simple, safe)
      4. "Verify parent directory exists" (simple, safe)
      5. "Try absolute vs relative paths" (moderate, safe)
    </step5>
    <step6>Attempt 1: prefer simple+safe → keep all 5</step6>
    <step7>Select top: "Use glob to find file"</step7>
    <step8>Prerequisites: glob tool available (yes) → No escalation needed</step8>
  </process>

  <output>
    <selected_strategy>Use glob to find file</selected_strategy>
    <approach>Use Glob tool to search for **/*Button*.tsx pattern to locate the actual file path. Start from project root or src/ directory. Check for common variations like button.tsx, ButtonComponent.tsx.</approach>
    <when_to_use>First retry when file path is wrong. File likely exists but at different location or with different name.</when_to_use>
    <success_indicators>Glob returns one or more matching file paths. Can then Read the correct file.</success_indicators>
    <risk_level>safe</risk_level>
    <prerequisites>None</prerequisites>
    <decision>SELECT_STRATEGY</decision>
  </output>
</example>

### Example 2: Second Retry - Test Failures

<example id="2">
  <input>
    <failure_category>Validation</failure_category>
    <task_type>test-run</task_type>
    <attempt_number>2</attempt_number>
    <prior_attempts>["Fix obvious syntax errors"]</prior_attempts>
    <failure_details>3 tests still failing after fixing import statement</failure_details>
  </input>

  <process>
    <step1>Load ALTERNATIVE_PATHS.md</step1>
    <step2>Filter task-type="test-run" OR "any" → 6 strategies remain</step2>
    <step3>Filter failure-category="Validation" OR "any" → 4 strategies remain</step3>
    <step4>Exclude "Fix obvious syntax errors" → 3 strategies remain</step4>
    <step5>Sort by difficulty+risk:
      1. "Run tests with verbose output" (simple, safe)
      2. "Run single failing test in isolation" (simple, safe)
      3. "Try alternative implementation approach" (moderate, moderate)
    </step5>
    <step6>Attempt 2: allow any difficulty, prefer safe/moderate → keep all 3</step6>
    <step7>Select top: "Run tests with verbose output"</step7>
    <step8>Prerequisites: test framework supports verbose flags (yes) → No escalation</step8>
  </process>

  <output>
    <selected_strategy>Run tests with verbose output</selected_strategy>
    <approach>Re-run test suite with maximum verbosity: npm test -- --verbose or jest --verbose. Examine detailed output to understand why tests are failing. Look for async issues, mock problems, or assertion details.</approach>
    <when_to_use>Second attempt when initial fix didn't resolve all failures. Need more information about what's actually going wrong.</when_to_use>
    <success_indicators>Verbose output reveals specific assertion failures, timing issues, or dependency problems that weren't visible before.</success_indicators>
    <risk_level>safe</risk_level>
    <prerequisites>Test framework must support verbose/debug mode</prerequisites>
    <decision>SELECT_STRATEGY</decision>
  </output>
</example>

### Example 3: Third Retry - Escalation Triggered

<example id="3">
  <input>
    <failure_category>Dependency</failure_category>
    <task_type>build-task</task_type>
    <attempt_number>3</attempt_number>
    <prior_attempts>["Install missing dependency", "Clear cache and reinstall"]</prior_attempts>
    <failure_details>Module @custom/internal-lib still not found after npm install</failure_details>
  </input>

  <process>
    <step1>Load ALTERNATIVE_PATHS.md</step1>
    <step2>Filter task-type="build-task" OR "any" → 7 strategies remain</step2>
    <step3>Filter failure-category="Dependency" OR "any" → 4 strategies remain</step3>
    <step4>Exclude prior_attempts → 2 strategies remain:
      - "Check package.json for correct version"
      - "Verify private registry authentication"
    </step4>
    <step5>Sort: both moderate difficulty, safe risk</step5>
    <step6>Attempt 3: allow all → 2 options available</step6>
    <step7>Select top: "Check package.json for correct version"</step7>
    <step8>Check escalation: attempt_number = 3 (max retries) → TRIGGER ESCALATION</step8>
  </process>

  <output>
    <selected_strategy>N/A</selected_strategy>
    <decision>ESCALATE</decision>
    <escalation_reason>
      Reached maximum retry attempts (3). Module @custom/internal-lib may require:
      (1) Private npm registry authentication
      (2) Manual installation from different source
      (3) Configuration of package.json to use correct registry
      User intervention needed to resolve authentication or registry configuration.
    </escalation_reason>
  </output>
</example>

### Example 4: No Applicable Strategies

<example id="4">
  <input>
    <failure_category>Resource</failure_category>
    <task_type>deployment</task_type>
    <attempt_number>1</attempt_number>
    <prior_attempts>[]</prior_attempts>
    <failure_details>SSH connection to production server timed out</failure_details>
  </input>

  <process>
    <step1>Load ALTERNATIVE_PATHS.md</step1>
    <step2>Filter task-type="deployment" → 2 strategies found</step2>
    <step3>Filter failure-category="Resource" → 0 strategies (deployment strategies cover validation/dependency, not resource failures)</step3>
    <step4>Try fallback: task-type="any" + failure-category="Resource" → 3 strategies found:
      - "Wait and retry with backoff" (network timeouts)
      - "Check resource availability" (disk, memory)
      - "Verify permissions" (file access)
    </step4>
    <step5>Evaluate: "Wait and retry with backoff" applies to SSH timeout</step5>
    <step6>Select this strategy</step6>
    <step7>BUT: Check context - deployment to production is high-stakes</step7>
    <step8>ESCALATE: Production deployment failures should not auto-retry without user awareness</step8>
  </process>

  <output>
    <selected_strategy>N/A</selected_strategy>
    <decision>ESCALATE</decision>
    <escalation_reason>
      Production deployment task with SSH timeout. While retry strategies exist, deployment failures should be escalated to user for awareness and approval before retrying. Network issues could indicate broader infrastructure problems.
    </escalation_reason>
  </output>
</example>

## Special Cases

### Case 1: Multiple Matching Strategies

When multiple strategies have same difficulty+risk, selection order:
1. Strategies mentioned in failure-analysis.md recommended_strategy first
2. Strategies that appeared earlier in ALTERNATIVE_PATHS.md file
3. Strategies with fewer prerequisites

### Case 2: Partial Matches

If no exact task-type + failure-category match:
1. Try task-type="any" + specific failure-category
2. Try specific task-type + failure-category="any"
3. Try task-type="any" + failure-category="any" (universal fallbacks)
4. If still none: ESCALATE

### Case 3: High-Stakes Tasks

For certain task types, escalate earlier than normal:
- `deployment` tasks: escalate after 1 attempt
- `data-migration` tasks: escalate after 1 attempt
- `git-operation` involving force push: escalate immediately
- Tasks with `checkpoint:human-verify` originally: escalate after 2 attempts

### Case 4: Repeated Identical Failures

If failure-details are identical across 2+ attempts despite using different strategies:
- Strategies aren't addressing root cause
- ESCALATE with note: "Different strategies produce identical failure - may need different approach entirely"

## Integration Points

This document integrates with:
- **failure-analysis.md**: Provides input (failure_category, decision to RETRY)
- **ALTERNATIVE_PATHS.md**: Source of strategies to select from
- **retry-execution engine** (Plan 01-03): Consumes selected strategy and executes approach
- **ATTEMPTS_LOG.md** (future): Logs which strategies were selected and outcomes

## Usage in Retry Loop

```
1. Task fails
2. Run failure-analysis.md → get failure_category, decision (RETRY/ESCALATE)
3. If decision = RETRY:
   a. Run path-selection.md with failure_category, task_type, attempt_number, prior_attempts
   b. If decision = SELECT_STRATEGY: execute approach from selected strategy
   c. If decision = ESCALATE: stop retry, convert to checkpoint or report to user
4. Log attempt and outcome
5. If still failing, increment attempt_number and loop to step 2 (max 3 attempts)
```

---

*Document-driven strategy selection for GSD automatic retry system*
