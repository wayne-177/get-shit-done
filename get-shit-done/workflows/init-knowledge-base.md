# Initialize Knowledge Base Workflow

This workflow bootstraps the global knowledge base directory structure at `~/.claude/gsd-knowledge/`. It is idempotent and safe to call multiple times.

## Purpose

Create and initialize the knowledge base directory structure for cross-project pattern learning. This workflow should be called:
- Before first pattern extraction from any project
- By other workflows that need to ensure knowledge base exists
- Manually when setting up GSD on a new machine

## Context Requirements

- `@get-shit-done/templates/knowledge-base-structure.md` - Directory structure reference

## Process

<step name="check_existence" number="1">
**Check if knowledge base directory exists**

**Process:**
1. Check if `~/.claude/gsd-knowledge/` directory exists
2. If exists AND contains valid `index.json` AND `config.json`:
   - Log: "Knowledge base already initialized at ~/.claude/gsd-knowledge/"
   - Skip to step 5 (verify structure)
3. If directory exists but files missing or corrupt:
   - Continue to step 2 (will repair missing files)
4. If directory does not exist:
   - Continue to step 2 (full initialization)

**Bash command:**
```bash
KB_DIR="${HOME}/.claude/gsd-knowledge"
if [ -d "${KB_DIR}" ] && [ -f "${KB_DIR}/index.json" ] && [ -f "${KB_DIR}/config.json" ]; then
  echo "EXISTING"
else
  echo "NEEDS_INIT"
fi
```

**Decision:**
- `EXISTING` → Skip to step 5
- `NEEDS_INIT` → Continue to step 2
</step>

<step name="create_directory_structure" number="2">
**Create directory structure**

Create the knowledge base directory and subdirectories.

**Process:**
1. Create main directory: `~/.claude/gsd-knowledge/`
2. Create patterns directory: `~/.claude/gsd-knowledge/patterns/`
3. Create pattern type subdirectories:
   - `~/.claude/gsd-knowledge/patterns/strategy-effectiveness/`
   - `~/.claude/gsd-knowledge/patterns/failure-pattern/`
   - `~/.claude/gsd-knowledge/patterns/task-insight/`

**Bash commands:**
```bash
KB_DIR="${HOME}/.claude/gsd-knowledge"
mkdir -p "${KB_DIR}/patterns/strategy-effectiveness"
mkdir -p "${KB_DIR}/patterns/failure-pattern"
mkdir -p "${KB_DIR}/patterns/task-insight"
```

**Error handling:**
- If permission denied: Log error and exit with failure message
- If disk full: Log error and exit with failure message
- Success: Continue to step 3
</step>

<step name="initialize_index" number="3">
**Initialize index.json**

Create the pattern index file if missing.

**Process:**
1. Check if `~/.claude/gsd-knowledge/index.json` exists
2. If missing, create with initial structure:

**Initial content:**
```json
{
  "version": "1.0.0",
  "last_updated": "[CURRENT_TIMESTAMP]",
  "total_patterns": 0,
  "by_type": {
    "strategy-effectiveness": [],
    "failure-pattern": [],
    "task-insight": []
  },
  "by_task_type": {},
  "by_failure_category": {}
}
```

**Bash command (create if not exists):**
```bash
KB_DIR="${HOME}/.claude/gsd-knowledge"
if [ ! -f "${KB_DIR}/index.json" ]; then
  cat > "${KB_DIR}/index.json" << 'EOF'
{
  "version": "1.0.0",
  "last_updated": "TIMESTAMP_PLACEHOLDER",
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
  # Replace timestamp placeholder with actual time
  sed -i '' "s/TIMESTAMP_PLACEHOLDER/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" "${KB_DIR}/index.json" 2>/dev/null || \
  sed -i "s/TIMESTAMP_PLACEHOLDER/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" "${KB_DIR}/index.json"
  echo "Created index.json"
else
  echo "index.json already exists"
fi
```

**Error handling:**
- If write fails: Log error, attempt to continue (step 4 may still succeed)
</step>

<step name="initialize_config" number="4">
**Initialize config.json**

Create the configuration file if missing.

**Process:**
1. Check if `~/.claude/gsd-knowledge/config.json` exists
2. If missing, create with default settings:

**Initial content:**
```json
{
  "version": "1.0.0",
  "created": "[CURRENT_TIMESTAMP]",
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
```

**Bash command (create if not exists):**
```bash
KB_DIR="${HOME}/.claude/gsd-knowledge"
if [ ! -f "${KB_DIR}/config.json" ]; then
  cat > "${KB_DIR}/config.json" << 'EOF'
{
  "version": "1.0.0",
  "created": "TIMESTAMP_PLACEHOLDER",
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
  # Replace timestamp placeholder with actual time
  sed -i '' "s/TIMESTAMP_PLACEHOLDER/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" "${KB_DIR}/config.json" 2>/dev/null || \
  sed -i "s/TIMESTAMP_PLACEHOLDER/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" "${KB_DIR}/config.json"
  echo "Created config.json"
else
  echo "config.json already exists"
fi
```

