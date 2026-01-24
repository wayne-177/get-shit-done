<purpose>
Query the global knowledge base for relevant patterns and return predictive hints for the current execution context.
</purpose>

<prerequisites>
Before importing patterns, verify knowledge base exists:

```bash
# Check if knowledge base is initialized
if [ -d ~/.claude/gsd-knowledge ] && [ -f ~/.claude/gsd-knowledge/index.json ]; then
  echo "KNOWLEDGE_BASE_READY"
else
  echo "KNOWLEDGE_BASE_NOT_FOUND"
fi
```

**If knowledge base not found:**
Skip pattern import silently. Display:
```
No patterns yet - hints will appear as you use GSD across projects
```
</prerequisites>

<process>

<step name="parse_plan_context">
**Extract task types from the PLAN.md being executed**

Parse the plan file to identify what kinds of tasks will be executed:

```bash
PLAN_FILE="$1"  # Path to PLAN.md

# Extract task types from plan content
# Look for indicators of different task categories:
# - bash-command: Commands, npm/yarn, git operations
# - file-edit: Create/modify files, code changes
# - test-run: Test execution, verification
# - build: Compilation, bundling
# - deploy: Deployment operations

TASK_TYPES=()

# Check for command-based tasks
if grep -qiE "(bash|npm|yarn|git|command|run|install|execute)" "$PLAN_FILE"; then
  TASK_TYPES+=("bash-command")
fi

# Check for file editing tasks
if grep -qiE "(create|modify|edit|write|file|code|implement)" "$PLAN_FILE"; then
  TASK_TYPES+=("file-edit")
fi

# Check for test tasks
if grep -qiE "(test|verify|check|validate|spec)" "$PLAN_FILE"; then
  TASK_TYPES+=("test-run")
fi

# Check for build tasks
if grep -qiE "(build|compile|bundle|transpile)" "$PLAN_FILE"; then
  TASK_TYPES+=("build")
fi

# Check for deployment tasks
if grep -qiE "(deploy|publish|release|push to)" "$PLAN_FILE"; then
  TASK_TYPES+=("deploy")
fi

echo "Detected task types: ${TASK_TYPES[@]}"
```

Store detected task types for querying.
</step>

<step name="query_strategy_effectiveness">
**Find strategy-effectiveness patterns matching detected task types**

Query patterns with confidence >= 0.6:

```bash
PATTERN_DIR=~/.claude/gsd-knowledge/patterns/strategy-effectiveness
MIN_CONFIDENCE="0.6"
STRATEGY_HINTS=()

for task_type in "${TASK_TYPES[@]}"; do
  for file in "$PATTERN_DIR"/*.json 2>/dev/null; do
    [ -f "$file" ] || continue

    # Check if pattern matches task type
    if grep -q "\"$task_type\"" "$file"; then
      # Get confidence
      if command -v jq &> /dev/null; then
        CONFIDENCE=$(jq -r '.confidence' "$file")
        SUCCESS_RATE=$(jq -r '.content.success_rate' "$file")
        STRATEGY_NAME=$(jq -r '.content.strategy_name' "$file")
        TOTAL_USES=$(jq -r '.evidence.occurrences' "$file")
        PROJECT_COUNT=$(jq -r '.evidence.project_count' "$file")
        DESCRIPTION=$(jq -r '.content.description' "$file")

        # Check confidence threshold
        if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )); then
          # Format success rate as percentage
          SUCCESS_PCT=$(echo "$SUCCESS_RATE * 100" | bc | cut -d'.' -f1)
          STRATEGY_HINTS+=("Strategy \"$STRATEGY_NAME\" has ${SUCCESS_PCT}% success rate ($TOTAL_USES uses across $PROJECT_COUNT projects)")
        fi
      else
        # Fallback without jq - just note pattern exists
        STRATEGY_HINTS+=("Pattern found in: $file")
      fi
    fi
  done
done

# Limit to top 3 strategy hints (already sorted by confidence in query)
STRATEGY_HINTS=("${STRATEGY_HINTS[@]:0:3}")
```
</step>

