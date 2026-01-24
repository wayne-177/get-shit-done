# Pattern Extraction Workflow

Extract patterns from project EXECUTION_HISTORY.md to the global knowledge base for cross-project learning.

## Purpose

This workflow parses EXECUTION_HISTORY.md, identifies patterns meeting extraction criteria, normalizes them to project-agnostic format, and writes them to `~/.claude/gsd-knowledge/` for use across all projects.

## Prerequisites

Before extracting patterns:

1. **EXECUTION_HISTORY.md must exist:**
   ```bash
   if [ ! -f .planning/EXECUTION_HISTORY.md ]; then
     echo "No EXECUTION_HISTORY.md found - nothing to extract"
     echo "Execution history is created during task execution with retries/failures"
     exit 0
   fi
   ```

2. **Adaptive features must be enabled:**
   ```bash
   ADAPTIVE_ENABLED=$(cat .planning/config.json 2>/dev/null | grep -o '"adaptive"' || echo "")
   if [ -z "$ADAPTIVE_ENABLED" ]; then
     echo "Adaptive features not configured in config.json"
     echo "Add 'adaptive.enabled: true' to enable pattern extraction"
     exit 0
   fi

   ENABLED=$(cat .planning/config.json | grep -A1 '"adaptive"' | grep -o '"enabled": *true' || echo "")
   if [ -z "$ENABLED" ]; then
     echo "Adaptive features disabled (config.adaptive.enabled = false)"
     echo "Enable to use cross-project learning features"
     exit 0
   fi
   ```

3. **Initialize knowledge base:**
   ```bash
   # Ensure knowledge base directory exists
   mkdir -p ~/.claude/gsd-knowledge/patterns/strategy-effectiveness
   mkdir -p ~/.claude/gsd-knowledge/patterns/failure-pattern
   mkdir -p ~/.claude/gsd-knowledge/patterns/task-insight

   # Create index.json if missing
   if [ ! -f ~/.claude/gsd-knowledge/index.json ]; then
     cat > ~/.claude/gsd-knowledge/index.json << 'EOF'
   {
     "version": "1.0.0",
     "last_updated": "TIMESTAMP",
     "total_patterns": 0,
     "by_type": {
       "strategy-effectiveness": [],
       "failure-pattern": [],
       "task-insight": []
     },
     "by_task_type": {},
     "by_failure_category": {}
   }
   EOF
     sed -i '' "s/TIMESTAMP/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" ~/.claude/gsd-knowledge/index.json 2>/dev/null || \
     sed -i "s/TIMESTAMP/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" ~/.claude/gsd-knowledge/index.json
   fi

   # Create config.json if missing
   if [ ! -f ~/.claude/gsd-knowledge/config.json ]; then
     cat > ~/.claude/gsd-knowledge/config.json << 'EOF'
   {
     "version": "1.0.0",
     "created": "TIMESTAMP",
     "settings": {
       "auto_extract": false,
       "min_confidence_for_extraction": 0.7,
       "max_patterns_per_type": 100,
       "pattern_expiry_days": 365
     },
     "statistics": {
       "total_extractions": 0,
       "projects_contributing": 0,
       "last_extraction": null
     }
   }
   EOF
     sed -i '' "s/TIMESTAMP/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" ~/.claude/gsd-knowledge/config.json 2>/dev/null || \
     sed -i "s/TIMESTAMP/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" ~/.claude/gsd-knowledge/config.json
   fi

   echo "Knowledge base initialized"
   ```

## Extraction Process

### Step 1: Parse Strategy Effectiveness Table

Extract rows from the `## Strategy Effectiveness` table in EXECUTION_HISTORY.md.

**Table format:**
```
| Strategy Name | Times Used | Success Rate | Avg Attempt # | Task Types | Notes |
```

