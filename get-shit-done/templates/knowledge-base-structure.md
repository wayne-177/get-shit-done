# Knowledge Base Directory Structure

---
type: template
target: ~/.claude/gsd-knowledge/
purpose: Global knowledge base for cross-project pattern learning
created: 2026-01-07
---

## Overview

The knowledge base stores project-agnostic patterns extracted from individual project execution histories. This enables predictive planning and pattern reuse across all GSD projects.

**Location:** `~/.claude/gsd-knowledge/`

**Relationship to per-project artifacts:**
- Per-project: `.planning/EXECUTION_HISTORY.md` (project-specific patterns)
- Global: `~/.claude/gsd-knowledge/` (normalized, cross-project patterns)

## Directory Structure

```
~/.claude/gsd-knowledge/
├── patterns/                   # Individual pattern files
│   ├── strategy-effectiveness/ # Patterns tracking which strategies work
│   │   └── *.json             # One file per pattern
│   ├── failure-pattern/        # Patterns tracking recurring failures
│   │   └── *.json             # One file per pattern
│   └── task-insight/           # Patterns tracking task-type learnings
│       └── *.json             # One file per pattern
├── index.json                  # Pattern index for quick lookup
└── config.json                 # Knowledge base settings
```

## File Descriptions

### patterns/

Contains individual pattern files organized by pattern type. Each pattern is a self-contained JSON file following the schema defined in `@get-shit-done/references/pattern-schema.md`.

**Subdirectories:**
- `strategy-effectiveness/` - Tracks which retry strategies work for which failure categories
- `failure-pattern/` - Tracks recurring failure patterns across projects
- `task-insight/` - Tracks learnings about specific task types (bash-command, file-edit, test-run, etc.)

**Naming convention:** `{pattern-id}.json` where pattern-id is a unique identifier (UUID or content hash)

### index.json

Quick-lookup index mapping pattern attributes to pattern files. Enables efficient pattern retrieval without scanning all files.

**Initial structure:**
```json
{
  "version": "1.0.0",
  "last_updated": "2026-01-07T00:00:00Z",
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

**Index entries:**
```json
{
  "by_type": {
    "strategy-effectiveness": [
      {
        "id": "se-abc123",
        "strategy_name": "retry-with-reruns",
        "task_types": ["test-run"],
        "success_rate": 0.85,
        "file": "patterns/strategy-effectiveness/se-abc123.json"
      }
    ]
  },
  "by_task_type": {
    "test-run": ["se-abc123", "fp-def456"],
    "bash-command": ["ti-ghi789"]
  },
  "by_failure_category": {
    "validation": ["se-abc123", "fp-def456"],
    "dependency": ["se-xyz999"]
  }
}
```

### config.json

Knowledge base configuration and metadata.

**Initial structure:**
```json
{
  "version": "1.0.0",
  "created": "2026-01-07T00:00:00Z",
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

**Settings explained:**
- `auto_extract`: Whether to automatically extract patterns after each project (future feature)
- `min_confidence_for_extraction`: Minimum confidence score required to promote pattern to knowledge base
- `max_patterns_per_type`: Maximum patterns per type before cleanup/consolidation
- `pattern_expiry_days`: Days after which unused patterns are considered stale

## Initialization

Use the initialization workflow to bootstrap a fresh knowledge base:

```
@get-shit-done/workflows/init-knowledge-base.md
```

This workflow:
1. Creates `~/.claude/gsd-knowledge/` if missing
2. Creates subdirectory structure
3. Initializes `index.json` with empty structure
4. Initializes `config.json` with default settings
5. Is idempotent - safe to run multiple times

## Usage Patterns

### Pattern Extraction (Future - Phase 5 Plan 02+)

When a project completes or at defined intervals, patterns from `.planning/EXECUTION_HISTORY.md` can be normalized and extracted to the global knowledge base:

1. Read project's EXECUTION_HISTORY.md
2. Identify patterns meeting confidence threshold
3. Normalize patterns (remove project-specific details)
4. Check for existing similar patterns in knowledge base
5. Merge or create new pattern entries
6. Update index.json

### Pattern Consumption (Future - Phase 5 Plan 03+)

When planning or selecting strategies, consult the knowledge base:

1. Query index.json by task type and/or failure category
2. Load relevant pattern files
3. Use pattern data to inform strategy selection
4. Weight recommendations by confidence and recency

## Design Principles

1. **Simple file structure:** JSON files over databases for portability and debuggability
2. **Self-contained patterns:** Each pattern file includes all information needed for interpretation
3. **Incremental growth:** Knowledge base grows as projects complete, no bulk migration required
4. **Opt-in by default:** Knowledge base features disabled until explicitly enabled
5. **Graceful degradation:** Missing or corrupt knowledge base never blocks project execution

## Related Files

- `@get-shit-done/workflows/init-knowledge-base.md` - Initialization workflow
- `@get-shit-done/references/pattern-schema.md` - JSON schema for pattern files
- `@get-shit-done/templates/execution-history.md` - Per-project history template