<step name="query_failure_patterns">
**Find common failures for detected task types**

```bash
PATTERN_DIR=~/.claude/gsd-knowledge/patterns/failure-pattern
FAILURE_HINTS=()

for task_type in "${TASK_TYPES[@]}"; do
  for file in "$PATTERN_DIR"/*.json 2>/dev/null; do
    [ -f "$file" ] || continue

    # Check if pattern matches task type
    if grep -q "\"$task_type\"" "$file"; then
      if command -v jq &> /dev/null; then
        CONFIDENCE=$(jq -r '.confidence' "$file")
        FAILURE_CAT=$(jq -r '.content.failure_category' "$file")
        TYPICAL_RECOVERY=$(jq -r '.content.typical_recovery' "$file")
        FREQUENCY=$(jq -r '.content.frequency' "$file")

        if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )); then
          FAILURE_HINTS+=("Watch for: $FAILURE_CAT failures ($TYPICAL_RECOVERY usually works)")
        fi
      fi
    fi
  done
done

# Limit to top 2 failure warnings
FAILURE_HINTS=("${FAILURE_HINTS[@]:0:2}")
```
</step>

<step name="query_task_insights">
**Find best practices and pitfalls for detected task types**

```bash
PATTERN_DIR=~/.claude/gsd-knowledge/patterns/task-insight
INSIGHT_HINTS=()

for task_type in "${TASK_TYPES[@]}"; do
  for file in "$PATTERN_DIR"/*.json 2>/dev/null; do
    [ -f "$file" ] || continue

    # Check if pattern matches task type
    if grep -q "\"$task_type\"" "$file"; then
      if command -v jq &> /dev/null; then
        CONFIDENCE=$(jq -r '.confidence' "$file")
        INSIGHT_CAT=$(jq -r '.content.insight_category' "$file")
        INSIGHT=$(jq -r '.content.insight' "$file")
        RECOMMENDATION=$(jq -r '.content.actionable_recommendation' "$file")

        if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )); then
          if [ "$INSIGHT_CAT" = "best-practice" ]; then
            INSIGHT_HINTS+=("Best practice: $INSIGHT")
          elif [ "$INSIGHT_CAT" = "common-pitfall" ]; then
            INSIGHT_HINTS+=("Pitfall to avoid: $INSIGHT")
          fi
        fi
      fi
    fi
  done
done

# Limit to top 2 insights
INSIGHT_HINTS=("${INSIGHT_HINTS[@]:0:2}")
```
</step>

<step name="format_results">
**Structure results as predictive hints display**

Combine all collected hints into formatted output block:

```bash
format_pattern_hints() {
  local has_hints=false

  # Check if any hints exist
  if [ ${#STRATEGY_HINTS[@]} -gt 0 ] || [ ${#FAILURE_HINTS[@]} -gt 0 ] || [ ${#INSIGHT_HINTS[@]} -gt 0 ]; then
    has_hints=true
  fi

  if [ "$has_hints" = false ]; then
    # No patterns found - could be new knowledge base
    echo ""
    return
  fi

  # Build formatted output
  cat << 'HINTS_HEADER'
===============================================
PREDICTIVE HINTS (from cross-project learning)
===============================================

HINTS_HEADER

  # Group hints by task type
  for task_type in "${TASK_TYPES[@]}"; do
    local type_hints=()

    # Collect relevant hints for this task type
    # (In practice, we'd filter hints by task_type more precisely)

    case "$task_type" in
      "test-run")
        echo "**For test-run tasks:**"
        for hint in "${STRATEGY_HINTS[@]}"; do
          if echo "$hint" | grep -qi "test\|run\|rerun"; then
            echo "- $hint"
          fi
        done
        ;;
      "bash-command")
        echo "**For bash-command tasks:**"
        for hint in "${FAILURE_HINTS[@]}"; do
          echo "- $hint"
        done
        ;;
      "file-edit")
        echo "**For file-edit tasks:**"
        for hint in "${INSIGHT_HINTS[@]}"; do
          echo "- $hint"
        done
        ;;
      *)
        echo "**For $task_type tasks:**"
        ;;
    esac
    echo ""
  done

  echo "==============================================="
}

# Generate formatted hints
FORMATTED_HINTS=$(format_pattern_hints)
echo "$FORMATTED_HINTS"
```

