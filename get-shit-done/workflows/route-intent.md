# Workflow: Route Intent

<purpose>
Main entry point for natural language workflow routing.
Combines intent matching, variable extraction, and confidence-based routing decisions.
</purpose>

<input>
- user_input: Natural language request from user
</input>

<process>

<step name="load_templates">
## Step 1: Load Template Index

Load template definitions:
```bash
cat ~/.claude/get-shit-done/workflow-templates/template-index.json
```

Parse JSON to get templates array. The index contains 18 workflow templates organized by category.
</step>

<step name="match_intent">
## Step 2: Match Intent

Run intent-matcher workflow mentally:
- Input: user_input, templates
- Output: MatchResult[] (sorted by confidence)

**Matching Algorithm:**
1. Normalize user input (lowercase, remove fillers)
2. For each template, calculate similarity score against:
   - patterns (weight: 0.6)
   - examples (weight: 0.3)
   - id (weight: 0.1)
3. Filter matches above threshold (score <= 0.6)
4. Sort by score ascending (best first)

**If no matches above threshold:**
- Route: 'claude_fallback'
- Reason: 'no_template_match'
- Proceed to AI workflow generation
- Route to generate-workflow.md with:
  - user_request: original input
  - match_context: { partial_matches: [...] }
</step>

<step name="evaluate_confidence">
## Step 3: Evaluate Confidence

Evaluate best match confidence based on Fuse.js score:

**HIGH confidence (score <= 0.3, confidence >= 0.7):**
- Route: 'direct'
- Proceed immediately to variable extraction
- Execute without confirmation

**MEDIUM confidence (0.3 < score <= 0.5, confidence 0.5-0.7):**
- Route: 'confirm'
- Present to user: "Did you mean: [template.id]? [template.description]"
- Wait for user confirmation (yes/no)
- If confirmed: proceed to variable extraction
- If rejected: try next match or route to Claude fallback

**LOW confidence (score > 0.5, confidence < 0.5):**
- Route: 'claude_fallback'
- Reason: 'low_confidence'
- Route to generate-workflow.md with:
  - user_request: original input
  - match_context: {
      category_hint: best_match.category,
      similar_workflows: [top 3 partial matches]
    }
</step>

<step name="extract_variables">
## Step 4: Extract Variables

Run variable-extractor workflow:
- Input: user_input, matched_template
- Output: ExtractionResult

**Process each variable slot:**
1. Try regex patterns against user input
2. Apply type conversion (number, string, path)
3. Track missing required variables

**If missing_required not empty:**
- Route: 'prompt'
- For each missing variable:
  - Present: missing.prompt
  - Collect user response
  - Add to variables
- Re-validate all required present

**If all required present:**
- Proceed to workflow execution
</step>

<step name="execute_workflow">
## Step 5: Execute Workflow

Once template matched and variables extracted:

**Build execution context:**
```
Workflow: [template.workflow]
Variables: [extracted variables]
Confidence: [match confidence]
Method: [direct | confirm | prompt]
```

**Present to user:**
```
Matched: [template.id] - [template.description]
Variables: [list extracted variables]

Executing [template.workflow]...
```

**Execute the workflow:**
Load and execute the matched workflow file with extracted variables passed as context.
</step>

</process>

<routing_decision_tree>
## Decision Tree

```
User Input
    │
    ▼
┌─────────────────────────┐
│   Load Template Index   │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│     Match Intent        │
│   (intent-matcher.md)   │
└─────────────────────────┘
    │
    ├── No matches ──────────────────────────► generate-workflow.md
    │                                          AI workflow generation
    │                                          with partial match context
    ▼
┌─────────────────────────┐
│   Check Confidence      │
│   (best match score)    │
└─────────────────────────┘
    │
    ├── HIGH (score <= 0.3) ────► Extract Variables ────► Execute
    │   confidence >= 0.7              │
    │                                  │
    │                                  └── Missing required? ──► Prompt ──► Execute
    │
    ├── MEDIUM (0.3 < score <= 0.5) ──► Confirm with User
    │   confidence 0.5-0.7                    │
    │                                         ├── Yes ──► Extract ──► Execute
    │                                         │
    │                                         └── No ──► Try Next Match
    │                                                    or generate-workflow.md
    │
    └── LOW (score > 0.5) ────────────────────► generate-workflow.md
        confidence < 0.5                        AI workflow generation
                                               with category hint + similar workflows
```
</routing_decision_tree>

<output>
```typescript
interface RouteResult {
  route: 'direct' | 'confirm' | 'prompt' | 'claude_fallback';
  template?: WorkflowTemplate;
  variables?: Record<string, string | number>;
  confidence?: number;
  reason?: string;
  workflow_path?: string;
}
```

**Route values:**
- **direct**: High confidence, execute immediately
- **confirm**: Medium confidence, ask user to confirm
- **prompt**: Missing variables, prompt for values
- **claude_fallback**: No match or low confidence, defer to Claude
</output>

<confirmation_flow>
## Confirmation Flow (MEDIUM Confidence)

When confidence is MEDIUM (score 0.3-0.5):

**Present:**
```
I think you want to run: [template.id]
[template.description]

Is this correct? (yes/no)
```

**If user says yes:**
- Proceed to variable extraction
- Execute workflow

**If user says no:**
- Check for next best match
- If another match exists with score <= 0.6:
  - Present that option
- If no more matches:
  - Route to claude_fallback
  - "I'm not sure what workflow you want. Try: /gsd:help"
