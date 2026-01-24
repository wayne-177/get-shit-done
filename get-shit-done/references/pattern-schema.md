# Pattern Schema Reference

This document defines the JSON schema for patterns stored in the global knowledge base (`~/.claude/gsd-knowledge/patterns/`). All patterns follow this schema to ensure consistency and enable automated processing.

## Overview

Patterns capture project-agnostic learnings from execution histories. Three pattern types exist:
- **strategy-effectiveness**: Tracks which retry strategies work for which failure categories
- **failure-pattern**: Tracks recurring failure patterns across projects
- **task-insight**: Tracks learnings about specific task types

## JSON Schema Definition

### Common Fields (All Pattern Types)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "GSD Knowledge Base Pattern",
  "type": "object",
  "required": ["id", "type", "version", "created", "last_updated", "confidence", "evidence", "content"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique pattern identifier (format: {type-prefix}-{uuid-or-hash})",
      "pattern": "^(se|fp|ti)-[a-f0-9]{8,}$"
    },
    "type": {
      "type": "string",
      "enum": ["strategy-effectiveness", "failure-pattern", "task-insight"],
      "description": "Pattern classification"
    },
    "version": {
      "type": "string",
      "description": "Schema version (semver)",
      "pattern": "^\\d+\\.\\d+\\.\\d+$"
    },
    "created": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp of pattern creation"
    },
    "last_updated": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp of last pattern update"
    },
    "confidence": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "description": "Confidence score (0.0-1.0) based on evidence strength"
    },
    "evidence": {
      "type": "object",
      "description": "Statistical backing for this pattern",
      "required": ["occurrences", "project_count"],
      "properties": {
        "occurrences": {
          "type": "integer",
          "minimum": 1,
          "description": "Total times this pattern was observed"
        },
        "project_count": {
          "type": "integer",
          "minimum": 1,
          "description": "Number of distinct projects contributing to this pattern"
        },
        "first_seen": {
          "type": "string",
          "format": "date-time",
          "description": "When pattern was first observed"
        },
        "last_seen": {
          "type": "string",
          "format": "date-time",
          "description": "Most recent observation"
        }
      }
    },
    "content": {
      "type": "object",
      "description": "Pattern-type-specific content (see type-specific schemas below)"
    },
    "tags": {
      "type": "array",
      "items": {"type": "string"},
      "description": "Optional searchable tags for discovery"
    },
    "analysis": {
      "type": "object",
      "description": "Analysis metadata (populated by analyze-patterns workflow)",
      "properties": {
        "effectiveness_score": {
          "type": "number",
          "description": "Calculated effectiveness score for ranking (see Effectiveness Score Calculation section)"
        },
        "validated": {
          "type": "boolean",
          "description": "True if pattern appears in 3+ projects (cross-project validated)"
        },
        "last_used": {
          "type": "string",
          "format": "date-time",
          "description": "When pattern was last displayed in predictive hints"
        },
        "usage_count": {
          "type": "integer",
          "description": "Times pattern was displayed in predictive hints"
        },
        "decay_scheduled": {
          "type": "string",
          "format": "date-time",
          "description": "When pattern is scheduled for decay review (Phase 8)"
        },
        "last_analyzed": {
          "type": "string",
          "format": "date-time",
          "description": "When pattern was last processed by analyze-patterns workflow"
        }
      }
    }
  }
}
```

### Strategy-Effectiveness Pattern

Tracks which retry strategies work for which failure categories.

```json
{
  "content": {
    "type": "object",
    "required": ["strategy_name", "task_types", "failure_categories", "success_rate", "avg_attempt_number"],
    "properties": {
      "strategy_name": {
        "type": "string",
        "description": "Name of the retry strategy (e.g., 'retry-with-reruns', 'reinstall-dependencies')"
      },
      "task_types": {
        "type": "array",
        "items": {
          "type": "string",
          "enum": ["bash-command", "file-edit", "test-run", "build", "deploy", "other"]
        },
        "description": "Task types where this strategy has been applied"
      },
      "failure_categories": {
        "type": "array",
        "items": {
          "type": "string",
          "enum": ["tool", "validation", "dependency", "logic", "resource", "permission", "network", "other"]
        },
        "description": "Failure categories this strategy addresses"
      },
      "success_rate": {
        "type": "number",
        "minimum": 0.0,
        "maximum": 1.0,
        "description": "Success rate as decimal (0.85 = 85%)"
      },
      "avg_attempt_number": {
        "type": "number",
        "minimum": 1.0,
        "description": "Average attempt number when this strategy succeeds (1 = first retry)"
      },
      "total_uses": {
        "type": "integer",
        "minimum": 1,
        "description": "Total times strategy was attempted"
      },
      "successful_uses": {
        "type": "integer",
        "minimum": 0,
        "description": "Times strategy led to success"
      },
      "description": {
        "type": "string",
        "description": "Human-readable summary of when/why this strategy works"
      },
      "contraindications": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Situations where this strategy should NOT be used"
      }
    }
  }
}
```

### Failure-Pattern Pattern

Tracks recurring failure patterns across projects.

```json
{
  "content": {
    "type": "object",
    "required": ["failure_category", "task_type", "frequency", "typical_recovery"],
    "properties": {
      "failure_category": {
        "type": "string",
        "enum": ["tool", "validation", "dependency", "logic", "resource", "permission", "network", "other"],
        "description": "Primary failure classification"
      },
      "task_type": {
        "type": "string",
        "enum": ["bash-command", "file-edit", "test-run", "build", "deploy", "other"],
        "description": "Task type where failure occurs"
      },
      "frequency": {
        "type": "integer",
        "minimum": 1,
        "description": "How often this pattern has been observed"
      },
      "typical_recovery": {
        "type": "string",
        "description": "Strategy that typically resolves this failure"
      },
      "error_signatures": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Error message patterns that indicate this failure (regex-safe)"
      },
      "root_causes": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Common root causes identified"
      },
      "escalation_appropriate": {
        "type": "boolean",
        "description": "Whether this failure type typically requires user intervention"
      },
      "notes": {
        "type": "string",
        "description": "Additional context about this failure pattern"
      }
    }
  }
}
```

### Task-Insight Pattern

Tracks learnings about specific task types.

```json
{
  "content": {
    "type": "object",
    "required": ["task_type", "insight_category", "insight"],
    "properties": {
      "task_type": {
        "type": "string",
        "enum": ["bash-command", "file-edit", "test-run", "build", "deploy", "other"],
        "description": "Task type this insight applies to"
      },
      "insight_category": {
        "type": "string",
        "enum": ["best-practice", "common-pitfall", "optimization", "prerequisite", "verification"],
        "description": "Category of insight"
      },
      "insight": {
        "type": "string",
        "description": "The actual learning (human-readable)"
      },
      "actionable_recommendation": {
        "type": "string",
        "description": "Specific action to take based on this insight"
      },
      "applies_when": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Conditions when this insight is relevant"
      },
      "source_examples": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Brief anonymized examples that led to this insight"
      }
    }
  }
}
```

## Pattern ID Format

Pattern IDs follow a consistent format for easy identification:

- **strategy-effectiveness**: `se-{hash}` (e.g., `se-a1b2c3d4`)
- **failure-pattern**: `fp-{hash}` (e.g., `fp-e5f6g7h8`)
- **task-insight**: `ti-{hash}` (e.g., `ti-i9j0k1l2`)

**Hash generation:**
- Use first 8 characters of SHA-256 hash of content
- Or use UUID v4 (first 8 chars after removing hyphens)
- Must be unique within the knowledge base

## Effectiveness Score Calculation

The effectiveness score ranks patterns for predictive hint display and identifies high-value patterns.

### Formula

**For strategy-effectiveness patterns:**
```
effectiveness_score = success_rate * confidence * log10(occurrences + 1)
```

**For failure-pattern and task-insight patterns:**
```
effectiveness_score = confidence * log10(occurrences + 1)
```

### Components

| Component | Range | Description |
|-----------|-------|-------------|
| `success_rate` | 0.0-1.0 | How often the strategy succeeds when applied (strategy-effectiveness only) |
| `confidence` | 0.0-1.0 | Evidence strength based on occurrences, project diversity, recency |
| `log10(occurrences + 1)` | 0.3-2.0+ | Logarithmic scaling rewards more observations with diminishing returns |

### Example Calculations

| Pattern | Type | Success Rate | Confidence | Occurrences | Score |
|---------|------|--------------|------------|-------------|-------|
| retry-with-reruns | strategy | 0.85 | 0.90 | 15 | 0.85 × 0.90 × log10(16) = **0.92** |
| reinstall-deps | strategy | 0.75 | 0.70 | 5 | 0.75 × 0.70 × log10(6) = **0.41** |
| dependency failure | failure | N/A | 0.80 | 10 | 0.80 × log10(11) = **0.83** |
| read-before-edit | insight | N/A | 0.90 | 20 | 0.90 × log10(21) = **1.19** |

### Score Interpretation

| Score Range | Interpretation | Action |
|-------------|----------------|--------|
| > 0.8 | High-value pattern | Prioritize in predictive hints |
| 0.5-0.8 | Useful pattern | Include in recommendations |
| < 0.5 | Low-value pattern | Candidate for decay review |

### When Scores Are Updated

- **During analysis:** The analyze-patterns workflow calculates and stores scores
- **During extraction:** Initial scores set based on first extraction data
- **During decay:** Scores decay over time if patterns aren't observed (Phase 8)

---

## Pattern Validation

Patterns are validated when they demonstrate cross-project consistency.

### Validation Criteria

- **Validated status:** `evidence.project_count >= 3`
- **Automatic:** Validation status is computed during analysis
- **Stored in:** `analysis.validated` field

### Benefits of Validated Patterns

1. **Priority in hints:** Validated patterns appear first in predictive hints
2. **Protected from decay:** Higher threshold before decay triggers
3. **Reliability indicator:** Users can trust validated patterns more

### Validation Process

1. Pattern is extracted from project EXECUTION_HISTORY.md
2. On extraction, `evidence.project_count` is incremented
3. When count reaches 3, `analysis.validated` is set to `true` during next analysis
4. Validated patterns get `decay_scheduled` set further in the future

---

## Validation Rules

### Required Validations

1. **ID uniqueness**: No two patterns may share the same ID
2. **Confidence bounds**: Must be 0.0 <= confidence <= 1.0
3. **Success rate bounds**: Must be 0.0 <= success_rate <= 1.0
4. **Consistency**: `successful_uses` <= `total_uses`
5. **Timestamp ordering**: `first_seen` <= `last_seen` <= `last_updated`
6. **Minimum evidence**: `occurrences` >= 1, `project_count` >= 1

### Confidence Calculation

Confidence scores should be calculated based on:

```
base_confidence = min(occurrences / 10, 1.0) * 0.5  // More observations = higher confidence
project_diversity = min(project_count / 3, 1.0) * 0.3  // More projects = more generalizable
recency = (1.0 if last_seen within 30 days else 0.7 if within 90 days else 0.5) * 0.2

