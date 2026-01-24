# Failure Classification Taxonomy

This document defines error categories and their recovery implications. Used by failure-analysis.md to map detected failures to appropriate recovery strategies.

## Purpose

Provide a structured classification system that maps failure types to recovery actions, enabling GSD to automatically select appropriate retry strategies or escalate intelligently.

## Taxonomy Overview

<taxonomy_structure>
Failures are classified into 5 primary categories:
1. Tool Failures - Problems with tool execution (Bash, Edit, Write, Read)
2. Validation Failures - Tests, builds, type checks, lints that fail
3. Dependency Failures - Missing packages, unmet prerequisites, environment issues
4. Logic Failures - Incorrect implementation, wrong approach, semantic errors
5. Resource Failures - Timeouts, memory issues, permissions, network problems

Each category has:
- Typical symptoms (what it looks like)
- Likely causes (why it happened)
- Recovery strategies (what to try)
- Escalation criteria (when to give up and ask user)
</taxonomy_structure>

## Category 1: Tool Failures

<failure_category name="tool">
  <description>
    Problems executing GSD tools (Bash, Edit, Write, Read, etc.) where the tool itself fails to perform the requested operation.
  </description>

  <symptoms>
    - Command not found (exit 127)
    - File not found for read/edit operations (ENOENT)
    - Invalid tool parameters
    - Bash command fails with non-zero exit
    - Edit fails due to old_string not matching
    - Write fails due to path issues
  </symptoms>

  <causes>
    - Typo in command or file path
    - Wrong working directory
    - File moved/renamed since last reference
    - Incorrect string matching for Edit tool
    - Tool parameter mismatch
  </causes>

  <recovery_strategies>
    <strategy priority="1">
      <approach>Verify file/command exists before retry</approach>
      <action>Use glob/grep to find correct path, verify command availability</action>
      <success_likelihood>high</success_likelihood>
    </strategy>

    <strategy priority="2">
      <approach>Adjust parameters and retry</approach>
      <action>Fix typos, update paths, expand old_string context for Edit</action>
      <success_likelihood>high</success_likelihood>
    </strategy>

    <strategy priority="3">
      <approach>Use alternative tool approach</approach>
      <action>Replace Edit with Write, try different command syntax</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>
  </recovery_strategies>

  <escalation_criteria>
    - File/command genuinely doesn't exist and should (user needs to create it)
    - Tool limitations prevent operation (need different tool)
    - 3 retry attempts exhausted without success
  </escalation_criteria>

  <alignment_with_gsd>
    Most tool failures can be auto-fixed. Rarely requires escalation to checkpoint:decision.
  </alignment_with_gsd>

  <examples>
    <example>
      <failure>Edit tool fails: old_string not found</failure>
      <recovery>Expand old_string with more context lines, verify via Read first</recovery>
      <outcome>Success on retry</outcome>
    </example>
    <example>
      <failure>Bash: command not found for 'tsc'</failure>
      <recovery>Check if typescript installed, use npx tsc instead</recovery>
      <outcome>Success with alternative approach</outcome>
    </example>
  </examples>
</failure_category>

## Category 2: Validation Failures

