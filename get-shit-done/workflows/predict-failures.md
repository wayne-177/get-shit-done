<purpose>
Predict failures BEFORE plan execution by analyzing task characteristics and matching them to historical failure patterns. Enables proactive failure prevention during planning.
</purpose>

<prerequisites>
Before running predictions, verify requirements:

```bash
# 1. Check if knowledge base exists
if [ ! -d ~/.claude/gsd-knowledge ] || [ ! -f ~/.claude/gsd-knowledge/index.json ]; then
  echo "KNOWLEDGE_BASE_NOT_FOUND"
  # Skip prediction silently - return empty assessment
  RISK_ASSESSMENT=""
  return 0
fi

# 2. Check if prediction is enabled in config
PREDICTION_ENABLED="true"  # Default when adaptive is enabled
if [ -f .planning/config.json ]; then
  # Check adaptive.prediction_enabled (defaults to true when adaptive.enabled is true)
  ADAPTIVE_ENABLED=$(grep -o '"enabled": *[^,}]*' .planning/config.json | head -1 | grep -o 'true\|false')
  PREDICTION_SETTING=$(grep -o '"prediction_enabled": *[^,}]*' .planning/config.json | grep -o 'true\|false')

  if [ "$ADAPTIVE_ENABLED" = "false" ]; then
    PREDICTION_ENABLED="false"
  elif [ -n "$PREDICTION_SETTING" ]; then
    PREDICTION_ENABLED="$PREDICTION_SETTING"
  fi
fi

if [ "$PREDICTION_ENABLED" = "false" ]; then
  echo "PREDICTION_DISABLED"
  RISK_ASSESSMENT=""
  return 0
fi

echo "PREDICTION_READY"
```

**If prerequisites not met:**
Set `RISK_ASSESSMENT=""` and return. Never block planning.
</prerequisites>

<process>

<step name="parse_plan_tasks">
**Extract task characteristics from PLAN.md**

Parse the plan file to build a comprehensive task profile:

```bash
parse_plan_tasks() {
  local PLAN_FILE="$1"

  # Initialize arrays for characteristics
  TASK_TYPES=()
  FILE_TYPES=()
  COMMAND_PATTERNS=()
  TASK_COUNT=0

  # Count tasks in plan
  TASK_COUNT=$(grep -c '<task type=' "$PLAN_FILE" 2>/dev/null || echo "0")

  # === Task Type Detection (reused from import-patterns.md) ===

  # Check for command-based tasks
  if grep -qiE "(bash|npm|yarn|pnpm|git|command|run|install|execute|curl|wget)" "$PLAN_FILE"; then
    TASK_TYPES+=("bash-command")
  fi

  # Check for file editing tasks
  if grep -qiE "(create|modify|edit|write|file|code|implement|add|update)" "$PLAN_FILE"; then
    TASK_TYPES+=("file-edit")
  fi

  # Check for test tasks
  if grep -qiE "(test|verify|check|validate|spec|jest|pytest|vitest)" "$PLAN_FILE"; then
    TASK_TYPES+=("test-run")
  fi

  # Check for build tasks
  if grep -qiE "(build|compile|bundle|transpile|webpack|vite|tsc)" "$PLAN_FILE"; then
    TASK_TYPES+=("build")
  fi

  # Check for deployment tasks
  if grep -qiE "(deploy|publish|release|push to|vercel|railway|fly\.io)" "$PLAN_FILE"; then
    TASK_TYPES+=("deploy")
  fi

  # === File Type Detection ===

  # Extract file extensions mentioned in plan
  if grep -qiE "\.tsx?\b" "$PLAN_FILE"; then
    FILE_TYPES+=("typescript")
  fi
  if grep -qiE "\.jsx?\b" "$PLAN_FILE"; then
    FILE_TYPES+=("javascript")
  fi
  if grep -qiE "\.py\b" "$PLAN_FILE"; then
    FILE_TYPES+=("python")
  fi
  if grep -qiE "\.json\b" "$PLAN_FILE"; then
    FILE_TYPES+=("json")
  fi
  if grep -qiE "\.md\b" "$PLAN_FILE"; then
    FILE_TYPES+=("markdown")
  fi
  if grep -qiE "\.ya?ml\b" "$PLAN_FILE"; then
    FILE_TYPES+=("yaml")
  fi

  # === Command Pattern Detection ===

  # Extract command keywords for matching
  if grep -qiE "npm (install|i|ci)" "$PLAN_FILE"; then
    COMMAND_PATTERNS+=("npm-install")
  fi
  if grep -qiE "npm (test|run test)" "$PLAN_FILE"; then
    COMMAND_PATTERNS+=("npm-test")
  fi
  if grep -qiE "npm (run )?build" "$PLAN_FILE"; then
    COMMAND_PATTERNS+=("npm-build")
  fi
  if grep -qiE "git (push|pull|merge|rebase)" "$PLAN_FILE"; then
    COMMAND_PATTERNS+=("git-operations")
  fi
  if grep -qiE "docker|container" "$PLAN_FILE"; then
    COMMAND_PATTERNS+=("docker")
  fi

  # === Complexity Indicators ===

  # Estimate task complexity
  ACTION_LENGTH=$(grep -A 20 '<action>' "$PLAN_FILE" | wc -c | tr -d ' ')
  if [ "$ACTION_LENGTH" -gt 2000 ]; then
    COMPLEXITY="high"
  elif [ "$ACTION_LENGTH" -gt 500 ]; then
    COMPLEXITY="medium"
  else
    COMPLEXITY="low"
  fi

  echo "Parsed $TASK_COUNT tasks: types=(${TASK_TYPES[*]}), files=(${FILE_TYPES[*]}), commands=(${COMMAND_PATTERNS[*]}), complexity=$COMPLEXITY"
}

# Run parsing
parse_plan_tasks "$PLAN_FILE"
```

