<purpose>
Analyze patterns in the global knowledge base to calculate frequency metrics, effectiveness scores, and identify high-value patterns for predictive hints and decay decisions.
</purpose>

<prerequisites>
Before running analysis, verify knowledge base exists and has patterns:

```bash
# Check knowledge base initialization
if [ ! -d ~/.claude/gsd-knowledge ]; then
  echo "ERROR: Knowledge base not initialized"
  echo "Run init-knowledge-base workflow first"
  exit 1
fi

# Check for patterns
TOTAL_PATTERNS=$(jq -r '.total_patterns // 0' ~/.claude/gsd-knowledge/index.json 2>/dev/null || echo "0")
if [ "$TOTAL_PATTERNS" -eq 0 ]; then
  echo "No patterns in knowledge base yet"
  echo "Patterns accumulate as you complete milestones with /gsd:complete-milestone"
  exit 0
fi

echo "Knowledge base ready: $TOTAL_PATTERNS patterns"
```

**If prerequisites fail:**
- Knowledge base not initialized → Exit with message to run init workflow
- No patterns yet → Exit with informational message (not an error)
</prerequisites>

<process>

<step name="load_all_patterns">
**Load patterns from knowledge base into memory for analysis**

```bash
KB_DIR=~/.claude/gsd-knowledge
PATTERNS_DIR="$KB_DIR/patterns"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Initialize counters
declare -A TYPE_COUNTS
declare -A TASK_TYPE_COUNTS
declare -A FAILURE_CAT_COUNTS

# Strategy-effectiveness patterns
SE_COUNT=0
for file in "$PATTERNS_DIR/strategy-effectiveness"/*.json 2>/dev/null; do
  [ -f "$file" ] || continue
  SE_COUNT=$((SE_COUNT + 1))

  # Count by task types
  if command -v jq &> /dev/null; then
    TASK_TYPES=$(jq -r '.content.task_types[]?' "$file" 2>/dev/null)
    for tt in $TASK_TYPES; do
      TASK_TYPE_COUNTS["$tt"]=$((${TASK_TYPE_COUNTS["$tt"]:-0} + 1))
    done

    # Count by failure categories
    FAIL_CATS=$(jq -r '.content.failure_categories[]?' "$file" 2>/dev/null)
    for fc in $FAIL_CATS; do
      FAILURE_CAT_COUNTS["$fc"]=$((${FAILURE_CAT_COUNTS["$fc"]:-0} + 1))
    done
  fi
done
TYPE_COUNTS["strategy-effectiveness"]=$SE_COUNT

# Failure-pattern patterns
FP_COUNT=0
for file in "$PATTERNS_DIR/failure-pattern"/*.json 2>/dev/null; do
  [ -f "$file" ] || continue
  FP_COUNT=$((FP_COUNT + 1))

  if command -v jq &> /dev/null; then
    TT=$(jq -r '.content.task_type // "other"' "$file" 2>/dev/null)
    FC=$(jq -r '.content.failure_category // "other"' "$file" 2>/dev/null)
    TASK_TYPE_COUNTS["$TT"]=$((${TASK_TYPE_COUNTS["$TT"]:-0} + 1))
    FAILURE_CAT_COUNTS["$FC"]=$((${FAILURE_CAT_COUNTS["$FC"]:-0} + 1))
  fi
done
TYPE_COUNTS["failure-pattern"]=$FP_COUNT

# Task-insight patterns
TI_COUNT=0
for file in "$PATTERNS_DIR/task-insight"/*.json 2>/dev/null; do
  [ -f "$file" ] || continue
  TI_COUNT=$((TI_COUNT + 1))

  if command -v jq &> /dev/null; then
    TT=$(jq -r '.content.task_type // "other"' "$file" 2>/dev/null)
    TASK_TYPE_COUNTS["$TT"]=$((${TASK_TYPE_COUNTS["$TT"]:-0} + 1))
  fi
done
TYPE_COUNTS["task-insight"]=$TI_COUNT

TOTAL_PATTERNS=$((SE_COUNT + FP_COUNT + TI_COUNT))
echo "Loaded $TOTAL_PATTERNS patterns"
```
</step>

<step name="calculate_frequency_metrics">
**Generate distribution statistics**

