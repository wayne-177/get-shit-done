<purpose>
Parse codebase analysis findings, query knowledge base for related patterns, and generate prioritized goal proposals.

Transforms raw ANALYSIS_REPORT.md into actionable GOAL_PROPOSALS.md for session-start awareness.
</purpose>

<prerequisites>
Before generating suggestions, verify analysis report exists:

```bash
# Check for analysis report
if [ ! -f .planning/ANALYSIS_REPORT.md ]; then
  echo "ERROR: No analysis report found at .planning/ANALYSIS_REPORT.md"
  echo ""
  echo "Run /gsd:analyze-codebase first to generate findings."
  exit 1
fi

# Get analysis timestamp for staleness tracking
ANALYSIS_TIMESTAMP=$(grep "^\*\*Generated:\*\*" .planning/ANALYSIS_REPORT.md | sed 's/.*: *//')
echo "Analysis report found (generated: $ANALYSIS_TIMESTAMP)"

# Check config for adaptive features (knowledge base integration)
ADAPTIVE_ENABLED="false"
if [ -f .planning/config.json ]; then
  if grep -q '"adaptive"' .planning/config.json; then
    ADAPTIVE_ENABLED=$(grep -A 2 '"adaptive"' .planning/config.json | grep '"enabled"' | grep -o 'true\|false' | head -1)
  fi
fi
echo "Adaptive features: $ADAPTIVE_ENABLED"
```

**If analysis report missing:** Error with guidance to run /gsd:analyze-codebase.
**If config missing:** Continue without knowledge base (graceful degradation).
</prerequisites>

<process>

<step name="parse_recommendations">
**Extract prioritized action items from ANALYSIS_REPORT.md**

Parse the "Recommendations for Suggestion Engine" section:

```bash
# Extract recommendations section
RECOMMENDATIONS=$(sed -n '/## Recommendations for Suggestion Engine/,/^## /p' .planning/ANALYSIS_REPORT.md | head -n -1)

# Parse each prioritized item
# Format: N. **[PRIORITY] Action description**
#         - Count: [N]
#         - Effort: quick-fix | moderate | significant
#         - Command: `<command>`
#         - Goal type: security-fix | dependency-update | tech-debt-cleanup

# Extract items into structured format
ITEMS=()
CURRENT_ITEM=""

while IFS= read -r line; do
  # Check for new numbered item
  if echo "$line" | grep -qE '^\s*[0-9]+\. \*\*\['; then
    # Save previous item if exists
    if [ -n "$CURRENT_ITEM" ]; then
      ITEMS+=("$CURRENT_ITEM")
    fi
    # Start new item
    PRIORITY=$(echo "$line" | grep -oE '\[CRITICAL\]|\[HIGH\]|\[MEDIUM\]|\[LOW\]' | tr -d '[]')
    TITLE=$(echo "$line" | sed 's/^[0-9]*\. \*\*\[[A-Z]*\] //' | sed 's/\*\*$//')
    CURRENT_ITEM="PRIORITY:$PRIORITY|TITLE:$TITLE"
  # Parse item details
  elif echo "$line" | grep -qE '^\s*- Count:'; then
    COUNT=$(echo "$line" | grep -oE '[0-9]+')
    CURRENT_ITEM="$CURRENT_ITEM|COUNT:$COUNT"
  elif echo "$line" | grep -qE '^\s*- Effort:'; then
    EFFORT=$(echo "$line" | sed 's/.*: //' | tr -d ' ')
    CURRENT_ITEM="$CURRENT_ITEM|EFFORT:$EFFORT"
  elif echo "$line" | grep -qE '^\s*- Command:'; then
    COMMAND=$(echo "$line" | sed 's/.*: //' | sed 's/`//g')
    CURRENT_ITEM="$CURRENT_ITEM|COMMAND:$COMMAND"
  elif echo "$line" | grep -qE '^\s*- Goal type:'; then
    GOAL_TYPE=$(echo "$line" | sed 's/.*: //' | tr -d ' ')
    CURRENT_ITEM="$CURRENT_ITEM|GOAL_TYPE:$GOAL_TYPE"
  fi