<failure_category name="validation">
  <description>
    Tests, builds, type checks, lints, or other quality gates fail, indicating code doesn't meet requirements.
  </description>

  <symptoms>
    - "X passing, Y failing" in test output
    - "build failed" or compilation errors
    - "type error" from TypeScript/type checkers
    - "lint error" from code quality tools
    - Non-zero exit from npm test, npm run build, etc.
  </symptoms>

  <causes>
    - Code implementation incorrect
    - Test expectations need updating
    - Breaking change introduced
    - Type definitions missing or wrong
    - Code style violations
  </causes>

  <recovery_strategies>
    <strategy priority="1">
      <approach>Analyze error details and fix implementation</approach>
      <action>Read error output, identify failing test/build step, correct code</action>
      <success_likelihood>high</success_likelihood>
    </strategy>

    <strategy priority="2">
      <approach>Update tests to match new implementation</approach>
      <action>If implementation is correct but tests outdated, update test expectations</action>
      <success_likelihood>medium</success_likelihood>
      <caution>Only if confident implementation is correct</caution>
    </strategy>

    <strategy priority="3">
      <approach>Try alternative implementation approach</approach>
      <action>Different algorithm, different library, refactor logic</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>

    <strategy priority="4">
      <approach>Add missing dependencies or type definitions</approach>
      <action>Install @types packages, add imports, configure tsconfig</action>
      <success_likelihood>high</success_likelihood>
    </strategy>
  </recovery_strategies>

  <escalation_criteria>
    - Test failures indicate requirement misunderstanding (need user clarification)
    - Build errors stem from architectural issue (need design decision)
    - Type errors suggest API misunderstanding (need user input)
    - 3 different fix attempts all fail tests
  </escalation_criteria>

  <alignment_with_gsd>
    Validation failures often require checkpoint:decision if they indicate wrong approach.
    Auto-retry appropriate for simple fixes (missing import, typo).
    Escalate for complex test failures that suggest design issue.
  </alignment_with_gsd>

  <examples>
    <example>
      <failure>5 tests failing after refactor</failure>
      <recovery>Analyze test errors, fix broken imports and function signatures</recovery>
      <outcome>Success after implementation correction</outcome>
    </example>
    <example>
      <failure>Build fails: Cannot find module '@types/node'</failure>
      <recovery>npm install --save-dev @types/node</recovery>
      <outcome>Success with dependency addition</outcome>
    </example>
    <example>
      <failure>Tests fail because implementation doesn't match requirements</failure>
      <recovery>Cannot auto-recover, requirements unclear</recovery>
      <outcome>Escalate to user for clarification</outcome>
    </example>
  </examples>
</failure_category>

## Category 3: Dependency Failures

<failure_category name="dependency">
  <description>
    Missing packages, unmet prerequisites, version conflicts, or environment configuration issues.
  </description>

  <symptoms>
    - "MODULE_NOT_FOUND" or "Cannot find module"
    - "command not found" for expected tools
    - "peer dependency" errors
    - "incompatible version" warnings with failure
    - "ENOENT" when trying to run installed executable
  </symptoms>

  <causes>
    - Package not installed (npm install not run)
    - Package installed locally but not globally (or vice versa)
    - Version conflict between dependencies
    - Missing system-level dependency
    - Wrong Node.js version
  </causes>

  <recovery_strategies>
    <strategy priority="1">
      <approach>Install missing dependency</approach>
      <action>npm install [package], npm install -g [package], or package.json update</action>
      <success_likelihood>high</success_likelihood>
    </strategy>

    <strategy priority="2">
      <approach>Use npx for commands instead of direct execution</approach>
      <action>Replace 'tsc' with 'npx tsc', ensures local package is used</action>
      <success_likelihood>high</success_likelihood>
    </strategy>

    <strategy priority="3">
      <approach>Resolve version conflicts</approach>
      <action>Update package.json version ranges, use npm install --legacy-peer-deps</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>

    <strategy priority="4">
      <approach>Use alternative package</approach>
      <action>Replace conflicting dependency with compatible alternative</action>
      <success_likelihood>low</success_likelihood>
      <caution>May require code changes</caution>
    </strategy>
  </recovery_strategies>

  <escalation_criteria>
    - System-level dependency missing (needs user installation: Python, Docker, etc.)
    - Version conflict requires architectural decision
    - Package doesn't exist or is deprecated
    - Network issues prevent package installation (see Resource Failures)
  </escalation_criteria>

  <alignment_with_gsd>
    Most dependency failures auto-recoverable via npm install.
    Escalate to checkpoint:decision for system dependencies or architectural conflicts.
  </alignment_with_gsd>

  <examples>
    <example>
      <failure>Error: Cannot find module 'express'</failure>
      <recovery>npm install express</recovery>
      <outcome>Success after installation</outcome>
    </example>
    <example>
      <failure>sh: typescript: command not found</failure>
      <recovery>npx tsc instead of tsc, or npm install -g typescript</recovery>
      <outcome>Success with npx approach</outcome>
    </example>
    <example>
      <failure>Peer dependency conflict between react@18 and library requiring react@17</failure>
      <recovery>Cannot auto-resolve, needs architectural decision</recovery>
      <outcome>Escalate to user for version strategy</outcome>
    </example>
  </examples>
</failure_category>

## Category 4: Logic Failures

