# Failure Detection Patterns

This document defines patterns for detecting failures in tool execution results. Used by failure-analysis.md to classify errors and select recovery strategies.

## Purpose

Enable GSD to automatically detect when tasks fail by recognizing common failure patterns in tool outputs (Bash, Edit, Write, Read, etc.).

## Tool Result Parsing

<parsing_strategy>
Tool results contain multiple signals that indicate failure:
- Exit codes (Bash tool)
- Error messages in stdout/stderr
- Stack traces
- Tool-specific error responses
- Timeout indicators
- Resource access errors
</parsing_strategy>

## Failure Indicators

### 1. Exit Code Failures

<failure_pattern type="exit_code">
  <indicator>Non-zero exit code from Bash tool</indicator>
  <detection>
    - Exit code 1: General error
    - Exit code 2: Misuse of shell builtin
    - Exit code 126: Command cannot execute
    - Exit code 127: Command not found
    - Exit code 128+N: Fatal error signal N
    - Exit code 130: Script terminated by Ctrl+C
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>npm install
Error: ENOENT: no such file or directory, open 'package.json'
Exit code: 1</output>
      <failure>true</failure>
      <category>dependency</category>
    </example>
  </examples>
</failure_pattern>

### 2. Timeout Failures

<failure_pattern type="timeout">
  <indicator>Command execution exceeds time limit</indicator>
  <detection>
    - "timed out" message
    - "ETIMEDOUT" error code
    - "timeout" in error message
    - Tool response indicates timeout exceeded
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>Command timed out after 120000ms</output>
      <failure>true</failure>
      <category>resource</category>
    </example>
  </examples>
</failure_pattern>

### 3. File Access Failures

<failure_pattern type="file_error">
  <indicator>File or directory cannot be accessed</indicator>
  <detection>
    - "ENOENT: no such file or directory"
    - "EACCES: permission denied"
    - "EISDIR: illegal operation on a directory"
    - "ENOTDIR: not a directory"
    - "EEXIST: file already exists" (when unexpected)
    - "cannot open file"
    - "file not found"
  </detection>
  <examples>
    <example>
      <tool>Read</tool>
      <output>Error: ENOENT: no such file or directory, open '/path/to/missing.js'</output>
      <failure>true</failure>
      <category>tool</category>
    </example>
    <example>
      <tool>Write</tool>
      <output>Error: EACCES: permission denied, open '/etc/protected.conf'</output>
      <failure>true</failure>
      <category>resource</category>
    </example>
  </examples>
</failure_pattern>

### 4. Permission Failures

<failure_pattern type="permission">
  <indicator>Insufficient permissions to perform operation</indicator>
  <detection>
    - "permission denied"
    - "EACCES"
    - "EPERM: operation not permitted"
    - "requires elevated privileges"
    - "sudo" mentioned in error context
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>mkdir: /usr/local/protected: Permission denied
Exit code: 1</output>
      <failure>true</failure>
      <category>resource</category>
    </example>
  </examples>
</failure_pattern>

### 5. Validation Failures

<failure_pattern type="validation">
  <indicator>Tests, builds, or type checks fail</indicator>
  <detection>
    - "test failed" or "tests failed"
    - "build failed"
    - "compilation error"
    - "type error"
    - "lint error"
    - "X passing, Y failing" (where Y > 0)
    - "FAIL" in test output
    - "ERROR" in build output
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>npm test
  16 passing (2s)
  3 failing

Exit code: 1</output>
      <failure>true</failure>
      <category>validation</category>
    </example>
    <example>
      <tool>Bash</tool>
      <output>npm run build
ERROR in ./src/main.js
Module not found: Error: Can't resolve 'missing-module'
Exit code: 1</output>
      <failure>true</failure>
      <category>validation</category>
    </example>
  </examples>
</failure_pattern>

### 6. Dependency Failures

<failure_pattern type="dependency">
  <indicator>Missing packages, unmet prerequisites, or environment issues</indicator>
  <detection>
    - "MODULE_NOT_FOUND"
    - "Cannot find module"
    - "package not found"
    - "command not found" (for installed tools)
    - "ENOENT" for expected executables
    - "peer dependency" warnings with failures
    - "incompatible version"
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>Error: Cannot find module 'express'
Require stack:
- /app/server.js
Exit code: 1</output>
      <failure>true</failure>
      <category>dependency</category>
    </example>
    <example>
      <tool>Bash</tool>
      <output>sh: typescript: command not found