**Extraction logic:**
```bash
# Extract Strategy Effectiveness table rows (skip header and separator)
HISTORY_FILE=".planning/EXECUTION_HISTORY.md"

# Find the Strategy Effectiveness section and extract table rows
sed -n '/^## Strategy Effectiveness/,/^## /p' "$HISTORY_FILE" | \
  grep '^|' | \
  grep -v '^| Strategy Name' | \
  grep -v '^|[-|]*$' | \
  grep -v '<!-- Example' | \
  while IFS='|' read -r _ strategy times_used success_rate avg_attempt task_types notes _; do
    # Trim whitespace
    strategy=$(echo "$strategy" | xargs)
    times_used=$(echo "$times_used" | xargs)
    success_rate=$(echo "$success_rate" | xargs)
    avg_attempt=$(echo "$avg_attempt" | xargs)
    task_types=$(echo "$task_types" | xargs)
    notes=$(echo "$notes" | xargs)

    # Skip if empty or below threshold
    [ -z "$strategy" ] && continue
    [ "$times_used" -lt 3 ] 2>/dev/null && continue

    echo "Found strategy: $strategy (used $times_used times)"

    # Extract numeric values
    # success_rate format: "80% (4/5)" or "80%"
    SUCCESS_PCT=$(echo "$success_rate" | grep -oE '[0-9]+%' | tr -d '%')
    SUCCESS_DECIMAL=$(echo "scale=2; $SUCCESS_PCT / 100" | bc)

    # Parse total uses from success_rate if in format "80% (4/5)"
    if echo "$success_rate" | grep -q '('; then
      SUCCESSFUL=$(echo "$success_rate" | grep -oE '\([0-9]+/' | tr -d '(/')
      TOTAL=$(echo "$success_rate" | grep -oE '/[0-9]+\)' | tr -d '/)')
    else
      TOTAL=$times_used
      SUCCESSFUL=$(echo "scale=0; $TOTAL * $SUCCESS_DECIMAL" | bc | cut -d. -f1)
    fi

    # Parse task types (comma-separated)
    TASK_TYPES_JSON=$(echo "$task_types" | sed 's/,/","/g' | sed 's/^/["/' | sed 's/$/"]/')

    echo "  Success rate: $SUCCESS_DECIMAL, Uses: $TOTAL, Successful: $SUCCESSFUL"
  done
```

**Pattern generation for each row:**
```bash
# Generate pattern ID
PATTERN_CONTENT="strategy_name:${strategy},task_types:${task_types}"
HASH=$(echo -n "$PATTERN_CONTENT" | shasum -a 256 | cut -c1-8)
PATTERN_ID="se-${HASH}"

# Calculate confidence (single project extraction)
# base_confidence = min(occurrences/10, 1.0) * 0.5
# project_diversity = min(1/3, 1.0) * 0.3 = 0.1 (single project)
# recency = 1.0 * 0.2 = 0.2 (just extracted)
BASE=$(echo "scale=2; if ($TOTAL/10 < 1) $TOTAL/10 else 1" | bc)
BASE_CONF=$(echo "scale=2; $BASE * 0.5" | bc)
DIVERSITY_CONF="0.10"  # Single project = 1/3 * 0.3
RECENCY_CONF="0.20"    # Just extracted = recent
CONFIDENCE=$(echo "scale=2; $BASE_CONF + $DIVERSITY_CONF + $RECENCY_CONF" | bc)

# Build pattern JSON
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
cat > /tmp/pattern-${PATTERN_ID}.json << EOF
{
  "id": "${PATTERN_ID}",
  "type": "strategy-effectiveness",
  "version": "1.0.0",
  "created": "${NOW}",
  "last_updated": "${NOW}",
  "confidence": ${CONFIDENCE},
  "evidence": {
    "occurrences": ${TOTAL},
    "project_count": 1,
    "first_seen": "${NOW}",
    "last_seen": "${NOW}"
  },
  "content": {
    "strategy_name": "${strategy}",
    "task_types": ${TASK_TYPES_JSON},
    "failure_categories": [],
    "success_rate": ${SUCCESS_DECIMAL},
    "avg_attempt_number": ${avg_attempt:-1.0},
    "total_uses": ${TOTAL},
    "successful_uses": ${SUCCESSFUL},
    "description": "${notes}"
  },
  "tags": []
}
EOF
```

### Step 2: Parse Common Failure Patterns Table

Extract rows from the `## Common Failure Patterns` table.

**Table format:**
```
| Failure Category | Task Type | Frequency | Typical Recovery | Last Seen | Notes |
```

**Extraction logic:**
```bash
# Extract Common Failure Patterns table rows
sed -n '/^## Common Failure Patterns/,/^## /p' "$HISTORY_FILE" | \
  grep '^|' | \
  grep -v '^| Failure Category' | \
  grep -v '^|[-|]*$' | \
  grep -v '<!-- Example' | \
  while IFS='|' read -r _ failure_cat task_type frequency typical_recovery last_seen notes _; do
    # Trim whitespace
    failure_cat=$(echo "$failure_cat" | xargs)
    task_type=$(echo "$task_type" | xargs)
    frequency=$(echo "$frequency" | xargs)
    typical_recovery=$(echo "$typical_recovery" | xargs)
    last_seen=$(echo "$last_seen" | xargs)
    notes=$(echo "$notes" | xargs)

    # Skip if empty or below threshold
    [ -z "$failure_cat" ] && continue
    [ "$frequency" -lt 3 ] 2>/dev/null && continue

    echo "Found failure pattern: $failure_cat + $task_type (frequency: $frequency)"
  done
```