Store results in global variables for subsequent steps.
</step>

<step name="query_failure_patterns">
**Match task characteristics to historical failure patterns**

Query failure-pattern entries with confidence >= 0.5 (lower threshold than hints for broader prediction):

```bash
query_failure_patterns() {
  local PATTERN_DIR=~/.claude/gsd-knowledge/patterns/failure-pattern
  local MIN_CONFIDENCE="0.5"

  # Initialize results array
  MATCHED_FAILURES=()

  for task_type in "${TASK_TYPES[@]}"; do
    for file in "$PATTERN_DIR"/*.json 2>/dev/null; do
      [ -f "$file" ] || continue

      # Check if pattern matches task type
      if grep -q "\"$task_type\"" "$file"; then
        if command -v jq &> /dev/null; then
          local CONFIDENCE=$(jq -r '.confidence' "$file")
          local FAILURE_CAT=$(jq -r '.content.failure_category' "$file")
          local FREQUENCY=$(jq -r '.content.frequency' "$file")
          local TYPICAL_RECOVERY=$(jq -r '.content.typical_recovery' "$file")
          local ERROR_SIGS=$(jq -r '.content.error_signatures | @json' "$file")
          local ROOT_CAUSES=$(jq -r '.content.root_causes | @json' "$file")
          local PROJECT_COUNT=$(jq -r '.evidence.project_count' "$file")
          local EFF_SCORE=$(jq -r '.analysis.effectiveness_score // 0' "$file")

          # Check confidence threshold
          if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )); then
            # Store as compound entry: task_type|failure_cat|confidence|frequency|recovery|project_count|eff_score
            MATCHED_FAILURES+=("$task_type|$FAILURE_CAT|$CONFIDENCE|$FREQUENCY|$TYPICAL_RECOVERY|$PROJECT_COUNT|$EFF_SCORE")
          fi
        else
          # Fallback without jq - basic grep
          if grep -q "\"confidence\":" "$file"; then
            MATCHED_FAILURES+=("$task_type|unknown|0.5|1|unknown|1|0")
          fi
        fi
      fi
    done
  done

  # Also match by command patterns
  for cmd_pattern in "${COMMAND_PATTERNS[@]}"; do
    for file in "$PATTERN_DIR"/*.json 2>/dev/null; do
      [ -f "$file" ] || continue

      # Check error_signatures or root_causes for command pattern matches
      if grep -qi "$cmd_pattern" "$file"; then
        if command -v jq &> /dev/null; then
          local CONFIDENCE=$(jq -r '.confidence' "$file")
          local FAILURE_CAT=$(jq -r '.content.failure_category' "$file")
          local FREQUENCY=$(jq -r '.content.frequency' "$file")
          local TYPICAL_RECOVERY=$(jq -r '.content.typical_recovery' "$file")
          local PROJECT_COUNT=$(jq -r '.evidence.project_count' "$file")
          local EFF_SCORE=$(jq -r '.analysis.effectiveness_score // 0' "$file")

          if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )); then
            MATCHED_FAILURES+=("command|$FAILURE_CAT|$CONFIDENCE|$FREQUENCY|$TYPICAL_RECOVERY|$PROJECT_COUNT|$EFF_SCORE")
          fi
        fi
      fi
    done
  done

  echo "Found ${#MATCHED_FAILURES[@]} matching failure patterns"
}

query_failure_patterns
```

