# Adaptive Path Selection Workflow

AI-powered alternative strategy selector using Claude to choose optimal recovery approaches based on execution context, not just fixed rules.

## Purpose

This workflow provides intelligent strategy selection that goes beyond hardcoded priority algorithms:

**Static selection (path-selection.md):**
- Fixed 8-step algorithm with predefined priority rules
- Sort by difficulty and risk (simple+safe first)
- Cannot consider execution history or failure nuances
- Same strategy selected for identical failure patterns

**Adaptive selection (this workflow):**
- AI-powered reasoning about strategy effectiveness
- Considers failure context and past attempt patterns
- Adapts recommendations to specific task situations
- Can prioritize strategies based on root cause analysis
- Falls back to static algorithm if AI selection fails

**When to use:**
- Enabled via `.planning/config.json` with `config.adaptive.enabled === true`
- Invoked from retry-orchestration.md step 4 (select_alternative)
- Provides smarter strategy selection than fixed priority rules

## Input Specification

<input_requirements>
  <required>
    <field name="failure_category">Category from failure analysis: Tool | Validation | Dependency | Logic | Resource</field>
    <field name="task_type">Type of task that failed: bash-command | file-edit | test-run | build-task | etc.</field>
    <field name="attempt_number">Current attempt count (1, 2, or 3)</field>
  </required>

  <optional>
    <field name="prior_attempts">List of strategy names already tried with outcomes</field>
    <field name="failure_details">Specific error context and root cause from failure analysis</field>
    <field name="task_context">What the task was trying to accomplish</field>
    <field name="recommended_strategy">Strategy recommendation from adaptive failure analysis (if available)</field>
  </optional>
</input_requirements>

## Process

<step name="prepare_selection_context" number="1">

**Assemble comprehensive strategy selection context:**

Build context for AI analysis including available strategies, failure patterns, and constraints.

**1. Load and filter available strategies from ALTERNATIVE_PATHS.md:**

Pre-filter strategies before sending to AI to avoid context bloat:

```markdown
## Available Strategies

Apply initial filtering (same as static algorithm steps 2-4):

1. Filter by task-type:
   - Keep strategies where <task-type> matches input task_type OR is "any"

2. Filter by failure-category:
   - Keep strategies where <failure-category> matches failure_category OR is "any"

3. Exclude already-tried:
   - Remove strategies where <strategy-name> appears in prior_attempts

4. Limit to reasonable set:
   - If >10 strategies remain after filtering, prioritize by:
     * Strategies matching specific task-type over "any"
     * Strategies matching specific failure-category over "any"
     * Take top 10 to avoid context bloat
```

**2. Format available strategies for AI:**

For each filtered strategy, include:
```markdown
### [strategy-name]
**Difficulty:** [simple | moderate | complex]
**Risk:** [safe | moderate | risky]

**Approach:**
[approach description]

**When to use:**
[when-to-use guidance]

**Success indicators:**
[success-indicators]

**Prerequisites:**
[prerequisites or "None"]
```

**3. Build failure context:**

```markdown
## Failure Context

**Task Type:** [task_type]
**Failure Category:** [failure_category]
**Attempt Number:** [attempt_number] of 3

**What failed:**
[task_context - what the task was trying to accomplish]

**Root Cause:**
[failure_details - specific error and analysis]
```

**4. Add attempt history (if available):**

```markdown
## Prior Attempts

[If prior_attempts provided, for each:]
**Attempt [N]:**
- Strategy: [strategy name]
- Approach: [what was tried]
- Outcome: [why it failed or what happened]
```

**5. Add risk tolerance guidance:**

```markdown
## Risk Tolerance for Attempt [N]

Attempt 1 (first retry):
- Strongly prefer simple + safe strategies
- Avoid risky or complex approaches
- Goal: Quick fix with minimal side effects

Attempt 2 (second retry):
- Allow moderate difficulty and risk
- Can try more investigative approaches
- Goal: Deeper diagnosis or alternative method

Attempt 3 (final retry):
- Allow any difficulty and risk level
- Last chance before escalation
- Goal: Exhaust all reasonable options
```

**6. Build selection prompt:**

