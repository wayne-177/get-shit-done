# Knowledge Base Storage API Reference

This document defines the storage operations for the global knowledge base at `~/.claude/gsd-knowledge/`. These operations are implemented as bash commands and patterns for use in GSD workflows.

## Overview

The knowledge base stores cross-project patterns in JSON files. All operations are:
- **File-based**: JSON files on disk, no database required
- **Idempotent**: Safe to retry on failure
- **Best-effort**: Failures don't block primary workflow execution

**Related files:**
- `@get-shit-done/templates/knowledge-base-structure.md` - Directory structure
- `@get-shit-done/references/pattern-schema.md` - Pattern JSON schema
- `@get-shit-done/workflows/init-knowledge-base.md` - Initialization workflow

## Operations

### 1. Initialize

**Purpose:** Bootstrap the knowledge base directory structure.

**When to call:** Before any other operation, to ensure the knowledge base exists.

**Implementation:** Delegates to `init-knowledge-base.md` workflow.

**Bash command:**
```bash
# Quick check if initialization needed
if [ -d ~/.claude/gsd-knowledge ] && [ -f ~/.claude/gsd-knowledge/index.json ] && [ -f ~/.claude/gsd-knowledge/config.json ]; then
  echo "Knowledge base ready"
else
  echo "Knowledge base needs initialization"
  # Follow init-knowledge-base.md workflow steps
fi
```

**Inline initialization (if workflow reference not available):**
```bash
# Create directory structure
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
```

**Error handling:**
- Permission denied → Log warning, disable knowledge base features for session
- Disk full → Log warning, disable knowledge base features for session
- Success → Continue with other operations

**Returns:** Nothing (void operation)

---

### 2. Write Pattern

**Purpose:** Add a new pattern or update an existing one.

**Input:**
- Pattern object (conforming to pattern-schema.md)
- If `id` provided: Update existing pattern
- If `id` not provided: Generate new ID and create pattern

**Process:**
1. Validate pattern against schema
2. Generate ID if new (format: `{type-prefix}-{8-char-hash}`)
3. Check for duplicate/similar patterns (deduplication)
4. Write pattern to `patterns/{type}/{id}.json`
5. Update `index.json` with pattern metadata

**Bash commands:**

**Generate pattern ID:**
```bash
# Generate ID from content hash
PATTERN_TYPE="strategy-effectiveness"  # or "failure-pattern" or "task-insight"
PATTERN_CONTENT="strategy_name:retry-with-reruns,task_types:test-run"

# Create hash from content
HASH=$(echo -n "$PATTERN_CONTENT" | shasum -a 256 | cut -c1-8)

# Build ID with type prefix
case "$PATTERN_TYPE" in
  "strategy-effectiveness") PREFIX="se" ;;
  "failure-pattern") PREFIX="fp" ;;
  "task-insight") PREFIX="ti" ;;
esac

PATTERN_ID="${PREFIX}-${HASH}"
echo "Generated ID: $PATTERN_ID"
```

**Write pattern file:**
```bash
PATTERN_ID="se-a1b2c3d4"
PATTERN_TYPE="strategy-effectiveness"
PATTERN_FILE=~/.claude/gsd-knowledge/patterns/${PATTERN_TYPE}/${PATTERN_ID}.json

# Write pattern JSON (example - actual content from caller)
cat > "$PATTERN_FILE" << 'EOF'
{
  "id": "se-a1b2c3d4",
  "type": "strategy-effectiveness",
  "version": "1.0.0",
  "created": "2026-01-07T12:00:00Z",
  "last_updated": "2026-01-07T12:00:00Z",
  "confidence": 0.85,
  "evidence": {
    "occurrences": 5,
    "project_count": 2,
    "first_seen": "2026-01-06T10:00:00Z",
    "last_seen": "2026-01-07T12:00:00Z"
  },
  "content": {
    "strategy_name": "retry-with-reruns",
    "task_types": ["test-run"],
    "failure_categories": ["validation"],
    "success_rate": 0.80,
    "avg_attempt_number": 2.0,
    "total_uses": 5,
    "successful_uses": 4,
    "description": "Re-running tests with isolation flags resolves flaky failures"
  },
  "tags": ["jest", "flaky-tests"]
}
EOF

echo "Pattern written to: $PATTERN_FILE"
```

