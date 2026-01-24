# Adaptive Failure Analysis Workflow

AI-powered failure diagnostic system using Claude to analyze task failures with richer context than static decision trees.

## Purpose

This workflow provides intelligent failure analysis that goes beyond hardcoded decision trees:

**Static analysis (failure-analysis.md):**
- Fixed decision tree with pre-programmed logic
- Pattern matching against known error types
- Cannot adapt to novel failure patterns
- Limited to predefined recovery strategies

**Adaptive analysis (this workflow):**
- AI-powered reasoning about failure context
- Understands nuanced error messages and stack traces
- Adapts recommendations to specific task context
- Can suggest creative recovery approaches
- Falls back to static analysis if AI analysis fails

**When to use:**
- Enabled via `.planning/config.json` with `config.adaptive.enabled === true`
- Invoked from retry-orchestration.md step 3 (analyze_failure)
- Provides richer diagnostics than static decision trees

## Input Specification

<input_requirements>
  <required>
    <field name="tool">Which tool was used (Bash, Edit, Write, Read, etc.)</field>
    <field name="command">Exact command or operation attempted</field>
    <field name="exit_code">Exit code (for Bash) or error indicator</field>
    <field name="stdout">Standard output from tool</field>
    <field name="stderr">Standard error from tool (if applicable)</field>
    <field name="task_context">What was the task trying to accomplish?</field>
  </required>

  <optional>
    <field name="task_goal">High-level objective of the task</field>
    <field name="attempt_number">Current attempt count (1, 2, or 3)</field>
    <field name="prior_attempts">What strategies were already tried and failed?</field>
    <field name="task_type">Type of task: auto, checkpoint, or checkpoint:decision</field>
  </optional>
</input_requirements>

## Process

<step name="prepare_context" number="1">

**Assemble comprehensive failure analysis prompt:**

Build a detailed context document for AI analysis including:

**1. Failure details:**
```markdown
## Failure Information

**Tool:** [tool name]
**Command:** [exact command/operation]
**Exit Code:** [exit code or error indicator]

**Output:**
```
[stdout content - first 1000 chars if very long]
```

**Error Messages:**
```
[stderr content - first 1000 chars if very long]
```
```

**2. Task context:**
```markdown
## Task Context

**Objective:** [what the task was trying to accomplish]
**Goal:** [high-level purpose of this task in the plan]
**Attempt:** [attempt number] of [max attempts]
```

**3. Prior attempt history (if available):**
```markdown
## Prior Attempts

[For each prior attempt:]
**Attempt [N]:**
- Strategy: [strategy name]
- Approach: [what was tried]
- Outcome: [why it failed]
```

**4. Failure taxonomy reference:**
```markdown
## Classification Framework

Task failures fall into 5 categories:
1. **Tool** - File not found, command not found, tool parameter errors
2. **Validation** - Tests failing, builds failing, type errors
3. **Dependency** - Missing packages, unmet prerequisites
4. **Logic** - Incorrect implementation, wrong approach, semantic errors
5. **Resource** - Timeouts, permissions, network issues, disk space

Examples from failure-detection.md:
- ENOENT (file not found) → Tool failure
- "3 tests failing" → Validation failure
- "Cannot find module" → Dependency failure
- "SyntaxError" → Logic failure
- "ETIMEDOUT" → Resource failure
```

**5. Analysis request:**
```markdown
## Analysis Request

Analyze this failure and provide:

1. **Classification:** Which of the 5 categories (Tool, Validation, Dependency, Logic, Resource)?
2. **Root Cause:** Specific description of what went wrong (not generic)
3. **Retry Recommendation:** Should we retry with alternative approach, or escalate to user?
4. **Confidence:** How confident are you in this analysis? (high/medium/low)

**Output format (structured markdown with clear sections):**

### Classification
[Category name]

### Root Cause
[Specific description: "File X not found at path Y", not "file error"]

### Retry Appropriate
[Yes/No - with brief reason]

### Confidence Level
[high/medium/low]

### Recommended Recovery
[If retry appropriate: specific action to take. If escalate: why escalation needed]

### Alternative Approaches
[2-3 backup strategies if recommended approach fails]
```

**Assembled prompt saved to variable:** `analysis_prompt`

</step>

<step name="invoke_ai_analyzer" number="2">

**Use Task tool to analyze failure:**

**Pattern from Phase 3 (understand-goal.md step 2):**

Invoke general-purpose agent with comprehensive context:

