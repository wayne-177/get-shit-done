# Workflow: Variable Extractor

<purpose>
Extract variable values from user input using regex patterns defined in the matched template.
Handles required vs optional variables, type conversion, and missing value prompts.
</purpose>

<input>
- user_input: The original natural language request
- template: The matched WorkflowTemplate containing variable definitions
</input>

<algorithm>
## Extraction Process

**Step 1: Iterate through template.variables**

For each VariableSlot in template.variables:

**Step 2: Try each pattern**

For each pattern in slot.patterns:
- Create regex with 'i' flag (case insensitive)
- Match against user_input
- If match found with capture group, extract value

**Step 3: Type conversion**

Based on slot.type:
- **"number"**: parseInt the captured value (e.g., "5" → 5)
- **"path"**: validate as file path format, normalize slashes
- **"string"**: use as-is (default)

**Step 4: Track missing required variables**

If slot.required === true and no value extracted:
- Add to missing_required array with slot.prompt

**Step 5: Return extraction results**

Return object with extracted variables, missing required list, and success flag.
</algorithm>

<common_patterns>
## Pre-defined Pattern Library

### Phase Numbers

Extract phase numbers from various formats:

```regex
phase\s+(\d+)              # "phase 5"
phase\s+#?(\d+)            # "phase #5"
(\d+)(?:st|nd|rd|th)?\s+phase  # "5th phase"
^(\d+)$                    # Just "5" (when context is clear)
```

**Examples:**
| Input | Pattern | Extracted |
|-------|---------|-----------|
| "plan phase 5" | `phase\s+(\d+)` | "5" → 5 |
| "the 3rd phase" | `(\d+)(?:st|nd|rd|th)?\s+phase` | "3" → 3 |
| "phase #12" | `phase\s+#?(\d+)` | "12" → 12 |

### Milestone Versions

Extract version numbers:

```regex
v?(\d+\.\d+)               # "v1.0" or "1.0"
version\s+(\d+\.\d+)       # "version 1.0"
milestone\s+v?(\d+\.\d+)   # "milestone v1.0"
```

**Examples:**
| Input | Pattern | Extracted |
|-------|---------|-----------|
| "milestone v2.0" | `milestone\s+v?(\d+\.\d+)` | "2.0" |
| "version 3.5" | `version\s+(\d+\.\d+)` | "3.5" |
| "3.0" | `v?(\d+\.\d+)` | "3.0" |

### Workflow Names

Extract workflow/command identifiers:

```regex
(?:run|execute|use)\s+([\w-]+)   # "run plan-phase"
workflow\s+["']?([\w-]+)["']?     # "workflow 'plan-phase'"
/([\w:-]+)                        # "/gsd:plan-phase"
```

**Examples:**
| Input | Pattern | Extracted |
|-------|---------|-----------|
| "run analyze-codebase" | `(?:run|execute|use)\s+([\w-]+)` | "analyze-codebase" |
| "workflow health-check" | `workflow\s+["']?([\w-]+)["']?` | "health-check" |
| "/gsd:progress" | `/([\w:-]+)` | "gsd:progress" |

### File Paths

Extract file path references:

```regex
(?:file|path)\s+["']?([^\s"']+)["']?   # "file src/app.ts"
([/.][\w/.\\-]+\.md)                    # "./path/to/PLAN.md"
([\w-]+/[\w-]+-PLAN\.md)               # "16-02/16-02-PLAN.md"
(\d+-\d+)                               # "16-02" (plan reference)
```

**Examples:**
| Input | Pattern | Extracted |
|-------|---------|-----------|
| "execute plan 16-02" | `(\d+-\d+)` | "16-02" |
| "file ./PLAN.md" | `(?:file|path)\s+["']?([^\s"']+)["']?` | "./PLAN.md" |

### Goal Descriptions

Extract free-form text:

```regex
understand\s+(.+)$         # "understand add user auth"
goal:\s*(.+)$              # "goal: implement caching"
["'](.+)["']               # Quoted text
```
</common_patterns>

<output>
```typescript
interface ExtractionResult {
  variables: Record<string, string | number>;  // Extracted values
  missing_required: MissingVariable[];         // Required but not found
  all_required_present: boolean;               // Quick check
}

interface MissingVariable {
  name: string;
  prompt: string;
  type: string;
}
```
</output>

<execution_guidance>
## When Claude Executes This Workflow

1. **Get variables array from matched template**
   ```javascript
   const variables = template.variables || [];
   ```