**Update index.json:**
```bash
# Read current index
INDEX_FILE=~/.claude/gsd-knowledge/index.json

# Use jq if available, otherwise manual update
if command -v jq &> /dev/null; then
  # Add to by_type array
  jq --arg id "$PATTERN_ID" \
     --arg type "$PATTERN_TYPE" \
     --arg file "patterns/${PATTERN_TYPE}/${PATTERN_ID}.json" \
     '.by_type[$type] += [{"id": $id, "file": $file}] | .total_patterns += 1 | .last_updated = now | todate' \
     "$INDEX_FILE" > "${INDEX_FILE}.tmp" && mv "${INDEX_FILE}.tmp" "$INDEX_FILE"
else
  # Manual update - read file, modify, write back
  # (More complex - prefer jq when available)
  echo "Warning: jq not available, index.json update may be incomplete"
fi
```

**Deduplication check:**
```bash
# Before writing, check if similar pattern exists
PATTERN_TYPE="strategy-effectiveness"
STRATEGY_NAME="retry-with-reruns"

# Search existing patterns for match
EXISTING=$(grep -l "\"strategy_name\": *\"${STRATEGY_NAME}\"" \
  ~/.claude/gsd-knowledge/patterns/${PATTERN_TYPE}/*.json 2>/dev/null | head -1)

if [ -n "$EXISTING" ]; then
  echo "DUPLICATE_FOUND: $EXISTING"
  # Merge instead of create new - increment occurrences, update stats
else
  echo "NO_DUPLICATE"
  # Safe to create new pattern
fi
```

**Error handling:**
- File write fails → Log error, return failure status
- Index update fails → Pattern file exists but index inconsistent (recoverable on next operation)
- Duplicate found → Merge patterns instead of creating new

**Returns:** Pattern ID (string) on success, empty on failure

---

### 3. Read Pattern

**Purpose:** Retrieve a pattern by its ID.

**Input:** Pattern ID (e.g., `se-a1b2c3d4`)

**Process:**
1. Parse ID to determine type from prefix
2. Construct file path
3. Read and return pattern JSON

**Bash commands:**

**Read pattern by ID:**
```bash
PATTERN_ID="se-a1b2c3d4"

# Determine type from prefix
case "${PATTERN_ID:0:2}" in
  "se") PATTERN_TYPE="strategy-effectiveness" ;;
  "fp") PATTERN_TYPE="failure-pattern" ;;
  "ti") PATTERN_TYPE="task-insight" ;;
  *) echo "ERROR: Invalid pattern ID prefix"; exit 1 ;;
esac

PATTERN_FILE=~/.claude/gsd-knowledge/patterns/${PATTERN_TYPE}/${PATTERN_ID}.json

if [ -f "$PATTERN_FILE" ]; then
  cat "$PATTERN_FILE"
else
  echo "null"  # Pattern not found
fi
```

**Read with validation:**
```bash
PATTERN_ID="se-a1b2c3d4"
PATTERN_TYPE="strategy-effectiveness"
PATTERN_FILE=~/.claude/gsd-knowledge/patterns/${PATTERN_TYPE}/${PATTERN_ID}.json

if [ -f "$PATTERN_FILE" ]; then
  # Validate JSON is parseable
  if python3 -c "import json; json.load(open('$PATTERN_FILE'))" 2>/dev/null || \
     jq . "$PATTERN_FILE" >/dev/null 2>&1; then
    cat "$PATTERN_FILE"
  else
    echo "ERROR: Pattern file exists but contains invalid JSON"
    echo "null"
  fi
else
  echo "null"
fi
```

