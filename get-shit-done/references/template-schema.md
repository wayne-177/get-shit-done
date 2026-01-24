# Template Schema

<overview>
Defines the JSON schema for workflow templates used in natural language intent matching. Templates enable GSD to understand user requests like "plan phase 5" or "what should I work on next" and route them to the appropriate workflow.
</overview>

<schema>
## WorkflowTemplate

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string | yes | Unique identifier (kebab-case, matches workflow name) |
| workflow | string | yes | Path to workflow file (relative to ~/.claude/get-shit-done/) |
| category | string | yes | Category for grouping (planning, execution, analysis, project, monitoring) |
| patterns | string[] | yes | Intent phrases to match (e.g., "plan phase", "break down phase") |
| variables | VariableSlot[] | no | Variables to extract from input |
| examples | string[] | yes | Full example utterances for fuzzy matching |
| description | string | yes | One-line description shown when matched |

## VariableSlot

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name | string | yes | Variable name (snake_case) |
| required | boolean | yes | Whether workflow fails without this |
| patterns | string[] | yes | Regex patterns to extract value (with capture group) |
| prompt | string | yes | Question to ask if not extracted |
| type | string | no | Expected type: number, string, path (default: string) |

</schema>

<matching_rules>
## Matching Configuration

Fuse.js options for template matching:

```javascript
{
  threshold: 0.6,           // Default, configurable via config.json
  keys: [
    { name: 'patterns', weight: 0.6 },
    { name: 'examples', weight: 0.3 },
    { name: 'id', weight: 0.1 }
  ],
  ignoreLocation: true,     // Match anywhere in string
  includeScore: true,       // For confidence routing
  minMatchCharLength: 2,    // Avoid single-char matches
  findAllMatches: true      // Don't stop at first match
}
```

## Confidence Thresholds

Fuse.js scores are inverted (0 = perfect match, 1 = no match).

| Score (Fuse.js) | Confidence | Action |
|-----------------|------------|--------|
| <= 0.3 | HIGH | Direct execution |
| 0.3-0.5 | MEDIUM | Confirm with user |
| > 0.5 | LOW | Claude fallback (Phase 17) |

## Score Interpretation

- **0.0-0.1:** Near-exact match (e.g., "plan phase 5" matches "plan phase" pattern)
- **0.1-0.3:** Strong match with minor variations (e.g., "planning phase 5" matches)
- **0.3-0.5:** Partial match, likely correct but confirm (e.g., "phase planning")
- **0.5-1.0:** Weak match, probably not the intended workflow

</matching_rules>

<categories>
## Workflow Categories

Categories group related workflows for organization and disambiguation:

- **planning**: Workflows that produce executable plans
  - plan-phase, discuss-phase, research-phase, list-phase-assumptions

- **execution**: Workflows that execute plans or run checks
  - execute-phase, health-check

- **analysis**: Workflows that analyze code or patterns
  - analyze-codebase, analyze-patterns, suggest, map-codebase

- **project**: Workflows for project/milestone lifecycle
  - new-project, new-milestone, discuss-milestone, complete-milestone

- **monitoring**: Workflows for progress tracking and session management
  - progress, consider-issues, pause-work, resume-work

## Category Disambiguation

When multiple workflows match with similar scores, prefer workflows in the most likely category based on context:

- User asks about "next steps" → monitoring (progress) first
- User mentions "phase X" → planning category first
- User mentions "project" or "milestone" → project category first

</categories>

<variables>
## Variable Extraction

Variables allow workflows to receive parameters from natural language input.

### Common Patterns

**phase_number** - Extract phase number from input:
```json
{
  "name": "phase_number",
  "required": false,
  "patterns": [
    "phase\\s+(\\d+)",
    "(\\d+)(?:st|nd|rd|th)?\\s+phase",
    "^(\\d+)$"
  ],
  "prompt": "Which phase number? (or press enter for next unplanned)",
  "type": "number"
}
```

**milestone_version** - Extract version number:
```json
{
  "name": "milestone_version",
  "required": false,
  "patterns": [
    "v?(\\d+\\.\\d+)",
    "version\\s+(\\d+\\.\\d+)",
    "milestone\\s+v?(\\d+\\.\\d+)"
  ],
  "prompt": "Which milestone version?",
  "type": "string"
}
```

**plan_path** - Extract file path:
```json
{
  "name": "plan_path",
  "required": false,
  "patterns": [
    "([\\w-]+/[\\w-]+-PLAN\\.md)",
    "(\\d+-\\d+-PLAN\\.md)",
    "plan\\s+at\\s+([\\w/.\\-]+)"
  ],
  "prompt": "Which plan file?",
  "type": "path"
}
```

### Pattern Best Practices

1. **Use capture groups** - `(\\d+)` not `\\d+`
2. **Case insensitivity** - Applied at matching time, patterns are case-sensitive in JSON
3. **Order matters** - More specific patterns first, fallback patterns last
4. **Test patterns** - Validate against expected inputs before deployment

</variables>

<examples>
## Template Examples

### plan-phase template (planning category)

```json
{
  "id": "plan-phase",
  "workflow": "workflows/plan-phase.md",
  "category": "planning",
  "description": "Create executable plan for a phase",
  "patterns": [
    "plan phase",
    "create plan for phase",
    "break down phase",
    "planning phase"
  ],
  "variables": [
    {
      "name": "phase_number",
      "required": false,
      "patterns": ["phase\\s+(\\d+)", "(\\d+)(?:st|nd|rd|th)?\\s+phase"],
      "prompt": "Which phase number? (or press enter for next unplanned)",
      "type": "number"
    }
  ],
  "examples": [
    "plan phase 5",
    "I need to plan phase 3",
    "create a plan for the next phase",
    "break down phase 7 into tasks",
    "let's plan phase 12",
    "planning for the 3rd phase"
  ]
}
```

### progress template (monitoring category)

```json
{
  "id": "progress",
  "workflow": "workflows/progress.md",
  "category": "monitoring",
  "description": "Check project progress and route to next action",
  "patterns": [
    "check progress",
    "show progress",
    "what's next",
    "where are we",
    "project status"
  ],
  "variables": [],
  "examples": [
    "what's the progress",
    "show me the project status",
    "where are we at",
    "what should I work on next",
    "check project progress",
    "how far along are we"
  ]
}
```

</examples>

<guidance>
## Adding New Templates

Add a template when:
1. A new workflow is created that users might invoke via natural language
2. An existing workflow gains common aliases or new use patterns
3. User feedback indicates missing pattern coverage

### Checklist for New Templates

- [ ] id matches workflow filename (without extension)
- [ ] workflow path is relative to ~/.claude/get-shit-done/
- [ ] category is one of: planning, execution, analysis, project, monitoring
- [ ] 3-5 patterns covering core intent variations
- [ ] 4-6 examples of full natural language utterances
- [ ] description is a single, clear line
- [ ] variables have tested regex patterns with capture groups

### Example Coverage Guidelines

Aim for diversity in examples:
- Different sentence structures ("plan phase 5", "I need to plan phase 5")
- With and without variables ("plan next phase" vs "plan phase 7")
- Informal variations ("let's plan" vs "create plan for")
- Questions and commands ("what phase is next" vs "plan next phase")

### Testing Templates

Before adding to template-index.json:
1. Test patterns against 5+ expected inputs
2. Verify variable extraction works
3. Check for conflicts with existing templates
4. Validate JSON syntax

</guidance>
