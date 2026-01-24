# Workflow: Generate Workflow

<purpose>
Generate valid GSD workflows from natural language when template matching fails. Called by route-intent.md on fallback to create new workflows dynamically using AI generation with safety validation and human confirmation.

**Principle:** AI-generated workflows must pass safety validation before execution. Users always confirm before running generated workflows.
</purpose>

<input>
- user_request: Natural language request that didn't match templates
- match_context: Optional partial match info
  - category_hint: Suggested category from partial matches (planning, execution, analysis, project, monitoring)
  - similar_workflows: IDs of partially matching workflows for context
</input>

<process>

<step name="load_context" priority="first">
## Step 1: Load Generation Context

Load required context files:

```bash
# Load project description
if [ -f .planning/PROJECT.md ]; then
  head -50 .planning/PROJECT.md | grep -A 10 "## Overview"
fi

# Load existing workflow list
cat ~/.claude/get-shit-done/workflow-templates/template-index.json | grep '"id"'

# Check for generated workflows already saved
if [ -f ~/.claude/get-shit-done/workflow-templates/generated/generated-index.json ]; then
  cat ~/.claude/get-shit-done/workflow-templates/generated/generated-index.json
fi
```

**Context assembly:**
- PROJECT.md overview (first 200 chars) → {project_description}
- Workflow IDs from template-index.json → {workflow_list}
- Category hint from match_context → {category_hint} (default: "analysis")

**Load references:**
- `@~/.claude/get-shit-done/templates/generation-prompt.md` - Prompt template
- `@~/.claude/get-shit-done/references/workflow-safety.md` - Safety validation rules
- `@~/.claude/get-shit-done/references/workflow-schema.json` - Output schema
</step>

<step name="select_examples">
## Step 2: Select Few-Shot Examples

Based on category hint, select 2 example workflows:

**Category-based selection:**

| Category | Example 1 | Example 2 |
|----------|-----------|-----------|
| planning | plan-phase.md | discuss-phase.md |
| execution | execute-plan.md | resume-work.md |
| analysis | analyze-codebase.md | suggest.md |
| project | new-project.md | complete-milestone.md |
| monitoring | progress.md | health-check.md |
| default | progress.md | plan-phase.md |

**Load example workflows:**

```bash
# Example: If category is "analysis"
EXAMPLE1_PATH=~/.claude/get-shit-done/workflows/analyze-codebase.md
EXAMPLE2_PATH=~/.claude/get-shit-done/workflows/suggest.md

# Extract key sections (truncate if very long)
head -100 "$EXAMPLE1_PATH"
head -100 "$EXAMPLE2_PATH"
```

**Truncation rule:**
If workflow > 100 lines, include:
- Full `<purpose>` section
- First 2-3 steps of `<process>`
- Full `<success_criteria>` section
- Add: `<!-- ... additional steps omitted for brevity ... -->`
</step>

<step name="build_prompt">
## Step 3: Build Generation Prompt

Fill generation-prompt.md template with collected context:

**Placeholders to fill:**
```
{user_request} ← from input (the natural language request)
{project_description} ← from PROJECT.md overview
{workflow_list} ← comma-separated IDs from template-index.json
{category_hint} ← from match_context or "analysis" default
{example1_id} ← selected example 1 filename
{example1_content} ← content from example 1 (truncated if needed)
{example2_id} ← selected example 2 filename
{example2_content} ← content from example 2 (truncated if needed)
```

**Validation before generation:**
- [ ] {user_request} is non-empty
- [ ] {workflow_list} contains at least 5 workflow IDs
- [ ] {example1_content} has `<purpose>` and `<process>` sections
- [ ] {example2_content} has `<purpose>` and `<process>` sections
</step>

<step name="generate_workflow">
## Step 4: Generate Workflow

Use Claude to generate workflow JSON following the schema.