```markdown
## Selection Task

Analyze the available strategies and select the BEST one for this failure context.

Consider:
1. **Root cause match:** Does strategy address the specific failure cause?
2. **Attempt appropriateness:** Does strategy fit risk tolerance for attempt [N]?
3. **Prior attempts:** Have similar approaches already failed? If so, avoid redundant strategies
4. **Success probability:** Which strategy has highest chance of resolving this specific issue?
5. **Failure analysis recommendation:** If adaptive failure analysis recommended a strategy, strongly consider it

**IMPORTANT:**
- You must select ONE strategy from the "Available Strategies" section above
- Do NOT invent new strategies - only choose from the provided list
- If recommended_strategy from failure analysis matches an available strategy, strongly consider it
- Consider why prior attempts failed when selecting next approach

**Output format (structured markdown):**

### Selected Strategy
[Exact strategy-name from Available Strategies section]

### Rationale
[Why this strategy is best for this specific failure - 2-3 sentences]

### Confidence Level
[high | medium | low]

### Fallback Consideration
[If this fails, what should be tried next - strategy name or "escalate"]
```

**Assembled prompt saved to variable:** `selection_prompt`

</step>

<step name="invoke_ai_selector" number="2">

**Use Task tool to select optimal strategy:**

**Pattern from adaptive-failure-analysis.md step 2:**

Invoke general-purpose agent with comprehensive context:

```javascript
// Pseudocode for invocation pattern
Task.create({
  subagent_type: "general-purpose",
  instruction: selection_prompt,  // From step 1
  context: [
    "@.planning/ALTERNATIVE_PATHS.md",  // Full library for reference
    "@get-shit-done/lib/failure-taxonomy.md"  // Category understanding
  ],
  timeout: 30000  // 30 second timeout for selection
})
```

**Actual invocation:**

Use Task tool with:
- Subagent type: general-purpose
- Instruction: Full selection prompt from step 1 (includes filtered strategies)
- Context: Load ALTERNATIVE_PATHS.md and failure-taxonomy.md for reference
- Expected output: Structured markdown response with selected strategy

**Wait for response:**
- AI analyzes failure context and available strategies
- Considers root cause, prior attempts, and risk tolerance
- Selects most promising strategy with rationale
- Provides confidence level and fallback consideration

**Capture output:** Store AI response in `ai_selection_result`

</step>

<step name="validate_selection" number="3">

**Ensure AI returned valid, actionable strategy selection:**

**Parse AI response for required sections:**

1. **Check for "Selected Strategy" section:**
   - Extract strategy name
   - Verify it matches one of the strategy names from Available Strategies (exact match)
   - If invalid or missing → mark as malformed

2. **Check for "Rationale" section:**
   - Verify it's specific (explains why this strategy fits this failure)
   - Not generic ("this should work", "good option")
   - If too generic or missing → mark as malformed

3. **Check for "Confidence Level" section:**
   - Extract high/medium/low
   - If missing → default to "medium"

4. **Retrieve full strategy details:**
   - Load selected strategy from ALTERNATIVE_PATHS.md
   - Extract: approach, when-to-use, success-indicators, prerequisites, difficulty, risk
   - Validate strategy exists in library

**Validation decision:**

```markdown
If selected strategy is valid AND exists in ALTERNATIVE_PATHS.md AND rationale is specific:
  → Use AI selection (proceed to output)

If any validation fails OR response is malformed OR strategy doesn't exist in library:
  → Fall back to static algorithm:
     1. Load @get-shit-done/lib/path-selection.md
     2. Run static 8-step selection algorithm with same inputs
     3. Use static selection output instead
     4. Log warning: "Adaptive selection failed validation, using static fallback"
```

**Fallback handling:**
- Adaptive selection is best-effort enhancement
- Static algorithm is always available as reliable fallback
- Never block retry workflow due to AI selection failure
- Log which selection method was used for debugging

**Error cases handled:**
- AI timeout → fallback to static
- Malformed response → fallback to static
- Invalid strategy name → fallback to static
- Strategy not in library → fallback to static
- Missing sections → fallback to static
- Task tool error → fallback to static

</step>

## Output Specification

<output_format>
  <field name="selection_method">adaptive | static</field>
  <field name="selected_strategy">The chosen strategy-name from ALTERNATIVE_PATHS.md</field>
  <field name="approach">Full approach description from the selected strategy</field>
  <field name="when_to_use">When-to-use guidance from selected strategy</field>
  <field name="success_indicators">Expected success-indicators for this strategy</field>
  <field name="difficulty">simple | moderate | complex</field>
  <field name="risk">safe | moderate | risky</field>
  <field name="prerequisites">Any prerequisites from strategy</field>
  <field name="confidence">high | medium | low (from AI or "medium" for static)</field>
  <field name="rationale">Why this strategy was selected (from AI or "static algorithm priority" for static)</field>
  <field name="available">true | false (false if no strategies available, triggers escalation)</field>
</output_format>

**Output structure (compatible with retry-orchestration.md expectations):**