</confirmation_flow>

<prompting_flow>
## Variable Prompting Flow

When required variables are missing:

**For each missing variable:**
```
[variable.prompt]
> [user enters value]
```

**Example:**
```
User: "plan phase"
→ Matched: plan-phase (HIGH confidence)
→ Variables: phase_number missing (optional)

Since phase_number is optional, proceed without prompting.
The workflow will determine next unplanned phase.
```

```
User: "discuss the milestone"
→ Matched: discuss-milestone (HIGH confidence)
→ Variables: (none required)

Proceed directly to execution.
```

**Re-extraction after prompt:**
After collecting prompted values, merge with original extraction:
```javascript
variables = { ...extracted, ...prompted }
```
</prompting_flow>

<claude_fallback>
## Claude Fallback: AI Workflow Generation

When route is 'claude_fallback', invoke generate-workflow.md to create a new workflow from scratch.

**Fallback triggers:**
- No matches above threshold (score > 0.6)
- All matches below LOW confidence (score > 0.5)
- User rejects all MEDIUM confidence suggestions

**Context passed to generation:**
- user_request: The original natural language input
- match_context: Partial match info for better generation
  - category_hint: Best-guess category from partial matches
  - similar_workflows: IDs of partially matching workflows

**After generation:**
- User confirms or rejects generated workflow
- If confirmed: workflow executes
- If saved: added to generated-index.json for future matching

**Fallback reasons:**
- `no_template_match`: Input didn't match any template → generate fresh
- `low_confidence`: Best match score > 0.5 → generate with category hint
- `user_rejected`: User rejected confirmation → generate as alternative
- `ambiguous`: Multiple templates matched equally → generate with clarification

**Example flow:**
```
User: "analyze my test coverage"
→ No template match found
→ Route to generate-workflow.md with:
    user_request: "analyze my test coverage"
    match_context: { category_hint: "analysis" }
→ AI generates new "analyze-test-coverage" workflow
→ User reviews and confirms
→ Workflow executes
→ (Optional) User saves for future use
```
</claude_fallback>

<examples>
## Routing Examples

### Example 1: HIGH Confidence Direct

**Input:** "plan phase 5"
**Match:** plan-phase (score: 0.1)
**Route:** direct

```
Flow:
1. Match: plan-phase (confidence: 0.9)
2. Extract: { phase_number: 5 }
3. Execute: workflows/plan-phase.md with phase_number=5
```

### Example 2: MEDIUM Confidence Confirm

**Input:** "break down the third phase into smaller tasks"
**Match:** plan-phase (score: 0.4)
**Route:** confirm

```
Flow:
1. Match: plan-phase (confidence: 0.6)
2. Confirm: "I think you want to run: plan-phase - Create executable plan for a phase. Is this correct?"
3. User: "yes"
4. Extract: { phase_number: 3 }
5. Execute: workflows/plan-phase.md with phase_number=3
```

### Example 3: LOW Confidence Fallback

**Input:** "fix all the bugs"
**Match:** (none good, best score: 0.7)
**Route:** claude_fallback

```
Flow:
1. Match: No good matches
2. Response: "I couldn't match your request to a known workflow."
3. Suggest: "Try /gsd:help to see available commands"
```

### Example 4: Prompt for Missing

**Input:** "create a new milestone"
**Match:** new-milestone (score: 0.05)
**Route:** direct → prompt

```
Flow:
1. Match: new-milestone (confidence: 0.95)
2. Extract: { milestone_version: undefined }
3. Variable optional, proceed without prompting
4. Execute: workflows/new-milestone.md
   (Workflow will ask for version if needed)
```

### Example 5: User Rejection

**Input:** "analyze"
**Match:** analyze-codebase (score: 0.45), analyze-patterns (score: 0.48)
**Route:** confirm → user rejects → try next

```
Flow:
1. Match: analyze-codebase (confidence: 0.55)
2. Confirm: "I think you want to run: analyze-codebase?"
3. User: "no"
4. Try next: analyze-patterns (confidence: 0.52)
5. Confirm: "Did you mean: analyze-patterns?"
6. User: "yes"
7. Execute: workflows/analyze-patterns.md
```
</examples>

<integration>
## Integration Points

**Uses:**
- intent-matcher.md: Intent matching algorithm
- variable-extractor.md: Variable extraction from input
- template-index.json: Template definitions
- template-schema.md: Schema reference

**Invokes:**
- generate-workflow.md: On fallback (no match or low confidence)

**Called from:**
- User natural language input (future: auto-routing from chat)
- /gsd:route command (explicit routing request)

**Outputs to:**
- Matched workflow file (e.g., workflows/plan-phase.md)
- Workflow receives extracted variables as context
- On fallback: generate-workflow.md receives user_request + match_context
</integration>

<configuration>
## Configuration

Routing behavior can be configured in .planning/config.json:

```json
{
  "routing": {
    "enabled": true,
    "confidence_thresholds": {
      "high": 0.3,
      "medium": 0.5
    },
    "auto_confirm_high": true,
    "show_alternatives": 3
  }
}
```

**Options:**
- `enabled`: Enable/disable natural language routing
- `confidence_thresholds.high`: Score threshold for direct execution (default: 0.3)
- `confidence_thresholds.medium`: Score threshold for confirmation (default: 0.5)
- `auto_confirm_high`: Skip confirmation for HIGH confidence (default: true)
- `show_alternatives`: Number of alternatives to show on fallback (default: 3)
</configuration>