```bash
# Calculate percentages
calc_pct() {
  local count=$1
  local total=$2
  if [ "$total" -gt 0 ]; then
    echo "scale=1; $count * 100 / $total" | bc
  else
    echo "0"
  fi
}

# Type distribution
SE_PCT=$(calc_pct $SE_COUNT $TOTAL_PATTERNS)
FP_PCT=$(calc_pct $FP_COUNT $TOTAL_PATTERNS)
TI_PCT=$(calc_pct $TI_COUNT $TOTAL_PATTERNS)

# Output frequency summary
echo "=== Pattern Distribution ==="
echo "strategy-effectiveness: $SE_COUNT ($SE_PCT%)"
echo "failure-pattern: $FP_COUNT ($FP_PCT%)"
echo "task-insight: $TI_COUNT ($TI_PCT%)"

# Task type distribution
echo ""
echo "=== By Task Type ==="
for tt in "${!TASK_TYPE_COUNTS[@]}"; do
  echo "$tt: ${TASK_TYPE_COUNTS[$tt]}"
done

# Failure category distribution
echo ""
echo "=== By Failure Category ==="
for fc in "${!FAILURE_CAT_COUNTS[@]}"; do
  echo "$fc: ${FAILURE_CAT_COUNTS[$fc]}"
done
```
</step>

<step name="calculate_effectiveness_scores">
**Calculate effectiveness scores for strategy-effectiveness patterns**

Effectiveness score formula:
```
effectiveness_score = success_rate * confidence * log10(occurrences + 1)
```

Where:
- `success_rate` = content.success_rate (for strategy-effectiveness patterns)
- `confidence` = pattern confidence value
- `occurrences` = evidence.occurrences

For non-strategy patterns:
```
effectiveness_score = confidence * log10(occurrences + 1)
```

```bash
# Calculate and store effectiveness scores
STRATEGY_SCORES=()  # Array of "score|file|strategy_name|success_rate|confidence"

for file in "$PATTERNS_DIR/strategy-effectiveness"/*.json 2>/dev/null; do
  [ -f "$file" ] || continue

  if command -v jq &> /dev/null; then
    SUCCESS_RATE=$(jq -r '.content.success_rate // 0' "$file")
    CONFIDENCE=$(jq -r '.confidence // 0' "$file")
    OCCURRENCES=$(jq -r '.evidence.occurrences // 1' "$file")
    STRATEGY_NAME=$(jq -r '.content.strategy_name // "unknown"' "$file")
    PATTERN_ID=$(jq -r '.id // "unknown"' "$file")

    # Calculate effectiveness score
    # score = success_rate * confidence * log10(occurrences + 1)
    # Using bc for floating point math
    LOG_OCC=$(echo "l($OCCURRENCES + 1)/l(10)" | bc -l 2>/dev/null || echo "0.3")
    SCORE=$(echo "$SUCCESS_RATE * $CONFIDENCE * $LOG_OCC" | bc -l 2>/dev/null || echo "0")

    # Store with score for sorting
    STRATEGY_SCORES+=("$SCORE|$file|$STRATEGY_NAME|$SUCCESS_RATE|$CONFIDENCE|$PATTERN_ID")
  fi
done

# Sort by score descending
IFS=$'\n' SORTED_STRATEGIES=($(printf '%s\n' "${STRATEGY_SCORES[@]}" | sort -t'|' -k1 -rn))
unset IFS

echo "Calculated effectiveness scores for ${#STRATEGY_SCORES[@]} strategy patterns"
```

**For failure-pattern and task-insight patterns:**

