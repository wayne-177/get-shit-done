# Failure Analysis Decision Tree

This document provides a diagnostic process for analyzing task failures and selecting appropriate recovery strategies. Integrates failure-detection.md patterns with failure-taxonomy.md categories to guide automatic retry logic.

## Purpose

Guide Claude during plan execution to:
1. Diagnose what went wrong when a task fails
2. Classify the failure into appropriate category
3. Extract actionable root cause information
4. Select best recovery strategy from available options
5. Decide whether to retry or escalate to user

## Input Specification

<analysis_input>
  <required>
    <field name="tool">Which tool was used (Bash, Edit, Write, Read, etc.)</field>
    <field name="command">Exact command or operation attempted</field>
    <field name="exit_code">Exit code (for Bash) or error indicator</field>
    <field name="stdout">Standard output from tool</field>
    <field name="stderr">Standard error from tool (if applicable)</field>
  </required>

  <optional>
    <field name="task_context">What was the task trying to accomplish?</field>
    <field name="task_type">auto, checkpoint, or checkpoint:decision</field>
    <field name="previous_attempts">How many times has this task been attempted?</field>
    <field name="previous_strategies">What recovery strategies were already tried?</field>
  </optional>
</analysis_input>

## Analysis Process

<diagnostic_workflow>
  <phase name="1_detect">
    <title>Detect Failure</title>
    <steps>
      <step>Check exit code: non-zero = failure (for Bash tool)</step>
      <step>Scan output for error keywords: Error, ERROR, FAIL, failed, exception</step>
      <step>Match against failure-detection.md patterns</step>
      <step>Confirm this is actual failure vs warning or partial success</step>
    </steps>
    <output>Failure confirmed: yes/no</output>
  </phase>

  <phase name="2_classify">
    <title>Classify Failure Type</title>
    <steps>
      <step>Match detected patterns to failure-taxonomy.md categories</step>
      <step>Identify primary category: Tool, Validation, Dependency, Logic, or Resource</step>
      <step>Note any secondary categories (failures can have multiple causes)</step>
    </steps>
    <output>Failure category with confidence level</output>
  </phase>

  <phase name="3_diagnose">
    <title>Extract Root Cause</title>
    <steps>
      <step>Which specific file/command/dependency caused failure?</step>
      <step>What is the exact error message?</step>
      <step>Is this a transient issue (network blip) or persistent?</step>
      <step>Is this a missing resource or incorrect usage?</step>
      <step>What context clues exist in the error output?</step>
    </steps>
    <output>Root cause statement with specifics</output>
  </phase>

  <phase name="4_strategize">
    <title>Select Recovery Strategy</title>
    <steps>
      <step>Review failure-taxonomy.md strategies for this category</step>
      <step>Filter out already-attempted strategies</step>
      <step>Select highest-priority untried strategy</step>
      <step>Verify strategy is applicable to this specific failure</step>
    </steps>
    <output>Recommended recovery strategy</output>
  </phase>

  <phase name="5_decide">
    <title>Retry or Escalate</title>
    <steps>
      <step>Check attempt count: have we hit max retries (3)?</step>
      <step>Check escalation criteria from failure-taxonomy.md</step>
      <step>Assess likelihood of success for proposed strategy</step>
      <step>Decide: retry with strategy, or escalate to user</step>
    </steps>
    <output>Decision: RETRY or ESCALATE</output>
  </phase>
</diagnostic_workflow>

## Decision Tree Questions