confidence = base_confidence + project_diversity + recency
```

Example:
- 15 occurrences across 4 projects, last seen 10 days ago
- `base = min(15/10, 1) * 0.5 = 0.5`
- `diversity = min(4/3, 1) * 0.3 = 0.3`
- `recency = 1.0 * 0.2 = 0.2`
- `confidence = 0.5 + 0.3 + 0.2 = 1.0`

### Minimum Thresholds for Extraction

Patterns should only be extracted to the knowledge base when:
- `occurrences >= 3` (pattern observed multiple times)
- `confidence >= 0.5` (minimum certainty threshold)
- For strategy-effectiveness: `total_uses >= 3`

## Normalization Guidance

When extracting patterns from project-specific EXECUTION_HISTORY.md to the knowledge base, normalize as follows:

### Remove Project-Specific Details

**Before (project-specific):**
```
Strategy "reinstall-dependencies" succeeded for npm install failure in /Users/john/myproject
```

**After (normalized):**
```json
{
  "content": {
    "strategy_name": "reinstall-dependencies",
    "task_types": ["bash-command"],
    "failure_categories": ["dependency"],
    "description": "Reinstalling dependencies resolves npm install failures"
  }
}
```

### Merge Similar Patterns

When a new pattern matches an existing one:
1. Increment `occurrences` and `total_uses`
2. Update `last_seen` and `last_updated`
3. Recalculate `success_rate` and `confidence`
4. Merge `task_types` and `failure_categories` arrays (union)
5. Keep the more informative `description`

### Pattern Matching for Merging

Two patterns should be merged when:
- Same `type`
- Same `strategy_name` (for strategy-effectiveness)
- Same `failure_category` + `task_type` (for failure-pattern)
- Same `task_type` + `insight_category` + similar `insight` (for task-insight)

## Example Patterns

### Example 1: Strategy-Effectiveness Pattern

```json
{
  "id": "se-a1b2c3d4",
  "type": "strategy-effectiveness",
  "version": "1.0.0",
  "created": "2026-01-06T10:00:00Z",
  "last_updated": "2026-01-07T15:30:00Z",
  "confidence": 0.85,
  "evidence": {
    "occurrences": 12,
    "project_count": 4,
    "first_seen": "2026-01-06T10:00:00Z",
    "last_seen": "2026-01-07T15:30:00Z"
  },
  "content": {
    "strategy_name": "retry-with-reruns",
    "task_types": ["test-run"],
    "failure_categories": ["validation"],
    "success_rate": 0.83,
    "avg_attempt_number": 2.0,
    "total_uses": 12,
    "successful_uses": 10,
    "description": "Running tests with --maxWorkers=1 --runInBand eliminates race conditions causing flaky test failures",
    "contraindications": ["Tests with intentional parallelism requirements", "Performance benchmarks"]
  },
  "tags": ["jest", "flaky-tests", "test-isolation"],
  "analysis": {
    "effectiveness_score": 0.8492,
    "validated": true,
    "last_used": "2026-01-07T14:00:00Z",
    "usage_count": 5,
    "last_analyzed": "2026-01-07T15:30:00Z"
  }
}
```

### Example 2: Failure-Pattern Pattern

```json
{
  "id": "fp-e5f6g7h8",
  "type": "failure-pattern",
  "version": "1.0.0",
  "created": "2026-01-05T08:00:00Z",
  "last_updated": "2026-01-07T12:00:00Z",
  "confidence": 0.75,
  "evidence": {
    "occurrences": 8,
    "project_count": 3,
    "first_seen": "2026-01-05T08:00:00Z",
    "last_seen": "2026-01-07T12:00:00Z"
  },
  "content": {
    "failure_category": "dependency",
    "task_type": "bash-command",
    "frequency": 8,
    "typical_recovery": "explicit-install-missing",
    "error_signatures": [
      "Cannot find module",
      "Module not found",
      "ERR_MODULE_NOT_FOUND"
    ],
    "root_causes": [
      "Package not in package.json",
      "Peer dependency not installed",
      "Cache corruption after node version change"
    ],
    "escalation_appropriate": false,
    "notes": "Usually auto-recoverable by installing the specific missing package"
  },
  "tags": ["npm", "node-modules", "dependency-resolution"],
  "analysis": {
    "effectiveness_score": 0.6769,
    "validated": true,
    "usage_count": 3,
    "last_analyzed": "2026-01-07T12:00:00Z"
  }
}
```

### Example 3: Task-Insight Pattern

```json
{
  "id": "ti-i9j0k1l2",
  "type": "task-insight",
  "version": "1.0.0",
  "created": "2026-01-04T14:00:00Z",
  "last_updated": "2026-01-07T09:00:00Z",
  "confidence": 0.90,
  "evidence": {
    "occurrences": 20,
    "project_count": 6,
    "first_seen": "2026-01-04T14:00:00Z",
    "last_seen": "2026-01-07T09:00:00Z"
  },
  "content": {
    "task_type": "file-edit",
    "insight_category": "best-practice",
    "insight": "Always read a file before editing to ensure the old_string matches current content",
    "actionable_recommendation": "Use Read tool before Edit tool for any file modification to verify exact content",
    "applies_when": [
      "Editing files that may have been modified since planning",
      "Edit tool returns 'old_string not found' error"
    ],
    "source_examples": [
      "Edit failed because file was auto-formatted since plan creation",
      "Edit failed because concurrent process modified file"
    ]
  },
  "tags": ["file-operations", "edit-tool", "content-verification"],
  "analysis": {
    "effectiveness_score": 1.1885,
    "validated": true,
    "last_used": "2026-01-07T08:00:00Z",
    "usage_count": 12,
    "last_analyzed": "2026-01-07T09:00:00Z"
  }
}
```

## Related Files

- `@get-shit-done/templates/knowledge-base-structure.md` - Directory structure documentation
- `@get-shit-done/workflows/init-knowledge-base.md` - Knowledge base initialization
- `@get-shit-done/templates/execution-history.md` - Per-project history (source for pattern extraction)