```bash
OTHER_SCORES=()  # "score|file|pattern_id|type|description"

# Score failure patterns
for file in "$PATTERNS_DIR/failure-pattern"/*.json 2>/dev/null; do
  [ -f "$file" ] || continue

  if command -v jq &> /dev/null; then
    CONFIDENCE=$(jq -r '.confidence // 0' "$file")
    OCCURRENCES=$(jq -r '.evidence.occurrences // 1' "$file")
    PATTERN_ID=$(jq -r '.id // "unknown"' "$file")
    FAILURE_CAT=$(jq -r '.content.failure_category // "unknown"' "$file")

    LOG_OCC=$(echo "l($OCCURRENCES + 1)/l(10)" | bc -l 2>/dev/null || echo "0.3")
    SCORE=$(echo "$CONFIDENCE * $LOG_OCC" | bc -l 2>/dev/null || echo "0")

    OTHER_SCORES+=("$SCORE|$file|$PATTERN_ID|failure-pattern|$FAILURE_CAT")
  fi
done

# Score task insights
for file in "$PATTERNS_DIR/task-insight"/*.json 2>/dev/null; do
  [ -f "$file" ] || continue

  if command -v jq &> /dev/null; then
    CONFIDENCE=$(jq -r '.confidence // 0' "$file")
    OCCURRENCES=$(jq -r '.evidence.occurrences // 1' "$file")
    PATTERN_ID=$(jq -r '.id // "unknown"' "$file")
    INSIGHT_CAT=$(jq -r '.content.insight_category // "unknown"' "$file")

    LOG_OCC=$(echo "l($OCCURRENCES + 1)/l(10)" | bc -l 2>/dev/null || echo "0.3")
    SCORE=$(echo "$CONFIDENCE * $LOG_OCC" | bc -l 2>/dev/null || echo "0")

    OTHER_SCORES+=("$SCORE|$file|$PATTERN_ID|task-insight|$INSIGHT_CAT")
  fi
done
```
</step>

<step name="identify_high_value_patterns">
**Identify top patterns by effectiveness score per type**

High-value patterns are:
1. Top 5 strategy-effectiveness patterns by score
2. Top 5 failure-pattern patterns by score
3. Top 5 task-insight patterns by score

```bash
# Top 5 strategy patterns
echo "=== High-Value Strategy Patterns (Top 5) ==="
TOP_STRATEGIES=()
count=0
for entry in "${SORTED_STRATEGIES[@]}"; do
  [ $count -ge 5 ] && break

  IFS='|' read -r score file name success_rate confidence pattern_id <<< "$entry"
  SUCCESS_PCT=$(echo "$success_rate * 100" | bc | cut -d'.' -f1)
  SCORE_FMT=$(printf "%.3f" $score)

  TOP_STRATEGIES+=("$pattern_id")
  echo "$((count+1)). $name"
  echo "   Score: $SCORE_FMT | Success: ${SUCCESS_PCT}% | Confidence: $confidence"
  echo "   ID: $pattern_id"

  count=$((count + 1))
done

# Top 5 other patterns per type
echo ""
echo "=== High-Value Failure Patterns (Top 5) ==="
IFS=$'\n' SORTED_FP=($(printf '%s\n' "${OTHER_SCORES[@]}" | grep "|failure-pattern|" | sort -t'|' -k1 -rn | head -5))
unset IFS
count=0
for entry in "${SORTED_FP[@]}"; do
  IFS='|' read -r score file pattern_id type desc <<< "$entry"
  SCORE_FMT=$(printf "%.3f" $score)
  echo "$((count+1)). $pattern_id: $desc (score: $SCORE_FMT)"
  count=$((count + 1))
done

echo ""
echo "=== High-Value Task Insights (Top 5) ==="
IFS=$'\n' SORTED_TI=($(printf '%s\n' "${OTHER_SCORES[@]}" | grep "|task-insight|" | sort -t'|' -k1 -rn | head -5))
unset IFS
count=0
for entry in "${SORTED_TI[@]}"; do
  IFS='|' read -r score file pattern_id type desc <<< "$entry"
  SCORE_FMT=$(printf "%.3f" $score)
  echo "$((count+1)). $pattern_id: $desc (score: $SCORE_FMT)"
  count=$((count + 1))
done
```
</step>

<step name="identify_cross_project_validated">
**Find patterns validated across 3+ projects**

```bash
VALIDATED_PATTERNS=()

for type_dir in strategy-effectiveness failure-pattern task-insight; do
  for file in "$PATTERNS_DIR/$type_dir"/*.json 2>/dev/null; do
    [ -f "$file" ] || continue

    if command -v jq &> /dev/null; then
      PROJECT_COUNT=$(jq -r '.evidence.project_count // 1' "$file")

      if [ "$PROJECT_COUNT" -ge 3 ]; then
        PATTERN_ID=$(jq -r '.id' "$file")

        # Get description based on type
        case "$type_dir" in
          "strategy-effectiveness")
            DESC=$(jq -r '.content.strategy_name' "$file")
            ;;
          "failure-pattern")
            DESC=$(jq -r '.content.failure_category + " on " + .content.task_type' "$file")
            ;;
          "task-insight")
            DESC=$(jq -r '.content.insight' "$file" | cut -c1-60)
            ;;
        esac

        VALIDATED_PATTERNS+=("$PATTERN_ID: $DESC ($PROJECT_COUNT projects)")
      fi
    fi
  done
done

echo ""
echo "=== Cross-Project Validated Patterns (3+ projects) ==="
if [ ${#VALIDATED_PATTERNS[@]} -eq 0 ]; then
  echo "No patterns validated across 3+ projects yet"
else
  for vp in "${VALIDATED_PATTERNS[@]}"; do
    echo "- $vp"
  done
fi
```
</step>