<decision_tree>
  <question id="Q1">
    <text>Is the exit code non-zero (for Bash) or error indicator present?</text>
    <yes>Go to Q2</yes>
    <no>Not a failure - check for warnings but continue</no>
  </question>

  <question id="Q2">
    <text>Does output contain "ENOENT", "no such file", or "command not found"?</text>
    <yes>Category: Tool or Dependency failure. Go to Q3</yes>
    <no>Go to Q4</no>
  </question>

  <question id="Q3">
    <text>Is it a file path (ENOENT for .js, .md, etc.) or command (command not found)?</text>
    <file>Tool failure - file doesn't exist. Strategy: verify path, use glob to find correct location</file>
    <command>Dependency failure - missing package/tool. Strategy: install dependency or use npx</command>
  </question>

  <question id="Q4">
    <text>Does output contain "EACCES", "EPERM", or "permission denied"?</text>
    <yes>Category: Resource failure (permissions). Go to Q5</yes>
    <no>Go to Q6</no>
  </question>

  <question id="Q5">
    <text>Can we work around by using different directory with write access?</text>
    <yes>Strategy: adjust path to user-writable location</yes>
    <no>Escalate: user needs to grant permissions or adjust system config</no>
  </question>

  <question id="Q6">
    <text>Does output contain test/build failure indicators (e.g., "3 failing", "build failed")?</text>
    <yes>Category: Validation failure. Go to Q7</yes>
    <no>Go to Q9</no>
  </question>

  <question id="Q7">
    <text>Are errors simple (missing import, typo) or complex (logic/algorithm wrong)?</text>
    <simple>Strategy: fix syntax/import error and retry</simple>
    <complex>Go to Q8</complex>
  </question>

  <question id="Q8">
    <text>Have we tried 2+ different implementation approaches?</text>
    <yes>Escalate: likely requirements misunderstanding or architectural issue</yes>
    <no>Strategy: try alternative implementation approach</no>
  </question>

  <question id="Q9">
    <text>Does output contain "MODULE_NOT_FOUND", "Cannot find module"?</text>
    <yes>Category: Dependency failure. Strategy: npm install missing package</yes>
    <no>Go to Q10</no>
  </question>

  <question id="Q10">
    <text>Does output contain timeout or network errors (ETIMEDOUT, ECONNREFUSED, ENOTFOUND)?</text>
    <yes>Category: Resource failure (network/timeout). Go to Q11</yes>
    <no>Go to Q12</no>
  </question>

  <question id="Q11">
    <text>Is this first or second attempt?</text>
    <first_or_second>Strategy: wait and retry (transient network issue), use exponential backoff</first_or_second>
    <third>Escalate: persistent network/connectivity issue needs user intervention</third>
  </question>

  <question id="Q12">
    <text>Does output contain syntax/runtime errors (SyntaxError, ReferenceError, TypeError)?</text>
    <yes>Category: Logic failure. Go to Q13</yes>
    <no>Go to Q14</no>
  </question>

  <question id="Q13">
    <text>Is error simple syntax fix (missing bracket, typo) or deeper logic issue?</text>
    <simple>Strategy: fix syntax error and retry</simple>
    <complex>Strategy: review implementation against requirements, try different approach. If fails again, escalate.</complex>
  </question>

  <question id="Q14">
    <text>Is this an Edit tool failure where old_string wasn't found?</text>
    <yes>Category: Tool failure. Strategy: expand old_string with more context, verify with Read first</yes>
    <no>Go to Q15</no>
  </question>

  <question id="Q15">
    <text>General error - does error message suggest specific remedy?</text>
    <yes>Strategy: follow error message guidance</yes>
    <no>Category: unclear. Log details, attempt general retry, or escalate if attempt count high</no>
  </question>
</decision_tree>

## Output Specification

<analysis_output>
  <field name="failure_detected">Boolean: true if genuine failure</field>
  <field name="category">Tool | Validation | Dependency | Logic | Resource</field>
  <field name="confidence">high | medium | low</field>
  <field name="root_cause">Specific description: "File X not found", "Module Y missing", "Test Z failed due to..."</field>
  <field name="error_context">
    Which file? Which command? Which dependency? Which test?
  </field>
  <field name="recommended_strategy">
    Specific action to take from failure-taxonomy.md recovery strategies
  </field>
  <field name="alternative_strategies">
    List of backup strategies if recommended one fails
  </field>
  <field name="decision">RETRY | ESCALATE</field>
  <field name="escalation_reason">
    If ESCALATE: why? (max attempts, user input needed, architectural decision, etc.)
  </field>