<failure_category name="logic">
  <description>
    Incorrect implementation, wrong approach, semantic errors, or algorithmic mistakes.
  </description>

  <symptoms>
    - Code runs but produces wrong results
    - Tests fail due to incorrect logic
    - Infinite loops or crashes
    - SyntaxError, ReferenceError, TypeError in execution
    - Logic doesn't match requirements
  </symptoms>

  <causes>
    - Misunderstood requirements
    - Algorithm error
    - Edge case not handled
    - Off-by-one error
    - Wrong API usage
    - Race condition or async/await mistake
  </causes>

  <recovery_strategies>
    <strategy priority="1">
      <approach>Analyze error and fix obvious logic bugs</approach>
      <action>Fix syntax errors, undefined references, typos in code</action>
      <success_likelihood>high</success_likelihood>
    </strategy>

    <strategy priority="2">
      <approach>Revisit requirements and adjust implementation</approach>
      <action>Re-read task description, align code with actual requirements</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>

    <strategy priority="3">
      <approach>Try completely different implementation approach</approach>
      <action>Different algorithm, different data structure, alternative library</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>

    <strategy priority="4">
      <approach>Research correct approach</approach>
      <action>Search documentation, look for examples, understand API properly</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>
  </recovery_strategies>

  <escalation_criteria>
    - Requirements genuinely ambiguous (need user clarification)
    - Multiple implementation attempts all fail tests
    - Algorithmic complexity beyond current capability
    - Approach fundamentally wrong (need architectural rethink)
  </escalation_criteria>

  <alignment_with_gsd>
    Logic failures often warrant escalation to checkpoint:decision.
    Auto-retry appropriate for simple bugs (syntax, typos).
    Escalate when approach is fundamentally wrong or requirements unclear.
    This is the category most likely to need user input.
  </alignment_with_gsd>

  <examples>
    <example>
      <failure>SyntaxError: Unexpected token '{'</failure>
      <recovery>Fix syntax error (missing semicolon, wrong bracket)</recovery>
      <outcome>Success after syntax correction</outcome>
    </example>
    <example>
      <failure>Function returns undefined instead of calculated value</failure>
      <recovery>Add missing return statement</recovery>
      <outcome>Success after logic fix</outcome>
    </example>
    <example>
      <failure>Implementation uses wrong algorithm, tests consistently fail</failure>
      <recovery>Try 2 alternative approaches, both fail</recovery>
      <outcome>Escalate to user - requirements may be misunderstood</outcome>
    </example>
  </examples>
</failure_category>

## Category 5: Resource Failures

