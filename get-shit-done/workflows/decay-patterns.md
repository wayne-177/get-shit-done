<purpose>
Manage pattern lifecycle through confidence decay based on age and usage, removing stale patterns while protecting validated ones.
</purpose>

<prerequisites>
Before running decay, verify knowledge base exists and has patterns:

```bash
# Check knowledge base initialization
if [ ! -d ~/.claude/gsd-knowledge ]; then
  echo "ERROR: Knowledge base not initialized"
  exit 1
fi

# Check for patterns
TOTAL_PATTERNS=$(jq -r '.total_patterns // 0' ~/.claude/gsd-knowledge/index.json 2>/dev/null || echo "0")
if [ "$TOTAL_PATTERNS" -eq 0 ]; then
  echo "No patterns to decay"
  exit 0
fi

echo "Decay review: $TOTAL_PATTERNS patterns"
```
</prerequisites>

<decay_model>
## Pattern Decay Model

Patterns decay over time based on recency of observation. The decay function reduces confidence when patterns haven't been observed or used recently.

### Decay Formula

```
days_since_last_seen = now - evidence.last_seen
decay_factor = 0.5 ^ (days_since_last_seen / half_life_days)
new_confidence = base_confidence * decay_factor
```

Where:
- `half_life_days` = config.settings.decay_half_life_days (default: 182 days)
- `base_confidence` = original calculated confidence (before decay)
- `decay_factor` ranges from 1.0 (just seen) to near-zero (very old)

### Example Calculations

| Days Since Last Seen | Half-Life | Decay Factor | Original Confidence | Decayed Confidence |
|----------------------|-----------|--------------|---------------------|-------------------|
| 0 | 182 | 1.000 | 0.90 | 0.90 |
| 91 | 182 | 0.707 | 0.90 | 0.64 |
| 182 | 182 | 0.500 | 0.90 | 0.45 |
| 365 | 182 | 0.250 | 0.90 | 0.23 |
| 548 | 182 | 0.125 | 0.90 | 0.11 |

### Protection for Validated Patterns

Patterns with `analysis.validated = true` (3+ projects) receive protection:
- Decay at half rate (effective half-life is doubled)
- Never auto-deleted (only marked as warning)
- Require manual review for deletion

```
# Validated pattern decay
if pattern.analysis.validated:
  effective_half_life = half_life_days * 2
  decay_factor = 0.5 ^ (days_since_last_seen / effective_half_life)
```
</decay_model>

<lifecycle_states>
## Pattern Lifecycle States

Patterns transition through states based on decayed confidence:

| State | Confidence Range | Description | Action |
|-------|------------------|-------------|--------|
| **Active** | >= 0.5 | Pattern actively used in hints | No action needed |
| **Warning** | 0.3 - 0.5 | Pattern flagged for review | Listed in decay report |
| **Expired** | < 0.3 | Pattern marked for deletion | Deleted (unless validated) |

### State Thresholds

Configurable via `~/.claude/gsd-knowledge/config.json`:

```json
{
  "settings": {
    "decay_half_life_days": 182,
    "warning_threshold": 0.5,
    "expiry_threshold": 0.3,
    "max_patterns_per_type": 100
  }
}
```

### State Transitions

```
                 time passes
Active ─────────────────────────> Warning
(conf >= 0.5)                    (0.3 <= conf < 0.5)
                                      │
                                      │ time passes
                                      v
                                  Expired
                                (conf < 0.3)
                                      │
                    ┌────────────────┴────────────────┐
                    │                                  │
             validated=true                     validated=false
                    │                                  │
                    v                                  v
             Stays Warning                        DELETED
           (manual review)                    (audit logged)
```
</lifecycle_states>

<process>

<step name="load_configuration">
**Load decay configuration from config.json**