<step name="generate_analysis_report">
**Create markdown analysis report**

```bash
REPORT_FILE=~/.claude/gsd-knowledge/ANALYSIS_REPORT.md

cat > "$REPORT_FILE" << EOF
# Pattern Analysis Report

**Generated:** $TIMESTAMP
**Total Patterns:** $TOTAL_PATTERNS

---

## Pattern Distribution

| Type | Count | % |
|------|-------|---|
| strategy-effectiveness | $SE_COUNT | ${SE_PCT}% |
| failure-pattern | $FP_COUNT | ${FP_PCT}% |
| task-insight | $TI_COUNT | ${TI_PCT}% |

## By Task Type

| Task Type | Pattern Count |
|-----------|---------------|
EOF

# Add task type rows
for tt in "${!TASK_TYPE_COUNTS[@]}"; do
  echo "| $tt | ${TASK_TYPE_COUNTS[$tt]} |" >> "$REPORT_FILE"
done

cat >> "$REPORT_FILE" << EOF

## By Failure Category

| Failure Category | Pattern Count |
|------------------|---------------|
EOF

# Add failure category rows
for fc in "${!FAILURE_CAT_COUNTS[@]}"; do
  echo "| $fc | ${FAILURE_CAT_COUNTS[$fc]} |" >> "$REPORT_FILE"
done

cat >> "$REPORT_FILE" << EOF

---

## High-Value Patterns

### Strategy Effectiveness (Top 5)

| Rank | Strategy | Success Rate | Confidence | Score |
|------|----------|--------------|------------|-------|
EOF

# Add top 5 strategy rows
count=0
for entry in "${SORTED_STRATEGIES[@]}"; do
  [ $count -ge 5 ] && break

  IFS='|' read -r score file name success_rate confidence pattern_id <<< "$entry"
  SUCCESS_PCT=$(echo "$success_rate * 100" | bc | cut -d'.' -f1 2>/dev/null || echo "0")
  SCORE_FMT=$(printf "%.3f" $score 2>/dev/null || echo "0.000")
  CONF_FMT=$(printf "%.2f" $confidence 2>/dev/null || echo "0.00")

  echo "| $((count+1)) | $name | ${SUCCESS_PCT}% | $CONF_FMT | $SCORE_FMT |" >> "$REPORT_FILE"
  count=$((count + 1))
done

cat >> "$REPORT_FILE" << EOF

### Failure Patterns (Top 5)

| Rank | Pattern ID | Category | Score |
|------|------------|----------|-------|
EOF

# Add top 5 failure pattern rows
count=0
for entry in "${SORTED_FP[@]}"; do
  [ $count -ge 5 ] && break

  IFS='|' read -r score file pattern_id type desc <<< "$entry"
  SCORE_FMT=$(printf "%.3f" $score 2>/dev/null || echo "0.000")

  echo "| $((count+1)) | $pattern_id | $desc | $SCORE_FMT |" >> "$REPORT_FILE"
  count=$((count + 1))
done

cat >> "$REPORT_FILE" << EOF

### Task Insights (Top 5)

| Rank | Pattern ID | Category | Score |
|------|------------|----------|-------|
EOF

# Add top 5 task insight rows
count=0
for entry in "${SORTED_TI[@]}"; do
  [ $count -ge 5 ] && break

  IFS='|' read -r score file pattern_id type desc <<< "$entry"
  SCORE_FMT=$(printf "%.3f" $score 2>/dev/null || echo "0.000")

  echo "| $((count+1)) | $pattern_id | $desc | $SCORE_FMT |" >> "$REPORT_FILE"
  count=$((count + 1))
done

cat >> "$REPORT_FILE" << EOF

---

## Cross-Project Validated

Patterns validated across 3+ projects receive priority in predictive hints.

EOF

# Add validated patterns
if [ ${#VALIDATED_PATTERNS[@]} -eq 0 ]; then
  echo "No patterns validated across 3+ projects yet." >> "$REPORT_FILE"
else
  for vp in "${VALIDATED_PATTERNS[@]}"; do
    echo "- $vp" >> "$REPORT_FILE"
  done
fi

cat >> "$REPORT_FILE" << EOF

---

## Recommendations

Based on analysis:

EOF

# Generate recommendations
if [ $SE_COUNT -eq 0 ]; then
  echo "- **Build strategy knowledge:** Complete more plans with retry scenarios to build strategy effectiveness patterns" >> "$REPORT_FILE"
fi

if [ ${#VALIDATED_PATTERNS[@]} -eq 0 ]; then
  echo "- **Increase cross-project usage:** Use GSD across multiple projects to validate patterns" >> "$REPORT_FILE"
fi

if [ $FP_COUNT -gt $SE_COUNT ]; then
  echo "- **Improve recovery strategies:** More failure patterns than strategies - consider documenting recovery approaches" >> "$REPORT_FILE"
fi

# Check for low-confidence patterns (candidates for decay)
LOW_CONF_COUNT=0
for type_dir in strategy-effectiveness failure-pattern task-insight; do
  for file in "$PATTERNS_DIR/$type_dir"/*.json 2>/dev/null; do
    [ -f "$file" ] || continue
    if command -v jq &> /dev/null; then
      CONF=$(jq -r '.confidence // 0' "$file")
      if (( $(echo "$CONF < 0.5" | bc -l 2>/dev/null || echo "0") )); then
        LOW_CONF_COUNT=$((LOW_CONF_COUNT + 1))
      fi
    fi
  done
done

if [ $LOW_CONF_COUNT -gt 0 ]; then
  echo "- **Review low-confidence patterns:** $LOW_CONF_COUNT patterns below 0.5 confidence may be candidates for decay" >> "$REPORT_FILE"
fi

cat >> "$REPORT_FILE" << EOF

---

*Report generated by analyze-patterns workflow*
*Next analysis: Run \`/gsd:analyze-patterns\` after accumulating more execution data*
EOF

echo "Analysis report saved to: $REPORT_FILE"
```
</step>