<failure_category name="resource">
  <description>
    Timeouts, memory issues, permission problems, disk space, network errors, or other environmental/resource constraints.
  </description>

  <symptoms>
    - "timed out" or ETIMEDOUT
    - "EACCES: permission denied"
    - "EPERM: operation not permitted"
    - "ENOSPC: no space left on device"
    - "EMFILE: too many open files"
    - "ECONNREFUSED" or network errors
    - Out of memory errors
  </symptoms>

  <causes>
    - Command takes too long (slow network, large computation)
    - Insufficient permissions
    - Network unavailable or service down
    - Disk full
    - Resource limits exceeded
    - Firewall or proxy blocking
  </causes>

  <recovery_strategies>
    <strategy priority="1">
      <approach>Retry with longer timeout</approach>
      <action>Increase timeout parameter, wait and retry</action>
      <success_likelihood>medium</success_likelihood>
    </strategy>

    <strategy priority="2">
      <approach>Retry same operation (transient network issues)</approach>
      <action>Wait briefly, then retry exact same command</action>
      <success_likelihood>medium</success_likelihood>
      <caution>Use exponential backoff, limit retries</caution>
    </strategy>

    <strategy priority="3">
      <approach>Work around permission issue</approach>
      <action>Use different directory with write permissions, adjust approach</action>
      <success_likelihood>low</success_likelihood>
    </strategy>

    <strategy priority="4">
      <approach>Use cached/offline alternative</approach>
      <action>If network operation, check for local cache or alternative source</action>
      <success_likelihood>low</success_likelihood>
    </strategy>
  </recovery_strategies>

  <escalation_criteria>
    - Permission issue requires sudo or system configuration change
    - Network persistently unavailable (can't reach registry, API, etc.)
    - Disk space genuinely insufficient (user needs to free space)
    - Timeout due to genuinely expensive operation (need different approach)
    - Resource limits need system-level adjustment
  </escalation_criteria>

  <alignment_with_gsd>
    Transient resource failures (network blips, temporary timeouts) can auto-retry.
    Persistent resource issues (permissions, disk space) escalate to user.
    Use exponential backoff for network retries.
  </alignment_with_gsd>

  <examples>
    <example>
      <failure>npm install fails with ETIMEDOUT</failure>
      <recovery>Wait 5 seconds, retry same command</recovery>
      <outcome>Success on retry (transient network issue)</outcome>
    </example>
    <example>
      <failure>mkdir: Permission denied for /usr/local/myapp</failure>
      <recovery>Cannot auto-fix, needs sudo or different location</recovery>
      <outcome>Escalate to user for permission grant or path change</outcome>
    </example>
    <example>
      <failure>Command timeout after 120s (default)</failure>
      <recovery>Retry with 300s timeout</recovery>
      <outcome>Success with longer timeout</outcome>
    </example>
  </examples>
</failure_category>

## Recovery Strategy Selection

<selection_algorithm>
  <step>1. Classify failure using failure-detection.md patterns</step>
  <step>2. Map to category: Tool, Validation, Dependency, Logic, or Resource</step>
  <step>3. Review recovery strategies in priority order</step>
  <step>4. Select highest-priority strategy not yet attempted</step>
  <step>5. Execute strategy and verify outcome</step>
  <step>6. If failure persists, move to next priority strategy</step>
  <step>7. If all strategies exhausted, check escalation criteria</step>
  <step>8. Escalate to checkpoint:decision or user input if warranted</step>
</selection_algorithm>

## Retry Limits

<retry_constraints>
  <max_attempts>3</max_attempts>
  <reasoning>Per GSD Evolution Master Plan risk mitigation - prevents infinite loops</reasoning>

  <backoff_strategy>
    <attempt>1</attempt>
    <delay>0s (immediate retry after fix)</delay>
  </backoff_strategy>

  <backoff_strategy>
    <attempt>2</attempt>
    <delay>5s (for transient network/resource issues)</delay>
  </backoff_strategy>

  <backoff_strategy>
    <attempt>3</attempt>
    <delay>10s (final retry with maximum backoff)</delay>
  </backoff_strategy>
</retry_constraints>

## Escalation Decision Matrix

<escalation_matrix>
  <category name="tool">
    <auto_recover>high</auto_recover>
    <escalate_after>3 attempts</escalate_after>
    <checkpoint_decision>rare</checkpoint_decision>
  </category>

  <category name="validation">
    <auto_recover>medium</auto_recover>
    <escalate_after>2-3 attempts</escalate_after>
    <checkpoint_decision>often (for design issues)</checkpoint_decision>
  </category>

  <category name="dependency">
    <auto_recover>high</auto_recover>
    <escalate_after>2 attempts</escalate_after>
    <checkpoint_decision>sometimes (for system deps)</checkpoint_decision>
  </category>

  <category name="logic">
    <auto_recover>low</auto_recover>
    <escalate_after>1-2 attempts</escalate_after>
    <checkpoint_decision>very often (usually needs rethink)</checkpoint_decision>
  </category>

  <category name="resource">
    <auto_recover>medium</auto_recover>
    <escalate_after>2-3 attempts</escalate_after>
    <checkpoint_decision>sometimes (for permissions/config)</checkpoint_decision>
  </category>
</escalation_matrix>

## Integration with GSD Task Types

<gsd_integration>
  <task_type name="auto">
    Auto tasks continue with automatic retry using this taxonomy.
    Escalate to user after max attempts or if escalation criteria met.
  </task_type>

  <task_type name="checkpoint">
    Checkpoint tasks already involve user, so failures can be discussed immediately.
    Taxonomy still useful for diagnosing issue before presenting to user.
  </task_type>

  <task_type name="checkpoint:decision">
    Use this task type when automatic retry exhausted and need user decision.
    Present failure category, attempted strategies, and recommended alternatives.
  </task_type>
</gsd_integration>

## Cross-References

This document is used by:
- `failure-analysis.md` - Maps detected failures to recovery strategies
- `../workflows/execute-plan.md` - (future) Implements retry logic using this taxonomy
- `ATTEMPTS_LOG.md` - (future) Logs which strategies were attempted

This document references:
- `failure-detection.md` - Provides failure pattern matching

---

*Document-driven failure classification for GSD automatic retry system*