Exit code: 127</output>
      <failure>true</failure>
      <category>dependency</category>
    </example>
  </examples>
</failure_pattern>

### 7. Network Failures

<failure_pattern type="network">
  <indicator>Network operations fail</indicator>
  <detection>
    - "ECONNREFUSED"
    - "ENOTFOUND"
    - "EHOSTUNREACH"
    - "network error"
    - "fetch failed"
    - "unable to connect"
    - "connection refused"
    - "DNS resolution failed"
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>npm install
npm ERR! code ENOTFOUND
npm ERR! errno ENOTFOUND
npm ERR! network request to https://registry.npmjs.org failed
Exit code: 1</output>
      <failure>true</failure>
      <category>resource</category>
    </example>
  </examples>
</failure_pattern>

### 8. Syntax/Logic Errors

<failure_pattern type="syntax">
  <indicator>Code contains syntax errors or logical mistakes</indicator>
  <detection>
    - "SyntaxError"
    - "unexpected token"
    - "parse error"
    - "invalid syntax"
    - "ReferenceError"
    - "TypeError" (may indicate logic error)
    - Stack trace with error location
  </detection>
  <examples>
    <example>
      <tool>Bash</tool>
      <output>node app.js
/app/app.js:15
  const data = {
               ^
SyntaxError: Unexpected token '{'
Exit code: 1</output>
      <failure>true</failure>
      <category>logic</category>
    </example>
  </examples>
</failure_pattern>

## Success vs Partial Failure

<success_criteria>
  <complete_success>
    - Exit code 0 (or no exit code for non-Bash tools)
    - No error messages in output
    - Expected outcomes achieved
    - Verification checks pass
  </complete_success>

  <partial_success>
    - Exit code 0 but warnings present
    - Operation completed but with degraded results
    - Some tests pass, others skipped (not failed)
    - Deprecated features used but still functional
  </partial_success>

  <degraded_success>
    - Primary goal achieved but secondary goals failed
    - Fallback approach succeeded after primary failed
    - Results obtained but quality/performance suboptimal
  </degraded_success>

  <complete_failure>
    - Non-zero exit code
    - Error messages indicate operation failed
    - No useful output produced
    - Verification checks fail
    - Task cannot proceed
  </complete_failure>
</success_criteria>

## Detection Algorithm

<detection_process>
  <step>1. Check exit code (if Bash tool): non-zero = likely failure</step>
  <step>2. Scan output for error keywords: Error, ERROR, FAIL, failed, etc.</step>
  <step>3. Match against specific failure patterns above</step>
  <step>4. Extract context: which file? which command? which dependency?</step>
  <step>5. Determine severity: blocking failure vs warning</step>
  <step>6. Classify using failure-taxonomy.md categories</step>
</detection_process>

## Pattern Matching Priority

When multiple patterns match, use this priority order:

<priority_order>
  1. Exit code (highest confidence)
  2. Explicit error messages (ENOENT, EACCES, etc.)
  3. Test/build failure indicators
  4. Stack traces
  5. Warning messages (lowest confidence, may not be failure)
</priority_order>

## Edge Cases

<edge_cases>
  <case>
    <scenario>Command succeeds but output contains "error" in non-error context</scenario>
    <handling>Verify exit code is 0, check for "Error:" prefix vs casual mention</handling>
  </case>

  <case>
    <scenario>Interactive commands that pause for input</scenario>
    <handling>Detect prompts like "Continue? (y/n)", classify as tool limitation not failure</handling>
  </case>

  <case>
    <scenario>Commands that intentionally return non-zero (like grep finding no matches)</scenario>
    <handling>Context-aware interpretation: grep exit 1 = no match, not error</handling>
  </case>

  <case>
    <scenario>Warnings that don't prevent operation success</scenario>
    <handling>Distinguish warnings from errors, don't classify as failure unless blocking</handling>
  </case>
</edge_cases>

## Integration

This document is referenced by:
- `failure-analysis.md` - Uses these patterns to diagnose failures
- `failure-taxonomy.md` - Maps detected patterns to recovery strategies

---

*Document-driven failure detection for GSD automatic retry system*