**Key differences from import-patterns.md:**
- Uses confidence >= 0.5 (vs 0.6) to catch more potential issues proactively
- Matches command patterns and file types, not just task types
- Captures frequency and effectiveness score for probability calculation
</step>

<step name="calculate_failure_probability">
**Compute failure probability for each matched pattern**

Calculate probability using pattern evidence:

```bash
calculate_failure_probability() {
  # Array to store task predictions
  TASK_PREDICTIONS=()

  for match in "${MATCHED_FAILURES[@]}"; do
    # Parse compound entry
    IFS='|' read -r TASK_TYPE FAILURE_CAT CONFIDENCE FREQUENCY RECOVERY PROJECT_COUNT EFF_SCORE <<< "$match"

    # === Base probability from frequency ===
    # More frequent failures = higher base probability
    # Cap at 0.8 (patterns can't predict certain failure)
    BASE_PROB=$(echo "scale=3; if ($FREQUENCY/20 < 0.8) $FREQUENCY/20 else 0.8" | bc)

    # === Adjust by confidence ===
    # Higher confidence patterns give more reliable predictions
    CONF_ADJUSTED=$(echo "scale=3; $BASE_PROB * $CONFIDENCE" | bc)

    # === Adjust by effectiveness score ===
    # Patterns with higher scores = more reliable predictions
    # If eff_score > 0.8, pattern is highly reliable, boost probability slightly
    # If eff_score < 0.5, pattern may be outdated, reduce probability
    if (( $(echo "$EFF_SCORE > 0.8" | bc -l) )); then
      SCORE_FACTOR="1.1"
    elif (( $(echo "$EFF_SCORE < 0.5" | bc -l) )); then
      SCORE_FACTOR="0.9"
    else
      SCORE_FACTOR="1.0"
    fi

    FINAL_PROB=$(echo "scale=3; $CONF_ADJUSTED * $SCORE_FACTOR" | bc)

    # === Cap at 0.95 (never predict certain failure) ===
    if (( $(echo "$FINAL_PROB > 0.95" | bc -l) )); then
      FINAL_PROB="0.95"
    fi

    # === Determine if cross-project validated ===
    VALIDATED="false"
    if [ "$PROJECT_COUNT" -ge 3 ]; then
      VALIDATED="true"
    fi

    # Store prediction: task_type|failure_cat|probability|confidence|recovery|validated
    TASK_PREDICTIONS+=("$TASK_TYPE|$FAILURE_CAT|$FINAL_PROB|$CONFIDENCE|$RECOVERY|$VALIDATED")
  done

  echo "Calculated ${#TASK_PREDICTIONS[@]} failure probability predictions"
}

calculate_failure_probability
```

**Probability formula:**
```
base = min(frequency/20, 0.8)
adjusted = base * confidence * effectiveness_factor
final = min(adjusted, 0.95)
```
</step>

<step name="aggregate_risk_assessment">
**Combine predictions into per-task and plan-level risk**

Aggregate multiple pattern matches per task:

```bash
aggregate_risk_assessment() {
  # Build risk profile
  declare -A TASK_RISK_MAP
  declare -A TASK_PATTERNS_MAP

  for prediction in "${TASK_PREDICTIONS[@]}"; do
    IFS='|' read -r TASK_TYPE FAILURE_CAT PROBABILITY CONFIDENCE RECOVERY VALIDATED <<< "$prediction"

    # Use highest probability for each task type (conservative approach)
    local CURRENT_PROB="${TASK_RISK_MAP[$TASK_TYPE]:-0}"
    if (( $(echo "$PROBABILITY > $CURRENT_PROB" | bc -l) )); then
      TASK_RISK_MAP[$TASK_TYPE]="$PROBABILITY"
    fi

    # Accumulate patterns per task type (include evidence for output)
    # Get original evidence from the match
    local OCCURRENCES=""
    local PROJECT_CNT=""
    for match in "${MATCHED_FAILURES[@]}"; do
      if echo "$match" | grep -q "^$TASK_TYPE|$FAILURE_CAT|"; then
        IFS='|' read -r _ _ _ OCCURRENCES _ PROJECT_CNT _ <<< "$match"
        break
      fi
    done
    TASK_PATTERNS_MAP[$TASK_TYPE]="${TASK_PATTERNS_MAP[$TASK_TYPE]}${FAILURE_CAT}:${PROBABILITY}:${CONFIDENCE}:${RECOVERY}:${VALIDATED}:${OCCURRENCES}:${PROJECT_CNT};"
  done

  # Calculate overall plan risk
  local TOTAL_RISK=0
  local RISK_COUNT=0
  local HIGH_RISK_COUNT=0
  local MEDIUM_RISK_COUNT=0

  for task_type in "${!TASK_RISK_MAP[@]}"; do
    local PROB="${TASK_RISK_MAP[$task_type]}"
    TOTAL_RISK=$(echo "scale=3; $TOTAL_RISK + $PROB" | bc)
    RISK_COUNT=$((RISK_COUNT + 1))

    # Categorize risk levels
    if (( $(echo "$PROB > 0.6" | bc -l) )); then
      HIGH_RISK_COUNT=$((HIGH_RISK_COUNT + 1))
    elif (( $(echo "$PROB >= 0.3" | bc -l) )); then
      MEDIUM_RISK_COUNT=$((MEDIUM_RISK_COUNT + 1))
    fi
  done

  # Calculate average plan risk
  if [ "$RISK_COUNT" -gt 0 ]; then
    PLAN_RISK=$(echo "scale=3; $TOTAL_RISK / $RISK_COUNT" | bc)
  else
    PLAN_RISK="0"
  fi

  # Determine plan risk level
  if (( $(echo "$PLAN_RISK > 0.6" | bc -l) )); then
    PLAN_RISK_LEVEL="HIGH"
  elif (( $(echo "$PLAN_RISK >= 0.3" | bc -l) )); then
    PLAN_RISK_LEVEL="MEDIUM"
  else
    PLAN_RISK_LEVEL="LOW"
  fi

  echo "Plan risk: $PLAN_RISK_LEVEL ($HIGH_RISK_COUNT high, $MEDIUM_RISK_COUNT medium)"

  # Export for formatting
  export TASK_RISK_MAP
  export TASK_PATTERNS_MAP
  export PLAN_RISK
  export PLAN_RISK_LEVEL
  export HIGH_RISK_COUNT
  export MEDIUM_RISK_COUNT
}

aggregate_risk_assessment
```

**Risk aggregation logic:**
- Multiple patterns for same task type: use highest probability
- Plan risk: average of all task type risks
- Track high-risk and medium-risk task counts
</step>

<step name="suggest_alternatives">
**For HIGH risk tasks, suggest preemptive alternatives**

Only triggered when task failure probability > 0.6 (HIGH risk threshold).