```javascript
// Pseudocode for invocation pattern
Task.create({
  subagent_type: "general-purpose",
  instruction: analysis_prompt,  // From step 1
  context: [
    "@get-shit-done/lib/failure-detection.md",  // Pattern examples
    "@get-shit-done/lib/failure-taxonomy.md"    // Category definitions
  ],
  timeout: 30000  // 30 second timeout for analysis
})
```

**Actual invocation:**

Use Task tool with:
- Subagent type: general-purpose
- Instruction: Full analysis prompt from step 1
- Context: Load failure-detection.md and failure-taxonomy.md for reference
- Expected output: Structured markdown response with sections

**Wait for response:**
- AI analyzes failure context
- Classifies into one of 5 categories
- Extracts specific root cause
- Recommends retry or escalation
- Provides confidence level

**Capture output:** Store AI response in `ai_analysis_result`

</step>

<step name="validate_output" number="3">

**Ensure AI returned valid, actionable analysis:**

**Parse AI response for required sections:**

1. **Check for "Classification" section:**
   - Extract category name
   - Verify it's one of: Tool, Validation, Dependency, Logic, Resource
   - If invalid or missing → mark as malformed

2. **Check for "Root Cause" section:**
   - Verify it's specific (mentions actual file/command/module names)
   - Not generic ("error occurred", "something failed")
   - If too generic or missing → mark as malformed

3. **Check for "Retry Appropriate" section:**
   - Extract Yes/No decision
   - If unclear or missing → mark as malformed

4. **Check for "Confidence Level" section:**
   - Extract high/medium/low
   - If missing → default to "medium"

**Validation decision:**

```markdown
If all required sections present AND category is valid AND root cause is specific:
  → Use AI analysis (proceed to output)

If any validation fails OR response is malformed:
  → Fall back to static analysis:
     1. Load @get-shit-done/lib/failure-analysis.md
     2. Run static decision tree workflow
     3. Use static analysis output instead
     4. Log warning: "Adaptive analysis failed validation, using static fallback"
```

**Fallback handling:**
- Adaptive analysis is best-effort enhancement
- Static analysis is always available as reliable fallback
- Never block retry workflow due to AI analysis failure
- Log which analysis method was used for debugging

**Error cases handled:**
- AI timeout → fallback to static
- Malformed response → fallback to static
- Invalid category → fallback to static
- Missing sections → fallback to static
- Task tool error → fallback to static

</step>

## Output Specification

<output_format>
  <field name="analysis_method">adaptive | static</field>
  <field name="failure_category">Tool | Validation | Dependency | Logic | Resource</field>
  <field name="root_cause">Specific description of what went wrong</field>
  <field name="retry_appropriate">true | false</field>
  <field name="confidence_level">high | medium | low</field>
  <field name="recommended_strategy">Specific action to take (if retry appropriate)</field>
  <field name="alternative_strategies">Array of 2-3 backup approaches</field>
  <field name="escalation_reason">Why escalation needed (if retry inappropriate)</field>
</output_format>

**Output structure (compatible with retry-orchestration.md expectations):**

```xml
<failure_analysis>
  <method>adaptive | static</method>
  <category>Tool | Validation | Dependency | Logic | Resource</category>
  <root_cause>[Specific description]</root_cause>
  <retry_appropriate>true | false</retry_appropriate>
  <confidence>[high | medium | low]</confidence>
  <recommended_strategy>[Specific action if retry appropriate]</recommended_strategy>
  <alternative_strategies>
    <strategy>[Backup approach 1]</strategy>
    <strategy>[Backup approach 2]</strategy>
    <strategy>[Backup approach 3]</strategy>
  </alternative_strategies>
  <escalation_reason>[Why escalate if retry inappropriate]</escalation_reason>
</failure_analysis>
```

**This output feeds directly into retry-orchestration.md step 4 (select_alternative) or step 7 (escalate).**

## Configuration

Adaptive analysis controlled by `.planning/config.json`:

```json
{
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true
  }
}
```

**Fields:**
- `enabled`: Whether to use adaptive analysis (default: false for backward compatibility)
- `fallback_to_static`: Whether to fall back to static on AI failure (default: true)

## Examples

### Example 1: Flaky Test Analysis