</analysis_output>

## Worked Examples

### Example 1: File Not Found

<example id="1">
  <input>
    <tool>Read</tool>
    <command>Read file_path=/app/src/components/Button.tsx</command>
    <exit_code>N/A (tool error)</exit_code>
    <stderr>Error: ENOENT: no such file or directory, open '/app/src/components/Button.tsx'</stderr>
    <task_context>Reading component file to add new prop</task_context>
    <previous_attempts>0</previous_attempts>
  </input>

  <analysis>
    <phase1_detect>
      - Exit code: N/A but tool returned error
      - Error keyword: "ENOENT", "no such file or directory"
      - Pattern match: File Access Failure (from failure-detection.md)
      - Conclusion: Genuine failure
    </phase1_detect>

    <phase2_classify>
      - Primary category: Tool failure
      - Pattern: ENOENT for specific file path
      - Confidence: High
    </phase2_classify>

    <phase3_diagnose>
      - Specific file: /app/src/components/Button.tsx
      - Root cause: File doesn't exist at expected path
      - Context: May have been moved, renamed, or never created
      - Transient: No (file won't magically appear)
    </phase3_diagnose>

    <phase4_strategize>
      - Strategy 1 (priority 1): Use glob to find Button.tsx in src/
      - Strategy 2 (priority 2): Use grep to search for "Button" component
      - Strategy 3 (priority 3): Check if file is named differently (button.tsx, ButtonComponent.tsx)
    </phase4_strategize>

    <phase5_decide>
      - Attempt count: 0 (first try)
      - Max retries: 3
      - Escalation criteria: Not met (can search for file)
      - Decision: RETRY with Strategy 1
    </phase5_decide>
  </analysis>

  <output>
    <failure_detected>true</failure_detected>
    <category>Tool</category>
    <confidence>high</confidence>
    <root_cause>File /app/src/components/Button.tsx does not exist</root_cause>
    <error_context>
      File: Button.tsx
      Expected location: /app/src/components/
      Operation: Read for editing
    </error_context>
    <recommended_strategy>Use Glob to search for **/*Button*.tsx to locate actual file path</recommended_strategy>
    <alternative_strategies>
      - Use Grep to search for "export.*Button" to find component definition
      - Check common naming variants (button.tsx, ButtonComponent.tsx)
    </alternative_strategies>
    <decision>RETRY</decision>
  </output>
</example>

### Example 2: Test Failures

<example id="2">
  <input>
    <tool>Bash</tool>
    <command>npm test</command>
    <exit_code>1</exit_code>
    <stdout>
      Test Suites: 2 passed, 1 failed, 3 total
      Tests:       16 passed, 3 failed, 19 total

      FAIL src/utils/calculator.test.js
        ● Calculator › should handle division by zero
          expect(received).toBe(expected)
          Expected: Infinity
          Received: undefined
    </stdout>
    <task_context>Implemented division function in calculator.js</task_context>
    <previous_attempts>0</previous_attempts>
  </input>

  <analysis>
    <phase1_detect>
      - Exit code: 1 (failure)
      - Error keyword: "failed", "FAIL"
      - Pattern match: Validation Failure (from failure-detection.md)
      - Conclusion: Genuine failure (tests failing)
    </phase1_detect>

    <phase2_classify>
      - Primary category: Validation failure (tests)
      - May also be Logic failure (implementation incorrect)
      - Confidence: High
    </phase2_classify>

    <phase3_diagnose>
      - Specific test: "should handle division by zero" in calculator.test.js
      - Root cause: Division function returns undefined instead of Infinity for x/0
      - Context: Implementation missing edge case handling
      - Transient: No (logic bug)
    </phase3_diagnose>

    <phase4_strategize>
      - Strategy 1: Fix division function to return Infinity for division by zero
      - Strategy 2: If test expectation wrong, verify requirements and update test
      - Strategy 3: Try different implementation approach for division
    </phase4_strategize>

    <phase5_decide>
      - Attempt count: 0 (first try)
      - Max retries: 3
      - Escalation criteria: Not met (clear fix path)
      - Decision: RETRY with Strategy 1
    </phase5_decide>
  </analysis>

  <output>
    <failure_detected>true</failure_detected>
    <category>Validation</category>
    <confidence>high</confidence>
    <root_cause>Division function returns undefined for x/0, expected Infinity</root_cause>
    <error_context>
      File: src/utils/calculator.js
      Test: calculator.test.js - "should handle division by zero"
      Issue: Missing edge case handling
    </error_context>
    <recommended_strategy>Read calculator.js, add check for divisor === 0 to return Infinity, re-run tests</recommended_strategy>
    <alternative_strategies>
      - Verify test expectation is correct (maybe should throw error instead of Infinity?)
      - Rewrite division function with different approach
    </alternative_strategies>
    <decision>RETRY</decision>
  </output>
</example>

### Example 3: Missing Dependency

<example id="3">
  <input>
    <tool>Bash</tool>
    <command>npm run build</command>
    <exit_code>1</exit_code>
    <stderr>
      Error: Cannot find module '@types/react'
      Require stack:
      - /app/node_modules/typescript/lib/typescript.js
    </stderr>
    <task_context>Running build after adding TypeScript types</task_context>
    <previous_attempts>0</previous_attempts>
  </input>

  <analysis>
    <phase1_detect>
      - Exit code: 1 (failure)
      - Error keyword: "Cannot find module"
      - Pattern match: Dependency Failure (from failure-detection.md)
      - Conclusion: Genuine failure
    </phase1_detect>

    <phase2_classify>
      - Primary category: Dependency failure
      - Pattern: MODULE_NOT_FOUND for @types/react
      - Confidence: High
    </phase2_classify>

    <phase3_diagnose>
      - Specific dependency: @types/react
      - Root cause: Type definitions not installed
      - Context: TypeScript build requires type definitions for dependencies
      - Transient: No (package missing)
    </phase3_diagnose>

    <phase4_strategize>
      - Strategy 1: npm install --save-dev @types/react
      - Strategy 2: Check if already in package.json devDependencies but not installed (run npm install)
      - Strategy 3: Verify compatible version and install
    </phase4_strategize>

    <phase5_decide>
      - Attempt count: 0 (first try)
      - Max retries: 3
      - Escalation criteria: Not met (straightforward npm install)
      - Decision: RETRY with Strategy 1
    </phase5_decide>
  </analysis>

  <output>
    <failure_detected>true</failure_detected>
    <category>Dependency</category>
    <confidence>high</confidence>
    <root_cause>Missing @types/react type definitions</root_cause>
    <error_context>
      Dependency: @types/react
      Context: TypeScript build process
      Operation: npm run build
    </error_context>
    <recommended_strategy>Run: npm install --save-dev @types/react</recommended_strategy>
    <alternative_strategies>
      - Check package.json and run npm install if already listed
      - Install specific version compatible with installed React version
    </alternative_strategies>
    <decision>RETRY</decision>
  </output>
</example>

### Example 4: Network Timeout

<example id="4">
  <input>
    <tool>Bash</tool>
    <command>npm install axios</command>
    <exit_code>1</exit_code>
    <stderr>
      npm ERR! code ETIMEDOUT
      npm ERR! errno ETIMEDOUT
      npm ERR! network request to https://registry.npmjs.org/axios failed, reason: connect ETIMEDOUT
    </stderr>
    <task_context>Installing HTTP client library</task_context>
    <previous_attempts>1</previous_attempts>
  </input>

  <analysis>
    <phase1_detect>
      - Exit code: 1 (failure)
      - Error keyword: "ETIMEDOUT", "network"
      - Pattern match: Network Failure (from failure-detection.md)
      - Conclusion: Genuine failure
    </phase1_detect>

    <phase2_classify>
      - Primary category: Resource failure (network)
      - Pattern: ETIMEDOUT for npm registry
      - Confidence: High
    </phase2_classify>

    <phase3_diagnose>
      - Specific resource: npm registry (registry.npmjs.org)
      - Root cause: Network timeout reaching registry
      - Context: Could be transient network issue or persistent connectivity problem
      - Transient: Likely (network issues often transient)
    </phase3_diagnose>

    <phase4_strategize>
      - Strategy 1: Wait 5-10 seconds, retry same command (attempt 2)
      - Strategy 2: Use longer timeout: npm install --timeout=60000 axios
      - Strategy 3: Check network connectivity, try alternative registry
    </phase4_strategize>

    <phase5_decide>
      - Attempt count: 1 (second attempt)
      - Max retries: 3 (one more left)
      - Escalation criteria: Not met yet (transient network issues get 2-3 retries)
      - Decision: RETRY with Strategy 1 (wait and retry)
    </phase5_decide>
  </analysis>

  <output>
    <failure_detected>true</failure_detected>
    <category>Resource</category>
    <confidence>high</confidence>
    <root_cause>Network timeout connecting to npm registry</root_cause>
    <error_context>
      Resource: registry.npmjs.org
      Operation: npm install axios
      Issue: ETIMEDOUT
    </error_context>
    <recommended_strategy>Wait 5 seconds (exponential backoff), then retry same command</recommended_strategy>
    <alternative_strategies>
      - Use longer timeout: npm install --timeout=60000 axios
      - Check network and try alternative registry if persistent
    </alternative_strategies>
    <decision>RETRY</decision>
  </output>
</example>

### Example 5: Logic Error - Escalation Needed

<example id="5">
  <input>
    <tool>Bash</tool>
    <command>npm test</command>
    <exit_code>1</exit_code>
    <stdout>
      Test Suites: 0 passed, 1 failed, 1 total
      Tests:       0 passed, 5 failed, 5 total

      FAIL src/auth/login.test.js
        ● All tests failing with "Expected user to be authenticated but got null"
    </stdout>
    <task_context>Implemented OAuth login flow</task_context>
    <previous_attempts>2</previous_attempts>
    <previous_strategies>
      - Attempt 1: Fixed async/await handling
      - Attempt 2: Tried different OAuth library
    </previous_strategies>
  </input>

  <analysis>
    <phase1_detect>
      - Exit code: 1 (failure)
      - Error keyword: "failed"
      - Pattern match: Validation Failure (tests)
      - Conclusion: Genuine failure
    </phase1_detect>

    <phase2_classify>
      - Primary category: Logic failure (implementation wrong)
      - Secondary: Validation failure (tests failing)
      - Confidence: High
    </phase2_classify>

    <phase3_diagnose>
      - All 5 tests failing consistently
      - Root cause: OAuth implementation not working
      - Context: Two different approaches tried, both failed
      - Persistent: Yes (not transient)
    </phase3_diagnose>

    <phase4_strategize>
      - Previous strategies exhausted: async fix, library change
      - Remaining options limited, likely fundamental misunderstanding
      - Could try: complete rewrite, different OAuth provider, manual auth
    </phase4_strategize>

    <phase5_decide>
      - Attempt count: 2 (third attempt would hit max)
      - Max retries: 3
      - Escalation criteria: MET - multiple approaches failed, suggests requirements misunderstanding
      - Decision: ESCALATE
    </phase5_decide>
  </analysis>

  <output>
    <failure_detected>true</failure_detected>
    <category>Logic</category>
    <confidence>high</confidence>
    <root_cause>OAuth login implementation consistently fails all tests across multiple approaches</root_cause>
    <error_context>
      File: src/auth/login.js
      Tests: All 5 tests in login.test.js failing
      Attempts: 2 (async fix, library change)
      Issue: Fundamental implementation problem
    </error_context>
    <recommended_strategy>N/A - escalation required</recommended_strategy>
    <alternative_strategies>N/A</alternative_strategies>
    <decision>ESCALATE</decision>
    <escalation_reason>
      Multiple implementation approaches failed (2 attempts). OAuth requirements may be misunderstood, or architectural decision needed (which OAuth provider? what flow?). Recommend converting to checkpoint:decision to discuss approach with user.
    </escalation_reason>
  </output>
</example>

### Example 6: Permission Error - Immediate Escalation

<example id="6">
  <input>
    <tool>Bash</tool>
    <command>mkdir /usr/local/myapp</command>
    <exit_code>1</exit_code>
    <stderr>mkdir: /usr/local/myapp: Permission denied</stderr>
    <task_context>Creating application directory</task_context>
    <previous_attempts>0</previous_attempts>
  </input>

  <analysis>
    <phase1_detect>
      - Exit code: 1 (failure)
      - Error keyword: "Permission denied"
      - Pattern match: Permission Failure (from failure-detection.md)
      - Conclusion: Genuine failure
    </phase1_detect>

    <phase2_classify>
      - Primary category: Resource failure (permissions)
      - Pattern: EACCES / permission denied for system directory
      - Confidence: High
    </phase2_classify>

    <phase3_diagnose>
      - Specific path: /usr/local/myapp (system directory)
      - Root cause: Insufficient permissions to create dir in /usr/local
      - Context: Requires sudo or different location
      - Transient: No (permissions won't change automatically)
    </phase3_diagnose>

    <phase4_strategize>
      - Strategy 1: Use user home directory instead (~/.local/myapp or ~/myapp)
      - Strategy 2: Escalate - user needs to grant permissions or choose different location
      - Note: Cannot auto-fix with sudo (security risk)
    </phase4_strategize>

    <phase5_decide>
      - Attempt count: 0 (first try)
      - Escalation criteria: MET - system permissions required, or architectural decision (where to install?)
      - Decision: ESCALATE (permissions in /usr/local are not auto-recoverable)
    </phase5_decide>
  </analysis>

  <output>
    <failure_detected>true</failure_detected>
    <category>Resource</category>
    <confidence>high</confidence>
    <root_cause>Permission denied creating directory in /usr/local (system location)</root_cause>
    <error_context>
      Path: /usr/local/myapp
      Operation: mkdir
      Issue: Insufficient permissions
    </error_context>
    <recommended_strategy>N/A - escalation required</recommended_strategy>
    <alternative_strategies>
      - Use user home directory: ~/myapp or ~/.local/myapp (can suggest to user)
    </alternative_strategies>
    <decision>ESCALATE</decision>
    <escalation_reason>
      Cannot auto-fix system permission issues. User needs to either: (1) grant sudo permissions, (2) choose different directory location. Recommend converting to checkpoint:decision to discuss installation location.
    </escalation_reason>
  </output>
</example>

## Quick Reference: Category → First Strategy

<quick_reference>
  <category name="Tool">
    <first_strategy>Verify file/command exists, use glob/grep to find correct path</first_strategy>
  </category>

  <category name="Validation">
    <first_strategy>Analyze error details, fix implementation bugs (imports, syntax, logic)</first_strategy>
  </category>

  <category name="Dependency">
    <first_strategy>Install missing dependency: npm install [package]</first_strategy>
  </category>

  <category name="Logic">
    <first_strategy>Fix obvious bugs (syntax, typos). If complex, escalate quickly (after 1-2 attempts)</first_strategy>
  </category>

  <category name="Resource">
    <first_strategy>Wait and retry (transient network). If persistent (permissions, disk), escalate</first_strategy>
  </category>
</quick_reference>

## Integration Points

This document integrates with:
- `failure-detection.md` - Input patterns for Phase 1 (Detect)
- `failure-taxonomy.md` - Category definitions and recovery strategies for Phases 2-4
- `../workflows/execute-plan.md` - (future) Will use this analysis to implement automatic retry
- `ATTEMPTS_LOG.md` - (future) Will log analysis outputs for each retry attempt

---

*Document-driven failure analysis for GSD automatic retry system*