```bash
KB_DIR=~/.claude/gsd-knowledge
CONFIG_FILE="$KB_DIR/config.json"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
NOW_EPOCH=$(date +%s)

# Load settings with defaults
if [ -f "$CONFIG_FILE" ] && command -v jq &> /dev/null; then
  HALF_LIFE_DAYS=$(jq -r '.settings.decay_half_life_days // 182' "$CONFIG_FILE")
  WARNING_THRESHOLD=$(jq -r '.settings.warning_threshold // 0.5' "$CONFIG_FILE")
  EXPIRY_THRESHOLD=$(jq -r '.settings.expiry_threshold // 0.3' "$CONFIG_FILE")
  MAX_PER_TYPE=$(jq -r '.settings.max_patterns_per_type // 100' "$CONFIG_FILE")
else
  HALF_LIFE_DAYS=182
  WARNING_THRESHOLD=0.5
  EXPIRY_THRESHOLD=0.3
  MAX_PER_TYPE=100
fi

echo "Decay configuration:"
echo "  Half-life: $HALF_LIFE_DAYS days"
echo "  Warning threshold: $WARNING_THRESHOLD"
echo "  Expiry threshold: $EXPIRY_THRESHOLD"
echo "  Max patterns per type: $MAX_PER_TYPE"
```
</step>

<step name="load_patterns">
**Load all patterns and calculate current state**

```bash
PATTERNS_DIR="$KB_DIR/patterns"

# Arrays to track results
declare -a ACTIVE_PATTERNS
declare -a WARNING_PATTERNS
declare -a EXPIRED_PATTERNS
declare -a DELETED_PATTERNS

# Process each pattern type
for type_dir in strategy-effectiveness failure-pattern task-insight; do
  for file in "$PATTERNS_DIR/$type_dir"/*.json 2>/dev/null; do
    [ -f "$file" ] || continue

    if command -v jq &> /dev/null; then
      # Extract pattern data
      PATTERN_ID=$(jq -r '.id' "$file")
      BASE_CONFIDENCE=$(jq -r '.confidence // 0.5' "$file")
      LAST_SEEN=$(jq -r '.evidence.last_seen // .created' "$file")
      IS_VALIDATED=$(jq -r '.analysis.validated // false' "$file")

      # Store for decay calculation
      echo "$PATTERN_ID|$BASE_CONFIDENCE|$LAST_SEEN|$IS_VALIDATED|$file|$type_dir"
    fi
  done
done > /tmp/patterns_to_decay.txt

TOTAL_TO_PROCESS=$(wc -l < /tmp/patterns_to_decay.txt | tr -d ' ')
echo "Loaded $TOTAL_TO_PROCESS patterns for decay review"
```
</step>

<step name="calculate_decay">
**Calculate decayed confidence for each pattern**