**Input:**
```xml
<failure_input>
  <tool>Bash</tool>
  <command>npm test</command>
  <exit_code>1</exit_code>
  <stdout>
    Test Suites: 2 passed, 1 failed, 3 total
    Tests: 16 passed, 1 failed, 17 total

    FAIL src/api/user.test.js
      ● UserAPI › should return 404 for missing user
        expect(received).toBe(expected)
        Expected: 404
        Received: 500

        Timeout: test exceeded 5000ms
  </stdout>
  <task_context>Run test suite to verify API implementation</task_context>
  <attempt_number>1</attempt_number>
</failure_input>
```

**AI Analysis:**
```markdown
### Classification
Validation

### Root Cause
Test "should return 404 for missing user" in src/api/user.test.js failing due to timeout (exceeded 5000ms) and wrong status code (500 instead of 404). This suggests both a performance issue and incorrect error handling in the API endpoint.

### Retry Appropriate
Yes - This appears to be a test flakiness issue combined with an implementation bug. The timeout suggests the test may be slow/flaky, and the 500 error suggests error handling needs fixing.

### Confidence Level
high

### Recommended Recovery
1. Fix the API endpoint to return 404 for missing users (check error handling in user route)
2. Increase test timeout to 10000ms or isolate with --runInBand to prevent timeout
3. Re-run tests

### Alternative Approaches
1. Run tests with --runInBand and --maxWorkers=1 to isolate and prevent timing issues
2. Check if API server is properly mocked - 500 error might indicate connection failure
3. Review error handling logic in user API endpoint for missing user case
```

**Output:**
```xml
<failure_analysis>
  <method>adaptive</method>
  <category>Validation</category>
  <root_cause>Test failing due to API returning 500 instead of 404 for missing user, plus test timeout exceeded 5000ms</root_cause>
  <retry_appropriate>true</retry_appropriate>
  <confidence>high</confidence>
  <recommended_strategy>Fix error handling in user API to return 404 for missing users, increase test timeout to 10000ms</recommended_strategy>
  <alternative_strategies>
    <strategy>Run tests with --runInBand --maxWorkers=1 to isolate timing issues</strategy>
    <strategy>Verify API server mocking setup - 500 may indicate connection issue</strategy>
    <strategy>Review and fix error handling in user endpoint</strategy>
  </alternative_strategies>
</failure_analysis>
```

---

### Example 2: Dependency with Context

**Input:**
```xml
<failure_input>
  <tool>Bash</tool>
  <command>npm run build</command>
  <exit_code>1</exit_code>
  <stderr>
    Error: Cannot find module '@aws-sdk/client-s3'
    Require stack:
    - /app/src/services/storage.js
    - /app/src/index.js
  </stderr>
  <task_context>Build application for deployment</task_context>
  <attempt_number>1</attempt_number>
  <prior_attempts>[]</prior_attempts>
</failure_input>
```

**AI Analysis:**
```markdown
### Classification
Dependency

### Root Cause
Missing @aws-sdk/client-s3 module required by src/services/storage.js. This AWS SDK package is not installed in node_modules.

### Retry Appropriate
Yes - This is a straightforward missing dependency that can be auto-installed.

### Confidence Level
high

### Recommended Recovery
Install the missing package: npm install @aws-sdk/client-s3

### Alternative Approaches
1. Check if it's listed in package.json but not installed - run npm install
2. Verify correct package name (might be @aws-sdk/s3-client vs client-s3)
3. If package.json already has it, try deleting node_modules and running npm install
```

**Output:**
```xml
<failure_analysis>
  <method>adaptive</method>
  <category>Dependency</category>
  <root_cause>Missing @aws-sdk/client-s3 package required by src/services/storage.js</root_cause>
  <retry_appropriate>true</retry_appropriate>
  <confidence>high</confidence>
  <recommended_strategy>npm install @aws-sdk/client-s3</recommended_strategy>
  <alternative_strategies>
    <strategy>Run npm install if package.json already lists dependency</strategy>
    <strategy>Verify package name is correct (@aws-sdk/client-s3)</strategy>
    <strategy>Delete node_modules and run fresh npm install</strategy>
  </alternative_strategies>
</failure_analysis>
```

---

### Example 3: Complex Logic Error (Escalation)

**Input:**
```xml
<failure_input>
  <tool>Bash</tool>
  <command>npm test</command>
  <exit_code>1</exit_code>
  <stdout>
    Test Suites: 0 passed, 1 failed, 1 total
    Tests: 0 passed, 5 failed, 5 total

    FAIL src/auth/oauth.test.js
      All 5 tests failing with "OAuth token validation returns null"
  </stdout>
  <task_context>Implement OAuth 2.0 login flow with Google provider</task_context>
  <attempt_number>2</attempt_number>
  <prior_attempts>
    <attempt>
      <number>1</number>
      <strategy>Fix async/await handling in OAuth callback</strategy>
      <outcome>All tests still failing</outcome>
    </attempt>
  </prior_attempts>
</failure_input>
```