```xml
<strategy_selection>
  <method>adaptive | static</method>
  <available>true | false</available>
  <strategy_name>[strategy name from ALTERNATIVE_PATHS.md]</strategy_name>
  <approach>[Full approach description]</approach>
  <when_to_use>[When to use guidance]</when_to_use>
  <success_indicators>[Success indicators]</success_indicators>
  <difficulty>[simple | moderate | complex]</difficulty>
  <risk>[safe | moderate | risky]</risk>
  <prerequisites>[Prerequisites or "None"]</prerequisites>
  <confidence>[high | medium | low]</confidence>
  <rationale>[Why selected]</rationale>
</strategy_selection>
```

**This output feeds directly into retry-orchestration.md step 5 (execute_alternative).**

**If available=false:** No applicable strategies remain, triggers escalation in step 7.

## Configuration

Adaptive selection controlled by `.planning/config.json`:

```json
{
  "adaptive": {
    "enabled": true,
    "fallback_to_static": true
  }
}
```

**Fields:**
- `enabled`: Whether to use adaptive selection (default: false for backward compatibility)
- `fallback_to_static`: Whether to fall back to static on AI failure (default: true)

## Examples

### Example 1: Test Failure with Flaky Tests

**Input:**
```xml
<selection_input>
  <failure_category>Validation</failure_category>
  <task_type>test-run</task_type>
  <attempt_number>1</attempt_number>
  <prior_attempts>[]</prior_attempts>
  <failure_details>
    Test "should return 404 for missing user" failing due to timeout (exceeded 5000ms) and wrong status code (500 instead of 404). Suggests test flakiness and error handling bug.
  </failure_details>
  <task_context>Run test suite to verify API implementation</task_context>
  <recommended_strategy>Run tests with --runInBand to isolate timing issues</recommended_strategy>
</selection_input>
```

**Available Strategies (after filtering):**
```markdown
### retry-with-reruns
**Difficulty:** simple
**Risk:** safe
**Approach:** Run with jest --maxWorkers=1 --runInBand to isolate tests

### increase-test-timeout
**Difficulty:** simple
**Risk:** safe
**Approach:** Increase timeout to 10000ms or higher

### run-single-test-isolation
**Difficulty:** simple
**Risk:** safe
**Approach:** Run only the failing test in isolation to debug
```

**AI Selection:**
```markdown
### Selected Strategy
retry-with-reruns

### Rationale
The failure shows both timeout and wrong status code, indicating test interference and possible race condition. Running with --runInBand isolates tests serially, preventing timing conflicts. This matches the recommended strategy from failure analysis and is safe for first attempt.

### Confidence Level
high

### Fallback Consideration
If this fails, try increase-test-timeout to rule out timing issues
```

**Output:**
```xml
<strategy_selection>
  <method>adaptive</method>
  <available>true</available>
  <strategy_name>retry-with-reruns</strategy_name>
  <approach>Run with jest --maxWorkers=1 --runInBand to isolate tests</approach>
  <when_to_use>Flaky tests with timing or race condition issues</when_to_use>
  <success_indicators>Tests pass consistently when run in isolation</success_indicators>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
  <prerequisites>None</prerequisites>
  <confidence>high</confidence>
  <rationale>Failure shows timeout and wrong status code indicating test interference. Running serially with --runInBand prevents race conditions and matches failure analysis recommendation.</rationale>
</strategy_selection>
```

---

### Example 2: Build Failure After Multiple Attempts

**Input:**
```xml
<selection_input>
  <failure_category>Dependency</failure_category>
  <task_type>build-task</task_type>
  <attempt_number>2</attempt_number>
  <prior_attempts>
    <attempt>
      <strategy>reinstall-dependencies</strategy>
      <outcome>Dependencies reinstalled but build still fails with "Cannot find module 'lodash'"</outcome>
    </attempt>
  </prior_attempts>
  <failure_details>Missing @types/lodash TypeScript definitions after npm install lodash</failure_details>
  <task_context>Build application for deployment</task_context>
</selection_input>
```

**Available Strategies (after filtering):**
```markdown
### explicit-install-missing
**Difficulty:** simple
**Risk:** safe
**Approach:** npm install [missing-package] to explicitly add dependency

### check-typescript-config
**Difficulty:** moderate
**Risk:** safe
**Approach:** Verify tsconfig.json allows JS modules and has correct module resolution

### install-type-definitions
**Difficulty:** simple
**Risk:** safe
**Approach:** Install @types packages for TypeScript type definitions
```