```bash
suggest_alternatives() {
  local STRATEGY_DIR=~/.claude/gsd-knowledge/patterns/strategy-effectiveness
  local MIN_CONFIDENCE="0.6"
  local MIN_SUCCESS_RATE="0.7"

  # Initialize alternatives map
  declare -gA TASK_ALTERNATIVES_MAP

  for task_type in "${!TASK_RISK_MAP[@]}"; do
    local PROB="${TASK_RISK_MAP[$task_type]}"

    # Only suggest alternatives for HIGH risk (> 0.6)
    if (( $(echo "$PROB <= 0.6" | bc -l) )); then
      continue
    fi

    local ALTERNATIVES=()

    # Get failure categories for this task type
    local PATTERNS="${TASK_PATTERNS_MAP[$task_type]}"
    IFS=';' read -ra PATTERN_ARRAY <<< "$PATTERNS"

    for pattern_entry in "${PATTERN_ARRAY[@]}"; do
      [ -z "$pattern_entry" ] && continue
      IFS=':' read -r FAIL_CAT _ _ _ _ _ _ <<< "$pattern_entry"

      # Query strategy-effectiveness patterns for this failure category
      for file in "$STRATEGY_DIR"/*.json 2>/dev/null; do
        [ -f "$file" ] || continue

        # Check if strategy matches failure category
        if grep -q "\"$FAIL_CAT\"" "$file"; then
          if command -v jq &> /dev/null; then
            local CONFIDENCE=$(jq -r '.confidence' "$file")
            local SUCCESS_RATE=$(jq -r '.content.success_rate' "$file")
            local STRATEGY_NAME=$(jq -r '.content.strategy_name' "$file")
            local DESCRIPTION=$(jq -r '.content.description // ""' "$file")
            local CONTRAINDICATIONS=$(jq -r '.content.contraindications // [] | @json' "$file")
            local EFF_SCORE=$(jq -r '.analysis.effectiveness_score // 0' "$file")

            # Check confidence and success rate thresholds
            if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )) && \
               (( $(echo "$SUCCESS_RATE >= $MIN_SUCCESS_RATE" | bc -l) )); then

              # Check contraindications against task context
              local SKIP_STRATEGY="false"

              # Parse contraindications and check against task characteristics
              if [ "$CONTRAINDICATIONS" != "[]" ] && [ "$CONTRAINDICATIONS" != "null" ]; then
                # Check common contraindications
                if echo "$CONTRAINDICATIONS" | grep -qi "performance" && \
                   grep -qiE "performance|benchmark|speed|timing" "$PLAN_FILE" 2>/dev/null; then
                  SKIP_STRATEGY="true"
                fi
                if echo "$CONTRAINDICATIONS" | grep -qi "real-time" && \
                   grep -qiE "real-time|realtime|live|streaming" "$PLAN_FILE" 2>/dev/null; then
                  SKIP_STRATEGY="true"
                fi
                if echo "$CONTRAINDICATIONS" | grep -qi "batch" && \
                   grep -qiE "batch|bulk|mass|parallel" "$PLAN_FILE" 2>/dev/null; then
                  SKIP_STRATEGY="true"
                fi
              fi

              if [ "$SKIP_STRATEGY" = "false" ]; then
                # Store: strategy_name|success_rate|description|eff_score
                local SUCCESS_PCT=$(echo "scale=0; $SUCCESS_RATE * 100" | bc | cut -d'.' -f1)
                ALTERNATIVES+=("$STRATEGY_NAME|$SUCCESS_PCT|$DESCRIPTION|$EFF_SCORE")
              fi
            fi
          fi
        fi
      done
    done

    # Sort alternatives by effectiveness score and limit to top 3
    if [ ${#ALTERNATIVES[@]} -gt 0 ]; then
      # Remove duplicates and sort by effectiveness score (field 4)
      local SORTED_ALTS=$(printf '%s\n' "${ALTERNATIVES[@]}" | sort -t'|' -k4 -rn | head -3)
      TASK_ALTERNATIVES_MAP[$task_type]="$SORTED_ALTS"
    fi
  done

  echo "Suggested alternatives for ${#TASK_ALTERNATIVES_MAP[@]} high-risk task types"
}

suggest_alternatives
```

**Query strategy-effectiveness patterns:**
- Match by task_type from the high-risk task
- Match by failure_category from predicted failure patterns
- Filter by confidence >= 0.6 and success_rate >= 0.7
- Sort by effectiveness_score descending
- Limit to top 3 alternatives per task

**Contraindication checking:**
- Parse contraindications array from strategy-effectiveness pattern
- Match against task characteristics (file types, commands, context in PLAN_FILE)
- Skip strategy if any contraindication matches task context
- Example: Skip "retry-with-reruns" for tasks with "Performance benchmarks" in action

**Non-blocking:** If alternative query fails, TASK_ALTERNATIVES_MAP stays empty and output shows "No alternatives available".
</step>

<step name="format_risk_assessment">
**Produce clear, actionable risk assessment output**

**Risk level thresholds:**
- **LOW:** probability < 0.3 (proceed normally)
- **MEDIUM:** probability 0.3-0.6 (be prepared for retries)
- **HIGH:** probability > 0.6 (consider alternatives before starting)