2. **For each variable slot:**

   a. **Try each regex pattern against input**
   ```javascript
   for (const patternStr of slot.patterns) {
     const pattern = new RegExp(patternStr, 'i');
     const match = input.match(pattern);
     if (match && match[1]) {
       // Found a match
       break;
     }
   }
   ```

   b. **If match found, store in result with type conversion**
   ```javascript
   if (slot.type === 'number') {
     result[slot.name] = parseInt(match[1], 10);
   } else {
     result[slot.name] = match[1];
   }
   ```

   c. **If no match and required, add to missing list**
   ```javascript
   if (slot.required && !result[slot.name]) {
     missing.push({
       name: slot.name,
       prompt: slot.prompt,
       type: slot.type || 'string'
     });
   }
   ```

3. **Return extraction result**
   ```javascript
   return {
     variables: result,
     missing_required: missing,
     all_required_present: missing.length === 0
   };
   ```

4. **If missing_required not empty, caller should prompt user**
   The route-intent.md workflow handles prompting for missing values.

## Type Conversion Rules

| Type | Conversion | Validation |
|------|------------|------------|
| number | parseInt(value, 10) | Must be valid integer |
| string | value (as-is) | None |
| path | value (as-is) | Optional: check format |

## Pattern Ordering

Patterns should be ordered from most specific to least specific:
1. Full phrase patterns first ("phase 5")
2. Partial patterns second ("5th phase")
3. Fallback patterns last (just "5")
</execution_guidance>

<examples>
## Extraction Examples

### Example 1: Plan Phase with Number

**Input:** "plan phase 5"
**Template:** plan-phase

```json
{
  "id": "plan-phase",
  "variables": [{
    "name": "phase_number",
    "required": false,
    "patterns": ["phase\\s+(\\d+)", "(\\d+)(?:st|nd|rd|th)?\\s+phase"],
    "type": "number"
  }]
}
```

**Result:**
```json
{
  "variables": { "phase_number": 5 },
  "missing_required": [],
  "all_required_present": true
}
```

### Example 2: New Milestone Missing Version

**Input:** "create a new milestone"
**Template:** new-milestone

```json
{
  "id": "new-milestone",
  "variables": [{
    "name": "milestone_version",
    "required": false,
    "patterns": ["v?(\\d+\\.\\d+)"],
    "prompt": "What version for this milestone? (e.g., 3.0)",
    "type": "string"
  }]
}
```

**Result:**
```json
{
  "variables": {},
  "missing_required": [],
  "all_required_present": true
}
```
(Version not required, so still valid)

### Example 3: Execute Plan with Reference

**Input:** "execute plan 16-02"
**Template:** execute-plan

```json
{
  "id": "execute-plan",
  "variables": [{
    "name": "plan_path",
    "required": false,
    "patterns": ["(\\d+-\\d+)", "([\\w-]+/[\\w-]+-PLAN\\.md)"],
    "type": "path"
  }]
}
```

**Result:**
```json
{
  "variables": { "plan_path": "16-02" },
  "missing_required": [],
  "all_required_present": true
}
```

### Example 4: Multiple Variables

**Input:** "research phase 7 for milestone 2.0"
**Template:** (hypothetical with multiple vars)

**Result:**
```json
{
  "variables": {
    "phase_number": 7,
    "milestone_version": "2.0"
  },
  "missing_required": [],
  "all_required_present": true
}
```

### Example 5: Required Variable Missing

**Input:** "run the workflow"
**Template:** (with required workflow_name)

**Result:**
```json
{
  "variables": {},
  "missing_required": [
    {
      "name": "workflow_name",
      "prompt": "Which workflow would you like to run?",
      "type": "string"
    }
  ],
  "all_required_present": false
}
```
</examples>

<edge_cases>
## Edge Case Handling

### Multiple Matches
If multiple patterns match, use the first match (patterns are ordered by specificity).

### Partial Matches
If a pattern partially matches but doesn't capture, continue to next pattern.

### Empty Capture Groups
If `match[1]` is undefined or empty string, treat as no match.

### Number Conversion Failures
If parseInt fails (NaN), treat as string type instead.

### Path Normalization
For path types, normalize forward/backward slashes to forward slashes.

### Optional Variables
If a variable is not required and not found, it's simply not included in the result.
</edge_cases>

<integration>
## Integration Points

**Invoked by:** route-intent.md (after intent matching)

**Receives from:** intent-matcher.md (matched template)

**Uses:**
- template-index.json: Variable slot definitions
- template-schema.md: VariableSlot schema reference
</integration>