**AI Selection:**
```markdown
### Selected Strategy
install-type-definitions

### Rationale
The failure explicitly mentions missing @types/lodash after lodash was installed. Prior attempt installed the runtime package but not TypeScript definitions. This is a common TypeScript dependency issue where both the package and its type definitions are needed.

### Confidence Level
high

### Fallback Consideration
If this fails, check-typescript-config to verify module resolution settings
```

**Output:**
```xml
<strategy_selection>
  <method>adaptive</method>
  <available>true</available>
  <strategy_name>install-type-definitions</strategy_name>
  <approach>Install @types packages for TypeScript type definitions (npm install --save-dev @types/lodash)</approach>
  <when_to_use>TypeScript build fails with missing type definitions after package installed</when_to_use>
  <success_indicators>Build completes without type errors, @types package in node_modules</success_indicators>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
  <prerequisites>TypeScript project with type checking enabled</prerequisites>
  <confidence>high</confidence>
  <rationale>Failure explicitly mentions missing @types/lodash after lodash installed. Prior attempt added runtime package but not type definitions, which is common TypeScript dependency pattern.</rationale>
</strategy_selection>
```

---

### Example 3: Malformed AI Response (Fallback)

**Input:**
```xml
<selection_input>
  <failure_category>Tool</failure_category>
  <task_type>file-edit</task_type>
  <attempt_number>1</attempt_number>
  <failure_details>Edit tool old_string not found in file</failure_details>
</selection_input>
```

**AI Response (malformed):**
```markdown
I think you should try reading the file first to see what's actually in it. That would help you find the right string to match.
```

**Validation:**
```
❌ Missing "Selected Strategy" section
❌ Missing "Rationale" section
❌ Missing "Confidence Level" section
❌ Response is conversational, not structured

→ FALLBACK TO STATIC ALGORITHM
```

**Static Algorithm Output:**
```xml
<strategy_selection>
  <method>static</method>
  <available>true</available>
  <strategy_name>read-before-edit</strategy_name>
  <approach>Read file first to verify exact string format and context, then use expanded old_string with surrounding lines</approach>
  <when_to_use>Edit fails because old_string doesn't match file contents exactly</when_to_use>
  <success_indicators>File contains the expected content after edit, no "not found" errors</success_indicators>
  <difficulty>simple</difficulty>
  <risk>safe</risk>
  <prerequisites>None</prerequisites>
  <confidence>medium</confidence>
  <rationale>Static algorithm priority: simple + safe strategy for Tool failure</rationale>
</strategy_selection>
```

**Note:** User never knows fallback occurred - output format is identical.

---

### Example 4: No Strategies Available (Escalation)

**Input:**
```xml
<selection_input>
  <failure_category>Resource</failure_category>
  <task_type>deployment</task_type>
  <attempt_number>1</attempt_number>
  <failure_details>SSH connection to production server timed out</failure_details>
  <prior_attempts>[]</prior_attempts>
</selection_input>
```

**After Step 1 Filtering:**
```
- Filter by task-type="deployment" → 2 strategies
- Filter by failure-category="Resource" → 0 strategies (no deployment+resource matches)
- Try fallback: task-type="any" + failure-category="Resource" → 3 strategies
- Check task criticality: deployment is high-stakes
```

**Decision:**
```
Even though fallback strategies exist, production deployment failures should escalate immediately for user awareness.
```

**Output:**
```xml
<strategy_selection>
  <method>adaptive</method>
  <available>false</available>
  <escalation_reason>Production deployment task with SSH timeout. Deployment failures should escalate to user for awareness and approval before retrying. Network issues could indicate broader infrastructure problems.</escalation_reason>
</strategy_selection>
```

## Integration

**Called by:**
- `retry-orchestration.md` step 4 (select_alternative) when `config.adaptive.enabled === true`

**Depends on:**
- `@.planning/ALTERNATIVE_PATHS.md` - Source of alternative strategies
- `@get-shit-done/lib/failure-taxonomy.md` - Category definitions for AI context
- `@get-shit-done/lib/path-selection.md` - Fallback static algorithm
- Task tool with general-purpose subagent (from Phase 3)

**Outputs to:**
- retry-orchestration.md step 5 (execute_alternative) if strategy selected
- retry-orchestration.md step 7 (escalate) if no strategies available

## Success Indicators

Adaptive selection working correctly when:
- AI selects strategies better suited to specific failure context than static priority
- Strategies match root cause from adaptive failure analysis
- Prior attempt patterns influence next strategy selection
- Malformed AI responses fall back gracefully to static algorithm
- Output format matches retry-orchestration.md expectations
- No breaking of retry workflow due to AI failures
- Selection considers execution history when available

---

*AI-powered strategy selection with static fallback for GSD v3.0 Intelligence*