**Pattern generation:**
```bash
# Generate pattern ID for failure pattern
PATTERN_CONTENT="failure_category:${failure_cat},task_type:${task_type}"
HASH=$(echo -n "$PATTERN_CONTENT" | shasum -a 256 | cut -c1-8)
PATTERN_ID="fp-${HASH}"

# Calculate confidence (same formula, single project)
BASE=$(echo "scale=2; if ($frequency/10 < 1) $frequency/10 else 1" | bc)
BASE_CONF=$(echo "scale=2; $BASE * 0.5" | bc)
CONFIDENCE=$(echo "scale=2; $BASE_CONF + 0.10 + 0.20" | bc)

# Build pattern JSON
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
cat > /tmp/pattern-${PATTERN_ID}.json << EOF
{
  "id": "${PATTERN_ID}",
  "type": "failure-pattern",
  "version": "1.0.0",
  "created": "${NOW}",
  "last_updated": "${NOW}",
  "confidence": ${CONFIDENCE},
  "evidence": {
    "occurrences": ${frequency},
    "project_count": 1,
    "first_seen": "${NOW}",
    "last_seen": "${NOW}"
  },
  "content": {
    "failure_category": "${failure_cat}",
    "task_type": "${task_type}",
    "frequency": ${frequency},
    "typical_recovery": "${typical_recovery}",
    "error_signatures": [],
    "root_causes": [],
    "escalation_appropriate": false,
    "notes": "${notes}"
  },
  "tags": []
}
EOF
```

### Step 3: Parse Task Type Analysis Sections

Extract insights from the `## Task Type Analysis` sections for each task type.

**Section format:**
```
### bash-command
**Success Rate:** X% (successes/total)
**Common Failures:**
- [category]: [description] ([frequency])
**Effective Strategies:**
- [strategy]: [success-rate] ([description])
**Insights:**
- [insight text]
```

**Extraction logic:**
```bash
# Extract insights for each task type section
for TASK_TYPE in bash-command file-edit test-run other; do
  # Find section and extract Insights
  sed -n "/^### ${TASK_TYPE}/,/^### /p" "$HISTORY_FILE" | \
    sed -n '/^\*\*Insights:\*\*/,/^\*\*/p' | \
    grep '^- ' | \
    sed 's/^- //' | \
    while read -r insight; do
      [ -z "$insight" ] && continue

      echo "Found insight for $TASK_TYPE: $insight"

      # Determine insight category based on keywords
      if echo "$insight" | grep -qiE 'best|should|always|prefer'; then
        CATEGORY="best-practice"
      elif echo "$insight" | grep -qiE 'avoid|don.t|never|pitfall|fail'; then
        CATEGORY="common-pitfall"
      elif echo "$insight" | grep -qiE 'faster|optimize|improve|performance'; then
        CATEGORY="optimization"
      else
        CATEGORY="best-practice"  # Default
      fi

      # Generate pattern
      PATTERN_CONTENT="task_type:${TASK_TYPE},category:${CATEGORY},insight:${insight}"
      HASH=$(echo -n "$PATTERN_CONTENT" | shasum -a 256 | cut -c1-8)
      PATTERN_ID="ti-${HASH}"

      echo "  Category: $CATEGORY, ID: $PATTERN_ID"
    done
done
```

**Pattern generation:**
```bash
# Build task-insight pattern JSON
NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
# Single observation = low base confidence
CONFIDENCE="0.50"  # 1 occurrence + single project + recent

cat > /tmp/pattern-${PATTERN_ID}.json << EOF
{
  "id": "${PATTERN_ID}",
  "type": "task-insight",
  "version": "1.0.0",
  "created": "${NOW}",
  "last_updated": "${NOW}",
  "confidence": ${CONFIDENCE},
  "evidence": {
    "occurrences": 1,
    "project_count": 1,
    "first_seen": "${NOW}",
    "last_seen": "${NOW}"
  },
  "content": {
    "task_type": "${TASK_TYPE}",
    "insight_category": "${CATEGORY}",
    "insight": "${insight}",
    "actionable_recommendation": "",
    "applies_when": [],
    "source_examples": []
  },
  "tags": []
}
EOF
```

