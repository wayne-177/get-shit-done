# Config Validation Library

Hand-rolled validation for `.planning/config.json` without external dependencies. Validates structure, types, and allowed values. Returns array of error messages (empty if valid).

## Purpose

Enable GSD to catch configuration errors early during plan execution by:
1. Validating JSON syntax (parse errors) separately from semantic validation
2. Checking types and allowed values for all known fields
3. Ignoring unknown fields for forward compatibility
4. Providing clear, actionable error messages with field paths

## Schema Definition

<config_schema>
The following fields are validated. Unknown fields are allowed (forward compatibility).

### Root Level

| Field | Type | Allowed Values | Required |
|-------|------|----------------|----------|
| `mode` | string | `interactive`, `autonomous`, `batch` | No |
| `depth` | string | `minimal`, `standard`, `thorough` | No |

### Gates Section

| Field | Type | Required |
|-------|------|----------|
| `gates.confirm_project` | boolean | No |
| `gates.confirm_phases` | boolean | No |
| `gates.confirm_roadmap` | boolean | No |
| `gates.confirm_breakdown` | boolean | No |
| `gates.confirm_plan` | boolean | No |
| `gates.execute_next_plan` | boolean | No |
| `gates.issues_review` | boolean | No |
| `gates.confirm_transition` | boolean | No |

### Safety Section

| Field | Type | Required |
|-------|------|----------|
| `safety.always_confirm_destructive` | boolean | No |
| `safety.always_confirm_external_services` | boolean | No |

### Retry Section

| Field | Type | Constraints | Required |
|-------|------|-------------|----------|
| `retry.enabled` | boolean | - | No |
| `retry.max_attempts` | integer | 1-5 | No |
| `retry.escalation_threshold` | integer | >= 1 | No |
| `retry.log_attempts` | boolean | - | No |

### Verification Section

| Field | Type | Required |
|-------|------|----------|
| `verification.enabled` | boolean | No |
| `verification.regression_check` | boolean | No |
| `verification.auto_rollback` | boolean | No |
| `verification.baseline_tests` | string | No |
| `verification.baseline_build` | string | No |

### Adaptive Section

| Field | Type | Required |
|-------|------|----------|
| `adaptive.enabled` | boolean | No |
| `adaptive.knowledge_base` | boolean | No |
| `adaptive.pattern_extraction` | boolean | No |
</config_schema>

## Validation Function

<validation_function>
```javascript
/**
 * Validate config.json structure
 * @param {object} config - Parsed JSON config object
 * @returns {string[]} Array of error messages (empty if valid)
 */
function validateConfig(config) {
  const errors = [];

  // Helper for type checking
  const checkType = (value, type, path) => {
    if (value !== undefined && typeof value !== type) {
      errors.push(`${path} must be a ${type}, got ${typeof value}`);
      return false;
    }
    return true;
  };

  // Helper for enum validation
  const checkEnum = (value, allowed, path) => {
    if (value !== undefined && !allowed.includes(value)) {
      errors.push(`${path} must be one of: ${allowed.join(', ')}`);
    }
  };

  // Helper for integer range validation
  const checkIntRange = (value, min, max, path) => {
    if (value !== undefined) {
      if (typeof value !== 'number' || !Number.isInteger(value)) {
        errors.push(`${path} must be an integer`);
      } else if (value < min || value > max) {
        errors.push(`${path} must be between ${min} and ${max}`);
      }
    }
  };

  // Helper for positive integer validation
  const checkPositiveInt = (value, path) => {
    if (value !== undefined) {
      if (typeof value !== 'number' || !Number.isInteger(value) || value < 1) {
        errors.push(`${path} must be a positive integer`);
      }
    }
  };

  // Root level
  checkEnum(config.mode, ['interactive', 'autonomous', 'batch'], 'mode');
  checkEnum(config.depth, ['minimal', 'standard', 'thorough'], 'depth');

  // Gates section
  if (config.gates !== undefined) {
    if (checkType(config.gates, 'object', 'gates')) {
      const gateKeys = [
        'confirm_project', 'confirm_phases', 'confirm_roadmap',
        'confirm_breakdown', 'confirm_plan', 'execute_next_plan',
        'issues_review', 'confirm_transition'
      ];
      for (const key of gateKeys) {
        checkType(config.gates[key], 'boolean', `gates.${key}`);
      }
    }
  }

  // Safety section
  if (config.safety !== undefined) {
    if (checkType(config.safety, 'object', 'safety')) {
      checkType(config.safety.always_confirm_destructive, 'boolean', 'safety.always_confirm_destructive');
      checkType(config.safety.always_confirm_external_services, 'boolean', 'safety.always_confirm_external_services');
    }
  }

  // Retry section
  if (config.retry !== undefined) {
    if (checkType(config.retry, 'object', 'retry')) {
      checkType(config.retry.enabled, 'boolean', 'retry.enabled');
      checkType(config.retry.log_attempts, 'boolean', 'retry.log_attempts');
      checkIntRange(config.retry.max_attempts, 1, 5, 'retry.max_attempts');
      checkPositiveInt(config.retry.escalation_threshold, 'retry.escalation_threshold');
    }
  }

  // Verification section
  if (config.verification !== undefined) {
    if (checkType(config.verification, 'object', 'verification')) {
      checkType(config.verification.enabled, 'boolean', 'verification.enabled');
      checkType(config.verification.regression_check, 'boolean', 'verification.regression_check');
      checkType(config.verification.auto_rollback, 'boolean', 'verification.auto_rollback');
      checkType(config.verification.baseline_tests, 'string', 'verification.baseline_tests');
      checkType(config.verification.baseline_build, 'string', 'verification.baseline_build');
    }
  }

  // Adaptive section
  if (config.adaptive !== undefined) {
    if (checkType(config.adaptive, 'object', 'adaptive')) {
      checkType(config.adaptive.enabled, 'boolean', 'adaptive.enabled');
      checkType(config.adaptive.knowledge_base, 'boolean', 'adaptive.knowledge_base');
      checkType(config.adaptive.pattern_extraction, 'boolean', 'adaptive.pattern_extraction');
    }
  }

  return errors;
}
```
</validation_function>