```bash
# Function to calculate days between dates
days_between() {
  local date1=$1
  local date2=$2

  # Convert ISO dates to epoch
  local epoch1=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$date1" +%s 2>/dev/null || \
                 date -d "$date1" +%s 2>/dev/null || echo "$NOW_EPOCH")
  local epoch2=$date2

  # Calculate days
  echo $(( (epoch2 - epoch1) / 86400 ))
}

# Process each pattern
while IFS='|' read -r pattern_id base_conf last_seen is_validated file type_dir; do
  # Calculate days since last seen
  DAYS_SINCE=$(days_between "$last_seen" "$NOW_EPOCH")

  # Determine effective half-life (doubled for validated patterns)
  if [ "$is_validated" = "true" ]; then
    EFFECTIVE_HALF_LIFE=$((HALF_LIFE_DAYS * 2))
  else
    EFFECTIVE_HALF_LIFE=$HALF_LIFE_DAYS
  fi

  # Calculate decay factor: 0.5 ^ (days / half_life)
  # Using bc for floating point
  DECAY_FACTOR=$(echo "scale=4; e(l(0.5) * $DAYS_SINCE / $EFFECTIVE_HALF_LIFE)" | bc -l 2>/dev/null || echo "1.0")

  # Calculate new confidence
  NEW_CONFIDENCE=$(echo "scale=4; $base_conf * $DECAY_FACTOR" | bc -l 2>/dev/null || echo "$base_conf")

  # Determine state based on new confidence
  if (( $(echo "$NEW_CONFIDENCE >= $WARNING_THRESHOLD" | bc -l) )); then
    STATE="active"
    ACTIVE_PATTERNS+=("$pattern_id|$NEW_CONFIDENCE|$DAYS_SINCE")
  elif (( $(echo "$NEW_CONFIDENCE >= $EXPIRY_THRESHOLD" | bc -l) )); then
    STATE="warning"
    # Calculate days until expiry
    DAYS_TO_EXPIRY=$(echo "scale=0; $EFFECTIVE_HALF_LIFE * l($NEW_CONFIDENCE / $EXPIRY_THRESHOLD) / l(0.5)" | bc -l 2>/dev/null || echo "?")
    WARNING_PATTERNS+=("$pattern_id|$NEW_CONFIDENCE|$DAYS_SINCE|$DAYS_TO_EXPIRY|$is_validated|$type_dir")
  else
    STATE="expired"
    EXPIRED_PATTERNS+=("$pattern_id|$NEW_CONFIDENCE|$DAYS_SINCE|$is_validated|$file|$type_dir")
  fi

  echo "$pattern_id: $STATE (confidence: $base_conf -> $NEW_CONFIDENCE, $DAYS_SINCE days old)"

done < /tmp/patterns_to_decay.txt

echo ""
echo "Decay results:"
echo "  Active: ${#ACTIVE_PATTERNS[@]}"
echo "  Warning: ${#WARNING_PATTERNS[@]}"
echo "  Expired: ${#EXPIRED_PATTERNS[@]}"
```
</step>

<step name="update_pattern_confidence">
**Update confidence values in pattern files**

```bash
echo "Updating pattern confidence values..."

while IFS='|' read -r pattern_id base_conf last_seen is_validated file type_dir; do
  # Recalculate (same as above)
  DAYS_SINCE=$(days_between "$last_seen" "$NOW_EPOCH")

  if [ "$is_validated" = "true" ]; then
    EFFECTIVE_HALF_LIFE=$((HALF_LIFE_DAYS * 2))
  else
    EFFECTIVE_HALF_LIFE=$HALF_LIFE_DAYS
  fi

  DECAY_FACTOR=$(echo "scale=4; e(l(0.5) * $DAYS_SINCE / $EFFECTIVE_HALF_LIFE)" | bc -l 2>/dev/null || echo "1.0")
  NEW_CONFIDENCE=$(echo "scale=4; $base_conf * $DECAY_FACTOR" | bc -l 2>/dev/null || echo "$base_conf")

  # Determine state
  if (( $(echo "$NEW_CONFIDENCE >= $WARNING_THRESHOLD" | bc -l) )); then
    STATE="active"
  elif (( $(echo "$NEW_CONFIDENCE >= $EXPIRY_THRESHOLD" | bc -l) )); then
    STATE="warning"
  else
    STATE="expired"
  fi

  # Update pattern file with new confidence and decay metadata
  if command -v jq &> /dev/null; then
    jq --arg conf "$NEW_CONFIDENCE" \
       --arg state "$STATE" \
       --arg timestamp "$TIMESTAMP" \
       '.confidence = ($conf | tonumber) |
        .analysis = (.analysis // {}) |
        .analysis.decay_state = $state |
        .analysis.last_decay_run = $timestamp' \
       "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
  fi

done < /tmp/patterns_to_decay.txt

echo "Pattern confidence values updated"
```
</step>

<step name="delete_expired_patterns">
**Delete expired patterns (unless validated)**