**Example output:**

```
===============================================
PREDICTIVE HINTS (from cross-project learning)
===============================================

**For test-run tasks:**
- Strategy "retry-with-reruns" has 85% success rate (12 uses across 4 projects)
- Common failure: validation errors - typically recovers with test isolation

**For bash-command tasks:**
- Watch for: dependency failures (reinstall usually works)
- Best practice: Always verify command exists before complex pipelines

===============================================
```
</step>

<step name="empty_knowledge_base_handling">
**Graceful handling when knowledge base is empty or unavailable**

```bash
handle_empty_knowledge_base() {
  local total_patterns=0

  # Check index for pattern count
  if [ -f ~/.claude/gsd-knowledge/index.json ]; then
    if command -v jq &> /dev/null; then
      total_patterns=$(jq -r '.total_patterns' ~/.claude/gsd-knowledge/index.json)
    else
      total_patterns=$(grep -o '"total_patterns": *[0-9]*' ~/.claude/gsd-knowledge/index.json | grep -o '[0-9]*')
    fi
  fi

  if [ "$total_patterns" -eq 0 ] || [ -z "$total_patterns" ]; then
    echo ""
    echo "No patterns yet - hints will appear as you use GSD across projects"
    echo ""
    return 0
  fi

  return 1  # Patterns exist, continue with query
}
```

This message is intentionally brief and non-intrusive when no patterns exist yet.
</step>

</process>

<output>
The workflow produces a `PATTERN_HINTS` variable containing the formatted hints block, or an empty string if:
- Knowledge base doesn't exist
- Knowledge base has no patterns
- No patterns match the detected task types
- All matching patterns are below confidence threshold (0.6)

**Usage in execute-phase.md:**

```bash
# Source or call import-patterns workflow
PATTERN_HINTS=$(import_patterns "$PLAN_PATH")

# Display if non-empty
if [ -n "$PATTERN_HINTS" ]; then
  echo "$PATTERN_HINTS"
fi
```

**Non-blocking guarantee:**
Pattern import failures are logged but never block plan execution. If any step fails:
- Log warning to stderr
- Set PATTERN_HINTS to empty string
- Continue with execution
</output>

<configuration>
Pattern import behavior can be configured in `.planning/config.json`:

```json
{
  "adaptive": {
    "enabled": true,
    "show_pattern_hints": true,
    "min_pattern_confidence": 0.6,
    "max_hints_per_category": {
      "strategy": 3,
      "failure": 2,
      "insight": 2
    }
  }
}
```

**Configuration options:**
- `show_pattern_hints`: Set false to disable predictive hints display entirely
- `min_pattern_confidence`: Minimum confidence threshold for pattern inclusion (default: 0.6)
- `max_hints_per_category`: Maximum hints to show per category (prevent overwhelming output)

**If config missing:** Use defaults (show hints, 0.6 confidence, 3/2/2 limits).
</configuration>

<related_files>
- `@get-shit-done/references/knowledge-base-api.md` - Storage operations
- `@get-shit-done/references/pattern-schema.md` - Pattern JSON structure
- `@get-shit-done/workflows/execute-phase.md` - Integration point (display_pattern_hints step)
- `@get-shit-done/workflows/init-knowledge-base.md` - Knowledge base initialization
</related_files>