## Usage Pattern

<usage>
### Separate Parse and Validate

Always handle JSON parse errors separately from validation errors. This provides clearer user feedback.

```javascript
/**
 * Load and validate config.json
 * @param {string} configPath - Path to config.json file
 * @returns {{ valid: boolean, config?: object, error?: string }}
 */
function loadAndValidateConfig(configPath) {
  const fs = require('fs');

  // Step 1: Read file
  let rawContent;
  try {
    rawContent = fs.readFileSync(configPath, 'utf8');
  } catch (error) {
    if (error.code === 'ENOENT') {
      // Config file doesn't exist - use defaults (this is OK)
      return { valid: true, config: {} };
    }
    return { valid: false, error: `Cannot read config: ${error.message}` };
  }

  // Step 2: Parse JSON (syntax check)
  let config;
  try {
    config = JSON.parse(rawContent);
  } catch (error) {
    return {
      valid: false,
      error: `Invalid JSON in config.json: ${error.message}\n  Check for missing commas, quotes, or brackets.`
    };
  }

  // Step 3: Validate structure (semantic check)
  const validationErrors = validateConfig(config);
  if (validationErrors.length > 0) {
    return {
      valid: false,
      error: `Config validation failed:\n  - ${validationErrors.join('\n  - ')}`
    };
  }

  return { valid: true, config };
}
```

### Integration with execute-phase.md

When loading config at plan execution start:

```markdown
1. Read .planning/config.json
2. If file doesn't exist: Continue with defaults (mode: interactive)
3. If JSON parse fails: STOP, show syntax error with line hint
4. If validation fails: STOP, show field-level errors
5. If valid: Proceed with execution
```

### Error Message Examples

**JSON Syntax Error:**
```
Error: Invalid JSON in config.json: Unexpected token '}' at position 45
  Check for missing commas, quotes, or brackets.
```

**Validation Error:**
```
Error: Config validation failed:
  - mode must be one of: interactive, autonomous, batch
  - retry.max_attempts must be between 1 and 5
  - gates.confirm_project must be a boolean, got string
```
</usage>

## Design Decisions

<design_decisions>
### Why Hand-Rolled Instead of JSON Schema

1. **Zero dependencies constraint** - PROJECT.md specifies no external npm dependencies
2. **Simple schema** - Only ~25 fields, max 2 levels of nesting, no complex features ($ref, anyOf, conditionals)
3. **Custom error messages** - Full control over actionable error messages

### Why Ignore Unknown Fields

1. **Forward compatibility** - Old validators should accept configs from newer GSD versions
2. **User extensions** - Users may add custom fields for their workflows
3. **Existing fields** - Config may have `_comment` or `_docs` fields

### Why Separate Parse and Validate

1. **Actionable errors** - "Invalid JSON" vs "Invalid field value" require different user actions
2. **Clear UX** - User knows whether to fix syntax or fix values
3. **Debugging** - Easier to isolate which layer failed
</design_decisions>

## Testing Guidance

<testing>
To verify the validation function works correctly:

### Valid Config (should return empty array)
```javascript
const validConfig = {
  mode: 'interactive',
  depth: 'standard',
  gates: { confirm_project: true },
  retry: { enabled: true, max_attempts: 3 }
};
console.assert(validateConfig(validConfig).length === 0, 'Valid config should pass');
```

### Invalid Mode (should return error)
```javascript
const invalidMode = { mode: 'invalid' };
const errors = validateConfig(invalidMode);
console.assert(errors.some(e => e.includes('mode must be one of')), 'Invalid mode should fail');
```

### Invalid Type (should return error)
```javascript
const invalidType = { retry: { max_attempts: 'five' } };
const errors = validateConfig(invalidType);
console.assert(errors.some(e => e.includes('must be an integer')), 'String for int should fail');
```

### Unknown Fields (should pass)
```javascript
const withUnknown = { mode: 'interactive', _comment: 'test', custom_field: true };
console.assert(validateConfig(withUnknown).length === 0, 'Unknown fields should be ignored');
```
</testing>

---

*Library: validate-config*
*Created: Phase 22 - Error Handling Hardening*
*Dependencies: None (hand-rolled)*