```bash
INDEX_FILE="$KB_DIR/index.json"
DELETED_COUNT=0
PROTECTED_COUNT=0

echo "Processing expired patterns..."

for entry in "${EXPIRED_PATTERNS[@]}"; do
  IFS='|' read -r pattern_id new_conf days_since is_validated file type_dir <<< "$entry"

  if [ "$is_validated" = "true" ]; then
    # Protected - don't delete, just mark as warning
    echo "PROTECTED: $pattern_id (validated pattern, stays at warning)"
    PROTECTED_COUNT=$((PROTECTED_COUNT + 1))

    # Update state to warning instead of deleting
    if command -v jq &> /dev/null; then
      jq --arg timestamp "$TIMESTAMP" \
         '.analysis.decay_state = "warning" |
          .analysis.protected_from_deletion = true |
          .analysis.protection_reason = "validated across 3+ projects"' \
         "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
    fi
  else
    # Not validated - delete
    echo "DELETING: $pattern_id (confidence: $new_conf, $days_since days old)"

    # Record deletion for audit
    DELETED_PATTERNS+=("$pattern_id|$new_conf|$days_since|$type_dir|$TIMESTAMP")

    # Remove from index
    if command -v jq &> /dev/null; then
      jq --arg id "$pattern_id" --arg type "$type_dir" \
         'del(.by_type[$type][] | select(.id == $id)) | .total_patterns -= 1' \
         "$INDEX_FILE" > "${INDEX_FILE}.tmp" && mv "${INDEX_FILE}.tmp" "$INDEX_FILE"
    fi

    # Delete file
    rm "$file"

    DELETED_COUNT=$((DELETED_COUNT + 1))
  fi
done

echo ""
echo "Deletion results:"
echo "  Deleted: $DELETED_COUNT"
echo "  Protected (validated): $PROTECTED_COUNT"
```
</step>

<step name="update_index">
**Update index.json with decay results**

```bash
if command -v jq &> /dev/null; then
  # Rebuild by_type arrays if needed (patterns may have been deleted)
  # Then update with decay metadata

  jq --arg timestamp "$TIMESTAMP" \
     --arg active "${#ACTIVE_PATTERNS[@]}" \
     --arg warning "${#WARNING_PATTERNS[@]}" \
     --arg deleted "$DELETED_COUNT" \
     --arg protected "$PROTECTED_COUNT" \
     '.last_updated = $timestamp |
      .decay = {
        "last_run": $timestamp,
        "active_count": ($active | tonumber),
        "warning_count": ($warning | tonumber),
        "deleted_count": ($deleted | tonumber),
        "protected_count": ($protected | tonumber)
      }' \
     "$INDEX_FILE" > "${INDEX_FILE}.tmp" && mv "${INDEX_FILE}.tmp" "$INDEX_FILE"

  echo "Updated index.json with decay metadata"
fi
```
</step>

<step name="generate_decay_report">
**Create decay report for user review**