### Step 4: Deduplicate and Merge

Before writing each pattern, check if a similar pattern already exists.

**Deduplication rules by type:**

| Type | Match criteria |
|------|----------------|
| strategy-effectiveness | Same `strategy_name` |
| failure-pattern | Same `failure_category` + `task_type` |
| task-insight | Same `task_type` + `insight_category` + similar `insight` |

**Deduplication logic:**
```bash
# For strategy-effectiveness patterns
check_duplicate_strategy() {
  local STRATEGY_NAME="$1"
  local PATTERN_DIR=~/.claude/gsd-knowledge/patterns/strategy-effectiveness

  # Search for existing pattern with same strategy name
  EXISTING=$(grep -l "\"strategy_name\": *\"${STRATEGY_NAME}\"" \
    "$PATTERN_DIR"/*.json 2>/dev/null | head -1)

  if [ -n "$EXISTING" ]; then
    echo "$EXISTING"  # Return path to existing pattern
  else
    echo ""  # No duplicate
  fi
}

# For failure-pattern patterns
check_duplicate_failure() {
  local FAILURE_CAT="$1"
  local TASK_TYPE="$2"
  local PATTERN_DIR=~/.claude/gsd-knowledge/patterns/failure-pattern

  for file in "$PATTERN_DIR"/*.json; do
    [ -f "$file" ] || continue
    if grep -q "\"failure_category\": *\"${FAILURE_CAT}\"" "$file" && \
       grep -q "\"task_type\": *\"${TASK_TYPE}\"" "$file"; then
      echo "$file"
      return
    fi
  done
  echo ""
}

# For task-insight patterns
check_duplicate_insight() {
  local TASK_TYPE="$1"
  local CATEGORY="$2"
  local INSIGHT="$3"
  local PATTERN_DIR=~/.claude/gsd-knowledge/patterns/task-insight

  for file in "$PATTERN_DIR"/*.json; do
    [ -f "$file" ] || continue
    if grep -q "\"task_type\": *\"${TASK_TYPE}\"" "$file" && \
       grep -q "\"insight_category\": *\"${CATEGORY}\"" "$file"; then
      # Check if insight is similar (first 50 chars match)
      EXISTING_INSIGHT=$(grep -o '"insight": *"[^"]*"' "$file" | cut -d'"' -f4)
      if [ "${EXISTING_INSIGHT:0:50}" = "${INSIGHT:0:50}" ]; then
        echo "$file"
        return
      fi
    fi
  done
  echo ""
}
```