```bash
format_risk_assessment() {
  # Check if any predictions exist
  if [ ${#TASK_PREDICTIONS[@]} -eq 0 ]; then
    if [ -f ~/.claude/gsd-knowledge/index.json ]; then
      local total_patterns=$(grep -o '"total_patterns": *[0-9]*' ~/.claude/gsd-knowledge/index.json | grep -o '[0-9]*')
      if [ "$total_patterns" -eq 0 ] || [ -z "$total_patterns" ]; then
        RISK_ASSESSMENT=""
        return
      fi
    fi
    # Patterns exist but none match
    cat << 'NO_MATCH'
================================================
PREDICTIVE RISK ASSESSMENT
================================================

All tasks LOW risk - no matching failure patterns found.
Proceed with confidence.

================================================
NO_MATCH
    return
  fi

  # Build formatted output
  cat << 'RISK_HEADER'
================================================
PREDICTIVE RISK ASSESSMENT
================================================

RISK_HEADER

  # Output per-task predictions
  for task_type in "${!TASK_RISK_MAP[@]}"; do
    local PROB="${TASK_RISK_MAP[$task_type]}"
    local PROB_PCT=$(echo "scale=0; $PROB * 100" | bc | cut -d'.' -f1)

    # Determine risk level emoji and label
    local RISK_LABEL="LOW"
    local RISK_EMOJI=""
    if (( $(echo "$PROB > 0.6" | bc -l) )); then
      RISK_LABEL="HIGH"
      RISK_EMOJI="[!]"
    elif (( $(echo "$PROB >= 0.3" | bc -l) )); then
      RISK_LABEL="MEDIUM"
      RISK_EMOJI="[~]"
    fi

    echo "$RISK_EMOJI **$task_type tasks:** $RISK_LABEL risk (${PROB_PCT}% failure probability)"

    # Parse and display contributing patterns
    local PATTERNS="${TASK_PATTERNS_MAP[$task_type]}"
    if [ -n "$PATTERNS" ]; then
      echo "   Predicted issues:"
      IFS=';' read -ra PATTERN_ARRAY <<< "$PATTERNS"
      for pattern_entry in "${PATTERN_ARRAY[@]}"; do
        [ -z "$pattern_entry" ] && continue
        IFS=':' read -r FAIL_CAT PAT_PROB PAT_CONF RECOVERY VALIDATED OCCURRENCES PROJECT_CNT <<< "$pattern_entry"

        local CONF_PCT=$(echo "scale=0; $PAT_CONF * 100" | bc | cut -d'.' -f1)
        local VALIDATED_NOTE=""
        if [ "$VALIDATED" = "true" ]; then
          VALIDATED_NOTE=" [cross-project validated]"
        fi

        # Include evidence strength
        local EVIDENCE_NOTE=""
        if [ -n "$OCCURRENCES" ] && [ -n "$PROJECT_CNT" ]; then
          EVIDENCE_NOTE=" (${OCCURRENCES} occurrences, ${PROJECT_CNT} projects)"
        fi

        echo "   - $FAIL_CAT: $RECOVERY usually works (${CONF_PCT}% confidence)$VALIDATED_NOTE"
        echo "     Based on:$EVIDENCE_NOTE"
      done
    fi

    # Display suggested alternatives for HIGH risk tasks only
    if [ "$RISK_LABEL" = "HIGH" ]; then
      local ALTS="${TASK_ALTERNATIVES_MAP[$task_type]}"
      if [ -n "$ALTS" ]; then
        echo ""
        echo "   **Suggested alternatives (consider before execution):**"
        local ALT_NUM=1
        while IFS= read -r alt_line; do
          [ -z "$alt_line" ] && continue
          IFS='|' read -r STRAT_NAME SUCCESS_PCT STRAT_DESC _ <<< "$alt_line"
          echo "   $ALT_NUM. $STRAT_NAME - ${SUCCESS_PCT}% success rate"
          if [ -n "$STRAT_DESC" ] && [ "$STRAT_DESC" != "null" ]; then
            echo "      $STRAT_DESC"
          fi
          ALT_NUM=$((ALT_NUM + 1))
        done <<< "$ALTS"
      else
        echo ""
        echo "   No proven alternatives available - proceed with caution, be ready to escalate"
      fi
    fi
    echo ""
  done

  # Plan summary
  local PLAN_RISK_PCT=$(echo "scale=0; $PLAN_RISK * 100" | bc | cut -d'.' -f1)

  echo "------------------------------------------------"
  echo "OVERALL PLAN RISK: $PLAN_RISK_LEVEL (${PLAN_RISK_PCT}% average)"

  if [ "$HIGH_RISK_COUNT" -gt 0 ]; then
    echo "$HIGH_RISK_COUNT task type(s) with HIGH risk - consider alternatives"
  fi
  if [ "$MEDIUM_RISK_COUNT" -gt 0 ]; then
    echo "$MEDIUM_RISK_COUNT task type(s) with MEDIUM risk - be prepared for retries"
  fi

  echo "================================================"
}

# Generate and store assessment
RISK_ASSESSMENT=$(format_risk_assessment)
```