```bash
REPORT_FILE="$KB_DIR/DECAY_REPORT.md"

cat > "$REPORT_FILE" << EOF
# Pattern Decay Report

**Run:** $TIMESTAMP
**Patterns Reviewed:** $TOTAL_TO_PROCESS

---

## Configuration

| Setting | Value |
|---------|-------|
| Half-life | $HALF_LIFE_DAYS days |
| Warning threshold | $WARNING_THRESHOLD |
| Expiry threshold | $EXPIRY_THRESHOLD |
| Max patterns per type | $MAX_PER_TYPE |

---

## Status Summary

| Status | Count |
|--------|-------|
| Active | ${#ACTIVE_PATTERNS[@]} |
| Warning | ${#WARNING_PATTERNS[@]} |
| Expired (deleted) | $DELETED_COUNT |
| Protected (validated) | $PROTECTED_COUNT |

---

## Warning Patterns (Review Recommended)

Patterns approaching expiry threshold. Consider:
- Refreshing with new observations (use in more projects)
- Manual retention if still valuable
- Allowing natural decay if outdated

| ID | Type | Confidence | Days Old | Days Until Expiry | Validated |
|----|------|------------|----------|-------------------|-----------|
EOF

# Add warning pattern rows
for entry in "${WARNING_PATTERNS[@]}"; do
  IFS='|' read -r pattern_id new_conf days_since days_to_expiry is_validated type_dir <<< "$entry"
  CONF_FMT=$(printf "%.3f" $new_conf 2>/dev/null || echo "$new_conf")
  echo "| $pattern_id | $type_dir | $CONF_FMT | $days_since | $days_to_expiry | $is_validated |" >> "$REPORT_FILE"
done

if [ ${#WARNING_PATTERNS[@]} -eq 0 ]; then
  echo "| (none) | - | - | - | - | - |" >> "$REPORT_FILE"
fi

cat >> "$REPORT_FILE" << EOF

---

## Deleted Patterns

Patterns that fell below expiry threshold ($EXPIRY_THRESHOLD) and were not protected by validation.

| ID | Type | Final Confidence | Days Old | Reason |
|----|------|------------------|----------|--------|
EOF

# Add deleted pattern rows
for entry in "${DELETED_PATTERNS[@]}"; do
  IFS='|' read -r pattern_id new_conf days_since type_dir del_time <<< "$entry"
  CONF_FMT=$(printf "%.3f" $new_conf 2>/dev/null || echo "$new_conf")
  echo "| $pattern_id | $type_dir | $CONF_FMT | $days_since | Confidence below $EXPIRY_THRESHOLD |" >> "$REPORT_FILE"
done

if [ $DELETED_COUNT -eq 0 ]; then
  echo "| (none deleted) | - | - | - | - |" >> "$REPORT_FILE"
fi

cat >> "$REPORT_FILE" << EOF

---

## Protected Patterns

Validated patterns (3+ projects) that would have been deleted but were protected.

| ID | Type | Confidence | Days Old | Protection Reason |
|----|------|------------|----------|-------------------|
EOF

# Add protected pattern rows (from EXPIRED_PATTERNS where validated=true)
PROTECTED_LOGGED=0
for entry in "${EXPIRED_PATTERNS[@]}"; do
  IFS='|' read -r pattern_id new_conf days_since is_validated file type_dir <<< "$entry"
  if [ "$is_validated" = "true" ]; then
    CONF_FMT=$(printf "%.3f" $new_conf 2>/dev/null || echo "$new_conf")
    echo "| $pattern_id | $type_dir | $CONF_FMT | $days_since | Validated across 3+ projects |" >> "$REPORT_FILE"
    PROTECTED_LOGGED=$((PROTECTED_LOGGED + 1))
  fi
done

if [ $PROTECTED_LOGGED -eq 0 ]; then
  echo "| (none protected) | - | - | - | - |" >> "$REPORT_FILE"
fi

cat >> "$REPORT_FILE" << EOF

---

## Recommendations

EOF

# Generate recommendations based on results
if [ ${#WARNING_PATTERNS[@]} -gt 5 ]; then
  echo "- **High warning count:** ${#WARNING_PATTERNS[@]} patterns approaching expiry. Consider running more projects to refresh observations." >> "$REPORT_FILE"
fi

if [ $DELETED_COUNT -gt 10 ]; then
  echo "- **Significant cleanup:** $DELETED_COUNT patterns deleted. This is normal for aging knowledge bases." >> "$REPORT_FILE"
fi

if [ ${#ACTIVE_PATTERNS[@]} -lt 10 ]; then
  echo "- **Low active patterns:** Only ${#ACTIVE_PATTERNS[@]} active patterns. Continue using GSD across projects to build knowledge." >> "$REPORT_FILE"
fi

if [ $DELETED_COUNT -eq 0 ] && [ ${#WARNING_PATTERNS[@]} -eq 0 ]; then
  echo "- **Healthy knowledge base:** All patterns are active with no decay concerns." >> "$REPORT_FILE"
fi

cat >> "$REPORT_FILE" << EOF

---

*Report generated by decay-patterns workflow*
*Next run: \`/gsd:analyze-patterns --decay\`*
EOF

echo "Decay report saved to: $REPORT_FILE"
```
</step>