**AI Analysis:**
```markdown
### Classification
Logic

### Root Cause
OAuth implementation consistently failing all 5 tests after attempting async/await fix. All tests report "OAuth token validation returns null", suggesting fundamental implementation issue with token exchange or validation logic. Two attempts with different approaches both failed.

### Retry Appropriate
No - After 2 attempts, with fundamentally different approaches both failing, this indicates either a misunderstanding of OAuth requirements or missing prerequisite information (API credentials, callback URLs, etc.)

### Confidence Level
high

### Recommended Recovery
N/A - Escalation required

### Alternative Approaches
N/A

**Escalation Reason:**
Multiple implementation approaches have failed. This suggests:
1. OAuth provider setup may be incomplete (missing credentials, wrong callback URL)
2. Requirements may be misunderstood (which OAuth flow? implicit vs authorization code?)
3. Test expectations may be incorrect
4. Fundamental architectural decision needed about authentication approach

Recommend converting to checkpoint:decision to:
- Verify OAuth provider is properly configured
- Confirm which OAuth flow is needed (authorization code, implicit, etc.)
- Check if test credentials/mocks are set up correctly
- Consider if OAuth is the right approach vs simpler auth
```

**Output:**
```xml
<failure_analysis>
  <method>adaptive</method>
  <category>Logic</category>
  <root_cause>OAuth implementation fails all 5 tests after 2 different fix attempts - token validation returns null consistently</root_cause>
  <retry_appropriate>false</retry_appropriate>
  <confidence>high</confidence>
  <escalation_reason>Multiple implementation approaches failed. Likely missing OAuth provider configuration, unclear requirements about which OAuth flow to use, or test setup issues. Recommend user clarification on OAuth setup and flow requirements.</escalation_reason>
</failure_analysis>
```

---

### Example 4: Malformed AI Response (Fallback)

**Input:**
```xml
<failure_input>
  <tool>Edit</tool>
  <command>Edit old_string="const foo = 'bar'" new_string="const foo = 'baz'"</command>
  <stderr>Error: old_string not found in file</stderr>
  <task_context>Update configuration value</task_context>
</failure_input>
```

**AI Response (malformed):**
```markdown
This looks like an edit error. The string might not exist in the file or might be formatted differently. You should try reading the file first.
```

**Validation:**
```
❌ Missing "Classification" section
❌ Missing "Root Cause" section
❌ Missing "Retry Appropriate" section
❌ Response is conversational, not structured

→ FALLBACK TO STATIC ANALYSIS
```

**Static Analysis (from failure-analysis.md):**
```xml
<failure_analysis>
  <method>static</method>
  <category>Tool</category>
  <root_cause>Edit tool old_string not found in file - string may not exist or formatting differs</root_cause>
  <retry_appropriate>true</retry_appropriate>
  <confidence>high</confidence>
  <recommended_strategy>Read file first to verify exact string format, expand old_string with more context</recommended_strategy>
  <alternative_strategies>
    <strategy>Use grep to find similar strings in file</strategy>
    <strategy>Use Write to replace entire file if small</strategy>
  </alternative_strategies>
</failure_analysis>
```

**Note:** User never knows fallback occurred - output format is identical.

## Integration

**Called by:**
- `retry-orchestration.md` step 3 (analyze_failure) when `config.adaptive.enabled === true`

**Depends on:**
- `@get-shit-done/lib/failure-detection.md` - Provides pattern examples for AI
- `@get-shit-done/lib/failure-taxonomy.md` - Provides category definitions
- `@get-shit-done/lib/failure-analysis.md` - Fallback static analysis
- Task tool with general-purpose subagent (from Phase 3)

**Outputs to:**
- retry-orchestration.md step 4 (select_alternative) if retry appropriate
- retry-orchestration.md step 7 (escalate) if retry inappropriate

## Success Indicators

Adaptive analysis working correctly when:
- AI provides more nuanced diagnostics than static decision trees
- Complex failures get better root cause explanations
- Recovery strategies are context-aware
- Malformed AI responses fall back gracefully to static analysis
- Output format matches retry-orchestration.md expectations
- No breaking of retry workflow due to AI failures

---

*AI-powered failure analysis with static fallback for GSD v3.0 Intelligence*