**Error handling:**
- File not found → Return null (pattern doesn't exist)
- Invalid JSON → Log warning, return null
- Permission error → Log warning, return null

**Returns:** Pattern JSON object or null if not found

---

### 4. Query Patterns

**Purpose:** Find patterns matching specified criteria.

**Input:** Filter criteria object with optional fields:
- `type`: Pattern type (strategy-effectiveness, failure-pattern, task-insight)
- `task_type`: Task type filter (bash-command, file-edit, test-run, etc.)
- `failure_category`: Failure category filter (tool, validation, dependency, etc.)
- `min_confidence`: Minimum confidence threshold (0.0-1.0)
- `strategy_name`: Strategy name filter (for strategy-effectiveness patterns)
- `limit`: Maximum results to return

**Process:**
1. Read index.json for quick filtering by type
2. Load matching pattern files
3. Apply additional filters
4. Sort by confidence (descending)
5. Return matching patterns

**Bash commands:**

**Query by type:**
```bash
QUERY_TYPE="strategy-effectiveness"
INDEX_FILE=~/.claude/gsd-knowledge/index.json

# Get all pattern IDs of this type from index
if command -v jq &> /dev/null; then
  jq -r ".by_type[\"$QUERY_TYPE\"][].id" "$INDEX_FILE"
else
  grep -o "\"id\": *\"${QUERY_TYPE:0:2}-[^\"]*\"" "$INDEX_FILE" | cut -d'"' -f4
fi
```

**Query by task type:**
```bash
QUERY_TASK_TYPE="test-run"
PATTERN_DIR=~/.claude/gsd-knowledge/patterns

# Search all pattern files for task type
grep -l "\"$QUERY_TASK_TYPE\"" "$PATTERN_DIR"/*/*.json 2>/dev/null
```

**Query with multiple filters:**
```bash
QUERY_TYPE="strategy-effectiveness"
QUERY_TASK_TYPE="test-run"
MIN_CONFIDENCE="0.7"
PATTERN_DIR=~/.claude/gsd-knowledge/patterns/${QUERY_TYPE}

# Find patterns matching criteria
for file in "$PATTERN_DIR"/*.json; do
  [ -f "$file" ] || continue

  # Check task type
  if ! grep -q "\"$QUERY_TASK_TYPE\"" "$file"; then
    continue
  fi

  # Check confidence (requires jq or python)
  if command -v jq &> /dev/null; then
    CONFIDENCE=$(jq -r '.confidence' "$file")
    if (( $(echo "$CONFIDENCE >= $MIN_CONFIDENCE" | bc -l) )); then
      echo "$file"
    fi
  else
    # Fallback: include all that match task type
    echo "$file"
  fi
done
```

**Query with sorting and limit:**
```bash
QUERY_TYPE="strategy-effectiveness"
LIMIT=5

# Get all patterns of type, sorted by confidence
if command -v jq &> /dev/null; then
  PATTERN_DIR=~/.claude/gsd-knowledge/patterns/${QUERY_TYPE}

  # Collect all patterns with confidence
  for file in "$PATTERN_DIR"/*.json; do
    [ -f "$file" ] || continue
    CONFIDENCE=$(jq -r '.confidence' "$file")
    echo "$CONFIDENCE $file"
  done | sort -rn | head -$LIMIT | cut -d' ' -f2-
fi
```

**Example: Find best strategy for test failures:**
```bash
# Query: "What's the most effective strategy for test-run failures with validation issues?"
TASK_TYPE="test-run"
FAILURE_CAT="validation"
PATTERN_DIR=~/.claude/gsd-knowledge/patterns/strategy-effectiveness

BEST_PATTERN=""
BEST_CONFIDENCE=0

for file in "$PATTERN_DIR"/*.json; do
  [ -f "$file" ] || continue

  # Check if pattern matches criteria
  if grep -q "\"$TASK_TYPE\"" "$file" && grep -q "\"$FAILURE_CAT\"" "$file"; then
    if command -v jq &> /dev/null; then
      CONFIDENCE=$(jq -r '.confidence' "$file")
      SUCCESS_RATE=$(jq -r '.content.success_rate' "$file")

      # Compare (simplified - use bc for float comparison)
      if (( $(echo "$CONFIDENCE > $BEST_CONFIDENCE" | bc -l) )); then
        BEST_CONFIDENCE=$CONFIDENCE
        BEST_PATTERN=$file
      fi
    fi
  fi
done

if [ -n "$BEST_PATTERN" ]; then
  echo "Best matching pattern:"
  cat "$BEST_PATTERN"
else
  echo "No matching patterns found"
fi
```

**Error handling:**
- Empty index → Return empty array
- No matches → Return empty array
- Partial read errors → Skip problematic files, return what's readable

**Returns:** Array of matching pattern objects (may be empty)

---

### 5. Delete Pattern

**Purpose:** Remove an outdated or invalid pattern.

**Input:** Pattern ID to delete

**Process:**
1. Locate pattern file
2. Remove from index.json
3. Delete pattern file

**Use cases:**
- Pattern has decayed below confidence threshold (Phase 8 decay function)
- Pattern was merged with another
- Pattern found to be incorrect

**Bash commands:**

**Delete pattern:**
```bash
PATTERN_ID="se-a1b2c3d4"

# Determine type from prefix
case "${PATTERN_ID:0:2}" in
  "se") PATTERN_TYPE="strategy-effectiveness" ;;
  "fp") PATTERN_TYPE="failure-pattern" ;;
  "ti") PATTERN_TYPE="task-insight" ;;
  *) echo "ERROR: Invalid pattern ID prefix"; exit 1 ;;
esac

PATTERN_FILE=~/.claude/gsd-knowledge/patterns/${PATTERN_TYPE}/${PATTERN_ID}.json
INDEX_FILE=~/.claude/gsd-knowledge/index.json

# Check if pattern exists
if [ ! -f "$PATTERN_FILE" ]; then
  echo "Pattern not found: $PATTERN_ID"
  exit 0  # Idempotent - already deleted
fi

# Remove from index first
if command -v jq &> /dev/null; then
  jq --arg id "$PATTERN_ID" --arg type "$PATTERN_TYPE" \
     'del(.by_type[$type][] | select(.id == $id)) | .total_patterns -= 1 | .last_updated = now | todate' \
     "$INDEX_FILE" > "${INDEX_FILE}.tmp" && mv "${INDEX_FILE}.tmp" "$INDEX_FILE"
fi

# Delete pattern file
rm "$PATTERN_FILE"

echo "Deleted pattern: $PATTERN_ID"
```

**Batch delete (for cleanup/decay):**
```bash
# Delete all patterns below confidence threshold
MIN_CONFIDENCE="0.3"
DELETED_COUNT=0

for type_dir in ~/.claude/gsd-knowledge/patterns/*/; do
  for file in "$type_dir"*.json; do
    [ -f "$file" ] || continue

    if command -v jq &> /dev/null; then
      CONFIDENCE=$(jq -r '.confidence' "$file")
      if (( $(echo "$CONFIDENCE < $MIN_CONFIDENCE" | bc -l) )); then
        PATTERN_ID=$(jq -r '.id' "$file")
        rm "$file"
        DELETED_COUNT=$((DELETED_COUNT + 1))
        echo "Deleted low-confidence pattern: $PATTERN_ID (confidence: $CONFIDENCE)"
      fi
    fi
  done
done

echo "Deleted $DELETED_COUNT patterns below threshold $MIN_CONFIDENCE"

# Rebuild index after batch delete
# (Call Initialize operation to repair index)
```

**Error handling:**
- Pattern not found → Success (idempotent - already deleted)
- Index update fails → Pattern deleted but index inconsistent (repair on next init)
- Permission error → Log error, return failure

**Returns:** Success/failure status

---

## Index.json Structure

The index provides fast lookups without scanning all pattern files:

```json
{
  "version": "1.0.0",
  "last_updated": "2026-01-07T12:00:00Z",
  "total_patterns": 15,
  "by_type": {
    "strategy-effectiveness": [
      {"id": "se-a1b2c3d4", "strategy_name": "retry-with-reruns", "success_rate": 0.85, "file": "patterns/strategy-effectiveness/se-a1b2c3d4.json"},
      {"id": "se-e5f6g7h8", "strategy_name": "reinstall-dependencies", "success_rate": 0.90, "file": "patterns/strategy-effectiveness/se-e5f6g7h8.json"}
    ],
    "failure-pattern": [
      {"id": "fp-i9j0k1l2", "failure_category": "dependency", "task_type": "bash-command", "file": "patterns/failure-pattern/fp-i9j0k1l2.json"}
    ],
    "task-insight": [
      {"id": "ti-m3n4o5p6", "task_type": "file-edit", "insight_category": "best-practice", "file": "patterns/task-insight/ti-m3n4o5p6.json"}
    ]
  },
  "by_task_type": {
    "test-run": ["se-a1b2c3d4"],
    "bash-command": ["se-e5f6g7h8", "fp-i9j0k1l2"],
    "file-edit": ["ti-m3n4o5p6"]
  },
  "by_failure_category": {
    "validation": ["se-a1b2c3d4"],
    "dependency": ["se-e5f6g7h8", "fp-i9j0k1l2"]
  }
}
```

**Index maintenance:**
- Updated on every Write and Delete operation
- Can be rebuilt by scanning pattern files if corrupted
- Initialize operation repairs missing/invalid index

---

## Error Handling Principles

1. **Never block primary workflow:** Knowledge base operations are best-effort. Failures log warnings but don't stop execution.

2. **Graceful degradation:** If knowledge base unavailable, features that depend on it are simply skipped.

3. **Idempotent operations:** All operations are safe to retry. Initialize can be called multiple times. Delete on missing pattern succeeds.

4. **Self-healing:** Index.json can be rebuilt from pattern files. Initialize repairs missing files.

5. **Log for debugging:** Errors are logged with enough context to diagnose issues without blocking.

**Example error wrapper:**
```bash
# Wrap knowledge base operation with graceful error handling
knowledge_base_op() {
  local result
  if result=$("$@" 2>&1); then
    echo "$result"
  else
    echo "WARNING: Knowledge base operation failed: $*" >&2
    echo "Error: $result" >&2
    echo ""  # Return empty/null
  fi
}

# Usage
PATTERN=$(knowledge_base_op cat ~/.claude/gsd-knowledge/patterns/strategy-effectiveness/se-abc123.json)
if [ -n "$PATTERN" ]; then
  echo "Pattern found"
else
  echo "Pattern not found or error occurred - continuing without it"
fi
```

---

## Concurrency Notes

- **Single-threaded execution:** Claude Code executes single-threaded, so file locking is not required
- **Atomic writes:** Use write-to-temp-then-move pattern for index.json updates
- **Read-after-write:** Always re-read files after writing to confirm success

---

## Usage in Workflows

**Pattern extraction (Phase 6):**
```
1. Read EXECUTION_HISTORY.md from completed project
2. Identify patterns meeting extraction criteria
3. For each pattern:
   a. Initialize (ensure knowledge base exists)
   b. Query (check for existing similar pattern)
   c. Write (create new or merge with existing)
```

**Strategy selection (Phase 7+):**
```
1. Initialize (ensure knowledge base exists)
2. Query patterns by failure_category and task_type
3. Sort by confidence and success_rate
4. Return top recommendation
```

**Decay/cleanup (Phase 8):**
```
1. Initialize
2. Query all patterns
3. For each pattern:
   a. Calculate new confidence based on age and usage
   b. If below threshold: Delete
   c. If above threshold: Update with new confidence
```