<step name="log_audit_trail">
**Append deleted patterns to audit log**

```bash
AUDIT_FILE="$KB_DIR/DECAY_AUDIT.md"

# Create audit file if doesn't exist
if [ ! -f "$AUDIT_FILE" ]; then
  cat > "$AUDIT_FILE" << EOF
# Pattern Decay Audit Log

This file tracks all patterns deleted by the decay process for debugging and potential recovery.

---

EOF
fi

# Append this run's deletions
if [ $DELETED_COUNT -gt 0 ]; then
  cat >> "$AUDIT_FILE" << EOF

## Decay Run: $TIMESTAMP

**Configuration:**
- Half-life: $HALF_LIFE_DAYS days
- Expiry threshold: $EXPIRY_THRESHOLD

**Deleted Patterns:**

| ID | Type | Final Confidence | Days Since Last Seen |
|----|------|------------------|---------------------|
EOF

  for entry in "${DELETED_PATTERNS[@]}"; do
    IFS='|' read -r pattern_id new_conf days_since type_dir del_time <<< "$entry"
    CONF_FMT=$(printf "%.3f" $new_conf 2>/dev/null || echo "$new_conf")
    echo "| $pattern_id | $type_dir | $CONF_FMT | $days_since |" >> "$AUDIT_FILE"
  done

  echo "" >> "$AUDIT_FILE"
  echo "---" >> "$AUDIT_FILE"

  echo "Audit trail updated: $AUDIT_FILE"
fi
```
</step>

<step name="cleanup">
**Clean up temporary files**

```bash
rm -f /tmp/patterns_to_decay.txt

echo ""
echo "=== Decay Complete ==="
echo "Active: ${#ACTIVE_PATTERNS[@]}"
echo "Warning: ${#WARNING_PATTERNS[@]}"
echo "Deleted: $DELETED_COUNT"
echo "Protected: $PROTECTED_COUNT"
echo ""
echo "Reports:"
echo "  - $KB_DIR/DECAY_REPORT.md"
if [ $DELETED_COUNT -gt 0 ]; then
  echo "  - $KB_DIR/DECAY_AUDIT.md"
fi
```
</step>

</process>

<output>
The workflow produces:

1. **Updated pattern files** with decayed confidence values and state metadata
2. **Updated index.json** with decay run statistics
3. **DECAY_REPORT.md** summarizing:
   - Pattern distribution by state
   - Warning patterns that need attention
   - Deleted patterns
   - Protected validated patterns
4. **DECAY_AUDIT.md** (appended) logging all deletions for recovery
</output>

<usage>
**Via command:**
```bash
/gsd:analyze-patterns --decay
```

**Manual execution:**
Follow workflow steps in order. Requires jq and bc for full functionality.

**Recommended frequency:**
- After every 5 completed milestones
- Monthly for active users
- Before major new projects (clean stale patterns)
</usage>

<error_handling>
- **jq not available:** Skip JSON updates, provide warnings
- **bc not available:** Use fallback calculations (less precise)
- **Permission errors:** Log and skip problematic files
- **Corrupted pattern files:** Skip and log for manual review
- **Never blocks:** Decay is best-effort, failures don't stop workflow
</error_handling>

<related_files>
- `@get-shit-done/workflows/analyze-patterns.md` - Pattern analysis workflow
- `@get-shit-done/references/pattern-schema.md` - Pattern structure
- `@get-shit-done/references/knowledge-base-api.md` - Storage operations
- `@get-shit-done/commands/analyze-patterns.md` - User command
</related_files>