**Error handling:**
- If write fails: Log error, continue to verification
</step>

<step name="verify_structure" number="5">
**Verify complete structure**

Validate that all required directories and files exist.

**Process:**
1. Check each required path exists
2. Validate JSON files are parseable
3. Report status

**Bash verification script:**
```bash
echo "Verifying knowledge base structure..."

KB_DIR="${HOME}/.claude/gsd-knowledge"

# Check directories
dirs=(
  "${KB_DIR}"
  "${KB_DIR}/patterns"
  "${KB_DIR}/patterns/strategy-effectiveness"
  "${KB_DIR}/patterns/failure-pattern"
  "${KB_DIR}/patterns/task-insight"
)

all_ok=true
for dir in "${dirs[@]}"; do
  if [ -d "$dir" ]; then
    echo "  [OK] $dir"
  else
    echo "  [MISSING] $dir"
    all_ok=false
  fi
done

# Check files
files=(
  "${KB_DIR}/index.json"
  "${KB_DIR}/config.json"
)

for file in "${files[@]}"; do
  if [ -f "$file" ]; then
    # Validate JSON
    if python3 -c "import json; json.load(open('$file'))" 2>/dev/null || \
       jq . "$file" >/dev/null 2>&1; then
      echo "  [OK] $file (valid JSON)"
    else
      echo "  [INVALID] $file (not valid JSON)"
      all_ok=false
    fi
  else
    echo "  [MISSING] $file"
    all_ok=false
  fi
done

if [ "$all_ok" = true ]; then
  echo ""
  echo "Knowledge base initialized successfully at ~/.claude/gsd-knowledge/"
else
  echo ""
  echo "WARNING: Knowledge base initialization incomplete - some components missing"
fi
```

**Output:**
- Success: "Knowledge base initialized successfully at ~/.claude/gsd-knowledge/"
- Partial: "WARNING: Knowledge base initialization incomplete - some components missing"
</step>

## Configuration

This workflow has no configuration - it uses hardcoded defaults for initial setup. Configuration can be modified after initialization by editing `~/.claude/gsd-knowledge/config.json`.

## Examples

### Example 1: Fresh Installation

```
$ # Knowledge base does not exist
$ ls ~/.claude/gsd-knowledge
ls: cannot access '/home/user/.claude/gsd-knowledge': No such file or directory

$ # Run initialization workflow
$ # (Workflow executes steps 1-5)

Verifying knowledge base structure...
  [OK] /home/user/.claude/gsd-knowledge
  [OK] /home/user/.claude/gsd-knowledge/patterns
  [OK] /home/user/.claude/gsd-knowledge/patterns/strategy-effectiveness
  [OK] /home/user/.claude/gsd-knowledge/patterns/failure-pattern
  [OK] /home/user/.claude/gsd-knowledge/patterns/task-insight
  [OK] /home/user/.claude/gsd-knowledge/index.json (valid JSON)
  [OK] /home/user/.claude/gsd-knowledge/config.json (valid JSON)

Knowledge base initialized successfully at ~/.claude/gsd-knowledge/
```

### Example 2: Already Initialized (Idempotent)

```
$ # Knowledge base already exists
$ ls ~/.claude/gsd-knowledge
config.json  index.json  patterns/

$ # Run initialization workflow
$ # (Step 1 detects EXISTING, skips to step 5)

Knowledge base already initialized at ~/.claude/gsd-knowledge/
Verifying knowledge base structure...
  [OK] /home/user/.claude/gsd-knowledge
  ...
Knowledge base initialized successfully at ~/.claude/gsd-knowledge/
```

### Example 3: Partial Repair

```
$ # index.json was accidentally deleted
$ ls ~/.claude/gsd-knowledge
config.json  patterns/

$ # Run initialization workflow
$ # (Step 1 detects NEEDS_INIT, runs steps 2-5)
$ # (Step 2 skips existing directories)
$ # (Step 3 creates missing index.json)

Created index.json
config.json already exists
Verifying knowledge base structure...
  ...
Knowledge base initialized successfully at ~/.claude/gsd-knowledge/
```

## Integration Notes

**Callable from other workflows:**

This workflow should be called before any knowledge base operation:

```xml
<step name="ensure_knowledge_base">
  <!-- Call init workflow - it's idempotent -->
  <workflow>@get-shit-done/workflows/init-knowledge-base.md</workflow>

  <!-- If init fails, log warning but continue -->
  <on_failure>
    <log level="warning">Knowledge base initialization failed - pattern extraction will be skipped</log>
    <set var="knowledge_base_available" value="false"/>
  </on_failure>
</step>
```

**Error handling principle:**
Knowledge base initialization should never block primary workflow execution. If initialization fails, log a warning and disable knowledge base features for that session.

## Success Indicators

Workflow completed successfully when:
- All five directories exist
- `index.json` exists and contains valid JSON
- `config.json` exists and contains valid JSON
- Verification step reports "Knowledge base initialized successfully"
