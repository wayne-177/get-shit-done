# Workflow: Intent Matcher

<purpose>
Match natural language user input to workflow templates using fuzzy search.
Returns matched template(s) with confidence scores.
</purpose>

<input>
- user_input: The natural language request from the user
- templates: Array of WorkflowTemplate objects from template-index.json
</input>

<algorithm>
## Matching Process

**Step 1: Normalize input**
- Convert to lowercase
- Remove extra whitespace (collapse multiple spaces to single)
- Strip common filler words: "please", "can you", "I want to", "help me", "could you", "I need to", "let's"
- Trim leading/trailing whitespace

**Step 2: Fuzzy match against patterns and examples**

Conceptually using Fuse.js configuration:
```javascript
{
  threshold: 0.6,           // Default match threshold
  keys: [
    { name: 'patterns', weight: 0.6 },  // Intent phrases weighted highest
    { name: 'examples', weight: 0.3 },  // Full examples secondary
    { name: 'id', weight: 0.1 }         // Workflow ID lowest
  ],
  ignoreLocation: true,     // Match anywhere in string
  includeScore: true,       // Return score with results
  minMatchCharLength: 2,    // Avoid single-char matches
  findAllMatches: true      // Don't stop at first match
}
```

**Step 3: Score interpretation**

Fuse.js returns score 0.0 (perfect match) to 1.0 (no match).
Convert to confidence: `confidence = 1 - score`

| Fuse.js Score | Confidence | Category | Action |
|---------------|------------|----------|--------|
| <= 0.3 | >= 0.7 | HIGH | Direct execution |
| 0.3 < score <= 0.5 | 0.5-0.7 | MEDIUM | Confirm with user |
| > 0.5 | < 0.5 | LOW | Claude fallback (Phase 17) |

**Step 4: Return results**

Return array of matches sorted by score (best first):
- template: matched WorkflowTemplate
- score: Fuse.js score (lower = better)
- confidence: 1 - score (higher = better)
- matched_on: which field matched (patterns, examples, id)
</algorithm>

<output>
```typescript
interface MatchResult {
  template: WorkflowTemplate;
  score: number;        // 0.0-1.0, lower is better
  confidence: number;   // 0.0-1.0, higher is better
  matched_on: 'patterns' | 'examples' | 'id';
}
```

Returns: MatchResult[] (empty if no matches above threshold)
</output>

<execution_guidance>
## When Claude Executes This Workflow

1. **Load template index**
   ```bash
   cat ~/.claude/get-shit-done/workflow-templates/template-index.json
   ```
   Parse JSON to get templates array.

2. **Normalize user input**
   Apply normalization rules:
   - Lowercase: "Plan Phase 5" → "plan phase 5"
   - Remove fillers: "can you help me plan phase 5" → "plan phase 5"
   - Trim whitespace: "  plan phase 5  " → "plan phase 5"

3. **For each template, calculate similarity**
   Check matches against:
   - **patterns**: Does normalized input contain or closely match any pattern phrase?
   - **examples**: Is normalized input similar to any example utterance?
   - **id**: Does normalized input mention the template id?

   Weighting: patterns (60%) > examples (30%) > id (10%)

4. **Rank by best match score**
   Sort results by score ascending (lower score = better match).

5. **Return top 3 matches above threshold**
   Filter out any matches with score > 0.6 (below threshold).
   Return up to 3 best matches.

## Decision Logic

```
IF no matches found:
  → Return empty array
  → Caller routes to Claude fallback

IF best match score <= 0.3:
  → HIGH confidence
  → Proceed to variable extraction
  → Execute directly

IF best match score 0.3-0.5:
  → MEDIUM confidence
  → Ask user to confirm: "Did you mean: [template.id]?"
  → Wait for confirmation

IF best match score > 0.5:
  → LOW confidence
  → Route to Claude fallback (Phase 17)
```
</execution_guidance>

<common_matches>
## Example Match Scenarios

### HIGH Confidence Matches (score <= 0.3)

| User Input | Matched Template | Score | Why |
|------------|------------------|-------|-----|
| "plan phase 5" | plan-phase | 0.1 | Direct pattern match |
| "check progress" | progress | 0.05 | Exact pattern match |
| "run health check" | health-check | 0.15 | Close pattern match |
| "suggest goals" | suggest | 0.1 | Direct pattern match |
| "new project" | new-project | 0.05 | Exact pattern match |

### MEDIUM Confidence Matches (score 0.3-0.5)

| User Input | Matched Template | Score | Why |
|------------|------------------|-------|-----|
| "break down phase 3 into tasks" | plan-phase | 0.35 | Example match |
| "what should we work on" | progress | 0.4 | Semantic similarity |
| "analyze the code" | analyze-codebase | 0.45 | Partial pattern match |

### LOW Confidence Matches (score > 0.5)

| User Input | Matched Template | Score | Action |
|------------|------------------|-------|--------|
| "fix the bug" | (none good) | 0.7 | Claude fallback |
| "deploy to production" | (none) | 0.9 | Claude fallback |
| "write tests" | (none good) | 0.65 | Claude fallback |

### Normalization Examples

| Raw Input | After Normalization |
|-----------|---------------------|
| "Can you help me plan phase 5?" | "plan phase 5?" |
| "I want to check the progress please" | "check the progress" |
| "Let's run a health check" | "run a health check" |
| "ANALYZE CODEBASE" | "analyze codebase" |
</common_matches>

<category_hints>
## Category-Based Disambiguation

When multiple templates match with similar scores, use category hints:

| User Context | Prefer Category |
|--------------|-----------------|
| Mentions "phase N" | planning |
| Mentions "project" or "milestone" | project |
| Asks "what's next" or "status" | monitoring |
| Mentions "run" or "execute" | execution |
| Asks about "issues" or "security" | analysis |

Categories (from template-schema.md):
- **planning**: plan-phase, discuss-phase, research-phase, list-phase-assumptions, understand
- **execution**: execute-plan, health-check
- **analysis**: analyze-codebase, analyze-patterns, suggest
- **project**: new-project, new-milestone, discuss-milestone, complete-milestone
- **monitoring**: progress, pause-work, resume-work, consider-issues
</category_hints>

<integration>
## Integration Points

**Invoked by:** route-intent.md (main routing workflow)

**Provides to:** variable-extractor.md (matched template for variable extraction)

**Uses:**
- template-index.json: Template definitions for matching
- template-schema.md: Schema reference for field definitions
</integration>