<step name="update_index_with_analysis">
**Add analysis metadata to index.json**

```bash
INDEX_FILE=~/.claude/gsd-knowledge/index.json

if command -v jq &> /dev/null; then
  # Build analysis metadata
  ANALYSIS_META=$(cat << EOF
{
  "last_analysis": "$TIMESTAMP",
  "total_patterns": $TOTAL_PATTERNS,
  "by_type_counts": {
    "strategy-effectiveness": $SE_COUNT,
    "failure-pattern": $FP_COUNT,
    "task-insight": $TI_COUNT
  },
  "validated_count": ${#VALIDATED_PATTERNS[@]},
  "low_confidence_count": $LOW_CONF_COUNT,
  "high_value_strategies": $(printf '%s\n' "${TOP_STRATEGIES[@]}" | jq -R -s 'split("\n") | map(select(. != ""))')
}
EOF
)

  # Update index with analysis metadata
  jq --argjson analysis "$ANALYSIS_META" '.analysis = $analysis' "$INDEX_FILE" > "${INDEX_FILE}.tmp" && \
    mv "${INDEX_FILE}.tmp" "$INDEX_FILE"

  echo "Updated index.json with analysis metadata"
else
  echo "Warning: jq not available, skipping index.json analysis metadata update"
fi
```
</step>

<step name="update_pattern_scores">
**Update individual patterns with effectiveness scores (optional)**

For patterns that will be queried frequently, store the calculated effectiveness score:

