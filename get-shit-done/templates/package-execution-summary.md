# Package Execution Summary Template

Template for per-package execution results used by coordinated-execution.md workflow.

---

## Markdown Template

```markdown
# Package: {package_name}

**Status:** {status}
**Duration:** {duration}
**Path:** {package_path}

---

## Tasks Executed

| Task | Status | Duration | Output |
|------|--------|----------|--------|
| {task_name} | {task_status} | {task_duration} | {task_summary} |

---

## Errors (if any)

{error_details}

---

## Dependencies

- **Depends on:** {dependencies_list}
- **Dependants:** {dependants_list}

---

*Executed at: {timestamp}*
```

---

## JSON Schema

Per-package result as stored in EXECUTION_RESULTS.package_results:

```json
{
  "package": "@scope/pkg-name",
  "path": "packages/pkg-name",
  "status": "success",
  "started": "2026-01-08T12:00:00Z",
  "completed": "2026-01-08T12:01:30Z",
  "duration_ms": 90000,
  "tasks": {
    "install": {
      "status": "success",
      "duration_ms": 30000
    },
    "build": {
      "status": "success",
      "duration_ms": 45000
    },
    "test": {
      "status": "success",
      "duration_ms": 15000
    }
  },
  "errors": [],
  "dependencies": ["@scope/shared"],
  "dependants": ["@scope/app"]
}
```

---

## Status Definitions

| Status | Description | Visual |
|--------|-------------|--------|
| `pending` | Not yet started | Gray |
| `running` | Currently executing | Blue |
| `success` | All tasks completed successfully | Green |
| `failed` | One or more tasks failed | Red |
| `skipped` | Skipped because a dependency failed | Yellow |
| `timeout` | Task exceeded timeout limit | Orange |

---

## Task Status Values

| Task Status | Description |
|-------------|-------------|
| `success` | Task completed with exit code 0 |
| `failed` | Task completed with non-zero exit code |
| `timeout` | Task exceeded configured timeout |
| `skipped` | Task was not executed (package failed earlier) |

---

## TypeScript Interface

```typescript
interface PackageExecutionSummary {
  /** Package name (e.g., "@scope/pkg-name") */
  package: string;

  /** Relative path to package (e.g., "packages/pkg-name") */
  path: string;

  /** Overall package execution status */
  status: 'pending' | 'running' | 'success' | 'failed' | 'skipped';

  /** ISO timestamp when execution started */
  started: string;

  /** ISO timestamp when execution completed */
  completed: string;

  /** Total execution duration in milliseconds */
  duration_ms: number;

  /** Per-task execution results */
  tasks: {
    [taskName: string]: TaskResult;
  };

  /** Error messages if any failures occurred */
  errors: string[];

  /** Workspace packages this package depends on */
  dependencies?: string[];

  /** Workspace packages that depend on this package */
  dependants?: string[];
}

interface TaskResult {
  /** Task execution status */
  status: 'success' | 'failed' | 'timeout' | 'skipped';

  /** Task execution duration in milliseconds */
  duration_ms: number;

  /** Exit code if task failed (0 for success) */
  exit_code?: number;

  /** Truncated output on failure (first 500 chars) */
  output?: string;
}
```

---

## Variable Reference

| Variable | Source | Description |
|----------|--------|-------------|
| `{package_name}` | MONOREPO_INFO.packages[].name | Full package name (e.g., "@scope/pkg") |
| `{package_path}` | MONOREPO_INFO.packages[].path | Relative path to package |
| `{status}` | Execution result | One of: pending, running, success, failed, skipped |
| `{duration}` | Calculated | Human-readable duration (e.g., "1m 30s", "450ms") |
| `{task_name}` | EXECUTION_CONFIG.tasks[] | Task identifier (e.g., "install", "build", "test") |
| `{task_status}` | Execution result | One of: success, failed, timeout, skipped |
| `{task_duration}` | Measured | Task execution time |
| `{task_summary}` | Execution output | Brief description or "OK" for success |
| `{error_details}` | Execution errors | Full error messages if failed |
| `{dependencies_list}` | MONOREPO_INFO.graph | Packages this depends on |
| `{dependants_list}` | Computed from graph | Packages depending on this |
| `{timestamp}` | date -u | ISO timestamp of execution |

---

## Duration Formatting

Duration is formatted for human readability:

| Raw (ms) | Formatted |
|----------|-----------|
| 450 | 450ms |
| 3200 | 3.2s |
| 65000 | 1m 5s |
| 125000 | 2m 5s |
| 3725000 | 1h 2m 5s |