**Merge logic:**
```bash
# Merge new pattern into existing pattern
merge_patterns() {
  local EXISTING_FILE="$1"
  local NEW_PATTERN_FILE="$2"

  # Read existing pattern values
  if command -v jq &> /dev/null; then
    OLD_OCCURRENCES=$(jq -r '.evidence.occurrences' "$EXISTING_FILE")
    OLD_PROJECT_COUNT=$(jq -r '.evidence.project_count' "$EXISTING_FILE")
    OLD_TOTAL_USES=$(jq -r '.content.total_uses // 0' "$EXISTING_FILE")
    OLD_SUCCESSFUL=$(jq -r '.content.successful_uses // 0' "$EXISTING_FILE")

    NEW_OCCURRENCES=$(jq -r '.evidence.occurrences' "$NEW_PATTERN_FILE")
    NEW_TOTAL_USES=$(jq -r '.content.total_uses // 0' "$NEW_PATTERN_FILE")
    NEW_SUCCESSFUL=$(jq -r '.content.successful_uses // 0' "$NEW_PATTERN_FILE")

    # Merge values
    MERGED_OCCURRENCES=$((OLD_OCCURRENCES + NEW_OCCURRENCES))
    MERGED_PROJECT_COUNT=$((OLD_PROJECT_COUNT + 1))
    MERGED_TOTAL=$((OLD_TOTAL_USES + NEW_TOTAL_USES))
    MERGED_SUCCESSFUL=$((OLD_SUCCESSFUL + NEW_SUCCESSFUL))

    # Recalculate success rate
    if [ "$MERGED_TOTAL" -gt 0 ]; then
      MERGED_RATE=$(echo "scale=2; $MERGED_SUCCESSFUL / $MERGED_TOTAL" | bc)
    else
      MERGED_RATE="0.00"
    fi

    # Recalculate confidence
    BASE=$(echo "scale=2; if ($MERGED_OCCURRENCES/10 < 1) $MERGED_OCCURRENCES/10 else 1" | bc)
    BASE_CONF=$(echo "scale=2; $BASE * 0.5" | bc)
    DIVERSITY=$(echo "scale=2; if ($MERGED_PROJECT_COUNT/3 < 1) $MERGED_PROJECT_COUNT/3 else 1" | bc)
    DIVERSITY_CONF=$(echo "scale=2; $DIVERSITY * 0.3" | bc)
    RECENCY_CONF="0.20"
    NEW_CONFIDENCE=$(echo "scale=2; $BASE_CONF + $DIVERSITY_CONF + $RECENCY_CONF" | bc)

    # Update existing pattern
    NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    jq --argjson occ "$MERGED_OCCURRENCES" \
       --argjson proj "$MERGED_PROJECT_COUNT" \
       --argjson total "$MERGED_TOTAL" \
       --argjson succ "$MERGED_SUCCESSFUL" \
       --argjson rate "$MERGED_RATE" \
       --argjson conf "$NEW_CONFIDENCE" \
       --arg now "$NOW" \
       '.evidence.occurrences = $occ |
        .evidence.project_count = $proj |
        .evidence.last_seen = $now |
        .last_updated = $now |
        .confidence = $conf |
        .content.total_uses = $total |
        .content.successful_uses = $succ |
        .content.success_rate = $rate' \
       "$EXISTING_FILE" > "${EXISTING_FILE}.tmp" && \
       mv "${EXISTING_FILE}.tmp" "$EXISTING_FILE"

    echo "Merged: occurrences=$MERGED_OCCURRENCES, projects=$MERGED_PROJECT_COUNT, confidence=$NEW_CONFIDENCE"
  else
    echo "Warning: jq not available, skipping merge"
  fi
}
```

### Step 5: Write Patterns to Knowledge Base

Write new patterns or merged updates to the knowledge base.

```bash
write_pattern() {
  local PATTERN_FILE="$1"
  local PATTERN_ID=$(grep -o '"id": *"[^"]*"' "$PATTERN_FILE" | cut -d'"' -f4)
  local PATTERN_TYPE=$(grep -o '"type": *"[^"]*"' "$PATTERN_FILE" | cut -d'"' -f4)

  local DEST_DIR=~/.claude/gsd-knowledge/patterns/${PATTERN_TYPE}
  local DEST_FILE="${DEST_DIR}/${PATTERN_ID}.json"

  # Copy pattern to knowledge base
  cp "$PATTERN_FILE" "$DEST_FILE"

  echo "Wrote pattern: $DEST_FILE"

  # Update index.json
  update_index "$PATTERN_ID" "$PATTERN_TYPE"
}

update_index() {
  local PATTERN_ID="$1"
  local PATTERN_TYPE="$2"
  local INDEX_FILE=~/.claude/gsd-knowledge/index.json

  if command -v jq &> /dev/null; then
    # Check if pattern already in index
    if jq -e ".by_type[\"$PATTERN_TYPE\"][] | select(.id == \"$PATTERN_ID\")" "$INDEX_FILE" >/dev/null 2>&1; then
      # Already indexed, just update timestamp
      jq --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
         '.last_updated = $now' \
         "$INDEX_FILE" > "${INDEX_FILE}.tmp" && mv "${INDEX_FILE}.tmp" "$INDEX_FILE"
    else
      # Add to index
      jq --arg id "$PATTERN_ID" \
         --arg type "$PATTERN_TYPE" \
         --arg file "patterns/${PATTERN_TYPE}/${PATTERN_ID}.json" \
         --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
         '.by_type[$type] += [{"id": $id, "file": $file}] |
          .total_patterns += 1 |
          .last_updated = $now' \
         "$INDEX_FILE" > "${INDEX_FILE}.tmp" && mv "${INDEX_FILE}.tmp" "$INDEX_FILE"
    fi
  else
    echo "Warning: jq not available, index.json not updated"
  fi
}
```

### Step 6: Track Extraction Results

Update statistics in knowledge base config.json.