**Output consumed by:** plan-phase.md during plan display (Phase 9 Plan 2 integration)
</step>

<step name="handle_errors">
**Non-blocking error handling**

Ensure prediction failures never block planning:

```bash
# Wrap entire prediction workflow
run_predictions_safely() {
  set +e  # Don't exit on error

  local PLAN_FILE="$1"

  # Attempt prediction
  {
    parse_plan_tasks "$PLAN_FILE"
    query_failure_patterns
    calculate_failure_probability
    aggregate_risk_assessment
    suggest_alternatives
    format_risk_assessment
  } 2>/dev/null

  # Check if we got any result
  if [ -z "$RISK_ASSESSMENT" ]; then
    # Prediction failed silently - this is fine
    RISK_ASSESSMENT=""
  fi

  set -e  # Restore error handling
}

# Usage
run_predictions_safely "$PLAN_FILE"
```

**Non-blocking guarantee:**
- All bash errors suppressed during prediction
- Empty assessment returned on any failure
- Planning proceeds regardless of prediction outcome
</step>

</process>

<output>
The workflow produces a `RISK_ASSESSMENT` variable containing formatted risk output, or empty string if:
- Knowledge base doesn't exist
- prediction_enabled is false in config
- No failure patterns match task characteristics
- Any error occurs during prediction

**Variable structure:**
```bash
# Empty when no predictions
RISK_ASSESSMENT=""

# Or formatted assessment block
RISK_ASSESSMENT="
================================================
PREDICTIVE RISK ASSESSMENT
================================================

[!] **bash-command tasks:** HIGH risk (65% failure probability)
   Predicted issues:
   - dependency: reinstall-dependencies usually works (80% confidence) [cross-project validated]
     Based on: (12 occurrences, 4 projects)

   **Suggested alternatives (consider before execution):**
   1. clean-install-strategy - 85% success rate
      Remove node_modules and reinstall with fresh lock file
   2. incremental-dependency - 78% success rate
      Install dependencies incrementally to isolate failures
   3. fallback-version-pin - 72% success rate
      Pin to last known working versions

[~] **test-run tasks:** MEDIUM risk (42% failure probability)
   Predicted issues:
   - validation: retry-with-reruns usually works (75% confidence)
     Based on: (8 occurrences, 2 projects)

------------------------------------------------
OVERALL PLAN RISK: MEDIUM (54% average)
1 task type(s) with HIGH risk - consider alternatives
1 task type(s) with MEDIUM risk - be prepared for retries
================================================
"
```

**Usage in plan-phase.md:**
```bash
# After generating plan, run predictions
source ~/.claude/get-shit-done/workflows/predict-failures.md
RISK_ASSESSMENT=$(run_predictions_safely "$PLAN_PATH")

# Display if non-empty
if [ -n "$RISK_ASSESSMENT" ]; then
  echo "$RISK_ASSESSMENT"
fi
```
</output>

<configuration>
Prediction behavior configured in `.planning/config.json`:

```json
{
  "adaptive": {
    "enabled": true,
    "prediction_enabled": true,
    "prediction_confidence_threshold": 0.5
  }
}
```

**Configuration options:**
- `adaptive.enabled`: Master switch for adaptive features (prediction respects this)
- `adaptive.prediction_enabled`: Enable/disable predictive risk assessment (default: true when adaptive.enabled)
- `adaptive.prediction_confidence_threshold`: Minimum confidence for pattern inclusion (default: 0.5)

**If config missing:** Use defaults (prediction enabled, 0.5 confidence threshold).
</configuration>

<related_files>
- `@get-shit-done/workflows/import-patterns.md` - Related workflow for execution-time hints
- `@get-shit-done/workflows/analyze-patterns.md` - Pattern analysis for effectiveness scores
- `@get-shit-done/references/pattern-schema.md` - Pattern JSON structure
- `@get-shit-done/workflows/plan-phase.md` - Integration point (Phase 9 Plan 2)
</related_files>