**Formatting logic:**
```javascript
function formatDuration(ms) {
  if (ms < 1000) return `${ms}ms`;
  if (ms < 60000) return `${(ms/1000).toFixed(1)}s`;
  if (ms < 3600000) {
    const m = Math.floor(ms / 60000);
    const s = Math.floor((ms % 60000) / 1000);
    return `${m}m ${s}s`;
  }
  const h = Math.floor(ms / 3600000);
  const m = Math.floor((ms % 3600000) / 60000);
  const s = Math.floor((ms % 60000) / 1000);
  return `${h}h ${m}m ${s}s`;
}
```

---

## Example Output

### Successful Package

```markdown
# Package: @scope/shared

**Status:** success
**Duration:** 1m 30s
**Path:** packages/shared

---

## Tasks Executed

| Task | Status | Duration | Output |
|------|--------|----------|--------|
| install | success | 30.2s | OK |
| build | success | 45.1s | OK |
| test | success | 15.0s | 12 tests passed |

---

## Errors (if any)

None

---

## Dependencies

- **Depends on:** (none - root package)
- **Dependants:** @scope/utils, @scope/ui, @scope/api

---

*Executed at: 2026-01-08T12:01:30Z*
```

### Failed Package

```markdown
# Package: @scope/api

**Status:** failed
**Duration:** 52.3s
**Path:** packages/api

---

## Tasks Executed

| Task | Status | Duration | Output |
|------|--------|----------|--------|
| install | success | 28.5s | OK |
| build | success | 18.2s | OK |
| test | failed | 5.6s | 3 tests failed |

---

## Errors (if any)

Test suite failed:
- api/routes/auth.test.ts: Expected 200, got 401
- api/routes/users.test.ts: Timeout after 5000ms
- api/routes/products.test.ts: Cannot read property 'id' of undefined

---

## Dependencies

- **Depends on:** @scope/shared, @scope/utils
- **Dependants:** @scope/app

---

*Executed at: 2026-01-08T12:02:22Z*
```

### Skipped Package

```markdown
# Package: @scope/app

**Status:** skipped
**Duration:** -
**Path:** apps/app

---

## Tasks Executed

| Task | Status | Duration | Output |
|------|--------|----------|--------|
| - | skipped | - | Dependency @scope/api failed |

---

## Errors (if any)

Skipped due to dependency failure. The following dependencies failed:
- @scope/api

---

## Dependencies

- **Depends on:** @scope/ui, @scope/api
- **Dependants:** (none - leaf package)

---

*Skipped at: 2026-01-08T12:02:22Z*
```

---

## Usage in coordinated-execution.md

The coordinated-execution.md workflow uses this template format for each package in the EXECUTION_RESULTS.package_results array:

```javascript
// Per-package result structure
const packageResult = {
  package: "@scope/pkg-name",
  path: "packages/pkg-name",
  status: "success",          // Set after all tasks complete
  started: "2026-01-08T12:00:00Z",
  completed: "2026-01-08T12:01:30Z",
  duration_ms: 90000,
  tasks: {
    install: { status: "success", duration_ms: 30000 },
    build: { status: "success", duration_ms: 45000 },
    test: { status: "success", duration_ms: 15000 }
  },
  errors: []
};

// Added to EXECUTION_RESULTS.package_results array
EXECUTION_RESULTS.package_results.push(packageResult);
```

**Status computation:**
1. Start with status = "pending"
2. When starting: status = "running"
3. If all tasks succeed: status = "success"
4. If any task fails: status = "failed"
5. If skipped due to dependency: status = "skipped"

---

## Dashboard Integration

Package execution summaries feed into EXECUTION_DASHBOARD.md:

```markdown
## Package Status

| # | Package | Status | Duration | Details |
|---|---------|--------|----------|---------|
| 1 | @scope/shared | success | 1m 30s | All tasks passed |
| 2 | @scope/utils | success | 45.2s | All tasks passed |
| 3 | @scope/api | failed | 52.3s | test failed |
| 4 | @scope/app | skipped | - | Dependency failed |
```

**Dashboard update flow:**
1. Initialize: All packages "pending"
2. Start package: Update to "running"
3. Complete package: Update to "success" or "failed" with duration
4. Skip package: Update to "skipped" with reason

---

*Package execution summary template for coordinated-execution workflow*
*Created: 2026-01-08*