```bash
update_extraction_stats() {
  local NEW_PATTERNS="$1"
  local MERGED_PATTERNS="$2"
  local CONFIG_FILE=~/.claude/gsd-knowledge/config.json

  if command -v jq &> /dev/null; then
    NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)

    # Read current stats
    CURRENT_EXTRACTIONS=$(jq -r '.statistics.total_extractions' "$CONFIG_FILE")
    CURRENT_PROJECTS=$(jq -r '.statistics.projects_contributing' "$CONFIG_FILE")

    # Increment
    NEW_EXTRACTIONS=$((CURRENT_EXTRACTIONS + 1))
    NEW_PROJECTS=$((CURRENT_PROJECTS + 1))

    # Update config
    jq --argjson ext "$NEW_EXTRACTIONS" \
       --argjson proj "$NEW_PROJECTS" \
       --arg now "$NOW" \
       '.statistics.total_extractions = $ext |
        .statistics.projects_contributing = $proj |
        .statistics.last_extraction = $now' \
       "$CONFIG_FILE" > "${CONFIG_FILE}.tmp" && mv "${CONFIG_FILE}.tmp" "$CONFIG_FILE"
  fi
}
```

### Step 7: Report Results

Display extraction summary to user.

```bash
report_extraction_results() {
  local SE_NEW="$1"
  local SE_MERGED="$2"
  local FP_NEW="$3"
  local FP_MERGED="$4"
  local TI_NEW="$5"
  local TI_MERGED="$6"

  local TOTAL_NEW=$((SE_NEW + FP_NEW + TI_NEW))
  local TOTAL_MERGED=$((SE_MERGED + FP_MERGED + TI_MERGED))

  cat << EOF
═══════════════════════════════════════════════════
PATTERN EXTRACTION COMPLETE
═══════════════════════════════════════════════════

Patterns extracted from: .planning/EXECUTION_HISTORY.md
Knowledge base: ~/.claude/gsd-knowledge/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RESULTS BY TYPE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Strategy Effectiveness:  ${SE_NEW} new, ${SE_MERGED} merged
Failure Patterns:        ${FP_NEW} new, ${FP_MERGED} merged
Task Insights:           ${TI_NEW} new, ${TI_MERGED} merged

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total new patterns:    ${TOTAL_NEW}
Total merged:          ${TOTAL_MERGED}
Total processed:       $((TOTAL_NEW + TOTAL_MERGED))

Patterns are now available for cross-project learning.
EOF

  # List high-confidence patterns if any
  if command -v jq &> /dev/null; then
    HIGH_CONF=$(find ~/.claude/gsd-knowledge/patterns -name "*.json" -exec jq -r 'select(.confidence > 0.8) | "\(.id): \(.content.strategy_name // .content.failure_category // .content.task_type) (confidence: \(.confidence))"' {} \; 2>/dev/null)

    if [ -n "$HIGH_CONF" ]; then
      echo ""
      echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
      echo "HIGH-CONFIDENCE PATTERNS (>0.8)"
      echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
      echo "$HIGH_CONF"
    fi
  fi
}
```

## Extraction Thresholds

Patterns are only extracted when they meet minimum thresholds:

| Metric | Threshold | Rationale |
|--------|-----------|-----------|
| Times Used / Frequency | >= 3 | Pattern must be observed multiple times |
| Confidence (calculated) | >= 0.5 | Minimum certainty for knowledge base |
| Success Rate (strategy) | >= 0.3 | Strategy must show some effectiveness |

Patterns below thresholds are skipped with a note:
```
Skipped: [pattern] - below threshold (times_used=2, min=3)
```

## Error Handling

All extraction operations are **best-effort**:

1. **Parse errors**: Skip malformed rows, log warning, continue
2. **File write errors**: Log error, continue with remaining patterns
3. **Index update errors**: Pattern exists but index inconsistent (repaired on next init)
4. **Knowledge base unavailable**: Log warning, exit gracefully

**Never block** the primary workflow for extraction failures.

## Integration Points

This workflow can be invoked:

1. **Manually**: Via `/gsd:extract-patterns` command at any time
2. **After phase completion**: Suggested in `execute-phase.md` after phases with failures
3. **On milestone completion**: Could be added to `complete-milestone.md` workflow
4. **Before new project**: Import accumulated patterns for predictive planning

## Related Files

- `@get-shit-done/references/pattern-schema.md` - Pattern JSON schema
- `@get-shit-done/references/knowledge-base-api.md` - Storage API operations
- `@get-shit-done/templates/execution-history.md` - Source data format
- `@get-shit-done/workflows/init-knowledge-base.md` - Knowledge base initialization
- `@get-shit-done/commands/extract-patterns.md` - User-facing command