**Generation process:**
1. Use filled generation-prompt.md as the prompt
2. Request structured output matching workflow-schema.json
3. Claude generates JSON with:
   - id: unique kebab-case identifier
   - description: one-line description
   - category: planning|execution|analysis|project|monitoring
   - patterns: 3-5 intent phrases
   - examples: 4-6 full utterances
   - variables: any required parameters (optional array)
   - workflow_content: full markdown workflow

**Example generated output:**
```json
{
  "id": "analyze-test-coverage",
  "description": "Analyze test coverage and identify files needing more tests",
  "category": "analysis",
  "patterns": [
    "analyze test coverage",
    "check test coverage",
    "coverage report",
    "which files need tests"
  ],
  "examples": [
    "analyze my test coverage",
    "show me which files need more tests",
    "check the test coverage of this project",
    "what files have low test coverage"
  ],
  "variables": [],
  "workflow_content": "<purpose>...</purpose><process>...</process><success_criteria>...</success_criteria>"
}
```

**If generation fails:**
- Log error
- Fall back to "I couldn't generate a workflow. Try describing what you want differently."
- Exit workflow
</step>

<step name="validate_workflow">
## Step 5: Validate Generated Workflow

Apply safety validation using workflow-safety.md rules.

**Validation sequence:**

1. **Schema validation:**
   - id matches pattern `^[a-z][a-z0-9-]*$`
   - description length 10-200 chars
   - category is valid enum
   - patterns has 3-8 items
   - examples has 4-10 items
   - workflow_content has required sections

2. **ID uniqueness:**
   - Check id not in template-index.json
   - Check id not in generated-index.json (if exists)

3. **Pattern conflict:**
   - No pattern should be exact match of existing workflow patterns
   - Warn if high similarity to existing patterns

4. **Workflow content structure:**
   - Must have `<purpose>` section
   - Must have `<process>` section with at least one `<step>`
   - Should have `<success_criteria>` section

5. **Safety validation (CRITICAL):**
   Load blocklist patterns from workflow-safety.md

   ```
   FOR each blocklist pattern:
     IF regex matches workflow_content:
       issues.append({level: "BLOCKED", pattern, description})
   ```

   Load allowlist commands from workflow-safety.md

   ```
   FOR each bash block in workflow_content:
     FOR each command:
       IF command not in allowlist:
         issues.append({level: "WARNING", command})
   ```

**Validation results:**
- **BLOCKED:** Dangerous pattern found - refuse to execute
- **WARNING:** Unrecognized commands - require user acknowledgment
- **SAFE:** All checks passed - safe to execute

**Build issues list:**
```
validation_result = {
  status: "BLOCKED" | "WARNING" | "SAFE",
  issues: [
    {level: "BLOCKED", pattern: "...", description: "..."},
    {level: "WARNING", command: "docker", line: 23}
  ]
}
```
</step>

<step name="present_to_user">
## Step 6: Present to User

Present generated workflow for human confirmation:

**If BLOCKED:**
```
BLOCKED: Cannot execute generated workflow

Dangerous patterns detected:
- [Line X]: [pattern] - [description]
- [Line Y]: [pattern] - [description]

This workflow cannot be executed. Please try a different request or use built-in workflows.

See /gsd:help for available commands.
```
Exit workflow.

**If WARNING or SAFE:**
```
Generated workflow for: "{user_request}"

ID: {id}
Category: {category}
Description: {description}

Intent patterns:
- {pattern 1}
- {pattern 2}
- {pattern 3}

[If WARNING:]
Validation warnings:
- [command] is not on allowlist (line X)

Options:
1. Execute this workflow now
2. View full workflow content
3. Save to generated templates (for future reuse)
4. Edit workflow before executing
5. Cancel

What would you like to do?
```

**Wait for user response.**
</step>

<step name="handle_response">
## Step 7: Handle User Response

Based on user choice:

**Option 1: Execute**
- If validation_status == "WARNING", confirm user acknowledges
- Execute the workflow_content directly
- Pass any extracted variables from original request
- Report execution result

**Option 2: View**
- Display full workflow_content in code block
- Return to options menu

**Option 3: Save**
- Call save_generated_workflow (see below)
- Confirm save location
- Return to options with option to execute

**Option 4: Edit**
- Display current workflow_content
- Accept user modifications
- Re-run validation on modified content
- Return to options

**Option 5: Cancel**
```
Cancelled workflow generation.

To try a different approach:
- /gsd:help - See available commands
- Describe your goal differently
```
Exit workflow.

**Execute workflow content:**

When executing, the workflow_content becomes the active workflow:
1. Parse <purpose>, <process>, <success_criteria> sections
2. Execute each <step> in order
3. Track success criteria
4. Report completion
</step>

<step name="save_generated_workflow">
## Step 8: Save Generated Workflow (Optional)

If user chooses to save:

```bash
# Ensure generated directory exists
mkdir -p ~/.claude/get-shit-done/workflow-templates/generated

# Load or create generated-index.json
if [ ! -f ~/.claude/get-shit-done/workflow-templates/generated/generated-index.json ]; then
  echo '{"version":"1.0","generated_workflows":[],"metadata":{"created":"'$(date -u +"%Y-%m-%d")'","last_updated":"'$(date -u +"%Y-%m-%d")'","workflow_count":0}}' > ~/.claude/get-shit-done/workflow-templates/generated/generated-index.json
fi
```

**Add to generated-index.json:**
```json
{
  "id": "{generated.id}",
  "workflow": "generated/{generated.id}.md",
  "category": "{generated.category}",
  "description": "{generated.description}",
  "patterns": [...],
  "examples": [...],
  "variables": [...],
  "generated_at": "2026-01-08T12:00:00Z",
  "original_request": "{user_request}",
  "approved_by": "user"
}
```

**Write workflow file:**
```bash
# Write workflow content to file
cat > ~/.claude/get-shit-done/workflow-templates/generated/{id}.md << 'EOF'
{workflow_content}
EOF
```

**Update metadata:**
```bash
# Update workflow_count and last_updated in generated-index.json
```

**Confirm:**
```
Saved workflow to:
- File: ~/.claude/get-shit-done/workflow-templates/generated/{id}.md
- Index: generated-index.json

This workflow will be available for future matching.
Note: Generated workflows are separate from built-in templates.
```
</step>

</process>

<success_criteria>
- [ ] Workflow generated from user request
- [ ] Safety validation completed (BLOCKED/WARNING/SAFE)
- [ ] User presented with options and made choice
- [ ] If executed: workflow ran successfully
- [ ] If saved: workflow added to generated-index.json
</success_criteria>

<error_handling>
## Error Handling

**Generation failure:**
- AI couldn't generate valid JSON
- Response: "I couldn't generate a workflow. Try describing what you want differently."

**Validation failure (BLOCKED):**
- Dangerous patterns detected
- Response: Show blocked patterns, refuse execution, suggest alternatives

**Schema validation failure:**
- Generated JSON doesn't match schema
- Response: "Generated workflow has invalid structure. Trying again..."
- Retry once, then fall back to error message

**Save failure:**
- Can't write to generated directory
- Response: "Couldn't save workflow. You can still execute it now."
- Offer to execute without saving
</error_handling>

<integration>
## Integration Points

**Called from:**
- route-intent.md: On fallback (no match or low confidence)
- Direct invocation (future): /gsd:generate command

**Uses:**
- workflow-schema.json: Output schema validation
- generation-prompt.md: Prompt template
- workflow-safety.md: Safety validation rules
- template-index.json: Existing workflow list for uniqueness check

**Outputs:**
- Executed workflow (if user chooses execute)
- Generated workflow file (if user chooses save)
- generated-index.json entry (if user chooses save)
</integration>