done <<< "$RECOMMENDATIONS"

# Save last item
if [ -n "$CURRENT_ITEM" ]; then
  ITEMS+=("$CURRENT_ITEM")
fi

echo "Parsed ${#ITEMS[@]} recommendation items"
```

**Output format per item:**
```
PRIORITY:CRITICAL|TITLE:Fix critical security vulnerabilities|COUNT:2|EFFORT:quick-fix|COMMAND:npm audit fix|GOAL_TYPE:security-fix
```
</step>

<step name="query_knowledge_base">
**Query knowledge base for relevant patterns (if adaptive.enabled)**

```bash
query_knowledge_base() {
  local goal_type="$1"
  local results=""

  # Skip if adaptive not enabled
  if [ "$ADAPTIVE_ENABLED" != "true" ]; then
    echo ""
    return
  fi

  # Check if knowledge base exists
  if [ ! -d ~/.claude/gsd-knowledge ] || [ ! -f ~/.claude/gsd-knowledge/index.json ]; then
    echo ""
    return
  fi

  # Map goal types to task types for pattern queries
  local task_types=""
  case "$goal_type" in
    "security-fix")
      task_types="bash-command"
      ;;
    "dependency-update")
      task_types="bash-command"
      ;;
    "tech-debt-cleanup")
      task_types="file-edit bash-command"
      ;;
  esac

  local failure_patterns=""
  local success_strategies=""

  # Query failure patterns (what commonly goes wrong)
  for file in ~/.claude/gsd-knowledge/patterns/failure-pattern/*.json 2>/dev/null; do
    [ -f "$file" ] || continue
    for task_type in $task_types; do
      if grep -q "\"$task_type\"" "$file" 2>/dev/null; then
        if command -v jq &> /dev/null; then
          local confidence=$(jq -r '.confidence // 0' "$file")
          local failure_cat=$(jq -r '.content.failure_category // "unknown"' "$file")
          local typical_recovery=$(jq -r '.content.typical_recovery // "retry"' "$file")

          # Check confidence threshold (0.6)
          if (( $(echo "$confidence >= 0.6" | bc -l) )); then
            failure_patterns="$failure_patterns|FAILURE:$failure_cat ($typical_recovery)"
          fi
        fi
      fi
    done
  done

  # Query strategy effectiveness (what works well)
  for file in ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/*.json 2>/dev/null; do
    [ -f "$file" ] || continue
    for task_type in $task_types; do
      if grep -q "\"$task_type\"" "$file" 2>/dev/null; then
        if command -v jq &> /dev/null; then
          local confidence=$(jq -r '.confidence // 0' "$file")
          local strategy=$(jq -r '.content.strategy_name // "unknown"' "$file")
          local success_rate=$(jq -r '.content.success_rate // 0' "$file")

          if (( $(echo "$confidence >= 0.6" | bc -l) )); then
            local pct=$(echo "$success_rate * 100" | bc | cut -d'.' -f1)
            success_strategies="$success_strategies|SUCCESS:$strategy (${pct}%)"
          fi
        fi
      fi
    done
  done

  # Query task insights (best practices)
  for file in ~/.claude/gsd-knowledge/patterns/task-insight/*.json 2>/dev/null; do
    [ -f "$file" ] || continue
    for task_type in $task_types; do
      if grep -q "\"$task_type\"" "$file" 2>/dev/null; then
        if command -v jq &> /dev/null; then
          local confidence=$(jq -r '.confidence // 0' "$file")
          local insight=$(jq -r '.content.insight // ""' "$file")

          if (( $(echo "$confidence >= 0.6" | bc -l) )); then
            results="$results|INSIGHT:$insight"
          fi
        fi
      fi
    done
  done

  echo "$failure_patterns$success_strategies$results"
}

# Count total patterns consulted
PATTERN_COUNT=0
if [ "$ADAPTIVE_ENABLED" = "true" ] && [ -f ~/.claude/gsd-knowledge/index.json ]; then
  if command -v jq &> /dev/null; then
    PATTERN_COUNT=$(jq -r '.total_patterns // 0' ~/.claude/gsd-knowledge/index.json)
  else
    PATTERN_COUNT=$(grep -o '"total_patterns": *[0-9]*' ~/.claude/gsd-knowledge/index.json | grep -o '[0-9]*' || echo "0")
  fi
fi
```

**Pattern query output per goal type:**
```
|FAILURE:dependency (reinstall)|SUCCESS:retry-with-clean (85%)|INSIGHT:Always verify deps
```
</step>

<step name="score_suggestions">
**Calculate priority score for each suggestion**

Scoring formula:
- Base score from priority: CRITICAL=100, HIGH=75, MEDIUM=50, LOW=25
- Effort modifier: quick-fix=1.2x, moderate=1.0x, significant=0.8x
- Pattern boost: +10 if knowledge base has relevant success patterns
- Pattern warning: Flag if high failure rate detected

```bash
calculate_score() {
  local priority="$1"
  local effort="$2"
  local patterns="$3"

  # Base score from priority
  local base_score=0
  case "$priority" in
    "CRITICAL") base_score=100 ;;
    "HIGH") base_score=75 ;;
    "MEDIUM") base_score=50 ;;
    "LOW") base_score=25 ;;
  esac

  # Effort modifier
  local effort_modifier=1.0
  case "$effort" in
    "quick-fix") effort_modifier=1.2 ;;
    "moderate") effort_modifier=1.0 ;;
    "significant") effort_modifier=0.8 ;;
  esac

  # Pattern boost
  local pattern_boost=0
  if echo "$patterns" | grep -q "SUCCESS:"; then
    pattern_boost=10
  fi

  # Calculate final score
  local adjusted=$(echo "$base_score * $effort_modifier" | bc)
  local final=$(echo "$adjusted + $pattern_boost" | bc | cut -d'.' -f1)

  echo "$final"
}

# Check for pattern warnings (high failure rate)
check_pattern_warning() {
  local patterns="$1"

  # Count failure patterns
  local failure_count=$(echo "$patterns" | grep -o '|FAILURE:' | wc -l | tr -d ' ')

  # If more failures than successes, warn
  local success_count=$(echo "$patterns" | grep -o '|SUCCESS:' | wc -l | tr -d ' ')

  if [ "$failure_count" -gt "$success_count" ] && [ "$failure_count" -gt 0 ]; then
    echo "true"
  else
    echo "false"
  fi
}

# Score all items
SCORED_ITEMS=()
for item in "${ITEMS[@]}"; do
  # Parse item fields
  PRIORITY=$(echo "$item" | grep -oE 'PRIORITY:[^|]*' | cut -d':' -f2)
  EFFORT=$(echo "$item" | grep -oE 'EFFORT:[^|]*' | cut -d':' -f2)
  GOAL_TYPE=$(echo "$item" | grep -oE 'GOAL_TYPE:[^|]*' | cut -d':' -f2)

  # Query patterns for this goal type
  PATTERNS=$(query_knowledge_base "$GOAL_TYPE")

  # Calculate score
  SCORE=$(calculate_score "$PRIORITY" "$EFFORT" "$PATTERNS")

  # Check for warnings
  HAS_WARNING=$(check_pattern_warning "$PATTERNS")

  # Add score and patterns to item
  SCORED_ITEM="$item|SCORE:$SCORE|PATTERNS:$PATTERNS|WARNING:$HAS_WARNING"
  SCORED_ITEMS+=("$SCORED_ITEM")
done

# Sort by score (descending)
IFS=$'\n' SORTED_ITEMS=($(for item in "${SCORED_ITEMS[@]}"; do
  score=$(echo "$item" | grep -oE 'SCORE:[0-9]*' | cut -d':' -f2)
  echo "$score $item"
done | sort -rn | cut -d' ' -f2-))
unset IFS

echo "Scored ${#SORTED_ITEMS[@]} suggestions"
```
</step>

<step name="generate_proposals">
**Create GOAL_PROPOSALS.md from scored suggestions**

```bash
generate_proposals() {
  local project_name=$(basename "$(pwd)")
  local timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

  # Count by priority
  local critical_count=0
  local high_count=0
  local medium_count=0
  local low_count=0
  local quick_wins=0

  for item in "${SORTED_ITEMS[@]}"; do
    priority=$(echo "$item" | grep -oE 'PRIORITY:[^|]*' | cut -d':' -f2)
    effort=$(echo "$item" | grep -oE 'EFFORT:[^|]*' | cut -d':' -f2)

    case "$priority" in
      "CRITICAL") ((critical_count++)) ;;
      "HIGH") ((high_count++)) ;;
      "MEDIUM") ((medium_count++)) ;;
      "LOW") ((low_count++)) ;;
    esac

    if [ "$effort" = "quick-fix" ]; then
      ((quick_wins++))
    fi
  done

  local total_count=${#SORTED_ITEMS[@]}
  local pattern_status="No knowledge base"
  if [ "$ADAPTIVE_ENABLED" = "true" ] && [ "$PATTERN_COUNT" -gt 0 ]; then
    pattern_status="$PATTERN_COUNT patterns consulted"
  fi

  # Generate file header
  cat > .planning/GOAL_PROPOSALS.md << HEADER
# Goal Proposals

**Project:** $project_name
**Generated:** $timestamp
**Analysis Report:** $ANALYSIS_TIMESTAMP
**Pattern Insights:** $pattern_status

---

## Executive Summary

**Total Suggestions:** $total_count
**By Priority:** $critical_count critical, $high_count high, $medium_count medium, $low_count low
**Estimated Quick Wins:** $quick_wins (quick-fix effort)

---

## Prioritized Suggestions

<!-- Sorted by score descending -->

HEADER

  # Generate each suggestion
  local rank=1
  for item in "${SORTED_ITEMS[@]}"; do
    # Parse fields
    local priority=$(echo "$item" | grep -oE 'PRIORITY:[^|]*' | cut -d':' -f2)
    local title=$(echo "$item" | grep -oE 'TITLE:[^|]*' | cut -d':' -f2-)
    local count=$(echo "$item" | grep -oE 'COUNT:[^|]*' | cut -d':' -f2)
    local effort=$(echo "$item" | grep -oE 'EFFORT:[^|]*' | cut -d':' -f2)
    local command=$(echo "$item" | grep -oE 'COMMAND:[^|]*' | cut -d':' -f2-)
    local goal_type=$(echo "$item" | grep -oE 'GOAL_TYPE:[^|]*' | cut -d':' -f2)
    local score=$(echo "$item" | grep -oE 'SCORE:[^|]*' | cut -d':' -f2)
    local patterns=$(echo "$item" | grep -oE 'PATTERNS:[^|]*' | cut -d':' -f2-)
    local has_warning=$(echo "$item" | grep -oE 'WARNING:[^|]*' | cut -d':' -f2)

    # Calculate score breakdown
    local base_score=0
    case "$priority" in
      "CRITICAL") base_score=100 ;;
      "HIGH") base_score=75 ;;
      "MEDIUM") base_score=50 ;;
      "LOW") base_score=25 ;;
    esac

    local effort_modifier=0
    case "$effort" in
      "quick-fix") effort_modifier=20 ;;  # 100 * 0.2 = 20 bonus
      "moderate") effort_modifier=0 ;;
      "significant") effort_modifier=-20 ;;  # 100 * -0.2 = -20 penalty
    esac

    local pattern_boost=0
    if echo "$patterns" | grep -q "SUCCESS:"; then
      pattern_boost=10
    fi

    cat >> .planning/GOAL_PROPOSALS.md << SUGGESTION
### $rank. $title

**Priority:** $priority
**Score:** $score (base $base_score + effort $effort_modifier + patterns $pattern_boost)
**Goal Type:** $goal_type
**Effort:** $effort
**Count:** ${count:-1} items

**Source:** Analysis Report
**Evidence:**
- Finding from ANALYSIS_REPORT.md: $title

SUGGESTION

    # Add pattern insights if available
    if [ -n "$patterns" ] && [ "$patterns" != ":" ]; then
      echo "**Pattern Insights:**" >> .planning/GOAL_PROPOSALS.md

      # Parse and display patterns
      for pattern in $(echo "$patterns" | tr '|' '\n' | grep -E '^(FAILURE|SUCCESS|INSIGHT):'); do
        local ptype=$(echo "$pattern" | cut -d':' -f1)
        local pvalue=$(echo "$pattern" | cut -d':' -f2-)
        case "$ptype" in
          "FAILURE") echo "- Warning: $pvalue commonly fails" >> .planning/GOAL_PROPOSALS.md ;;
          "SUCCESS") echo "- Strategy: $pvalue works well" >> .planning/GOAL_PROPOSALS.md ;;
          "INSIGHT") echo "- Tip: $pvalue" >> .planning/GOAL_PROPOSALS.md ;;
        esac
      done

      if [ "$has_warning" = "true" ]; then
        echo "- **WARNING:** High failure rate detected for this type of task" >> .planning/GOAL_PROPOSALS.md
      fi
      echo "" >> .planning/GOAL_PROPOSALS.md
    fi

    # Add suggested action
    cat >> .planning/GOAL_PROPOSALS.md << ACTION

**Suggested Action:**
\`\`\`
${command:-Review and address manually}
\`\`\`

---

ACTION

    ((rank++))
  done

  # Add grouping by goal type
  cat >> .planning/GOAL_PROPOSALS.md << GROUPING

## Suggestions by Goal Type

### Security Fixes
GROUPING

  for item in "${SORTED_ITEMS[@]}"; do
    if echo "$item" | grep -q 'GOAL_TYPE:security-fix'; then
      local title=$(echo "$item" | grep -oE 'TITLE:[^|]*' | cut -d':' -f2-)
      local score=$(echo "$item" | grep -oE 'SCORE:[^|]*' | cut -d':' -f2)
      echo "- $title (score: $score)" >> .planning/GOAL_PROPOSALS.md
    fi
  done

  local sec_count=$(for item in "${SORTED_ITEMS[@]}"; do echo "$item" | grep -q 'GOAL_TYPE:security-fix' && echo 1; done | wc -l | tr -d ' ')
  if [ "$sec_count" = "0" ]; then
    echo "_None_" >> .planning/GOAL_PROPOSALS.md
  fi

  cat >> .planning/GOAL_PROPOSALS.md << DEPS

### Dependency Updates
DEPS

  for item in "${SORTED_ITEMS[@]}"; do
    if echo "$item" | grep -q 'GOAL_TYPE:dependency-update'; then
      local title=$(echo "$item" | grep -oE 'TITLE:[^|]*' | cut -d':' -f2-)
      local score=$(echo "$item" | grep -oE 'SCORE:[^|]*' | cut -d':' -f2)
      echo "- $title (score: $score)" >> .planning/GOAL_PROPOSALS.md
    fi
  done

  local dep_count=$(for item in "${SORTED_ITEMS[@]}"; do echo "$item" | grep -q 'GOAL_TYPE:dependency-update' && echo 1; done | wc -l | tr -d ' ')
  if [ "$dep_count" = "0" ]; then
    echo "_None_" >> .planning/GOAL_PROPOSALS.md
  fi

  cat >> .planning/GOAL_PROPOSALS.md << DEBT

### Tech Debt Cleanup
DEBT

  for item in "${SORTED_ITEMS[@]}"; do
    if echo "$item" | grep -q 'GOAL_TYPE:tech-debt-cleanup'; then
      local title=$(echo "$item" | grep -oE 'TITLE:[^|]*' | cut -d':' -f2-)
      local score=$(echo "$item" | grep -oE 'SCORE:[^|]*' | cut -d':' -f2)
      echo "- $title (score: $score)" >> .planning/GOAL_PROPOSALS.md
    fi
  done

  local debt_count=$(for item in "${SORTED_ITEMS[@]}"; do echo "$item" | grep -q 'GOAL_TYPE:tech-debt-cleanup' && echo 1; done | wc -l | tr -d ' ')
  if [ "$debt_count" = "0" ]; then
    echo "_None_" >> .planning/GOAL_PROPOSALS.md
  fi

  # Add metadata footer
  cat >> .planning/GOAL_PROPOSALS.md << FOOTER

---

## Metadata

**Scoring Formula:**
- Base: CRITICAL=100, HIGH=75, MEDIUM=50, LOW=25
- Effort modifier: quick-fix=+20%, moderate=0%, significant=-20%
- Pattern boost: +10 if success patterns exist
- Pattern warning: Flagged if >40% failure rate in knowledge base

**Sources:**
- Analysis Report: .planning/ANALYSIS_REPORT.md
- Knowledge Base: ~/.claude/gsd-knowledge/ (if available)

**Configuration:**
- adaptive.enabled: $ADAPTIVE_ENABLED
- Confidence threshold: 0.6

---

*Generated by generate-suggestions workflow*
*Location: .planning/GOAL_PROPOSALS.md*
FOOTER

  echo "Generated .planning/GOAL_PROPOSALS.md with $total_count suggestions"
}

generate_proposals
```
</step>

<step name="handle_empty_report">
**Handle case where analysis report has no recommendations**

```bash
# Check if any items were parsed
if [ ${#ITEMS[@]} -eq 0 ]; then
  echo ""
  echo "No actionable recommendations found in analysis report."
  echo ""
  echo "This likely means your codebase is in good health."
  echo ""

  # Generate minimal proposals file
  cat > .planning/GOAL_PROPOSALS.md << EMPTY
# Goal Proposals

**Project:** $(basename "$(pwd)")
**Generated:** $(date -u +"%Y-%m-%dT%H:%M:%SZ")
**Analysis Report:** $ANALYSIS_TIMESTAMP
**Pattern Insights:** $([[ "$ADAPTIVE_ENABLED" = "true" ]] && echo "$PATTERN_COUNT patterns available" || echo "No knowledge base")

---

## Executive Summary

**Total Suggestions:** 0
**By Priority:** 0 critical, 0 high, 0 medium, 0 low
**Estimated Quick Wins:** 0

---

## Status

No actionable recommendations found in the analysis report.

This indicates your codebase is in good health:
- No critical or high-severity security vulnerabilities
- Dependencies are reasonably up to date
- Tech debt is within acceptable levels

---

## Next Steps

Consider running analysis again after:
- Major dependency updates
- New feature implementations
- Before milestone completions

---

*Generated by generate-suggestions workflow*
*Location: .planning/GOAL_PROPOSALS.md*
EMPTY

  echo "Generated empty GOAL_PROPOSALS.md (healthy codebase)"
  exit 0
fi
```
</step>

</process>

<output>
Creates `.planning/GOAL_PROPOSALS.md` containing:
- Executive summary with counts by priority
- Prioritized suggestions with scores
- Knowledge base insights (if available)
- Suggestions grouped by goal type
- Scoring formula documentation

**File location:** `.planning/GOAL_PROPOSALS.md`

**Success message:**
```
Generated .planning/GOAL_PROPOSALS.md with N suggestions
- N critical, N high, N medium, N low priority
- N quick wins identified
- N patterns consulted (if adaptive enabled)
```

**Error cases:**
- No ANALYSIS_REPORT.md → Error with guidance to run /gsd:analyze-codebase
- No recommendations → Generate minimal healthy-codebase file
- Knowledge base unavailable → Continue without pattern insights
</output>

<configuration>
Behavior configured via `.planning/config.json`:

```json
{
  "adaptive": {
    "enabled": true
  }
}
```

**adaptive.enabled:**
- `true`: Query knowledge base for pattern insights, display in proposals
- `false`: Skip knowledge base queries, generate proposals from analysis only

Knowledge base location: `~/.claude/gsd-knowledge/`
Confidence threshold: 0.6 (consistent with import-patterns.md)
</configuration>

<related_files>
- `@get-shit-done/templates/analysis-report.md` - Input format (Phase 11)
- `@get-shit-done/templates/goal-proposals.md` - Output format (this phase)
- `@~/.claude/get-shit-done/workflows/import-patterns.md` - Pattern query reference
- `@~/.claude/get-shit-done/references/knowledge-base-api.md` - Storage operations
</related_files>