```bash
# Only update if jq is available (for safe JSON modification)
if command -v jq &> /dev/null; then
  for entry in "${SORTED_STRATEGIES[@]}"; do
    IFS='|' read -r score file name success_rate confidence pattern_id <<< "$entry"

    # Add analysis.effectiveness_score to pattern
    SCORE_FMT=$(printf "%.4f" $score 2>/dev/null || echo "0.0000")

    jq --arg score "$SCORE_FMT" \
       --arg timestamp "$TIMESTAMP" \
       '.analysis = (.analysis // {}) | .analysis.effectiveness_score = ($score | tonumber) | .analysis.last_analyzed = $timestamp' \
       "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
  done

  # Update validated patterns
  for vp_entry in "${VALIDATED_PATTERNS[@]}"; do
    PATTERN_ID=$(echo "$vp_entry" | cut -d':' -f1)

    # Determine file path from ID prefix
    case "${PATTERN_ID:0:2}" in
      "se") TYPE_DIR="strategy-effectiveness" ;;
      "fp") TYPE_DIR="failure-pattern" ;;
      "ti") TYPE_DIR="task-insight" ;;
    esac

    PATTERN_FILE="$PATTERNS_DIR/$TYPE_DIR/${PATTERN_ID}.json"
    if [ -f "$PATTERN_FILE" ]; then
      jq --arg timestamp "$TIMESTAMP" \
         '.analysis = (.analysis // {}) | .analysis.validated = true | .analysis.last_analyzed = $timestamp' \
         "$PATTERN_FILE" > "${PATTERN_FILE}.tmp" && mv "${PATTERN_FILE}.tmp" "$PATTERN_FILE"
    fi
  done

  echo "Updated pattern files with analysis metadata"
fi
```
</step>

</process>

<output>
The workflow produces:

1. **ANALYSIS_REPORT.md** at `~/.claude/gsd-knowledge/ANALYSIS_REPORT.md`
   - Pattern distribution by type
   - Distribution by task type and failure category
   - Top 5 high-value patterns per type
   - Cross-project validated patterns
   - Actionable recommendations

2. **Updated index.json** with analysis metadata:
   - `analysis.last_analysis`: Timestamp of analysis
   - `analysis.total_patterns`: Count
   - `analysis.by_type_counts`: Distribution
   - `analysis.validated_count`: Cross-project validated count
   - `analysis.high_value_strategies`: Top strategy IDs

3. **Updated pattern files** with analysis fields:
   - `analysis.effectiveness_score`: Calculated score
   - `analysis.validated`: Boolean for 3+ project validation
   - `analysis.last_analyzed`: Timestamp
</output>

<usage>
**Manual analysis:**
```bash
# Run full analysis
# Follow workflow steps in order
```

**Future command integration:**
The `/gsd:analyze-patterns` command (Phase 8 plan 2) will invoke this workflow.

**Scheduled analysis:**
Consider running after every 5 completed milestones or monthly for active users.
</usage>

<effectiveness_formula>
## Effectiveness Score Calculation

**For strategy-effectiveness patterns:**
```
effectiveness_score = success_rate * confidence * log10(occurrences + 1)
```

**For failure-pattern and task-insight patterns:**
```
effectiveness_score = confidence * log10(occurrences + 1)
```

**Component weights:**
- `success_rate` (0.0-1.0): How often the strategy works when applied
- `confidence` (0.0-1.0): Evidence strength based on occurrences, project diversity, recency
- `log10(occurrences + 1)`: Logarithmic scaling rewards more observations but with diminishing returns

**Example calculations:**

| Pattern | Success Rate | Confidence | Occurrences | Score |
|---------|--------------|------------|-------------|-------|
| retry-with-reruns | 0.85 | 0.90 | 15 | 0.85 * 0.90 * log10(16) = 0.92 |
| reinstall-deps | 0.75 | 0.70 | 5 | 0.75 * 0.70 * log10(6) = 0.41 |
| (failure pattern) | N/A | 0.80 | 10 | 0.80 * log10(11) = 0.83 |

**Score interpretation:**
- Score > 0.8: High-value pattern, prioritize in hints
- Score 0.5-0.8: Useful pattern, include in recommendations
- Score < 0.5: Low-value pattern, candidate for decay review
</effectiveness_formula>

<validation_criteria>
## Cross-Project Validation

Patterns are marked as "validated" when:
- `evidence.project_count >= 3`

**Benefits of validation:**
1. Higher priority in predictive hints display
2. Protected from early decay
3. Marked as reliable in analysis reports

**Validation is automatic:**
- During pattern extraction, project_count is tracked
- During analysis, validated status is computed and stored
- No manual validation required
</validation_criteria>

<related_files>
- `@get-shit-done/references/pattern-schema.md` - Pattern JSON structure
- `@get-shit-done/references/knowledge-base-api.md` - Storage operations
- `@get-shit-done/workflows/extract-patterns.md` - Pattern extraction from projects
- `@get-shit-done/workflows/import-patterns.md` - Pattern import for hints
</related_files>
